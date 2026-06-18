# RSS-Bridge 访问控制机制：Whitelist 与 Token 鉴权协作

## 一、整体架构概览

### 1.1 请求处理流水线

请求进入 RSS-Bridge 后，按以下顺序经过中间件（`lib/RssBridge.php:25-39`），最终到达具体 Action：

```
请求 → BasicAuthMiddleware → CacheMiddleware → ExceptionMiddleware
     → SecurityMiddleware → MaintenanceMiddleware → TokenAuthenticationMiddleware
     → Action (Frontpage / Display / List / ...)
```

**关键点**：Whitelist 检查不在中间件层，而是在各 Action 内部通过 `BridgeFactory::isEnabled()` 执行。Token 鉴权在中间件层完成，通过后向 Request 注入 `token` attribute。

### 1.2 配置源与加载顺序

配置加载顺序（后者覆盖前者），见 `lib/Configuration.php:18-82`：

| 优先级 | 配置源 | 说明 |
|--------|--------|------|
| 1（最低） | `config.default.ini.php` | 默认配置，随版本更新，不可修改 |
| 2 | `config.ini.php` | 用户自定义配置 |
| 3 | `whitelist.txt` | 遗留快捷方式，仅设置 `system.enabled_bridges` |
| 4（最高） | 环境变量 `RSSBRIDGE_*` | Docker 部署常用 |

> **注意**：`whitelist.txt` 的内容仅映射到 `system.enabled_bridges`，与 config.ini.php 中同名字段等价。

---

## 二、Whitelist：管理员声明可用 Bridge 的放行规则

### 2.1 Whitelist 的三种声明方式

**方式一：config.ini.php（推荐）**

```ini
[system]
enabled_bridges[] = *              ; 启用全部 bridge
; 或按需启用
;enabled_bridges[] = TwitchBridge
;enabled_bridges[] = TwitterBridge
```

**方式二：whitelist.txt（遗留）**

```bash
echo '*' > whitelist.txt                    # 全部启用
echo -e "TwitchBridge\nTwitterBridge" > whitelist.txt   # 指定启用
```

**方式三：环境变量**

```bash
RSSBRIDGE_system_enabled_bridges="TwitchBridge,TwitterBridge"
```

### 2.2 BridgeFactory 的启用判定逻辑

`lib/BridgeFactory.php:25-42` 在构造时完成桥接启用列表构建：

```
Configuration::getConfig('system', 'enabled_bridges')
    │
    ├─ 包含 '*'  → 将 bridges/ 目录下扫描到的所有 *Bridge.php 类名
    │              全部加入 $this->enabledBridges
    │
    └─ 列出具体名称 → 逐个 normalize（补全 Bridge 后缀、大小写匹配）
                     存在则加入 $this->enabledBridges
                     不存在则记录到 $this->missingEnabledBridges 并日志告警
```

判定接口：`BridgeFactory::isEnabled(string $bridgeName): bool`
→ 简单检查 `in_array($bridgeName, $this->enabledBridges)`

### 2.3 各 Action 中的放行检查点

Whitelist 检查**非全局中间件**，而是在各 Action 内部按需执行：

| Action | 检查位置 | 未放行行为 |
|--------|----------|-----------|
| **DisplayAction** | `actions/DisplayAction.php:36-38` | 返回 HTTP 400，消息 "This bridge is not whitelisted" |
| **FrontpageAction** | `actions/FrontpageAction.php:30-36` | 不在首页渲染 bridge 卡片，统计不计入 `active_bridges` |
| **ListAction** | `actions/ListAction.php:22-24` | **不拦截**，但将 status 标记为 `"inactive"`，前端可自行过滤 |

> **设计意图**：DisplayAction 是核心数据出口，必须严格校验；Frontpage 仅展示过滤；ListAction 提供全景视图供 API 客户端自行判断。

---

## 三、Token 模式下访问者身份识别

### 3.1 Token 鉴权触发条件

在 `middlewares/TokenAuthenticationMiddleware.php:9-11`：

```php
if (! Configuration::getConfig('authentication', 'token')) {
    return $next($request);  // token 为空字符串 → 直接放行，不鉴权
}
```

即：**当且仅当 config 中 `authentication.token` 被设置为非空值时，Token 模式才激活**。

### 3.2 Token 校验流程

```
请求到达 TokenAuthenticationMiddleware
    │
    ├─ config 中 authentication.token 为空？
    │     └─ 是 → 放行，不做任何处理
    │
    ├─ 请求参数中缺失 token？
    │     └─ 是 → 返回 HTTP 401，渲染 token.html.php 输入表单（"Missing token"）
    │
    ├─ hash_equals(config.token, 请求token) 不匹配？
    │     └─ 是 → 返回 HTTP 401，渲染 token.html.php 输入表单（"Invalid token"）
    │
    └─ 全部通过
          ├─ 将 token 写入 Request attribute: $request->withAttribute('token', $token)
          └─ 调用 $next($request) 放行
```

**安全细节**：使用 `hash_equals()` 做时序安全比较（`middlewares/TokenAuthenticationMiddleware.php:22`），防止侧信道攻击。

### 3.3 Token 通过后的身份透传

Token 验证成功后，系统**不做细粒度角色区分**（没有 admin/user 分级），仅有"已认证/未认证"二元状态。

Token 在后续流程中的两处使用：

1. **FrontpageAction 渲染表单**（`actions/FrontpageAction.php:15-16, 163-169`）：
   ```php
   $token = $request->getAttribute('token');
   // ... 若 token 模式开启且存在，则在所有 bridge 表单中注入隐藏字段
   if (Configuration::getConfig('authentication', 'token') && $token) {
       $form .= '<input type="hidden" name="token" value="{$token}" />';
   }
   ```
   → 保证用户在首页点击"Generate feed"生成的 URL 自动携带 token，无需手动拼接。

2. **DisplayAction 剥离参数**（`actions/DisplayAction.php:77`）：
   ```php
   $remove = ['token', 'action', 'bridge', 'format', ...];
   $input = array_diff_key($requestArray, array_fill_keys($remove, ''));
   $bridge->setInput($input);
   ```
   → token 不会传递给具体 bridge 的业务逻辑，仅用于访问控制。

### 3.4 与 Basic Auth 的协作关系

`BasicAuthMiddleware` 排在流水线**最前端**（`lib/RssBridge.php:26`），Token 中间件排在**最后**。两者为**独立开关**的并行关系：

| 场景 | authentication.enable | authentication.token | 效果 |
|------|----------------------|----------------------|------|
| 无鉴权 | false | 空 | 全部放行 |
| 仅 Basic Auth | true + 用户名/密码 | 空 | 需通过 HTTP Basic，通过后无需 token |
| 仅 Token | false | 非空 | 需通过 URL ?token=xxx |
| 两者同开 | true + 用户名/密码 | 非空 | **先过 Basic，再过 Token**，两道关卡 |

---

## 四、Whitelist 开关切换时的行为差异

### 4.1 维度一：从「全部放行」到「部分放行」

**状态迁移**：`enabled_bridges = ['*']` → `enabled_bridges = ['TwitterBridge', 'TwitchBridge']`

| 受影响模块 | 行为变化 |
|-----------|---------|
| **首页（FrontpageAction）** | 所有未列入白名单的 bridge 卡片消失，`active_bridges` 计数减少 |
| **DisplayAction** | 已保存的其他 bridge feed URL 立即返回 HTTP 400 "not whitelisted" |
| **ListAction** | status 字段从 `"active"` 变为 `"inactive"`，JSON 结构不变 |
| **BridgeFactory 启动** | 若列表中包含不存在的 bridge 名，记录 warning 级日志，不中断启动 |

### 4.2 维度二：从「部分放行」到「全部放行」

**状态迁移**：`enabled_bridges = ['TwitterBridge']` → `enabled_bridges = ['*']`

| 受影响模块 | 行为变化 |
|-----------|---------|
| **首页** | bridges/ 目录下全部 bridge 被渲染，`active_bridges` = 总数 |
| **DisplayAction** | 所有历史保存的 feed URL 恢复可访问 |
| **ListAction** | 所有 bridge status 变为 `"active"` |

### 4.3 维度三：新增白名单条目

**场景**：`enabled_bridges = ['TwitterBridge']` → 添加 `TwitchBridge`

| 检查点 | 行为 |
|--------|------|
| 名称规范化 | 支持 `Twitch` / `TwitchBridge` / `twitchbridge`，大小写不敏感（`lib/BridgeFactory.php:54-64`） |
| 缺失容忍 | 若填写拼写错误的 bridge 名，仅记录 info 日志到 `missingEnabledBridges`，首页顶部展示 Warning 横幅 |
| 生效时机 | 下次 HTTP 请求时自动生效（BridgeFactory 每次请求重新构造），无需重启服务 |

### 4.4 维度四：移除白名单条目 vs Token 失效的对比

| 现象 | 移除 Whitelist 条目 | Token 失效/缺失 |
|------|-------------------|-----------------|
| HTTP 状态码 | 400 Bad Request | 401 Unauthorized |
| 返回页面 | error.html.php 模板 | token.html.php 表单模板 |
| 错误消息 | "This bridge is not whitelisted" | "Missing token" / "Invalid token" |
| 首页表现 | bridge 卡片消失 | 展示 token 输入表单，无 bridge 列表 |
| 缓存行为 | CacheMiddleware 缓存 5~15 分钟（400 错误码） | CacheMiddleware 缓存 5~15 分钟（401 错误码） |

---

## 七、运行时变更代码走向深度分析

### 7.1 场景一：Token 运行时变更对正在进行请求的影响

**问题**：管理员在 config.ini.php 中重命名或删除 token 条目时，正好有访问者在用旧 token 拉数据，这条请求是被截断返 401 还是按旧 token 继续完成？

#### 7.1.1 关键代码基础

**配置加载时机**（`index.php:13-14`）：
```php
require __DIR__ . '/lib/bootstrap.php';
require __DIR__ . '/lib/config.php';  // 这里调用 Configuration::loadConfiguration()
```

**Configuration 存储结构**（`lib/Configuration.php:12`）：
```php
private static $config = [];  // 静态变量，进程生命周期内常驻内存
```

**Token 中间件读取方式**（`middlewares/TokenAuthenticationMiddleware.php:9,22`）：
```php
if (! Configuration::getConfig('authentication', 'token')) { ... }
if (! hash_equals(Configuration::getConfig('authentication', 'token'), $token)) { ... }
```

#### 7.1.2 请求生命周期时序分析

**单个请求的完整时间线**：
```
T0: 进程启动，index.php 开始执行
    → require lib/config.php
    → Configuration::loadConfiguration() 读取 config.ini.php
    → 将 authentication.token 值写入 self::$config 静态变量
T1: TokenAuthenticationMiddleware::__invoke() 执行
    → 从 self::$config 静态变量读取 token 值进行比较
T2: DisplayAction 执行，isEnabled() 检查，collectData() 拉取数据
T3: Response 返回
```

**管理员变更 token 发生在 T1.5（Token 校验通过后，数据拉取中）**：

```
T0: 配置加载，token = "old-secret" 写入静态变量
T1: Token 中间件校验通过（用的是内存中的 "old-secret"）
    ← 管理员此时修改 config.ini.php，token = "new-secret"
T1.5: 正在执行 bridge->collectData()，可能耗时几秒
T2: DisplayAction 正常完成，返回 200
    （全程没有重新读取配置文件）
```

#### 7.1.3 结论：不会被截断

**代码层面的铁证**：
1. `Configuration::$config` 是 `private static` 静态变量，在请求开始时一次性加载
2. `getConfig()` 直接从静态变量读取，**不会重新读取文件**
3. 没有任何 `reloadConfiguration()` 或动态刷新机制
4. `TokenAuthenticationMiddleware` 只在流水线入口执行一次，通过后不再复检

**所以**：正在进行的请求会**按旧 token 继续完成**，不会中途被截断返 401。只有**下一个新请求**才会加载新配置，使用新 token 校验。

---

### 7.2 场景二：Whitelist 热更新与 BridgeFactory 重建

**问题**：Whitelist 重新打开之前已下线的 bridge 时，BridgeFactory 是否真能在不重启进程的前提下恢复其访问？缓存路径在哪段代码处理？

#### 7.2.1 对象生命周期分析

**PHP Share-Nothing 架构前提**：每个 HTTP 请求对应独立的 PHP 进程，请求结束进程销毁。

**Container 单例范围**（`lib/Container.php:8,21-24`）：
```php
private array $resolved = [];  // 非静态，每个 Container 实例独立

public function offsetGet($offset)
{
    if (!isset($this->resolved[$offset])) {
        $this->resolved[$offset] = $this->values[$offset]($this);  // 首次访问时创建
    }
    return $this->resolved[$offset];
}
```

**BridgeFactory 注册**（`lib/dependencies.php:35-37`）：
```php
$container['bridge_factory'] = function ($c) {
    return new BridgeFactory($c['cache'], $c['logger']);  // 每次新建 Container 都会重新构造
};
```

**BridgeFactory 构造时读取白名单**（`lib/BridgeFactory.php:25`）：
```php
$enabledBridges = Configuration::getConfig('system', 'enabled_bridges');
```

#### 7.2.2 热更新生效路径

```
请求 A（白名单 = [Twitter]）:
  → 新进程，新 Container
  → BridgeFactory 新建，enabledBridges = [Twitter]
  → isEnabled("Twitch") = false → 返回 400
  → 进程结束，Container 销毁

管理员修改 config.ini.php: enabled_bridges[] = TwitchBridge

请求 B（白名单 = [Twitter, Twitch]）:
  → 全新进程，全新 Container
  → BridgeFactory 全新构造，enabledBridges = [Twitter, Twitch]
  → isEnabled("Twitch") = true → 正常返回 200
```

**结论**：**BridgeFactory 确实能在不重启服务的前提下热更新**，因为：
1. 每次请求创建全新的 Container 和 BridgeFactory
2. BridgeFactory 构造时重新读取 Configuration（每次请求也重新加载）
3. 无需重启 php-fpm 或 web 服务器

#### 7.2.3 缓存陷阱：400 错误缓存导致的"假失效"

**关键发现**：即使 Whitelist 已更新，已缓存的 400 响应可能导致"看起来仍然不能访问"。

**缓存 key 构成**（`lib/http.php:248-251`）：
```php
public function toArray(): array
{
    return $this->get;  // 返回所有 $_GET 参数，包括 token, bridge, format 等
}
```

**CacheMiddleware 两处使用相同的 cacheKey**：
- `middlewares/CacheMiddleware.php:24`: `$cacheKey = 'http_' . json_encode($request->toArray());`
- `actions/DisplayAction.php:50`: `$cacheKey = 'http_' . json_encode($request->toArray());`

**400 错误缓存策略**（`middlewares/CacheMiddleware.php:48-50`）：
```php
} elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
    // Cache these responses for about ~10 mins on average
    $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
}
```

#### 7.2.4 完整的缓存链路时序

```
请求 1（Twitch 未在白名单）:
  CacheMiddleware:
    → cacheKey = http_{"action":"display","bridge":"Twitch",...}
    → cache->get() 不存在
    → 继续执行
  TokenAuthenticationMiddleware: 通过
  DisplayAction:
    → isEnabled("TwitchBridge") = false
    → 返回 400 "not whitelisted"
  CacheMiddleware 后置处理:
    → 400 属于错误缓存列表
    → cache->set(cacheKey, 400_response, 300~900秒)

管理员将 Twitch 加入白名单

请求 2（同一 URL，缓存期内）:
  CacheMiddleware:
    → cacheKey 完全相同
    → cache->get() 命中 → 直接返回 400
    → **根本不会执行到 DisplayAction 的 isEnabled() 检查**
    → 用户感知："还是不能访问"

请求 N（缓存过期后）:
  CacheMiddleware:
    → cache->get() 不存在
    → 继续执行
  DisplayAction:
    → isEnabled("TwitchBridge") = true
    → 正常返回 200
```

#### 7.2.5 缓存代码路径索引

| 执行阶段 | 代码位置 | 行为 |
|---------|---------|------|
| **缓存命中检查**（最前置） | `middlewares/CacheMiddleware.php:23-41` | 先于所有业务中间件，命中直接返回 |
| **缓存 key 计算** | `middlewares/CacheMiddleware.php:24` / `lib/http.php:248-251` | 包含所有 GET 参数，不包含 attributes |
| **200 成功缓存** | `actions/DisplayAction.php:56-63` | DisplayAction 内部缓存，使用 bridge 自身的 TTL |
| **4xx/5xx 错误缓存** | `middlewares/CacheMiddleware.php:46-54` | 固定 5~15 分钟随机 TTL |
| **缓存 key 不包含** | `actions/DisplayAction.php:77` | token 等控制参数会从 bridge 输入中剥离，但**已在 cacheKey 中** |

---

### 7.3 场景三：紧急下线 bridge 的代码入口与缓存清除

**问题**：紧急处理下线 bridge 的时间点在代码中对应哪个入口？是否存在手动清除 400 错误缓存的接口？若没有，需等待多长 TTL？

#### 7.3.1 紧急下线 bridge 的代码入口

紧急下线 bridge 的操作是通过**修改配置**实现的，没有独立的"紧急下线"代码入口。生效路径如下：

```
管理员操作（三种等价方式）:
  ① 修改 config.ini.php 中 [system] 下的 enabled_bridges，移除目标 bridge
  ② 编辑 whitelist.txt，移除目标 bridge 行或改写内容
  ③ 修改环境变量 RSSBRIDGE_system_enabled_bridges

↓ 下次 HTTP 请求时自动生效

配置加载入口:
  → index.php:14  require lib/config.php
  → lib/config.php:13  Configuration::loadConfiguration($config, getenv())
  → lib/Configuration.php:46-53  读取 whitelist.txt（如果存在）
  → lib/Configuration.php:55-81  读取 RSSBRIDGE_* 环境变量覆盖

↓

BridgeFactory 构造时读取白名单:
  → lib/dependencies.php:35-37  Container 注册 bridge_factory
  → lib/BridgeFactory.php:25-42  __construct 中根据 enabled_bridges 构建白名单
  → lib/BridgeFactory.php:49-51  isEnabled() 判断

↓

各 Action 拦截:
  actions/DisplayAction.php:36-38    ← 核心拦截点，返回 400
  actions/FrontpageAction.php:30-36  ← 不渲染卡片
  actions/ListAction.php:22-24       ← status 变为 inactive
```

**生效时机**：从管理员保存配置文件起，**第一个未被缓存命中的新请求**即可生效。

#### 7.3.2 手动清除缓存接口：不存在

**代码层面的全面排查结论**：

| 排查范围 | 结果 |
|---------|------|
| `actions/` 目录下 7 个 Action | 无任何 CacheClearAction、PurgeAction 等管理类 Action |
| 全代码库搜索 `->clear()`、`->delete(` 调用 | 仅 `tests/CacheTest.php` 中测试代码调用，无生产代码 |
| README.md 提到的 `bin/cache-clear` | 项目中不存在该文件（可能是文档遗留或规划中功能） |
| docker-entrypoint.sh | 无缓存清除逻辑 |
| CacheInterface 接口能力 | 有 `clear()`、`delete()`、`prune()` 三个方法，但**无任何 Action 暴露这些能力** |

**结论：不存在通过 HTTP 请求手动清除缓存的接口。**

#### 7.3.3 被动清除路径：1% 概率的 prune

唯一的被动清除机制在 `middlewares/CacheMiddleware.php:56-60`：

```php
// For 1% of requests, prune cache
if (rand(1, 100) === 1) {
    // This might be resource intensive!
    $this->cache->prune();
}
```

`prune()` 的具体行为（各缓存实现一致）：
- **FileCache** (`caches/FileCache.php:88-113`)：遍历所有缓存文件，删除 `expiration <= time()` 的文件
- **SQLiteCache** (`caches/SQLiteCache.php:112-124`)：执行 `DELETE FROM storage WHERE updated > 0 AND updated <= now()`
- **MemcachedCache** (`caches/MemcachedCache.php:63-66`)：空实现，Memcached 自身管理过期
- **ArrayCache** (`caches/ArrayCache.php:49-58`)：遍历内存数组删除过期项

**注意**：`prune()` **只删除已过期的缓存项**，不会主动删除未过期的 400 错误缓存。即使触发了 1% 概率的 prune，TTL 未到的 400 缓存依然存在。

#### 7.3.4 400 错误缓存 TTL 精确值

**TTL 计算代码**在 `middlewares/CacheMiddleware.php:48-50`：

```php
} elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
    // Cache these responses for about ~10 mins on average
    $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
}
```

**精确分析**：

| 参数 | 计算 | 值 |
|------|------|-----|
| 基础值 | `60 * 5` | 300 秒（5 分钟） |
| 随机增量 | `rand(1, 60 * 10)` | 1 ~ 600 秒（1 秒 ~ 10 分钟） |
| **最小 TTL** | 300 + 1 | **301 秒（5 分 1 秒）** |
| **最大 TTL** | 300 + 600 | **900 秒（15 分钟）** |
| **平均 TTL** | 300 + 300.5 | **约 600.5 秒（10 分钟）** |

使用随机 TTL 的设计意图：避免"缓存雪崩"——防止大量缓存同时过期导致后端瞬间压力飙升。

#### 7.3.5 紧急下线后等待时间表

管理员执行紧急下线（从 enabled_bridges 中移除某 bridge）后，不同场景下的实际生效时间：

| 场景 | 实际生效时间 | 说明 |
|------|-------------|------|
| 该 bridge URL **从未被访问过** | **即时**（下一个请求） | 无缓存，直接进入 DisplayAction:36 的 isEnabled 检查 |
| 该 bridge URL **之前访问成功（200 缓存）** | 需等待 bridge 自身的 `getCacheTimeout()`（通常几分钟到几小时） | 200 缓存 TTL 由各 bridge 决定，不受 5~15 分钟限制 |
| 该 bridge URL **之前访问失败（400 缓存，最常见）** | **301 秒 ~ 900 秒（5 分 1 秒 ~ 15 分钟）** | 由 CacheMiddleware 随机 TTL 决定 |
| 恰好触发了 1% 概率的 prune | 不加速 | prune 只删已过期项，未过期的 400 缓存仍保留 |

#### 7.3.6 运维侧手动清缓存的替代方案

由于系统没有暴露清缓存接口，管理员如需立即生效，需**直接操作存储层**：

| 缓存类型 | 手动清除方法 |
|---------|-------------|
| **FileCache**（默认） | 删除 `cache/` 目录下所有 `*.cache` 文件，或仅删除 md5  hash 匹配的目标文件 |
| **SQLiteCache** | 执行 `DELETE FROM storage WHERE key LIKE '%bridge_name%'` 或清空整个表 |
| **MemcachedCache** | 执行 `flush_all` 命令清空全部，或按 sha1(key) 删除单项 |
| **ArrayCache**（dev 环境） | 无需操作，每个请求后进程销毁，缓存不跨请求 |

---

## 五、协作场景完整示例

### 场景 A：标准私用部署（Token + 全量 Bridge）

**配置**：
- `authentication.token = "my-secret-token"`
- `enabled_bridges[] = *`

**访问链路**：
```
用户访问 /?token=my-secret-token
  → BasicAuthMiddleware（跳过，authentication.enable=false）
  → TokenAuthenticationMiddleware（校验通过，注入 attribute）
  → FrontpageAction
       → 遍历所有 bridge，isEnabled() 全部为 true
       → 渲染所有卡片，每个表单内嵌 <input type=hidden name=token>
用户点击某 bridge Generate feed
  → 跳转到 /?action=display&bridge=Xxx&format=Html&token=my-secret-token
  → TokenAuthenticationMiddleware（通过）
  → DisplayAction
       → isEnabled("XxxBridge") = true
       → 正常返回 feed 数据
```

### 场景 B：公开只读部署（无 Token + 白名单限制）

**配置**：
- `authentication.token = ""` （空）
- `enabled_bridges[] = TwitterBridge`

**访问链路**：
```
匿名用户访问 /
  → TokenAuthenticationMiddleware（token 为空，直接放行）
  → FrontpageAction
       → 仅 isEnabled("TwitterBridge") = true，其他 bridge 不渲染
匿名用户构造 ?action=display&bridge=TwitchBridge
  → TokenAuthenticationMiddleware（放行）
  → DisplayAction
       → isEnabled("TwitchBridge") = false
       → 返回 400 "This bridge is not whitelisted"
```

### 场景 C：严格企业部署（Basic + Token + 部分白名单）

**配置**：
- `authentication.enable = true`，`username/password` 已设置
- `authentication.token = "corp-token-xyz"`
- `enabled_bridges[] = GithubIssueBridge`

**访问链路**：
```
请求进入
  → BasicAuthMiddleware 检查 Authorization 头
       → 缺失 → 401 + WWW-Authenticate: Basic 弹窗
       → 错误 → 401 重试
       → 正确 → 继续
  → TokenAuthenticationMiddleware 检查 ?token=
       → 缺失/错误 → 401 + token 输入表单
       → 正确 → 注入 attribute
  → DisplayAction 请求 GithubIssueBridge
       → isEnabled = true → 正常返回数据
  → DisplayAction 请求 TwitterBridge
       → isEnabled = false → 400 not whitelisted
```

---

## 六、关键代码索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|---------|---------|
| 中间件流水线编排 | `lib/RssBridge.php` | 25-39 |
| Whitelist 文件加载 | `lib/Configuration.php` | 46-53 |
| enabled_bridges 校验 | `lib/Configuration.php` | 88-90 |
| 环境变量 enabled_bridges 解析 | `lib/Configuration.php` | 71-74 |
| Configuration 静态存储结构 | `lib/Configuration.php` | 12 |
| Configuration getConfig 读取 | `lib/Configuration.php` | 158-164 |
| BridgeFactory 白名单构建 | `lib/BridgeFactory.php` | 25-42 |
| BridgeFactory isEnabled | `lib/BridgeFactory.php` | 49-52 |
| Bridge 名称规范化 | `lib/BridgeFactory.php` | 66-75 |
| DisplayAction 白名单检查 | `actions/DisplayAction.php` | 36-38 |
| DisplayAction 剥离 token | `actions/DisplayAction.php` | 77 |
| DisplayAction 200 缓存写入 | `actions/DisplayAction.php` | 56-63 |
| FrontpageAction 过滤未启用 bridge | `actions/FrontpageAction.php` | 30-36 |
| FrontpageAction 表单注入 token | `actions/FrontpageAction.php` | 163-169 |
| ListAction status 标记 | `actions/ListAction.php` | 22-24 |
| Token 中间件主逻辑 | `middlewares/TokenAuthenticationMiddleware.php` | 7-32 |
| Basic Auth 中间件 | `middlewares/BasicAuthMiddleware.php` | 10-37 |
| CacheMiddleware 缓存命中检查 | `middlewares/CacheMiddleware.php` | 23-41 |
| CacheMiddleware 错误缓存策略 | `middlewares/CacheMiddleware.php` | 46-54 |
| CacheMiddleware 1% 概率 prune | `middlewares/CacheMiddleware.php` | 56-60 |
| 400 错误 TTL 精确计算 | `middlewares/CacheMiddleware.php` | 48-50 |
| CacheInterface 接口定义 | `lib/CacheInterface.php` | 3-14 |
| FileCache prune 实现 | `caches/FileCache.php` | 88-113 |
| SQLiteCache prune 实现 | `caches/SQLiteCache.php` | 112-124 |
| FileCache 缓存文件名（md5） | `caches/FileCache.php` | 116-118 |
| SQLiteCache 缓存 key（sha1） | `caches/SQLiteCache.php` | 131-134 |
| 配置加载入口 | `index.php` | 13-14 |
| 配置加载执行 | `lib/config.php` | 13 |
| Container 单例缓存 | `lib/Container.php` | 8, 21-24 |
| BridgeFactory 容器注册 | `lib/dependencies.php` | 35-37 |
| Request toArray 构成 | `lib/http.php` | 248-251 |
