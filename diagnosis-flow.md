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

### 2.5 调试模式触发与系统影响

```
lib/Configuration.php:38-44
```

项目根目录存在 `DEBUG` 文件时，强制覆盖两项配置：
- `system.env` 强制设为 `dev`
- `cache.type` 强制设为 `array`（内存缓存，避免持久化）

### 2.6 调试模式对错误输出粒度的影响

调试模式（`env=dev`）与生产模式（`env=prod`）在错误处理上有三处关键差异：

**差异 1：错误处理的严格程度**
```
index.php:26-44
```
- **dev**：`set_error_handler()` 中检测到 PHP 运行时错误后，立即 `throw new ErrorException()`，将任何 E_WARNING/E_NOTICE 升级为致命异常，强制暴露问题。这意味着开发模式下一行代码的 Notice 都会导致整个请求崩溃并显示异常栈。
- **prod**：仅通过 `$logger->warning()` 记录为警告日志，请求继续正常执行，用户不会感知到警告级别的错误。

**差异 2：默认日志过滤级别**
```
lib/dependencies.php:48-64
```
- **dev**：`ErrorLogHandler(Logger::DEBUG)` — 所有级别（DEBUG/INFO/WARNING/ERROR）全部输出到 error_log，包括桥接器的 ClientException、限流异常等"正常业务异常"。
- **prod**：`ErrorLogHandler(Logger::INFO)` — 过滤掉 DEBUG 级别，只保留 INFO 及以上。这意味着 ClientException、RateLimitException 等 DEBUG 日志在生产环境的 error_log 中完全不可见，只能通过文件日志（如果配置了）查看。

**差异 3：异常页面信息量**
异常页面模板 `templates/exception.html.php:1-147` 在 dev 和 prod 中**完全相同**，始终输出以下完整信息：
- 异常类型、代码、消息、文件、行号
- `trace_from_exception()` 生成的完整调用栈（带 GitHub 源码链接格式）
- Query String、版本号、OS、PHP 版本等上下文
- 注意：生产环境如果想隐藏异常栈，需要自定义修改模板或前端反向代理层过滤。系统本身没有提供"生产环境简化错误页"的开关。

### 2.7 调试模式对正常运行性能的影响

调试模式开启后，两项变更对性能影响显著：

**影响 1：缓存类型变更为 ArrayCache（性能双刃剑）**
```
caches/ArrayCache.php:8-59
```

| 维度 | FileCache（生产默认） | ArrayCache（调试模式） |
|-----|----------------------|----------------------|
| 存储位置 | 磁盘文件 `cache/*.cache` | PHP 进程内存 `$this->data[]` |
| 跨请求共享 | ✅ 多个请求共享同一份缓存 | ❌ 每次请求后进程结束，缓存全部失效 |
| 缓存命中 | 第 2 次请求后开始命中 | 永远 0 命中（单次请求内可重复利用） |
| 磁盘 I/O | 每次 get/set 都有读写开销 | 0 次磁盘 I/O |
| 源站请求量 | 缓存 TTL 内不重复抓取 | 每个请求都重新抓取所有外部 URL |
| 单次请求延迟 | 低（命中缓存时） | 高（每次都走网络） |
| 源站压力 | 小 | 极大（调试时可能触发对方限流） |

结论：**调试模式极大地牺牲了整体吞吐，但在单次请求内部的纯 PHP 计算上会略快（因为没有磁盘 I/O）。** 调试模式适合本地开发和问题排查，绝对不能在生产环境开启。

**影响 2：DEBUG 日志输出量增大**
- SimpleLogger 在 dev 模式会记录所有 DEBUG 级别日志（包括桥接器参数验证失败、用户输入错误等"正常"业务异常）
- error_log 写入在 PHP 中是同步阻塞的，日志量的增加会轻微增加每个请求的耗时
- 如果配置了文件日志（StreamHandler），磁盘写入量也会增大

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

### 4.0 代理服务器配置注入链路

代理配置从 `config.ini.php` 到 curl 执行的完整路径分三层传递，每层都有开关条件：

```
config.default.ini.php:93-106  [proxy] 配置段
  ├─ proxy.url       = ""      // 代理地址，如 "tcp://192.168.0.1:8080"
  ├─ proxy.name      = "Hidden proxy name"  // 前端显示名
  └─ proxy.by_bridge = false   // 是否允许用户单请求关闭代理
       │
       ▼
Configuration::loadConfiguration()
  └─ 校验 proxy.url 必须为 string、proxy.by_bridge 必须为 bool、proxy.name 必须为 string
       │
       ▼
DisplayAction::__invoke() 前置开关 (actions/DisplayAction.php:40-48)
  if (proxy.url 配置了          // 有代理
      && proxy.by_bridge=true   // 允许用户关
      && _noproxy 参数存在)     // 用户明确要求跳过
  {
      define('NOPROXY', true);  // 定义常量，全局生效
  }
       │
       ▼
getContents() 函数注入 (lib/contents.php:100-102)
  if (Configuration::getConfig('proxy', 'url') && !defined('NOPROXY')) {
      $config['proxy'] = Configuration::getConfig('proxy', 'url');
  }
  // 注意：这里 proxy.url 为空时即使定义了 NOPROXY 也不会走代理（双重保险）
       │
       ▼
CurlHttpClient::request() 默认值合并 (lib/http.php:69-74)
  'proxy' => null,  // 默认不使用代理，被上面 $config 覆盖
       │
       ▼
cURL 选项设置 (lib/http.php:133-134)
  if ($config['proxy']) {
      curl_setopt($ch, CURLOPT_PROXY, $config['proxy']);
  }
```

**三个关键边界条件（任意一个不满足就不走代理）：**

| 条件 | 说明 | 控制方 |
|-----|------|-------|
| `Configuration::getConfig('proxy', 'url')` 非空 | 运维必须在 config.ini.php 配置了代理地址 | 运维/服务器管理员 |
| `!defined('NOPROXY')` | 当前请求没有被定义为跳过代理 | 用户 + 运维共同控制 |
| `proxy.by_bridge = true` 时才接受 `_noproxy` 参数 | 运维可以锁死所有请求必须走代理 | 运维 |

**对桥接器抓取链路的影响：**
- **全链路生效**：`getContents()` / `getSimpleHTMLDOM()` / `getSimpleHTMLDOMCached()` 三个 API 都走同一套配置，不存在"某个请求不走代理"的可能（桥接器代码中绕过 getContents 自建 curl 的除外）
- **缓存与代理解耦**：缓存 key 是 `server_{url}`，不包含代理地址。**这意味着切换代理配置后，旧缓存会被直接复用，不会重新抓取。** 如果需要强制换新代理抓，必须手动清理缓存或改 URL 参数。
- **特殊桥接器（PixivBridge）**：PixivBridge 的 `proxy_url` 配置项是**图像代理**，不是 HTTP 请求代理。它用于将 `https://i.pximg.net/xxx.png` 重写为 `https://proxy.example.com/xxx.png`，解决图片防盗链问题，与 CURLOPT_PROXY 的 HTTP CONNECT 隧道代理是两套机制。

**前端 UI 开关（FrontpageAction）：**
```
actions/FrontpageAction.php:62-72
```
只有当 `proxy.url` 和 `proxy.by_bridge` 同时为 true 时，才会在桥接器表单上渲染一个 `_noproxy` 复选框，显示名为 `Disable proxy ({proxy.name})`。用户勾选后会在请求参数里带上 `_noproxy=on`。

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

### 4.3.1 错误反馈链接：GitHub Issue URL 生成、脱敏与提交流程

当 `error.output = feed` 时，错误条目中会内嵌"Find similar bugs"和"Create GitHub Issue"两个按钮。这两个链接的完整生成与脱敏链路如下：

```
DisplayAction::createFeedItemFromException()  (actions/DisplayAction.php:146-170)
  │
  ├─ 嵌套渲染 bridge-error.html.php 模板
  │    ├─ $error    = 渲染 exception.html.php（异常详情）
  │    ├─ $searchUrl = createGithubSearchUrl()
  │    ├─ $issueUrl  = createGithubIssueUrl()
  │    └─ $maintainer = $bridge->getMaintainer()
  │
  ▼
DisplayAction::createGithubSearchUrl()  (actions/DisplayAction.php:223-229)
  return 'https://github.com/RSS-Bridge/rss-bridge/issues?q='
       . urlencode('is:issue is:open ' . $bridge->getName())
  // 构造搜索条件：只搜索该桥接器名称相关的 open issue
  // 注意：这里 bridge->getName() 来自桥接器类定义，不含用户输入
       │
       ▼
DisplayAction::createGithubIssueUrl()  (actions/DisplayAction.php:193-221)
  │
  ├─ 提取维护者列表（逗号分隔，trim 去空白）
  │
  ├─ 组装 GitHub /issues/new query：
  │    ├─ title  = "{BridgeName} failed with: {异常消息}"
  │    ├─ labels = "Bridge-Broken"
  │    ├─ assignee = 第一个维护者的 GitHub handle
  │    └─ body（核心脱敏流程，见下）
  │
  └─ return 'https://github.com/RSS-Bridge/rss-bridge/issues/new?'
          . http_build_query($query)
```

**body 内容的脱敏与组装清单（按写入顺序）：**

| 字段 | 来源 | 脱敏处理 | 说明 |
|-----|------|---------|------|
| 异常消息 | `create_sane_exception_message($e)` | `sanitize_root()` 移除服务器绝对路径 | lib/utils.php:53-64 |
| 调用栈 | `trace_to_call_points(trace_from_exception($e))` | 每帧 file 路径都经 `sanitize_root()` 脱敏 | lib/utils.php:72-122 |
| Query String | `$_SERVER['QUERY_STRING']` | **直接写入，无脱敏** ⚠️ | 可能包含用户敏感参数 |
| 版本号 | `Configuration::getVersion()` | 系统常量，无敏感 | |
| 操作系统 | `PHP_OS_FAMILY` | 系统常量，无敏感 | |
| PHP 版本 | `phpversion()` | 系统常量，无敏感 | |
| 维护者 | `$bridge->getMaintainer()` | 来自桥接器类定义，无用户输入 | 以 `@user` 格式引用 |

**模板层的二次转义保护：**

错误条目最终嵌入 feed 条目 content，经过两层转义：

```
第一层：exception.html.php 模板
  templates/exception.html.php:103 - <?= e(sanitize_root($e->getMessage())) ?>
  templates/exception.html.php:107 - <?= e(sanitize_root($e->getFile())) ?>
  // e() = htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8')
  // sanitize_root() = 移除项目根目录绝对路径前缀

第二层：bridge-error.html.php 模板
  templates/bridge-error.html.php:2  - <?= raw($error) ?>        // raw() 不转义（因为上层已经 e() 过）
  templates/bridge-error.html.php:4  - <?= raw($searchUrl) ?>    // URL 经 urlencode() 过
  templates/bridge-error.html.php:8  - <?= raw($issueUrl) ?>     // URL 经 http_build_query() 过
  templates/bridge-error.html.php:13 - <?= e($maintainer) ?>     // 维护者名字仍转义
```

**用户输入的参数在错误反馈中的流转（完整链路）：**

```
用户 URL 参数（如 ?bridge=X&user=secret&password=123）
    │
    ├─ 存入 Request 对象 → DisplayAction 传给 bridge
    │
    ├─ 异常抛出时：异常消息可能包含参数（由桥接器自行决定）
    │   └─ 经 sanitize_root() + e() 转义后显示在异常页面
    │
    ├─ $_SERVER['QUERY_STRING'] 原样进入 GitHub Issue body
    │   └─ ⚠️ 如果 URL 中包含敏感参数（如 API token），
    │       用户点击 "Create GitHub Issue" 时会带到 GitHub issue
    │       用户可在提交前手动修改/删除
    │
    └─ 缓存层不存：错误计数缓存 key 是 error_reporting_{bridgeName}_{errorCode}，
                    不含用户参数，不会将用户输入持久化
```

**ParameterValidator 层的输入验证（桥接器参数层面）：**
```
lib/ParameterValidator.php:8-59
```
桥接器声明的参数（PARAMETERS 常量）在被 bridge 使用前，会经 `ParameterValidator::validateInput()` 校验：

| 参数类型 | 校验方式 | 非法处理 |
|---------|---------|---------|
| text | `filter_var($value)` 或正则匹配 | → null |
| number | `FILTER_VALIDATE_INT` | → null |
| checkbox | `FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE` | → null |
| list | `filter_var` + `in_array($expectedValues)` | → null |

校验失败的参数会置 null 并加入错误列表，但**不会阻止异常消息中携带原始用户输入**（异常抛出在校验之前/之外）。

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

### 5.5 缓存键失效与强制刷新的完整触发条件

系统中缓存失效分为 **自然过期**、**被动清理**、**主动清理**、**不写入即不缓存** 四种模式。以下是所有触发条件的完整清单：

#### 条件 1：自然过期（TTL 到期，读取时检测）
发生在所有 `cache->get($key)` 调用中，五种缓存实现各自的过期判断逻辑：

| 缓存类型 | 过期判断位置 | 判断逻辑 |
|---------|------------|---------|
| FileCache | `caches/FileCache.php:40-45` | `$expiration === 0 \|\| $expiration > time()`，不满足则 delete + return default |
| SQLiteCache | `caches/SQLiteCache.php:64-77` | 同上，**注意发现过期不主动 delete，仅返回 default（遗留问题，过期条目等 prune 清理）** |
| MemcachedCache | `caches/MemcachedCache.php:23-30` | 交给 Memcached 服务端原生 TTL 管理，PHP 侧不判断 |
| ArrayCache | `caches/ArrayCache.php:18-24` | 同 FileCache，过期后 delete |
| NullCache | `caches/NullCache.php:8-10` | 永远返回 default（等于全过期） |

**特别注意：** SQLiteCache 的过期条目在 get 时不删除，会在磁盘上越积越多，直到下一次 prune 触发才会被清掉。这与 FileCache/ArrayCache 的"读时即删"策略不一致。

#### 条件 2：被动清理（读时检测到异常就删）
除了过期，读取过程中检测到数据损坏也会触发删除：

- **FileCache 反序列化失败** (`caches/FileCache.php:35-38`)
  ```php
  $item = unserialize($data);
  if ($item === false) {
      $this->logger->warning('Failed to unserialize: {path}');
      $this->delete($key);    // 损坏立即删除
      return $default;
  }
  ```
  - 典型原因：缓存文件被截断、磁盘写满写了一半、不同 PHP 版本序列化格式不兼容。

- **SQLiteCache 反序列化失败** (`caches/SQLiteCache.php:68-72`)
  - 只记录 ERROR 日志，**不主动 delete**（与 FileCache 行为再次不一致，下一次 get 还会再次失败）。

#### 条件 3：不写入即不缓存（根本就不产生缓存键）
以下情况会导致 `cache->set()` 被跳过或提前返回，相当于"强制不缓存"：

**A. TTL 特殊值拦截**
- 所有五种缓存类型的 `set()` 方法第一行都判断：
  ```php
  if ($ttl === 0) {
      return;  // TTL 为 0 直接不存储
  }
  ```
- 触发场景：用户传了 `_cache_timeout=0` + `cache.custom_timeout=true`，或者桥接器内部 `saveCacheValue($k, $v, 0)`。

**B. 响应 Cache-Control 头拦截（HTTP 请求内容缓存）**
```
lib/contents.php:110-118
  switch ($response->getCode()) {
      case 200: case 201: case 202:
          $cacheControl = $response->getHeader('cache-control');
          if ($cacheControl) {
              $directives = explode(',', $cacheControl);
              $directives = array_map('trim', $directives);
              if (in_array('no-cache', $directives) || in_array('no-store', $directives)) {
                  break;  // 跳过后续的 cache->set()，直接不缓存
              }
          }
          $cache->set($cacheKey, $response, 86400 * 10);
          break;
  }
```
- 触发条件：源站返回了 `Cache-Control: no-cache` 或 `Cache-Control: no-store`。
- 实际案例：`bridges/RobinhoodSnacksBridge.php:23-24` 主动在请求头中加 `Cache-Control: no-cache`，但这是**请求**头不会被识别；只有对方**响应**带 no-cache 才会生效。

**C. HTTP 非 2xx/304 响应跳过缓存（getContents）**
- `contents.php:126-133` 中 301/302/303 显式标注 `todo: cache`，暂不缓存；
- 4xx/5xx 直接 throw HttpException，也走不到 cache->set()。

**D. 桥接器 CACHE_TIMEOUT = 0（响应缓存）**
- 当桥接器定义 `const CACHE_TIMEOUT = 0`，`getCacheTimeout()` 返回 0，
- DisplayAction `actions/DisplayAction.php:57-63` 中 `$ttl` 为 0，
- 最终 `cache->set()` 的 `$ttl === 0` 判断命中，不存储响应。

#### 条件 4：主动清理（显式调用 prune / delete / clear）

**A. 概率触发 prune（全局）**
```
middlewares/CacheMiddleware.php:56-60
  if (rand(1, 100) === 1) {
      $this->cache->prune();
  }
```
- 触发概率：**1%**（每个请求独立判断）
- 五种缓存的 prune 行为差异：

| 缓存类型 | prune 实现 | 资源消耗 |
|---------|----------|---------|
| FileCache | `scandir()` 遍历全部文件 → 逐个读取 unserialize → 判断过期 → `unlink()` | 高（大量文件时遍历磁盘 I/O 大） |
| SQLiteCache | 单条 SQL `DELETE FROM storage WHERE updated > 0 AND updated <= :now` | 低（数据库原生索引） |
| MemcachedCache | 空方法 `{}`，服务端自己管理 | 0 |
| ArrayCache | 遍历 PHP 数组 unset 过期条目 | 低（仅内存操作） |
| NullCache | 空方法 `{}` | 0 |

- prune 触发点在 **CacheMiddleware 的返程**，也就是说响应已经处理完、即将发送给用户时才执行，所以 prune 的延迟不会影响用户响应速度（但会占用 PHP-FPM 进程）。
- `enable_purge` 配置项：FileCache/SQLiteCache 的 prune 第一行检查 `$this->config['enable_purge']`，false 时直接 return 不执行。配置在各自的 `[FileCache]` / `[SQLiteCache]` section。

**B. 显式 delete（缓存 key 被删除，强制下次刷新）**
系统中**没有**给用户或管理员提供"通过 URL 参数强制刷新某个 bridge 缓存"的功能。
也就是说不存在 `?action=display&bridge=X&_flush=1` 这种调用 `cache->delete()` 的入口。

只能通过以下方式实现强制刷新：
1. **运维侧手动删**：直接删除 `cache/` 目录下对应的 `.cache` 文件（或清空 SQLite 表、flush Memcached）
2. **改 URL 参数**：哪怕改一个无关参数或加 `&_=<timestamp>`，因为缓存 key 是 `json_encode($request->toArray())`，参数变了 key 就变了，自然不命中。（Feed 阅读器有时会自动加 `_=1718...`，这会导致缓存完全失效，所以 DisplayAction 在 `lib/BridgeAbstract.php:76-88` 的 `remove` 列表中显式排除了 `_` 参数。）
3. **等 TTL 自然过期**

**C. clear 全量清空**
- 代码中没有找到任何显式调用 `cache->clear()` 的位置。
- 提供给管理员的操作只能手动删除 cache 目录下所有文件 / 执行 `DELETE FROM storage` / `memflush`。

#### 条件 5：限流状态缓存（桥接器控制的特殊短 TTL 缓存）
除了以上通用机制，部分桥接器还有自己的短 TTL 缓存控制下一次请求的行为：
```
bridges/SpotifyBridge.php:109-110
  $retryAfter = $e->response->getHeader('Retry-After') ?? (60 * 5);
  $this->cache->set('spotify_rate_limit', true, $retryAfter);

bridges/RedditBridge.php:122-137
  检测到 429 / X-Ratelimit-Remaining=0 时，缓存 reddit_rate_limit 61 分钟

bridges/Vk2Bridge.php:196-323
  连续失败 5 次缓存 5 秒、失败更多缓存 30 分钟
```
这些缓存条目的失效 = 下次可以正常请求，是"软熔断"机制。它们的 TTL 由对方服务的响应头决定，不是配置文件的固定值。

### 5.6 五种缓存实现的行为差异汇总

| 维度 | NullCache | ArrayCache | FileCache | SQLiteCache | MemcachedCache |
|-----|----------|-----------|-----------|-------------|----------------|
| 存储位置 | 无 | 进程内存 | 磁盘文件 | SQLite DB 文件 | 外部服务端 |
| 跨请求共享 | ❌ | ❌ | ✅ | ✅ | ✅ |
| get 过期是否删除 | N/A | ✅ 立即删 | ✅ 立即删 | ❌ 等 prune | 服务端管 |
| get 反序列化失败是否删除 | N/A | N/A | ✅ 删 | ❌ 不删 | 服务端管 |
| set TTL=0 行为 | 不存 | 不存 | 不存 | 不存 | 不存 |
| prune 资源消耗 | 0 | 低 | 高（扫磁盘） | 低（SQL） | 0 |
| enable_purge 控制 | ❌ | ❌ | ✅ | ✅ | ❌ |
| 日志输出 | 无 | 无 | unserialize/write 失败 WARNING | unserialize 失败 ERROR + 写/删失败 WARNING | set 失败 WARNING（带 5 项错误码） |

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

### 6.3 CurlHttpClient 超时与重试的完整代码路径

桥接器通过 `getContents()` / `getSimpleHTMLDOM()` / `getSimpleHTMLDOMCached()` 发起 HTTP 请求，超时与重试参数从配置到 curl 执行的完整链路如下：

```
配置层 (config.default.ini.php)
  ├─ http.timeout = 5         // 单次请求超时（秒）
  └─ http.retries = 1         // 失败后的重试次数
       │
       ▼
配置加载 (Configuration::loadConfiguration)
  └─ 存储到内存，支持环境变量 RSSBRIDGE_HTTP_TIMEOUT / RSSBRIDGE_HTTP_RETRIES 覆盖
       │
       ▼
getContents() 函数读取配置 (lib/contents.php:52-57)
  ├─ 'timeout' => Configuration::getConfig('http', 'timeout'),
  └─ 'retries' => Configuration::getConfig('http', 'retries'),
       │
       ▼
CurlHttpClient::request() 默认值合并 (lib/http.php:69-101)
  ├─ 默认 timeout = 5（如果配置为空则用 5）
  └─ 默认 retries = 2（如果配置为空则用 2）
       │
       ├─ 注意：这里存在两处默认值不一致的问题！
       │     config.default.ini.php 写 retries = 1，
       │     但 CurlHttpClient 代码默认 retries = 2。
       │     最终生效：配置文件值覆盖代码默认值，所以实际默认是 1 次重试。
       │
       ▼
cURL 选项设置 (lib/http.php:115)
  curl_setopt($ch, CURLOPT_TIMEOUT, $config['timeout']);
  // 设置 cURL 允许执行的最大秒数（包含 DNS 解析、连接建立、数据传输全程）
  // 注意：这是 CURLOPT_TIMEOUT，不是 CURLOPT_CONNECTTIMEOUT，
  // 意味着 5 秒内没完成整个请求就强制超时中断
       │
       ▼
重试循环执行 (lib/http.php:170-197)
  $tries = 0;
  while (true) {
      $tries++;
      $body = curl_exec($ch);
      if ($body !== false) {
          break;                              // 成功，退出循环
      }
      if ($tries <= $config['retries']) {
          continue;                          // 重试次数未用完，再来一次
      }
      // 超过重试次数，彻底失败
      throw new HttpException(
          'cURL error {msg}: {errno} for {url}'
      );
  }
```

**重试的触发条件（非常关键）：**
- **仅触发**：`curl_exec()` 返回 `false`（网络层失败）
  - DNS 解析失败
  - TCP 连接被拒绝/超时
  - SSL/TLS 握手失败
  - 连接中途被重置（RST）
  - 超过 CURLOPT_TIMEOUT 强制中断
  - 任何 curl_errno 非 0 的情况

- **不触发重试**：HTTP 协议层面的错误（因为 curl_exec 返回了 body，不是 false）
  - HTTP 404 Not Found ❌ 不重试
  - HTTP 500 Internal Server Error ❌ 不重试
  - HTTP 429 Too Many Requests ❌ 不重试
  - HTTP 503 Service Unavailable ❌ 不重试
  - 这些错误会走到 `lib/contents.php:130-133` 的 switch default 分支，抛出 `HttpException`

**HTTP 状态码异常的补救（桥接器自定义）：**
由于框架层对 HTTP 4xx/5xx 不重试，某些桥接器会自己实现应用层重试逻辑。典型模式：

```
bridges/SpotifyBridge.php:98-115  // 429 限流特殊处理
  ├─ catch (HttpException $e)
  │   ├─ 如果是 429：
  │   │   ├─ 读取响应头 Retry-After
  │   │   ├─ 将限流状态写入缓存 spotify_rate_limit，TTL = Retry-After
  │   │   └─ throwRateLimitException()
  │   └─ 其他状态码：继续抛出
  └─ 下次请求先检查缓存中是否有限流标记，有则直接抛 RateLimitException
```

```
bridges/RedditBridge.php:119-137  // 类似模式：检测 X-Ratelimit-Remaining 头
bridges/Vk2Bridge.php:196-323     // 限流标记缓存 5 秒/30 分钟两级
```

### 6.4 缓存超时 TTL 的完整传递路径

除了 HTTP 请求超时，**缓存超时（TTL）** 也是另一个"时间"相关的关键参数。它控制缓存条目多久后失效：

```
用户 URL 参数 _cache_timeout
  │
  └─ 仅当 cache.custom_timeout = true 时生效（actions/FrontpageAction.php:74-80）
       │
       ▼
DisplayAction 缓存 TTL 决策 (actions/DisplayAction.php:57-63)
  if (cache.custom_timeout && isset($_cache_timeout)) {
      $ttl = (int)$_cache_timeout;           // 用户自定义
  } else {
      $ttl = $bridge->getCacheTimeout();     // 桥接器默认
  }
       │
       ▼
桥接器 getCacheTimeout() 覆盖 (bridges/InstagramBridge.php:75-82)
  public function getCacheTimeout() {
      $customTimeout = $this->getOption('cache_timeout');  // 从 config.ini 读取桥接器级配置
      if ($customTimeout) return $customTimeout;
      return parent::getCacheTimeout();                    // 回退到 CACHE_TIMEOUT 常量
  }
       │
       ▼
默认回退 (lib/BridgeAbstract.php:114-117)
  return static::CACHE_TIMEOUT;  // 默认为 3600 秒 = 1 小时
       │
       ▼
最终写入 cache->set($cacheKey, $response, $ttl)
```

**TTL 特殊值语义：**
- `TTL = 0`：`set()` 方法直接 return，不存储（等同于禁用缓存）
- `TTL = null`：`expiration = 0`，意为永不过期（只在桥接器内部缓存 saveCacheValue 有机会出现，响应缓存 TTL 一定有值）

---

## 七、输出格式系统（RSS / Atom / MRSS 切换

**文件：** `lib/FormatFactory.php`, `lib/FormatAbstract.php`, `formats/`

### 7.1 格式选择与注入入口

格式切换从 URL 参数 `format=` 进入，到渲染输出的完整链路：

```
DisplayAction::createResponse()  (actions/DisplayAction.php:126-143)
  │
  ├─ $format = $request->get('format')   // 从 URL 参数提取
  │
  ├─ $formatFactory = new FormatFactory()
  │    └─ 构造时扫描 formats/ 目录，匹配 *Format.php 文件
  │
  ├─ $format = $formatFactory->create($format)
  │    │
  │    ├─ 正则校验：/^[a-zA-Z0-9-]*$/（非法字符直接 InvalidArgumentException）
  │    ├─ sanitizeName():
  │    │    ├─ ucfirst(strtolower($name))  // 大小写不敏感
  │    │    ├─ 去掉尾缀 .php 或 Format
  │    │    └─ 与已知格式名白名单对比
  │    └─ 匹配成功 → 实例化 \XxxFormat::class
  │    └─ 不匹配 → throw "Unknown format given"
  │
  ├─ $format->setItems($items)          // 写入 items（FeedItem[]）
  ├─ $format->setFeed($bridge->getFeed())
  ├─ $format->setLastModified(time())
  │
  ├─ Response Headers:
  │    ├─ Content-Type: {format->getMimeType()}; charset=UTF-8
  │    └─ Last-Modified: {GMT时间}
  │
  └─ $body = $format->render()            // 各格式自定义渲染
```

**FormatFactory 的白名单来源：
```
lib/FormatFactory.php:9-16
```
- 扫描 `formats/` 目录，正则 `/^([^.]+)Format\.php$/U`
- 自动发现所有 *Format.php 文件，排序后作为可用格式列表
- 当前 6 种格式：Atom / Html / Json / Mrss / Plaintext / Sfeed

### 7.2 六种格式实现对比与差异分支

所有格式继承自 FormatAbstract，共享 setFeed/setItems/setLastModified，差异仅在 render() 方法。

| 格式 | MIME Type | 输出结构 | 特有分支逻辑 |
|-----|-----------|----------|-------------|
| **MrssFormat** | `application/rss+xml` | RSS 2.0 + Media RSS 命名空间 | 根节点 `<rss version="2.0">` → `<channel>` → `<item>` |
| **AtomFormat** | `application/atom+xml` | RFC 4287 Atom | 根节点 `<feed xmlns="http://www.w3.org/2005/Atom">` → `<entry>` |
| **JsonFormat** | `application/json` | JSON Feed 1.0 | 字段映射：title→title, uri→url, categories→tags, enclosures→attachments |
| **HtmlFormat** | `text/html` | HTML 网页 | 渲染 html-format.html.php 模板，还会列出其他所有格式的链接 |
| **PlaintextFormat** | `text/plain` | PHP print_r 调试输出 | `print_r($feed, true)`，最简单的字符串 |
| **SfeedFormat** | `text/plain` | Sfeed 制表符分隔 | `timestamp\ttitle\turi\tcontent\thtml\t\tenclosure\tauthor\tenclosure\tcategories` 一行一条 |

### 7.3 MrssFormat 与 AtomFormat 核心差异分支

**MRSS（RSS 2.0 + Media RSS）：

**A. 命名空间声明差异：

```
MrssFormat.php:40-44
  <rss version="2.0" xmlns:atom="..." xmlns:media="http://search.yahoo.com/mrss/">
    <channel>
AtomFormat.php:24-26
  <feed xmlns="http://www.w3.org/2005/Atom" xmlns:media="...">
```

**B. 日期格式差异：**

```
MrssFormat.php:169-171  pubDate → gmdate(DATE_RFC2822, $timestamp)
  // 例: Mon, 15 Aug 2005 15:16:00 +0000

AtomFormat.php:130-139  published / updated → gmdate(DATE_ATOM, $timestamp)
  // 例: 2005-08-15T15:16:00+00:00
```

**C. 条目 ID 处理分支：

```
MrssFormat.php:120-129  <guid isPermaLink="false">
  ├─ uid 存在 → 直接使用 uid，isPermaLink="false"
  ├─ uid 不存在但有 uri → 用 uri，isPermaLink="true"
  └─ 都没有 → sha1(title + content)

AtomFormat.php:96-108  <id>
  ├─ uid 存在 → urn:sha1:{uid}
  ├─ 有 uri → 直接用 uri
  └─ 都没有 → urn:sha1:{hash(title+content)}
```

**D. Feed 级 icon 分支：

```
MrssFormat.php:82-96  icon 映射到 <image><url><title><link>（RSS 标准 image 元素
  AtomFormat.php:38-46  icon → <icon> + <logo> 两个独立元素
```

**E. enclosure 分支（两者的 enclosure 都用 Media RSS 命名空间：**

```
MrssFormat.php:180-185  <media:content url="..." type="..."> （MRSS 支持多 enclosure）
AtomFormat.php:181-187  <link rel="enclosure" type="..." href="..."> （Atom 原生 enclosure）
```

**两者共同点（相同字段缺失降级：**
- 两者都支持 iTunes 播客扩展（itunes 命名空间 + enclosure 属性
- thumbnail 字段有值时都注入 `xmlns:media="http://search.yahoo.com/mrss/

### 7.4 特殊字段的条件分支

**iTunes 播客命名空间（条件触发：**

仅当 Feed 或 item 中存在 `itunes` 字段时，两种格式都会动态添加 `xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd` 命名空间：

```
MrssFormat.php:97-103 (channel级 itunes 字段 → 加命名空间 + 逐项输出
MrssFormat.php:140-155  item  item 级 itunes 字段
AtomFormat.php:62-64 + 146-159 同理
```

注意：AtomFormat 的 Feed 级 itunes 字段的 `// todo: skip? 注释说明这部分可能还没实现暂未渲染，仅 item 级 itunes 有效。

**thumbnail 缩略图：**

item 有 thumbnail 字段时：
- MrssFormat：目前未单独输出（代码中没有 thumbnail 逻辑缺失（看 item 没有缩略图目前两个格式都支持但 AtomFormat.php:195-199 `<media:thumbnail url="...">

### 7.5 FeedItem 字段过滤与容错

FeedItem 类是格式的 setter 自带严格过滤，会过滤后传给所有格式共享：

```
lib/FeedItem.php
```

| 字段 | 输入过滤逻辑 |
|-----|---------|
| uri | 必须 `https?:// 开头，否则丢弃 |
| title | truncate() 截断到 150 字符 |
| timestamp | 数字或 strtotime() 解析失败则丢弃 |
| author | 仅字符串直接用 |
| content | HTML DOM 节点自动转 string |
| enclosures | 必须通过 FILTER_VALIDATE_URL |
| uid | 已有 sha1 原样保留，否则对输入做 sha1 哈希 |
| 其他字段 | 存入 misc 数组，JSON 格式单独输出 |

JsonFormat 的额外的 misc 字段单独输出为 `_rssbridge` vendor 前缀命名空间；XML 格式通过 `toArray()['misc']` 合并输出。

### 7.6 Content-Type 与输出

最终通过 `$format->getMimeType()`：

- `FormatAbstract` 通过 `static::MIME_TYPE` 常量定义，各格式覆盖：

| 格式 | MIME |
|-----|------|
| Mrss | application/rss+xml |
| Atom | application/atom+xml |
| Json | application/json |
| Html | text/html |
| Plaintext/Sfeed | text/plain |

---

## 八、辅助工具函数

**文件：** `lib/utils.php`

### 8.1 路径脱敏

```
lib/utils.php:129-140
```

`sanitize_root()` - 移除文件路径中的项目根目录前缀，避免泄露服务器路径信息。

### 8.2 异常栈格式化

```
lib/utils.php:72-122
```

- `trace_from_exception()` - 从异常提取调用栈，反转顺序（从调用者到被调者）
- `trace_to_call_points()` - 将栈帧转换为可读的调用点字符串数组
- `frame_to_call_point()` - 单帧格式化：`文件(行号): 类->方法()`

### 8.3 异常类型

```
lib/utils.php:250-283
```

- `ClientException` - 客户端错误（用户输入问题），仅 DEBUG 日志
- `throwClientException()` - 抛出客户端异常
- `throwServerException()` - 抛出服务端异常
- `throwRateLimitException()` - 抛出限流异常

---

## 九、完整调用链路总结

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
    ├─ ★ 代理前置开关：proxy.url + proxy.by_bridge + _noproxy 参数
    │   └─ 命中 → define('NOPROXY', true)  后续所有 getContents 跳过代理
    ├─ 创建 Bridge 实例
    ├─ try { bridge->collectData() }
    │   ├─ 成功：
    │   │   ├─ items 传入 FeedItem（字段 URI/标题/时间等严格过滤）
    │   │   └─ ★ 格式切换：format= 参数 → FormatFactory 白名单校验 → 实例化对应 Format
    │   │       ├─ setItems/setFeed/setLastModified
    │   │       ├─ Response Header: Content-Type = Format::MIME_TYPE
    │   │       └─ $format->render()  (Mrss / Atom / Json / Html / Plaintext / Sfeed)
    │   │           └─ 写入缓存（200 响应 + TTL）
    │   └─ 失败 → 按异常类型分级处理
    │       ├─ ClientException → DEBUG 日志
    │       ├─ RateLimitException → DEBUG 日志 + 429
    │       ├─ HttpException(429/503) → DEBUG 日志 + 对应状态码
    │       └─ 其他 → ERROR 日志 + 错误报告计数
    │           └─ 达阈值 → 按 error.output 策略
    │               ├─ feed：包装为 FeedItem
    │               │   └─ ★ 错误反馈：createGithubIssueUrl() 生成 issue 链接
    │               │       ├─ 异常消息 + 调用栈（sanitize_root() 脱敏）
    │               │       ├─ $_SERVER['QUERY_STRING']（原样无脱敏 ⚠️）
    │               │       └─ 版本/OS/PHP/维护者信息
    │               ├─ http：500 异常页面
    │               └─ none：静默空 feed
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

## 十、关键配置项速查

| 配置项 | 默认值 | 说明 |
|-------|-------|------|
| `system.env` | `prod` | 运行环境。dev=DEBUG 日志+错误升级为异常；prod=INFO 日志+警告仅记录不崩溃 |
| `system.enable_maintenance_mode` | `false` | 维护模式，开启后所有请求返回 503 |
| `cache.type` | `file` | 缓存类型：file/sqlite/memcached/array/null。DEBUG 文件存在时强制为 array |
| `cache.custom_timeout` | `false` | 是否允许用户通过 `_cache_timeout` URL 参数自定义缓存 TTL |
| `logging.file_path` | - | 日志文件路径，不设置则仅用 PHP error_log() |
| `logging.file_level` | - | 文件日志级别：DEBUG/INFO/WARNING/ERROR |
| `error.output` | `feed` | 错误输出方式：feed（包装为条目）/http（500 页面）/none（静默） |
| `error.report_limit` | `1` | 错误报告阈值，同一 bridge+errorCode 出现 N 次后才向用户展示 |
| `http.timeout` | `5` | cURL 总超时秒数（CURLOPT_TIMEOUT，含 DNS+连接+传输全程） |
| `http.retries` | `1` | 网络层失败重试次数（仅 curl_exec 返回 false 时触发，HTTP 4xx/5xx 不重试） |
| `http.max_filesize` | `20` | 单个 HTTP 响应最大体积，单位 MB |
| `FileCache.path` | `cache/` | FileCache 存储目录 |
| `FileCache.enable_purge` | `true` | prune() 是否真正删除过期文件。false 时 1% 概率的 prune 啥也不做 |
| `SQLiteCache.file` | `cache.sqlite` | SQLite 数据库文件路径 |
| `SQLiteCache.timeout` | `5000` | SQLite 忙等待超时（毫秒），高并发场景调大避免锁超时 |
| `SQLiteCache.enable_purge` | `true` | prune() 是否真正执行 DELETE SQL |
| `MemcachedCache.host` | `localhost` | Memcached 服务地址 |
| `MemcachedCache.port` | `11211` | Memcached 服务端口 |
| `proxy.url` | `""` | HTTP 代理地址，如 "tcp://192.168.0.1:8080"。空表示不使用代理。通过 CURLOPT_PROXY 注入所有桥接器请求 |
| `proxy.name` | `"Hidden proxy name"` | 前端 UI 上显示的代理名称，仅用于 FrontpageAction 的 _noproxy 复选框文案 |
| `proxy.by_bridge` | `false` | 是否允许用户单请求关闭代理。true 时 URL 上带 `_noproxy=on` 即可跳过代理 |

