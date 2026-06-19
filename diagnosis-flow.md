# RSS-Bridge 日志与诊断能力分析

按代码执行顺序，从入口到输出，梳理错误采集、调试输出和缓存状态的完整脉络。

---

## 一、入口阶段：全局错误捕获

**文件：** `index.php`

应用启动时，注册三层全局错误/异常处理器，确保任何未被业务代码捕获的问题都能被记录。

### 1.1 异常处理器 (`set_exception_handler`)

```
index.php:20-24
```

- 捕获所有未被捕获的 `\Throwable` 异常
- 渲染 `templates/exception.html.php` 错误页面，返回 500 状态码
- 通过 `$logger->error()` 记录错误，上下文包含异常对象 `e`

### 1.2 错误处理器 (`set_error_handler`)

```
index.php:26-44
```

- 捕获 PHP 运行时错误（如 E_WARNING、E_NOTICE 等）
- 若错误被 `error_reporting()` 掩码屏蔽则忽略（如 deprecation 消息）
- **dev 环境**：将错误升级为 `ErrorException` 抛出，强制暴露问题
- **prod 环境**：通过 `$logger->warning()` 记录警告级别日志，包含文件和行号
- 使用 `sanitize_root()` 对文件路径进行脱敏处理

### 1.3 关闭函数 (`register_shutdown_function`)

```
index.php:47-59
```

- 处理前述两种方式无法捕获的致命错误（fatal error）
- 通过 `error_get_last()` 获取最后一次错误
- 通过 `$logger->error()` 记录错误，前缀标记 `(shutdown)`

---

## 二、日志系统初始化

**文件：** `lib/logger.php`, `lib/dependencies.php`

### 2.1 日志接口与实现

```
lib/logger.php:5-26  (Logger 接口)
lib/logger.php:28-101 (SimpleLogger 实现)
```

日志级别（数值越大越严重）：
- `DEBUG (10)` - 调试信息
- `INFO (20)` - 一般信息
- `WARNING (30)` - 警告
- `ERROR (40)` - 错误

### 2.2 SimpleLogger 核心逻辑

```
lib/logger.php:69-100
```

日志过滤（静默忽略特定异常）：
- `RateLimitException` - 限流异常不记录
- 消息以 "Format name invalid"、"Unknown format given"、"Unable to find" 开头的异常

分发机制：
- 遍历所有已注册的 handler，将日志记录传递给每个 handler

### 2.3 日志处理器

**StreamHandler** (`lib/logger.php:103-150`)
- 将日志写入文件流（如 `/var/log/rss-bridge.log`）
- 支持级别过滤，低于设定级别的日志不输出
- 对异常上下文进行格式化：提取 type、code、message、file、line、url、trace
- 输出格式：`[时间] 名称.级别 消息 {上下文JSON}`

**ErrorLogHandler** (`lib/logger.php:152-198`)
- 通过 PHP 内置 `error_log()` 函数输出
- 格式化逻辑与 StreamHandler 相同
- 无换行符（`error_log` 自动添加）

**NullLogger** (`lib/logger.php:200-217`)
- 空实现，丢弃所有日志，用于测试或静默模式

### 2.4 日志配置与初始化

```
lib/dependencies.php:48-64
```

日志器名称固定为 `rssbridge`。

**环境决定默认 handler 级别：**
- `dev` 环境：`ErrorLogHandler(Logger::DEBUG)` - 输出 DEBUG 及以上
- `prod` 环境：`ErrorLogHandler(Logger::INFO)` - 输出 INFO 及以上

**文件日志（可选）：**
- 配置项：`logging.file_path` 和 `logging.file_level`
- 两者都配置时，添加 `StreamHandler` 写入指定文件
- 级别从配置字符串转换为常量（DEBUG/INFO/WARNING/ERROR）

### 2.5 调试模式触发

```
lib/Configuration.php:38-44
```

项目根目录存在 `DEBUG` 文件时：
- `system.env` 强制设为 `dev`
- `cache.type` 强制设为 `array`（内存缓存，避免持久化）

---

## 三、中间件层：请求生命周期诊断

**文件：** `lib/RssBridge.php`, `middlewares/`

### 3.1 中间件执行顺序

```
lib/RssBridge.php:25-38
```

中间件按数组逆序包装，实际执行顺序（由外到内）：

```
请求 → BasicAuthMiddleware
     → CacheMiddleware
     → ExceptionMiddleware
     → SecurityMiddleware
     → MaintenanceMiddleware
     → TokenAuthenticationMiddleware
     → Action 处理器
```

### 3.2 ExceptionMiddleware

```
middlewares/ExceptionMiddleware.php:14-23
```

- 包裹整个 Action 执行，捕获所有 `\Throwable`
- 记录 `ERROR` 级别日志：`"Exception in ExceptionMiddleware"`
- 返回 500 响应，渲染异常页面模板

### 3.3 CacheMiddleware（HTTP 响应缓存）

```
middlewares/CacheMiddleware.php:14-63
```

**缓存命中检查：**
- 仅对 `DisplayAction` 生效
- 缓存 key：`http_` + 请求参数 JSON 编码的哈希
- 命中缓存时检查 `If-Modified-Since` 头，支持 304 响应

**缓存写入：**
- 200 响应：由 DisplayAction 自行处理缓存（有自定义 TTL 逻辑）
- 400/403/404/429/500/503 响应：缓存 5~15 分钟（随机，防雪崩）
- 其他状态码：缓存 5 分钟

**缓存清理：**
- 1% 概率触发 `$cache->prune()` 清理过期缓存（概率触发，避免每次都扫）

---

## 四、DisplayAction：业务层错误处理

**文件：** `actions/DisplayAction.php`

### 4.1 桥接器执行与错误分类

```
actions/DisplayAction.php:73-124
```

`try-catch` 包裹桥接器数据采集，按异常类型分类处理：

| 异常类型 | 日志级别 | 处理方式 |
|---------|---------|---------|
| `ClientException` | DEBUG | 用户输入错误，仅调试日志 |
| `RateLimitException` | DEBUG | 返回 429 状态码 |
| `HttpException` (429/503) | DEBUG | 直接返回对应状态码 |
| `HttpException` (其他) | 不记录 | 继续向下走通用错误逻辑 |
| 其他异常 | ERROR | 完整异常栈记录 |

### 4.2 错误报告计数（缓存辅助）

```
actions/DisplayAction.php:108-112, 172-191
```

**`logBridgeError()` 方法：**
- 缓存 key：`error_reporting_{bridgeName}_{errorCode}`
- 缓存结构：`{error, time, count}`，TTL 5 天
- 每次调用计数 +1，更新时间戳
- 用于实现错误报告阈值（`error.report_limit` 配置）

### 4.3 错误输出策略

```
actions/DisplayAction.php:107, 114-123
```

当错误计数达到 `report_limit` 阈值时，按 `error.output` 配置输出：

- **`feed`**（默认）：将错误包装为 feed 条目返回
  - 标题：`Bridge returned error {code}! ({日期标识})`
  - 内容：异常详情 + GitHub 搜索/issue 链接 + 维护者信息
  - 每日生成唯一标识符，避免 feed 阅读器重复提醒

- **`http`**：直接返回 HTTP 500 错误页面

- **`none`**：静默，返回空 feed

### 4.4 响应缓存写入

```
actions/DisplayAction.php:50, 56-64
```

- 200 响应时写入缓存
- TTL 优先级：用户指定 `_cache_timeout`（需 `cache.custom_timeout` 开启） > 桥接器 `CACHE_TIMEOUT` 常量
- 缓存 key 与 CacheMiddleware 一致：`http_` + 请求参数 JSON

---

## 五、缓存系统详解

**文件：** `lib/CacheInterface.php`, `lib/CacheFactory.php`, `caches/`

### 5.1 缓存接口

```
lib/CacheInterface.php:3-14
```

标准 PSR-16 风格接口：
- `get(key, default)` - 读取
- `set(key, value, ttl)` - 写入
- `delete(key)` - 删除
- `clear()` - 清空全部
- `prune()` - 清理过期项

### 5.2 缓存工厂

```
lib/CacheFactory.php:15-110
```

支持的缓存类型：
- `null` → `NullCache` - 空缓存
- `file` → `FileCache` - 文件缓存
- `sqlite` → `SQLiteCache` - SQLite 数据库缓存
- `memcached` → `MemcachedCache` - Memcached 分布式缓存
- `array` → `ArrayCache` - 内存数组缓存（单次请求内有效）

### 5.3 FileCache 实现

```
caches/FileCache.php
```

**存储结构：**
- 文件命名：`md5(key).cache`
- 文件内容：序列化数组 `{key, expiration, value}`
- `expiration = 0` 表示永不过期

**get 流程：**
1. 文件不存在 → 返回默认值
2. 反序列化失败 → 记录 WARNING 日志，删除文件，返回默认值
3. 已过期 → 删除文件，返回默认值
4. 有效 → 返回 value

**set 流程：**
1. TTL 为 0 → 直接返回（不存储）
2. 序列化写入文件
3. 写入失败 → 记录 WARNING 日志（通常是磁盘满）

**prune 流程：**
- 遍历缓存目录所有文件
- 检查过期时间，删除已过期文件
- 反序列化失败的文件也会被清理

### 5.4 缓存的三种使用场景

**场景 1：HTTP 响应缓存**（CacheMiddleware + DisplayAction）
- Key 格式：`http_{请求参数JSON}`
- 存储整个 Response 对象
- TTL：由桥接器或用户指定

**场景 2：HTTP 请求内容缓存**（getContents）
- Key 格式：`server_{url}_{bodyHash}`
- 存储 Response 对象
- TTL：固定 10 天（200/201/202 状态码）
- 支持 `If-Modified-Since` 和 `ETag` 协商缓存

**场景 3：桥接器内部缓存**（BridgeAbstract）
- Key 格式：`{bridgeShortName}_{key}`
- 桥接器通过 `loadCacheValue()` / `saveCacheValue()` 使用
- 默认 TTL：1 天

**场景 4：错误计数缓存**（DisplayAction）
- Key 格式：`error_reporting_{bridgeName}_{errorCode}`
- TTL：5 天

---

## 六、HTTP 层：请求与错误

**文件：** `lib/http.php`, `lib/contents.php`

### 6.1 HTTP 异常体系

```
lib/http.php:6-56
```

- `RateLimitException` - 限流异常（utils.php 中定义）
- `HttpException` - HTTP 请求异常，包含 Response 对象
- `CloudFlareException` - Cloudflare 拦截，继承自 HttpException

**CloudFlare 检测：**
- 通过响应体中的 `<title>` 标签判断
- 识别标题："Just a moment..."、"Please Wait..."、"Attention Required!" 等

### 6.2 getContents 缓存逻辑

```
lib/contents.php:36-138
```

**读取缓存：**
1. 生成缓存 key：`server_{url}_{requestBodyHash}`
2. 有缓存时，提取 `last-modified` 和 `etag`
3. 下次请求带上 `If-Modified-Since` 和 `If-None-Match` 头

**处理响应：**
- 200/201/202：检查 `Cache-Control` 头，无 `no-cache`/`no-store` 则缓存 10 天
- 304：使用缓存的 body
- 301/302/303：暂不缓存（todo）
- 其他：抛出 `HttpException`

### 6.3 CurlHttpClient 重试机制

```
lib/http.php:170-197
```

- 配置 `http.retries` 指定重试次数
- cURL 执行失败（网络错误等）时自动重试
- 超过重试次数后抛出 `HttpException`，包含 curl 错误号和错误信息

---

## 七、辅助工具函数

**文件：** `lib/utils.php`

### 7.1 路径脱敏

```
lib/utils.php:129-140
```

`sanitize_root()` - 移除文件路径中的项目根目录前缀，避免泄露服务器路径信息。

### 7.2 异常栈格式化

```
lib/utils.php:72-122
```

- `trace_from_exception()` - 从异常提取调用栈，反转顺序（从调用者到被调者）
- `trace_to_call_points()` - 将栈帧转换为可读的调用点字符串数组
- `frame_to_call_point()` - 单帧格式化：`文件(行号): 类->方法()`

### 7.3 异常类型

```
lib/utils.php:250-283
```

- `ClientException` - 客户端错误（用户输入问题），仅 DEBUG 日志
- `throwClientException()` - 抛出客户端异常
- `throwServerException()` - 抛出服务端异常
- `throwRateLimitException()` - 抛出限流异常

---

## 八、完整调用链路总结

```
HTTP 请求
    │
    ▼
index.php
  ├─ 注册全局异常/错误/关闭处理器
  ├─ 加载配置 (Configuration)
  ├─ 初始化依赖容器 (dependencies.php)
  │   ├─ 创建 Logger (SimpleLogger + ErrorLogHandler)
  │   ├─ 创建 CacheFactory
  │   └─ 创建 Cache 实例
  └─ RssBridge->main()
       │
       ▼
  中间件链（外→内）
  1. BasicAuthMiddleware         - 基础认证
  2. CacheMiddleware             - HTTP 响应缓存检查
  3. ExceptionMiddleware         - 异常捕获与日志
  4. SecurityMiddleware          - 安全检查
  5. MaintenanceMiddleware       - 维护模式
  6. TokenAuthenticationMiddleware - Token 认证
       │
       ▼
  DisplayAction
    ├─ 校验参数
    ├─ 创建 Bridge 实例
    ├─ try { bridge->collectData() }
    │   ├─ 成功 → 渲染 feed → 写入缓存
    │   └─ 失败 → 按异常类型分级处理
    │       ├─ ClientException → DEBUG 日志
    │       ├─ RateLimitException → DEBUG 日志 + 429
    │       ├─ HttpException(429/503) → DEBUG 日志 + 对应状态码
    │       └─ 其他 → ERROR 日志 + 错误报告计数
    │           └─ 达阈值 → 按 error.output 策略输出
    └─ 返回 Response
       │
       ▼
  CacheMiddleware（返程）
    └─ 非 200 响应 → 写入缓存（5~15分钟）
    └─ 1% 概率 → 触发 prune 清理过期缓存
       │
       ▼
  Response->send() → 输出到浏览器
```

---

## 九、关键配置项速查

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `system.env` | `prod` | 运行环境，dev 时输出 DEBUG 日志 |
| `cache.type` | `file` | 缓存类型：file/sqlite/memcached/array/null |
| `cache.custom_timeout` | `false` | 是否允许用户自定义缓存超时 |
| `logging.file_path` | - | 日志文件路径，不设置则仅用 error_log |
| `logging.file_level` | - | 文件日志级别：DEBUG/INFO/WARNING/ERROR |
| `error.output` | `feed` | 错误输出方式：feed/http/none |
| `error.report_limit` | `1` | 错误报告阈值，需出现多少次才显示 |
| `http.timeout` | `5` | HTTP 请求超时（秒） |
| `http.retries` | `1` | HTTP 请求重试次数 |
