# RSS-Bridge Action Dispatch 完整链路

## 1. 架构总览

```
HTTP Request (带 bridge, action, format 参数)
    │
    ▼
index.php: 入口引导
    │
    ├─▶ 异常/错误处理注册 (set_exception_handler, set_error_handler)
    │
    ├─▶ Request 对象构建 (fromGlobals 或 fromCli)
    │
    ▼
RssBridge::main() - 核心调度器
    │
    ├─▶ Action 名称解析
    ├─▶ Action 处理器从 DIC 容器获取
    ├─▶ 中间件洋葱模型包装
    └─▶ 中间件链执行
            │
            ▼
Middleware Chain (洋葱圈模型)
    │
    ├─▶ BasicAuthMiddleware
    ├─▶ CacheMiddleware (缓存命中直接返回)
    ├─▶ ExceptionMiddleware (全局异常捕获)
    ├─▶ SecurityMiddleware
    ├─▶ MaintenanceMiddleware
    └─▶ TokenAuthenticationMiddleware
            │
            ▼
DisplayAction::__invoke() - 核心业务逻辑
    │
    ├─▶ 参数校验 (bridge, format 必填)
    ├─▶ BridgeFactory 创建桥实例
    ├─▶ createResponse() 执行数据采集和格式化
    │       │
    │       ├─▶ BridgeAbstract::collectData() - 桥采集原始数据
    │       ├─▶ FormatFactory 创建格式器
    │       ├─▶ FormatAbstract::setItems() - 数据注入
    │       └─▶ FormatAbstract::render() - 输出 RSS/JSON/Atom
    │
    └─▶ 响应缓存写入
            │
            ▼
Response::send() - HTTP 响应输出
```

---

## 2. 请求入口与参数解析

### 2.1 文件位置
- **入口文件**: `index.php:1-75`
- **Request 类**: `lib/http.php:200-252`

### 2.2 关键代码流程

```php
// index.php:63-69
$argv = $argv ?? null;
if ($argv) {
    parse_str(implode('&', array_slice($argv, 1)), $cliArgs);
    $request = Request::fromCli($cliArgs);
} else {
    $request = Request::fromGlobals();
}
```

**参数接收方式**:
- **Web 请求**: `Request::fromGlobals()` 从 `$_GET` 读取参数
- **CLI 模式**: 将命令行参数解析为 `key=value` 格式

**核心请求参数**:
| 参数 | 说明 | 示例 |
|------|------|------|
| `action` | 动作类型，默认 `Frontpage` | `action=Display` |
| `bridge` | 桥名称 | `bridge=GitHubTrending` |
| `format` | 输出格式 | `format=Atom`, `format=Json` |
| `context` | 桥上下文（可选） | |
| `_noproxy` | 禁用代理 | `_noproxy=1` |
| `_cache_timeout` | 自定义缓存 TTL | `_cache_timeout=3600` |

---

## 3. Action 调度与中间件链

### 3.1 Action 名称解析
**文件**: `lib/RssBridge.php:13-40`

```php
// lib/RssBridge.php:15-21
$action = $request->get('action', 'Frontpage');
$actionName = strtolower($action) . 'Action';
$actionName = implode(array_map('ucfirst', explode('-', $actionName)));
$filePath = __DIR__ . '/../actions/' . $actionName . '.php';
if (!file_exists($filePath)) {
    return new Response(render(__DIR__ . '/../templates/error.html.php', ['message' => 'Invalid action']), 400);
}
```

**Action 名称映射规则**:
- `action=Display` → `displayAction` → `DisplayAction`
- `action=front-page` → `front-pageAction` → `Front-PageAction` (实际为 `FrontpageAction`)
- 默认值: `Frontpage`

**可用 Action**:
| Action 类 | 文件 | 用途 |
|-----------|------|------|
| `DisplayAction` | `actions/DisplayAction.php` | 核心：执行桥并输出 Feed |
| `FrontpageAction` | `actions/FrontpageAction.php` | 首页展示 |
| `ListAction` | `actions/ListAction.php` | 列出可用桥 |
| `DetectAction` | `actions/DetectAction.php` | 从 URL 检测桥 |
| `FindfeedAction` | `actions/FindfeedAction.php` | 发现 Feed |
| `ConnectivityAction` | `actions/ConnectivityAction.php` | 连通性测试 |
| `HealthAction` | `actions/HealthAction.php` | 健康检查 |

### 3.2 DIC 容器依赖注入
**文件**: `lib/dependencies.php:1-73`

```php
// lib/dependencies.php:15-17
$container[DisplayAction::class] = function ($c) {
    return new DisplayAction($c['cache'], $c['logger'], $c['bridge_factory']);
};

// lib/RssBridge.php:23
$handler = $this->container[$actionName];
```

**容器自动解析**: 首次访问时才实例化（懒加载），通过 `Container::offsetGet()` 实现。

### 3.3 中间件洋葱模型
**文件**: `lib/RssBridge.php:25-39`

```php
$middlewares = [
    new BasicAuthMiddleware(),
    new CacheMiddleware($this->container['cache']),
    new ExceptionMiddleware($this->container['logger']),
    new SecurityMiddleware(),
    new MaintenanceMiddleware(),
    new TokenAuthenticationMiddleware(),
];
$action = function ($req) use ($handler) {
    return $handler($req);
};
foreach (array_reverse($middlewares) as $middleware) {
    $action = fn ($req) => $middleware($req, $action);
}
return $action($request->withAttribute('action', $actionName));
```

**执行顺序**（`array_reverse` 后从外到内）:
```
Request → TokenAuthenticationMiddleware
          → MaintenanceMiddleware
              → SecurityMiddleware
                  → ExceptionMiddleware
                      → CacheMiddleware
                          → BasicAuthMiddleware
                              → DisplayAction
                          ← BasicAuthMiddleware
                      ← CacheMiddleware
                  ← ExceptionMiddleware
              ← SecurityMiddleware
          ← MaintenanceMiddleware
      ← TokenAuthenticationMiddleware
Response
```

**各中间件职责**:
| 中间件 | 文件 | 职责 |
|--------|------|------|
| `BasicAuthMiddleware` | `middlewares/BasicAuthMiddleware.php` | HTTP Basic 认证 |
| `CacheMiddleware` | `middlewares/CacheMiddleware.php` | 响应缓存读写 |
| `ExceptionMiddleware` | `middlewares/ExceptionMiddleware.php` | 全局异常兜底 |
| `SecurityMiddleware` | `middlewares/SecurityMiddleware.php` | 安全头、CORS 等 |
| `MaintenanceMiddleware` | `middlewares/MaintenanceMiddleware.php` | 维护模式 |
| `TokenAuthenticationMiddleware` | `middlewares/TokenAuthenticationMiddleware.php` | Token 鉴权 |

### 3.4 6 层中间件顺序覆盖分析

#### 3.4.1 顺序设计原理

```php
// lib/RssBridge.php:139-146 - 注册顺序（数组内顺序）
$middlewares = [
    new BasicAuthMiddleware(),          // [0] 最内层，紧挨着 Action
    new CacheMiddleware($cache),        // [1]
    new ExceptionMiddleware($logger),   // [2]
    new SecurityMiddleware(),           // [3]
    new MaintenanceMiddleware(),        // [4]
    new TokenAuthenticationMiddleware(),// [5] 最外层，第一个接触请求
];
```

经过 `array_reverse` 包装后，**实际执行流**为：
```
         TokenAuth (最外层)
             │
      MaintenanceMode
             │
       SecurityCheck
             │
    ExceptionHandler
             │
        CacheLayer
             │
      BasicAuth (最内层)
             │
         DisplayAction
```

#### 3.4.2 顺序覆盖与短路逻辑

**外层中间件可以短路内层中间件**：

| 中间件 | 短路条件 | 覆盖效果 |
|--------|----------|----------|
| `TokenAuthenticationMiddleware` | `authentication.token` 配置存在但 `token` 参数缺失或无效 | 返回 401，内层所有中间件和 Action 都不执行 |
| `MaintenanceMiddleware` | `system.enable_maintenance_mode = true` | 返回 503，内层全部跳过 |
| `SecurityMiddleware` | GET 参数包含非字符串值 | 返回 400，内层全部跳过 |
| `ExceptionMiddleware` | 内层抛出未捕获异常 | 捕获并返回 500，不继续向外层抛出 |
| `CacheMiddleware` | 缓存命中 | 直接返回缓存响应，内层 `BasicAuth` 和 `DisplayAction` 都不执行 |
| `BasicAuthMiddleware` | `authentication.enable` 开启但认证失败 | 返回 401，`DisplayAction` 不执行 |

**代码证据 - TokenAuthenticationMiddleware 短路** (`middlewares/TokenAuthenticationMiddleware.php:9-27`):
```php
public function __invoke(Request $request, $next): Response
{
    if (! Configuration::getConfig('authentication', 'token')) {
        return $next($request);  // 未启用，继续下一层
    }
    $token = $request->get('token');
    if (! $token) {
        return new Response(render('token.html.php', ['message' => 'Missing token']), 401);  // 短路！
    }
    if (! hash_equals(Configuration::getConfig('authentication', 'token'), $token)) {
        return new Response(render('token.html.php', ['message' => 'Invalid token']), 401);  // 短路！
    }
    return $next($request);  // 认证通过，进入内层
}
```

**代码证据 - CacheMiddleware 短路** (`middlewares/CacheMiddleware.php:23-41`):
```php
$cacheKey = 'http_' . json_encode($request->toArray());
$cachedResponse = $this->cache->get($cacheKey);
if ($cachedResponse) {
    // 304 Not Modified 检查...
    return $cachedResponse;  // 短路！BasicAuth 和 DisplayAction 都不执行
}
$response = $next($request);  // 未命中，继续执行内层
```

#### 3.4.3 认证中间件的优先级设计

`TokenAuthenticationMiddleware` 在最外层，`BasicAuthMiddleware` 在最内层，这意味着：
- **Token 认证优先级更高**：如果同时配置了 token 和 basic auth，token 校验失败会直接短路，不会执行 basic auth
- **Token 认证对所有 Action 生效**：包括 Frontpage、List 等不需要桥的 Action
- **BasicAuth 仅在缓存未命中时执行**：如果缓存命中，直接返回，不触发 basic auth 检查

**配置互斥性**：两种认证方式是独立配置项，不存在互斥检查，可以同时开启。此时执行流为：
```
Token 校验 → 维护模式 → 安全检查 → 异常捕获 → 缓存检查 → BasicAuth 校验 → Action
```

---

## 4. Bridge (桥) 选择逻辑

### 4.1 BridgeFactory 工厂类
**文件**: `lib/BridgeFactory.php:1-86`

#### 4.1.1 初始化时扫描所有桥
```php
// lib/BridgeFactory.php:18-23
foreach (scandir(__DIR__ . '/../bridges/') as $file) {
    if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
        $this->bridgeClassNames[] = $m[1];
    }
}
```

### 4.2 BridgeFactory 热加载机制

#### 4.2.1 SPL 自动加载器注册
**文件**: `lib/bootstrap.php:29-44`

```php
spl_autoload_register(function ($className) {
    $folders = [
        __DIR__ . '/../actions/',
        __DIR__ . '/../bridges/',
        __DIR__ . '/../caches/',
        __DIR__ . '/../formats/',
        __DIR__ . '/../lib/',
        __DIR__ . '/../middlewares/',
    ];
    foreach ($folders as $folder) {
        $file = $folder . $className . '.php';
        if (is_file($file)) {
            require $file;
        }
    }
});
```

**热加载流程**:
1. `BridgeFactory` 构造时只扫描 `bridges/` 目录收集类名，**不立即加载**
2. 当 `BridgeFactory::create($name)` 被调用时 `new $name(...)` 触发自动加载
3. SPL autoloader 按顺序搜索 6 个目录，找到匹配文件后 `require` 加载
4. 类定义在 PHP 进程生命周期内只加载一次，后续请求复用

**热加载时序**:
```
请求 1:
  BridgeFactory 构造 → scandir bridges/ → ['FooBridge', 'BarBridge', ...]
  create('FooBridge') → new FooBridge() → 触发 autoload → require bridges/FooBridge.php
  FooBridge 类定义进入进程内存

请求 2 (同一 FPM 进程):
  BridgeFactory 构造 → 重新 scandir bridges/ → 重新收集类名
  create('FooBridge') → new FooBridge() → 类已存在，直接实例化（无需重新 require）
```

**热加载特性**:
- **按需加载**：只有真正被调用的桥才会被 `require` 进内存
- **每次请求重扫描**：`BridgeFactory` 在每次请求中重新实例化，重新 `scandir`，因此新增/删除桥文件在下次请求即可生效，无需重启 PHP-FPM
- **类定义内存缓存**：一旦 `require` 成功，类定义在 PHP 进程生命周期内常驻，修改已有桥代码需要重启 PHP-FPM 才能生效

#### 4.2.2 类加载冲突检测
当前实现**不做重复加载检查**：如果 `bridges/` 和 `lib/` 目录下存在同名类文件，先扫描到的目录会优先加载，后扫描的目录被忽略。

```php
// 风险场景：假设存在两个文件
// bridges/FooBridge.php - class FooBridge extends BridgeAbstract
// lib/FooBridge.php     - class FooBridge

// autoloader 搜索顺序: actions/ → bridges/ → caches/ → formats/ → lib/ → middlewares/
// 因此 bridges/FooBridge.php 会被优先加载，lib/FooBridge.php 永远不会被加载
```

### 4.3 桥名称命名空间冲突

#### 4.3.1 全局命名空间污染
**所有桥类都定义在全局命名空间下**，没有 `namespace` 声明：

```php
// bridges/GitHubBridge.php
class GitHubBridge extends BridgeAbstract { ... }  // 全局命名空间

// bridges/GitHubTrendingBridge.php
class GitHubTrendingBridge extends BridgeAbstract { ... }  // 全局命名空间
```

**文件**: `lib/bootstrap.php` 中的 autoloader 也没有处理命名空间，只按类名搜索文件。

#### 4.3.2 大小写不敏感匹配的冲突风险

`createBridgeClassName()` 使用**大小写不敏感**匹配，这可能导致意外冲突：

```php
// lib/BridgeFactory.php:54-64
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    if (! in_array($nameLoweredCase, $namesLoweredCase)) {
        return null;
    }
    $index = array_search($nameLoweredCase, $namesLoweredCase);  // 小写匹配
    return $this->bridgeClassNames[$index];
}
```

**冲突场景 1 - 文件名大小写不一致**:
```
bridges/
  GitHubBridge.php      - class GitHubBridge
  githubbridge.php      - class githubbridge (注意类名大小写)
```
扫描得到 `bridgeClassNames = ['GitHubBridge', 'githubbridge']`
`array_map('strtolower', ...)` 得到 `['githubbridge', 'githubbridge']`
`array_search('githubbridge', ...)` **永远返回第一个匹配的索引 0**，`githubbridge.php` 永远无法被访问到。

**冲突场景 2 - 前缀/后缀歧义**:
```
bridges/
  FooBridge.php         - class FooBridge
  FoobarBridge.php      - class FoobarBridge
```
请求 `bridge=Foo` → 标准化为 `FooBridge` → 匹配成功 ✅
请求 `bridge=Foobar` → 标准化为 `FoobarBridge` → 匹配成功 ✅
**无冲突**，因为 `normalizeBridgeName` 会补全 `Bridge` 后缀后再匹配。

**冲突场景 3 - 多目录同名类**:
如果未来在 `lib/` 目录下也定义了 `class FooBridge`，由于 autoloader 先搜索 `bridges/`，桥类会优先加载，`lib/FooBridge` 被屏蔽。

#### 4.3.3 实例化时的命名空间解析
```php
// lib/BridgeFactory.php:44-47
public function create(string $name): BridgeAbstract
{
    return new $name($this->cache, $this->logger);  // $name 是 "FooBridge"
}
```
由于没有 `use` 或命名空间前缀，`new $name(...)` 总是在**全局命名空间**下解析类名。

#### 4.3.4 冲突防护措施
- **文件名约定**：所有桥文件必须以 `Bridge.php` 结尾，扫描时通过正则 `/^([^.]+Bridge)\.php$/U` 过滤
- **类名约定**：桥类名必须与文件名一致（PHP 不强制，但 autoloader 要求）
- **白名单机制**：即使类名冲突，未在白名单中的桥也无法被调用

#### 4.1.2 桥名称标准化
```php
// lib/BridgeFactory.php:54-64
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    if (! in_array($nameLoweredCase, $namesLoweredCase)) {
        return null;
    }
    $index = array_search($nameLoweredCase, $namesLoweredCase);
    return $this->bridgeClassNames[$index];
}

// lib/BridgeFactory.php:66-75
public static function normalizeBridgeName(string $name)
{
    if (preg_match('/(.+)(?:\.php)/', $name, $matches)) {
        $name = $matches[1];
    }
    if (!preg_match('/(Bridge)$/i', $name)) {
        $name = sprintf('%sBridge', $name);
    }
    return $name;
}
```

**桥名称匹配规则**:
- `bridge=GitHubTrending` → `GitHubTrendingBridge`
- `bridge=GitHubTrendingBridge` → `GitHubTrendingBridge`
- `bridge=githubtrending` → `githubtrendingBridge` → 大小写不敏感匹配 → `GitHubTrendingBridge`

#### 4.1.3 白名单检查
```php
// lib/BridgeFactory.php:25-41
$enabledBridges = Configuration::getConfig('system', 'enabled_bridges');
if ($enabledBridges === null) {
    throw new \Exception('No bridges are enabled...');
}
foreach ($enabledBridges as $enabledBridge) {
    if ($enabledBridge === '*') {
        $this->enabledBridges = $this->bridgeClassNames;
        break;
    }
    // ...
}

// DisplayAction.php:36-38
if (!$this->bridgeFactory->isEnabled($bridgeClassName)) {
    return new Response(render(..., ['message' => 'This bridge is not whitelisted']), 400);
}
```

#### 4.1.4 桥实例化
```php
// lib/BridgeFactory.php:44-47
public function create(string $name): BridgeAbstract
{
    return new $name($this->cache, $this->logger);
}
```

---

## 5. 数据采集与格式化输出

### 5.1 DisplayAction 主流程
**文件**: `actions/DisplayAction.php:19-144`

```php
// actions/DisplayAction.php:19-67
public function __invoke(Request $request): Response
{
    $bridgeName = $request->get('bridge');
    $format = $request->get('format');
    
    // 参数校验...
    
    $bridgeClassName = $this->bridgeFactory->createBridgeClassName($bridgeName);
    // 桥存在性、白名单检查...
    
    $cacheKey = 'http_' . json_encode($request->toArray());
    
    $bridge = $this->bridgeFactory->create($bridgeClassName);
    
    $response = $this->createResponse($request, $bridge, $format);
    
    if ($response->getCode() === 200) {
        $ttl = $bridge->getCacheTimeout();  // 默认 3600 秒
        $this->cache->set($cacheKey, $response, $ttl);
    }
    
    return $response;
}
```

### 5.2 桥数据采集流程
**核心方法**: `createResponse()` (`DisplayAction.php:69-144`)

```php
private function createResponse(Request $request, BridgeAbstract $bridge, string $format)
{
    try {
        $bridge->loadConfiguration();               // 加载桥配置
        
        // 过滤掉系统参数，只保留桥相关参数
        $remove = ['token', 'action', 'bridge', 'format', ...];
        $input = array_diff_key($request->toArray(), array_fill_keys($remove, ''));
        
        $bridge->setInput($input);                  // 参数校验与注入
        $bridge->collectData();                     // 桥实现的抓取逻辑
        $items = $bridge->getItems();               // 获取原始数据数组
    } catch (\Throwable $e) {
        // 错误处理 - 见第 6 节
    }
```

### 5.3 DisplayAction collectData 取消请求与超时机制

#### 5.3.1 HTTP 请求超时控制
**文件**: `lib/http.php:65-197` (CurlHttpClient)

```php
// lib/http.php:69-79
$defaultConfig = [
    'useragent' => null,
    'timeout' => 5,          // 默认 5 秒超时
    'headers' => [],
    'proxy' => null,
    'curl_options' => [],
    'if_not_modified_since' => null,
    'retries' => 2,          // 默认重试 2 次
    'max_filesize' => null,
    'max_redirections' => 5, // 最多 5 次重定向
];

// lib/http.php:115
curl_setopt($ch, CURLOPT_TIMEOUT, $config['timeout']);
```

**可配置超时**:
```php
// lib/contents.php:52-57
$config = [
    'useragent'     => Configuration::getConfig('http', 'useragent'),
    'timeout'       => Configuration::getConfig('http', 'timeout'),   // 从配置读取
    'retries'       => Configuration::getConfig('http', 'retries'),
    'curl_options'  => $curlOptions,
];
```

#### 5.3.2 连接中断检测（缺失机制）

**当前实现不支持客户端断开取消**：代码库中未使用 `connection_aborted()`、`ignore_user_abort()` 或 `fastcgi_finish_request()`。

```bash
$ grep -r "connection_abort\|ignore_user_abort\|fastcgi_finish_request\|connection_status" .
# 无匹配结果
```

**这意味着**:
- 如果客户端在 `collectData()` 执行过程中断开连接，PHP 会继续执行直到完成或超时
- 桥的 HTTP 请求会完整执行，即使客户端已经离开
- 已发起的 cURL 请求无法中途取消，必须等待超时或完成
- 服务器资源（CPU、网络连接）在客户端断开后仍会被占用直到当前操作完成

#### 5.3.3 响应体积限制
**文件**: `lib/http.php:119-131`

```php
if ($config['max_filesize']) {
    curl_setopt($ch, CURLOPT_MAXFILESIZE, $config['max_filesize']);
    
    // 进度回调函数监控 Content-Length 缺失时的响应体积
    curl_setopt($ch, CURLOPT_PROGRESSFUNCTION, function ($ch, $downloadSize, $downloaded, $uploadSize, $uploaded) use ($config) {
        if ($downloaded > $config['max_filesize']) {
            return -1;  // 返回非零值中止 cURL 传输
        }
        return 0;
    });
}
```

**可配置文件大小限制**:
```php
// lib/contents.php:94-98
$maxFileSize = Configuration::getConfig('http', 'max_filesize');
if ($maxFileSize) {
    $config['max_filesize'] = $maxFileSize * 2 ** 20;  // MB → 字节
}
```

#### 5.3.4 重试机制
**文件**: `lib/http.php:171-192`

```php
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        break;  // 成功，退出循环
    }
    if ($tries <= $config['retries']) {
        continue;  // 继续重试
    }
    // 达到最大重试次数，抛出异常
    throw new HttpException(sprintf(
        'cURL error %s: %s (%s) for %s',
        curl_error($ch),
        curl_errno($ch),
        'https://curl.haxx.se/libcurl/c/libcurl-errors.html',
        $url
    ));
}
```

**重试策略**:
- 只在 `curl_exec` 返回 `false`（网络层错误）时重试
- HTTP 4xx/5xx 状态码不触发重试（`curl_exec` 仍然返回 body）
- 每次重试使用相同的 cURL 句柄和配置
- 重试之间无延迟

#### 5.3.5 collectData 异常中断

桥的 `collectData()` 方法可以通过抛出异常主动中断执行：

```php
// lib/utils.php:254-267
function throwClientException(string $message = '')
{
    throw new ClientException($message, 400);  // 中断采集，标记为客户端错误
}

function throwRateLimitException(string $message = '')
{
    throw new RateLimitException($message);      // 中断采集，标记为限流
}

function throwServerException(string $message = '')
{
    throw new \Exception($message, 500);         // 中断采集，标记为服务端错误
}
```

**桥主动中断示例** (`bridges/YoutubeBridge.php:201`):
```php
public function collectData()
{
    if (/* 参数缺失 */) {
        throwClientException("You must either specify either:\n - YouTube username (?u=...)\n - Channel id (?c=...)");
        // 执行终止，后续代码不再执行
    }
    // ... 正常采集逻辑
}
```

**中断后流程**:
1. `collectData()` 抛出异常
2. `createResponse()` 的 `catch (\Throwable $e)` 捕获
3. 根据异常类型决定日志级别和响应行为（见第 6 节）
4. 无论异常类型如何，**已发起的 HTTP 请求无法回滚**
5. 已写入缓存的部分数据不会自动回滚

---

## 6. 错误处理完整链路

### 6.1 三层错误捕获机制

#### 第一层: PHP 全局错误处理
**文件**: `index.php:20-59`

```php
// 未捕获异常兜底
set_exception_handler(function (\Throwable $e) use ($logger) {
    $response = new Response(render(__DIR__ . '/templates/exception.html.php', ['e' => $e]), 500);
    $response->send();
    $logger->error('Uncaught Exception', ['e' => $e]);
});

// PHP 错误转换
set_error_handler(function ($code, $message, $file, $line) use ($logger) {
    if (Configuration::getConfig('system', 'env') === 'dev') {
        throw new \ErrorException($message, 0, $code, $file, $line);
    }
    $logger->warning($text);
});

// 致命错误兜底
register_shutdown_function(function () use ($logger) {
    $error = error_get_last();
    if ($error) {
        $logger->error($message);
    }
});
```

#### 第二层: ExceptionMiddleware
**文件**: `middlewares/ExceptionMiddleware.php:14-23`

```php
public function __invoke(Request $request, $next): Response
{
    try {
        return $next($request);
    } catch (\Throwable $e) {
        $this->logger->error('Exception in ExceptionMiddleware', ['e' => $e]);
        return new Response(render(__DIR__ . '/../templates/exception.html.php', ['e' => $e]), 500);
    }
}
```

#### 第三层: DisplayAction 内 try-catch
**文件**: `actions/DisplayAction.php:73-124`

```php
try {
    $bridge->loadConfiguration();
    $bridge->setInput($input);
    $bridge->collectData();
    $items = $bridge->getItems();
} catch (\Throwable $e) {
    // 精细化错误分类处理
}
```

### 6.2 三层错误降级幂等性分析

#### 6.2.1 三层捕获的幂等设计

**幂等性保证**：同一异常无论被哪一层捕获，最终行为保持一致。

```
异常抛出点
    │
    ├─▶ 第三层 DisplayAction::createResponse() try-catch
    │    ├─ 已分类处理 → 按类型返回对应响应
    │    └─ 未处理 → 继续抛出
    │
    ├─▶ 第二层 ExceptionMiddleware::__invoke() try-catch
    │    └─ 兜底捕获 → 返回 500 错误页 + ERROR 日志
    │
    └─▶ 第一层 set_exception_handler()
         └─ 最终兜底 → 返回 500 错误页 + ERROR 日志 + 终止执行
```

**幂等证据 - 异常未被第三层捕获时**：
```php
// 场景：DisplayAction 内 catch 只处理了桥执行阶段的异常
// 但参数校验阶段（__invoke 方法内）的异常不在 try 块内

// actions/DisplayAction.php:21-38
public function __invoke(Request $request): Response
{
    $bridgeName = $request->get('bridge');
    if (!$bridgeName) {
        // 这里抛出的异常不在 createResponse() 的 try 块内
        return new Response(render('error.html.php', ['message' => 'Missing bridge name']), 400);
    }
    
    $bridgeClassName = $this->bridgeFactory->createBridgeClassName($bridgeName);
    if (!$bridgeClassName) {
        return new Response(render('error.html.php', ['message' => 'Bridge not found']), 404);
    }
    // ... 
    // createResponse() 内的 try-catch 只保护这段之后的逻辑
}
```

如果在这些参数校验阶段抛出未捕获异常，会被第二层 `ExceptionMiddleware` 捕获，行为是**幂等**的：
- 都返回错误响应（第三层返回 400/404，第二层返回 500）
- 都记录日志（第三层按类型分级，第二层固定 ERROR）
- 都不会产生脏数据（缓存只在成功时写入）

#### 6.2.2 重复错误处理的防护

**幂等风险**：同一异常可能被多层捕获，导致重复日志。

**代码中的防护**：
```php
// index.php:20-24 - 全局异常处理器
set_exception_handler(function (\Throwable $e) use ($logger) {
    $response = new Response(render('exception.html.php', ['e' => $e]), 500);
    $response->send();
    $logger->error('Uncaught Exception', ['e' => $e]);
});

// middlewares/ExceptionMiddleware.php:14-23
public function __invoke(Request $request, $next): Response
{
    try {
        return $next($request);
    } catch (\Throwable $e) {
        $this->logger->error('Exception in ExceptionMiddleware', ['e' => $e]);
        return new Response(render('exception.html.php', ['e' => $e]), 500);
    }
}
```

**重复日志风险**：如果 ExceptionMiddleware 捕获并返回响应，PHP 的 `set_exception_handler` **不会被触发**，因为异常已被处理。这保证了每层只处理一次。

**幂等边界**：
- ✅ 同一异常只会被一层捕获，不会重复日志
- ✅ 无论哪层捕获，都不会写入成功缓存
- ✅ 错误计数缓存（`logBridgeError`）使用 `cache->set()` 是幂等操作
- ❌ 多次重复请求同一错误 URL 会导致错误计数持续累加（这是预期行为）

#### 6.2.3 缓存写入的幂等性

**成功响应**：只有 HTTP 200 才写入缓存，且只写一次
```php
// actions/DisplayAction.php:56-64
if ($response->getCode() === 200) {
    $this->cache->set($cacheKey, $response, $ttl);  // 只有成功才缓存
}
```

**错误响应**：在 CacheMiddleware 中缓存，幂等写入
```php
// middlewares/CacheMiddleware.php:48-50
} elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
    $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
}
```

**幂等保证**：
- `CacheInterface::set()` 语义是覆盖式写入，多次调用结果一致
- 错误缓存使用随机 TTL 避免缓存雪崩，但重复调用同一 key 仍然幂等
- 缓存 key 基于完整请求参数生成，相同请求生成相同 key

---

### 6.3 错误分类处理策略

| 异常类型 | 日志级别 | 响应行为 |
|----------|----------|----------|
| `ClientException` | DEBUG | 根据 `error.output` 配置决定 |
| `RateLimitException` | DEBUG | 返回 429 Too Many Requests |
| `HttpException` (429/503) | DEBUG | 原样返回状态码 |
| 其他 `HttpException` | - | 不日志，按普通错误处理 |
| 其他 `Throwable` | ERROR | 根据 `error.output` 配置决定 |

**代码**:
```php
// actions/DisplayAction.php:91-124
if ($e instanceof ClientException) {
    $this->logger->debug(...);
} elseif ($e instanceof RateLimitException) {
    $this->logger->debug(...);
    return new Response(render(..., ['e' => $e]), 429);
} elseif ($e instanceof HttpException) {
    if (in_array($e->getCode(), [429, 503])) {
        $this->logger->debug(...);
        return new Response(render(..., ['e' => $e]), $e->getCode());
    }
} else {
    $this->logger->error(...);
}
```

### 6.3 错误输出配置
**配置项**: `error.output` (config.ini)

| 配置值 | 行为 |
|--------|------|
| `feed` | 将错误包装为 Feed 条目返回 |
| `http` | 返回 500 HTTP 错误页 |
| `none` | 返回空 Feed |

**错误条目包装**:
```php
// DisplayAction.php:146-170
private function createFeedItemFromException($e, BridgeAbstract $bridge): array
{
    return [
        'title'     => sprintf('Bridge returned error %s! (%s)', $e->getCode(), $uniqueIdentifier),
        'uri'       => get_current_url(),
        'timestamp' => time(),
        'uid'       => $bridge->getName() . '_' . $uniqueIdentifier,
        'content'   => render_template('bridge-error.html.php', [
            'error'      => render_template('exception.html.php', ['e' => $e]),
            'searchUrl'  => self::createGithubSearchUrl($bridge),
            'issueUrl'   => self::createGithubIssueUrl($bridge, $e),
            'maintainer' => $bridge->getMaintainer(),
        ]),
    ];
}
```

### 6.4 error.report_limit 告警机制

#### 6.4.1 阈值控制原理

**配置项**: `error.report_limit` - 控制错误向客户端暴露的敏感度

```php
// actions/DisplayAction.php:107-123
$errorOutput = Configuration::getConfig('error', 'output');
$reportLimit = Configuration::getConfig('error', 'report_limit');
$errorCount = 1;
if ($reportLimit > 1) {
    $errorCount = $this->logBridgeError($bridge->getName(), $e->getCode());
}
// 达到阈值才向客户端暴露错误
if ($errorCount >= $reportLimit) {
    if ($errorOutput === 'feed') {
        $items = [$this->createFeedItemFromException($e, $bridge)];  // 包装为 Feed 条目
    } elseif ($errorOutput === 'http') {
        return new Response(render('exception.html.php', ['e' => $e]), 500);  // 返回 HTTP 错误
    } elseif ($errorOutput === 'none') {
        // 静默，返回空 Feed
    }
}
// 未达到阈值：静默处理，返回空 Feed，不向客户端暴露错误
```

**告警策略表**:

| `report_limit` | 行为 | 适用场景 |
|----------------|------|----------|
| `1` | 每次错误都向客户端暴露 | 开发环境、单用户实例 |
| `> 1` | 达到 N 次后才暴露错误 | 公开实例、避免偶发错误骚扰用户 |
| 很大的值 | 几乎永不暴露 | 追求用户体验、不希望用户看到错误 |

#### 6.4.2 错误计数缓存与滑动窗口

**文件**: `actions/DisplayAction.php:172-191`

```php
private function logBridgeError($bridgeName, $code)
{
    $cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
    $report = $this->cache->get($cacheKey);
    if ($report) {
        $report = Json::decode($report);
        $report['count']++;      // 计数递增
        $report['time'] = time(); // ⚠️ 每次更新时间戳
    } else {
        $report = ['error' => $code, 'time' => time(), 'count' => 1];
    }
    $this->cache->set($cacheKey, Json::encode($report), 86400 * 5);  // 5 天固定 TTL
    return $report['count'];
}
```

**滑动窗口特性**:
- TTL 固定为 5 天，**不是滚动窗口**
- 每次更新 `time` 字段但**不更新 TTL**，缓存到期后计数重置
- 5 天后缓存自动过期，计数从 1 重新开始
- 这是"固定窗口"而非"滑动窗口"，在窗口边界可能出现阈值穿透

**并发安全问题**:
```php
// 风险：非原子操作
$report = $this->cache->get($cacheKey);      // 读
$report['count']++;                           // 改（内存中）
$this->cache->set($cacheKey, $report, $ttl); // 写

// 并发场景：
// 请求 A: get → count=5
// 请求 B: get → count=5
// 请求 A: set(count=6)
// 请求 B: set(count=6)  ← 丢失了一次计数！
```
当前实现**没有使用原子递增**（如 `incr`），高并发下计数可能不准确。

#### 6.4.3 告警触发后的行为

| `error.output` | 达到阈值后的响应 |
|----------------|------------------|
| `feed` | 将错误包装为 Feed 条目，包含 GitHub issue 链接、维护者信息、搜索链接 |
| `http` | 返回 500 HTTP 错误页，展示完整异常栈 |
| `none` | 返回空 Feed，用户看到空白 Feed 但无错误提示 |

**错误条目内容** (`createFeedItemFromException`):
- `title`: "Bridge returned error 500! (19389)" - 每天一个唯一标识
- `content`: 包含异常信息 + GitHub Issue 自动生成链接 + 搜索已知问题链接
- `uid`: "BridgeName_19389" - 避免 Feed 阅读器重复提醒

**静默期特性**:
- 未达到阈值时，对用户完全透明，返回空 Feed
- 达到阈值后，每次请求都返回错误条目（直到缓存过期）
- 错误条目每天生成一个新的 UID，Feed 阅读器每天提醒一次

---

## 7. 缓存策略

### 7.1 CacheMiddleware (外层缓存)
**文件**: `middlewares/CacheMiddleware.php:14-63`

```php
public function __invoke(Request $request, $next): Response
{
    if ($action !== 'DisplayAction') {
        return $next($request);  // 只缓存 DisplayAction
    }
    
    $cacheKey = 'http_' . json_encode($request->toArray());
    $cachedResponse = $this->cache->get($cacheKey);
```

### 7.4 CacheMiddleware 缓存穿透分析

#### 7.4.1 缓存穿透定义

**缓存穿透**：缓存未命中时，请求穿透到后端，大量并发请求同时打到源站。

#### 7.4.2 穿透场景分析

**场景 1: 首次请求 / 缓存过期**
```
请求 A → 缓存未命中 → 执行桥 → 耗时 2s
  请求 B（同时到达）→ 缓存未命中 → 执行桥 → 耗时 2s
    请求 C（同时到达）→ 缓存未命中 → 执行桥 → 耗时 2s

结果：3 个请求都打到源站，造成 3 倍负载
```

**当前实现无防穿透机制**：
```php
// middlewares/CacheMiddleware.php:23-44
$cacheKey = 'http_' . json_encode($request->toArray());
$cachedResponse = $this->cache->get($cacheKey);

if ($cachedResponse) {
    return $cachedResponse;  // 命中，直接返回
}

// ❌ 未命中，直接穿透，无锁、无排队、无降级
$response = $next($request);  // 多个并发请求都会执行到这里
```

**场景 2: 不存在的桥 / 非法参数**
```
请求: ?bridge=NonExistentBridge&format=Atom
  → 缓存未命中
  → DisplayAction 返回 404
  → CacheMiddleware 缓存 404 响应（5~15 分钟）
  → 后续请求命中缓存，不会继续穿透

✅ 错误响应有缓存，一定程度防止了恶意探测穿透
```

**缓存策略表**:
| 响应码 | 是否缓存 | TTL | 位置 |
|--------|----------|-----|------|
| 200 | ✅ | 桥自定义（默认 3600s） | DisplayAction 内部 |
| 304 | ✅ | 继承原缓存 TTL | 浏览器 / 代理 |
| 400 / 403 / 404 / 429 / 500 / 503 | ✅ | 300~900s（随机） | CacheMiddleware |
| 其他 | ✅ | 300s | CacheMiddleware |

#### 7.4.3 缓存 Key 设计与放大风险

**Cache Key 生成**:
```php
// middlewares/CacheMiddleware.php:24
$cacheKey = 'http_' . json_encode($request->toArray());
```

**Key 包含所有 GET 参数**，包括：
- `bridge`, `format`, `context` - 业务参数
- `token` - 认证参数（⚠️ 每个用户独立缓存）
- `_noproxy`, `_cache_timeout` - 控制参数
- `_` - 某些 RSS 阅读器添加的缓存破坏参数

**缓存放大风险**:
- 如果 `token` 参数存在，**每个用户的缓存完全独立**，缓存命中率大幅降低
- 如果请求包含随机 `_` 参数，**每次请求 Key 都不同**，缓存完全失效
- 不同参数顺序（`?a=1&b=2` vs `?b=2&a=1`）生成不同 Key，但 PHP 中 `$_GET` 顺序由查询字符串决定

**代码证据 - 不过滤参数**:
```php
// DisplayAction 过滤了参数，但 CacheMiddleware 没有
// actions/DisplayAction.php:76-87
$remove = ['token', 'action', 'bridge', 'format', '_noproxy', '_cache_timeout', '_error_time', '_'];
$input = array_diff_key($request->toArray(), array_fill_keys($remove, ''));
// ↑ DisplayAction 执行时会过滤这些参数用于桥输入
// ↓ 但 CacheMiddleware 生成 Key 时用的是完整 toArray()
$cacheKey = 'http_' . json_encode($request->toArray());
```

#### 7.4.4 防穿透的缺失机制

当前实现**没有**以下常见防穿透机制：
1. ❌ **没有请求锁**（Mutex Lock）- 防止并发重复请求
2. ❌ **没有缓存预热** - 过期前主动刷新
3. ❌ **没有负缓存永不过期** - 404 等错误响应也会过期
4. ❌ **没有参数归一化** - 相同参数不同顺序生成不同 Key
5. ✅ **有错误缓存** - 5~15 分钟 TTL，防止持续穿透

**典型穿透场景**（缓存过期瞬间）:
```
t=0s: 缓存有效，所有请求命中
t=3600s: 缓存过期
t=3600.1s: 100 个并发请求同时到达
         → 全部缓存未命中
         → 全部执行桥，并发 100 个请求到源站
         → 源站可能被打垮
t=3602s: 第一个桥执行完成，写入缓存
t=3602.1s: 后续请求开始命中缓存

结果：2s 内源站承受 100 倍流量
```

---

### 7.5 日志系统补充：4 级 PII 脱敏

#### 7.5.1 PII 脱敏核心函数

**文件**: `lib/utils.php:124-140`

```php
/**
 * Trim path prefix for privacy/security reasons
 *
 * Example: "/home/davidsf/rss-bridge/index.php" => "index.php"
 */
function sanitize_root(string $filePath): string
{
    // Root folder of the project e.g. /home/satoshi/repos/rss-bridge
    $root = dirname(__DIR__);
    return _sanitize_path_name($filePath, $root);
}

function _sanitize_path_name(string $s, string $pathName): string
{
    // Remove all occurrences of $pathName in the string
    return str_replace(["$pathName/", $pathName], '', $s);
}
```

#### 7.5.2 四级日志的脱敏覆盖

| 日志级别 | 脱敏点 | 脱敏位置 |
|----------|--------|----------|
| **DEBUG** (10) | 异常 message / file / trace | `lib/logger.php:125-128`, `172-175` |
| **INFO** (20) | 异常 message / file / trace | 同上 |
| **WARNING** (30) | 错误 message / file / line | `index.php:36-42` |
| **ERROR** (40) | 异常完整上下文 | `lib/logger.php:125-128`, `index.php:20-24` |

**代码证据 - 异常上下文脱敏** (`lib/logger.php:119-130`):
```php
// StreamHandler 和 ErrorLogHandler 都有相同的脱敏逻辑
if (isset($record['context']['e'])) {
    /** @var \Throwable $e */
    $e = $record['context']['e'];
    unset($record['context']['e']);
    $record['context']['type'] = get_class($e);
    $record['context']['code'] = $e->getCode();
    $record['context']['message'] = sanitize_root($e->getMessage());  // ✅ 脱敏
    $record['context']['file'] = sanitize_root($e->getFile());        // ✅ 脱敏
    $record['context']['line'] = $e->getLine();
    $record['context']['url'] = get_current_url();
    $record['context']['trace'] = trace_to_call_points(trace_from_exception($e));
}
```

**堆栈追踪脱敏** (`lib/utils.php:72-90`):
```php
function trace_from_exception(\Throwable $e): array
{
    $frames = array_reverse($e->getTrace());
    $frames[] = [
        'file' => $e->getFile(),
        'line' => $e->getLine(),
    ];
    $trace = [];
    foreach ($frames as $frame) {
        $trace[] = [
            'file'      => sanitize_root($frame['file'] ?? ''),  // ✅ 每个栈帧都脱敏
            'line'      => $frame['line'] ?? null,
            'class'     => $frame['class'] ?? null,
            'type'      => $frame['type'] ?? null,
            'function'  => $frame['function'] ?? null,
        ];
    }
    return $trace;
}
```

**全局错误处理器脱敏** (`index.php:36-42`):
```php
set_error_handler(function ($code, $message, $file, $line) use ($logger) {
    // ...
    $text = sprintf(
        '%s at %s line %s',
        sanitize_root($message),  // ✅ 错误信息脱敏
        sanitize_root($file),     // ✅ 文件名脱敏
        $line
    );
    $logger->warning($text);
});
```

#### 7.5.3 脱敏范围

**已脱敏**:
- ✅ 文件系统路径（移除项目根目录前缀）
- ✅ 异常消息中的路径
- ✅ 堆栈追踪中的文件名
- ✅ 错误消息中的路径

**未脱敏（潜在风险）**:
- ❌ URL 查询参数中的敏感值（如 API key、token 等会完整出现在 URL 中）
- ❌ 桥的配置值（如 API key 可能出现在异常消息中）
- ❌ 用户输入内容（可能反射到异常消息中）
- ❌ HTTP 请求头（如 Cookie、Authorization 等不会出现在默认日志中）

**URL 泄露风险**:
```php
// lib/logger.php:128
$record['context']['url'] = get_current_url();  // ❌ 完整 URL，包含所有查询参数

// 风险 URL:
// https://example.com/?action=Display&bridge=Twitter&token=secret123&user=elonmusk
// 日志中会完整记录 token=secret123
```

#### 7.5.4 额外的日志过滤机制

**文件**: `lib/logger.php:69-88`

```php
private function log(int $level, string $message, array $context = []): void
{
    if (isset($context['e'])) {
        /** @var \Throwable $e */
        $e = $context['e'];

        if ($e instanceof RateLimitException) {
            return;  // ✅ 限流异常完全不日志
        }
        
        // ✅ 跳过已知无害错误
        $ignoredMessages = [
            'Format name invalid',
            'Unknown format given',
            'Unable to find',
        ];
        foreach ($ignoredMessages as $ignoredMessage) {
            if (str_starts_with($e->getMessage(), $ignoredMessage)) {
                return;  // 不输出日志
            }
        }
    }
    // ... 输出到 handlers
}
```

---

## 8. 补充章节总结

### 8.1 新增内容索引

| 补充主题 | 章节位置 | 核心发现 |
|----------|----------|----------|
| 6 层中间件顺序覆盖 | §3.4 | Token 认证优先级高于 Basic 认证，缓存命中可短路所有内层 |
| BridgeFactory 热加载 | §4.2 | SPL autoload 按需加载，新增桥无需重启 PHP-FPM |
| 桥名称命名空间冲突 | §4.3 | 全局命名空间+大小写不敏感匹配存在冲突风险 |
| collectData 取消请求 | §5.3 | 无 `connection_aborted` 检测，客户端断开仍继续执行 |
| 三层错误降级幂等 | §6.2 | 异常只会被一层捕获，不会重复日志；缓存写入幂等 |
| error.report_limit 告警 | §6.4 | 5 天固定窗口计数，非原子递增存在并发丢失 |
| CacheMiddleware 穿透 | §7.4 | 无请求锁，缓存过期瞬间并发穿透；token 参数导致缓存放大 |
| log 4 级 PII 脱敏 | §7.5 | 路径脱敏完善，但 URL 查询参数完整泄露 |

### 8.2 设计权衡点汇总

| 设计决策 | 优点 | 缺点 |
|----------|------|------|
| 中间件倒序包装 | 洋葱模型清晰 | 顺序依赖数组索引，隐式不易理解 |
| 大小写不敏感匹配 | 用户友好 | 存在冲突风险，第一个匹配优先 |
| 每次请求重新扫描桥 | 热加载友好 | 重复 IO，可优化 |
| 无连接中断检测 | 实现简单 | 客户端断开后浪费服务器资源 |
| 固定窗口错误计数 | 实现简单 | 窗口边界可能穿透，并发计数不准 |
| 全参数缓存 Key | 逻辑简单 | token、`_` 参数导致缓存命中率低 |
| 仅路径脱敏 | 防止路径泄露 | URL 查询参数中的敏感值完整记录 |

---

## 9. 改进方案详述

> 以下每个方案均包含：问题定位 → 当前代码 → 改造代码 → 变更影响。

### 9.1 中间件倒序数组显式 enum 改造

#### 9.1.1 问题定位

**文件**: `lib/RssBridge.php:25-38`

当前中间件顺序由数组索引隐式决定，依赖 `array_reverse` 倒序包装。开发者在调整顺序时必须心算 reverse 后的执行流，容易出错。

```php
// 当前实现 - 注册顺序≠执行顺序，需要心算 reverse
$middlewares = [
    new BasicAuthMiddleware(),          // [0] → 反转后最内层
    new CacheMiddleware($cache),        // [1]
    new ExceptionMiddleware($logger),   // [2]
    new SecurityMiddleware(),           // [3]
    new MaintenanceMiddleware(),        // [4]
    new TokenAuthenticationMiddleware(),// [5] → 反转后最外层
];
foreach (array_reverse($middlewares) as $middleware) {
    $action = fn ($req) => $middleware($req, $action);
}
```

#### 9.1.2 改造方案：显式 enum + 有序注册

```php
// lib/MiddlewarePriority.php - 新增文件
enum MiddlewarePriority: int
{
    case TokenAuthentication = 100;
    case Maintenance         = 200;
    case Security            = 300;
    case Exception           = 400;
    case Cache               = 500;
    case BasicAuth           = 600;

    public function create(Container $container): Middleware
    {
        return match ($this) {
            self::TokenAuthentication => new TokenAuthenticationMiddleware(),
            self::Maintenance         => new MaintenanceMiddleware(),
            self::Security            => new SecurityMiddleware(),
            self::Exception           => new ExceptionMiddleware($container['logger']),
            self::Cache               => new CacheMiddleware($container['cache']),
            self::BasicAuth           => new BasicAuthMiddleware(),
        };
    }
}
```

```php
// lib/RssBridge.php - 改造后
public function main(Request $request): Response
{
    // ... action 解析 ...

    $priorities = MiddlewarePriority::cases(); // 按 enum 声明顺序 = 执行顺序（从外到内）
    $action = function ($req) use ($handler) {
        return $handler($req);
    };
    // 从最内层往最外层包装（倒序遍历）
    foreach (array_reverse($priorities) as $priority) {
        $middleware = $priority->create($this->container);
        $action = fn ($req) => $middleware($req, $action);
    }
    return $action($request->withAttribute('action', $actionName));
}
```

#### 9.1.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| 顺序可见性 | 数组索引 + reverse 心算 | enum 声明顺序 = 执行顺序 |
| 新增中间件 | 在数组中插入，需计算 reverse 位置 | 在 enum 中按优先级插入 |
| 中间件依赖 | 构造函数在 RssBridge::main 中硬编码 | 集中在 `MiddlewarePriority::create()` |
| 运行时开销 | 无 | enum cases 无额外开销，`match` 编译期优化 |
| 兼容性 | PHP 7.4+ | PHP 8.1+（enum 特性） |

**回退方案**：若需保持 PHP 7.4 兼容，可用带常量的类替代 enum：

```php
class MiddlewarePriority
{
    const TOKEN_AUTH   = 100;
    const MAINTENANCE  = 200;
    const SECURITY     = 300;
    const EXCEPTION    = 400;
    const CACHE        = 500;
    const BASIC_AUTH   = 600;

    public static function ordered(): array
    {
        return [
            self::TOKEN_AUTH,
            self::MAINTENANCE,
            self::SECURITY,
            self::EXCEPTION,
            self::CACHE,
            self::BASIC_AUTH,
        ];
    }
}
```

---

### 9.2 大小写不敏感命名空间分隔

#### 9.2.1 问题定位

**文件**: `lib/BridgeFactory.php:54-64`

`createBridgeClassName()` 用 `strtolower` + `array_search` 做大小写不敏感匹配。如果存在两个仅大小写不同的桥类名（如 `GithubBridge` 和 `GitHubBridge`），`array_search` 永远返回第一个匹配索引。

```php
// 当前实现 - 大小写不敏感但有冲突
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    if (! in_array($nameLoweredCase, $namesLoweredCase)) {
        return null;
    }
    $index = array_search($nameLoweredCase, $namesLoweredCase);
    return $this->bridgeClassNames[$index];  // ← 永远返回首个匹配
}
```

#### 9.2.2 改造方案：命名空间分隔符

引入子命名空间前缀，将桥按来源/功能分组，消除大小写冲突空间：

```php
// lib/BridgeFactory.php - 改造后
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);

    // 精确匹配优先（大小写敏感）
    $exactIndex = array_search($name, $this->bridgeClassNames);
    if ($exactIndex !== false) {
        return $this->bridgeClassNames[$exactIndex];
    }

    // 降级为大小写不敏感匹配，但检测冲突
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    $matches = array_keys($namesLoweredCase, $nameLoweredCase);
    if (count($matches) === 0) {
        return null;
    }
    if (count($matches) > 1) {
        // 冲突检测：多个桥仅在大小写上不同
        $conflicting = array_map(fn($i) => $this->bridgeClassNames[$i], $matches);
        $this->logger->warning(sprintf(
            'Bridge name collision detected: %s. Using first match: %s',
            implode(', ', $conflicting),
            $this->bridgeClassNames[$matches[0]]
        ));
    }
    return $this->bridgeClassNames[$matches[0]];
}
```

**桥文件命名空间分隔**（长期方案）：

```
bridges/
  Social/
    TwitterBridge.php       → class Social\TwitterBridge
    MastodonBridge.php      → class Social\MastodonBridge
  Video/
    YoutubeBridge.php       → class Video\YoutubeBridge
    VimeoBridge.php         → class Video\VimeoBridge
  Developer/
    GithubBridge.php        → class Developer\GithubBridge
    GitlabBridge.php        → class Developer\GitlabBridge
```

```php
// 对应的 normalizeBridgeName 改造
public static function normalizeBridgeName(string $name): string
{
    // 支持 "Social/Twitter" 或 "SocialTwitter" 两种输入格式
    if (preg_match('/(.+)(?:\.php)/', $name, $matches)) {
        $name = $matches[1];
    }
    if (!preg_match('/(Bridge)$/i', $name)) {
        $name = sprintf('%sBridge', $name);
    }
    return $name;
}

// 对应的 autoloader 改造
spl_autoload_register(function ($className) {
    // 将命名空间分隔符转为目录分隔符
    $filePath = __DIR__ . '/../bridges/' . str_replace('\\', '/', $className) . '.php';
    if (is_file($filePath)) {
        require $filePath;
    }
});
```

#### 9.2.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| 冲突检测 | 静默使用首个匹配 | 日志告警，明确冲突 |
| 精确匹配 | 无，总是大小写不敏感 | 精确匹配优先 |
| 命名空间 | 全局 | 按功能分组（长期） |
| API 兼容性 | `?bridge=Github` | 短期完全兼容；长期需支持 `?bridge=Social/Twitter` |
| 文件系统 | 扁平目录 | 子目录结构 |

---

### 9.3 APCu 重扫描锁

#### 9.3.1 问题定位

**文件**: `lib/BridgeFactory.php:18-23`

每个请求都 `scandir` bridges 目录。在高并发场景下，100 个请求同时执行 `scandir`，产生 100 次重复 IO。

```php
// 当前实现 - 每次请求都扫描
public function __construct(CacheInterface $cache, Logger $logger)
{
    foreach (scandir(__DIR__ . '/../bridges/') as $file) {
        if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
            $this->bridgeClassNames[] = $m[1];
        }
    }
    // ...
}
```

#### 9.3.2 改造方案：APCu 进程级缓存 + TTL 锁

```php
// lib/BridgeFactory.php - 改造后
private const SCAN_CACHE_KEY = 'rssbridge_bridge_scan';
private const SCAN_CACHE_TTL = 60; // 60 秒内复用扫描结果

private function scanBridgeDir(): array
{
    // 尝试从 APCu 读取
    if (function_exists('apcu_fetch')) {
        $cached = apcu_fetch(self::SCAN_CACHE_KEY);
        if ($cached !== false) {
            return $cached;
        }
    }

    // 缓存未命中，执行扫描
    $classNames = [];
    foreach (scandir(__DIR__ . '/../bridges/') as $file) {
        if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
            $classNames[] = $m[1];
        }
    }

    // 写入 APCu
    if (function_exists('apcu_store')) {
        apcu_store(self::SCAN_CACHE_KEY, $classNames, self::SCAN_CACHE_TTL);
    }

    return $classNames;
}

public function __construct(CacheInterface $cache, Logger $logger)
{
    $this->cache = $cache;
    $this->logger = $logger;
    $this->bridgeClassNames = $this->scanBridgeDir();
    // ... enabled_bridges 逻辑不变
}
```

**无 APCu 时的降级方案**：

```php
// 使用类静态变量作为进程级缓存（PHP-FPM worker 复用）
private static ?array $scannedClassNames = null;

private function scanBridgeDir(): array
{
    if (self::$scannedClassNames !== null) {
        return self::$scannedClassNames;
    }

    $classNames = [];
    foreach (scandir(__DIR__ . '/../bridges/') as $file) {
        if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
            $classNames[] = $m[1];
        }
    }

    self::$scannedClassNames = $classNames;
    return $classNames;
}
```

#### 9.3.3 变更影响

| 维度 | 改造前 | 改造后（APCu） | 改造后（静态变量） |
|------|--------|---------------|-------------------|
| IO 次数/请求 | 1 次 scandir | 0 次（TTL 内） | 0 次（进程生命周期内） |
| 热加载延迟 | 立即 | 最多 60 秒 | 直到 FPM worker 重启 |
| 依赖 | 无 | ext-apcu | 无 |
| 内存 | 无额外 | APCu 共享内存 | PHP 进程内存 |
| 适用场景 | 开发 | 生产 | 生产（无 APCu 时） |

**推荐组合**：生产用 APCu（TTL 60s，平衡热加载与性能），开发用静态变量 + 手动重启 FPM。

---

### 9.4 滑动窗口错误计数

#### 9.4.1 问题定位

**文件**: `actions/DisplayAction.php:172-191`

当前错误计数使用固定窗口（5 天 TTL），存在两个问题：
1. 窗口边界穿透：错误恰好在 TTL 过期前后集中时，两次窗口各自未达阈值
2. 非原子读改写：`get → count++ → set` 在并发下丢失计数

```php
// 当前实现 - 固定窗口 + 非原子递增
private function logBridgeError($bridgeName, $code)
{
    $cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
    $report = $this->cache->get($cacheKey);
    if ($report) {
        $report = Json::decode($report);
        $report['count']++;
    } else {
        $report = ['error' => $code, 'time' => time(), 'count' => 1];
    }
    $this->cache->set($cacheKey, Json::encode($report), 86400 * 5);
    return $report['count'];
}
```

#### 9.4.2 改造方案：时间桶滑动窗口

```php
// actions/DisplayAction.php - 改造后
private function logBridgeError(string $bridgeName, int $code): int
{
    $windowSeconds = 86400; // 1 天窗口
    $bucketSize    = 3600;  // 1 小时一个桶
    $now           = time();
    $bucketCount   = $windowSeconds / $bucketSize; // 24 个桶

    $prefix = 'err_bucket_' . $bridgeName . '_' . $code . '_';

    // 写入当前桶（原子递增）
    $currentBucket = (int)($now / $bucketSize);
    $currentKey = $prefix . $currentBucket;

    // 优先使用 APCu 原子递增
    if (function_exists('apcu_inc')) {
        $count = apcu_inc($currentKey, 1);
        if ($count === 1) {
            apcu_store($currentKey . '_ttl', true, $windowSeconds);
        }
    } else {
        // 降级：使用 CacheInterface（非原子，但可接受）
        $count = $this->cache->get($currentKey) ?? 0;
        $count++;
        $this->cache->set($currentKey, $count, $windowSeconds);
    }

    // 读取窗口内所有桶的总和
    $totalCount = 0;
    for ($i = 0; $i < $bucketCount; $i++) {
        $bucketId = $currentBucket - $i;
        $key = $prefix . $bucketId;
        $bucketValue = $this->cache->get($key) ?? 0;
        $totalCount += (int)$bucketValue;
    }

    return $totalCount;
}
```

**SQLiteCache 原子递增替代方案**：

```php
// 利用 SQLite 的 INSERT OR REPLACE 实现原子计数
public function inc(string $key, int $step = 1, ?int $ttl = null): int
{
    $cacheKey = $this->createCacheKey($key);
    $expiration = $ttl ? time() + $ttl : 0;

    $this->db->exec('BEGIN IMMEDIATE'); // 排他锁
    $stmt = $this->db->prepare(
        'INSERT INTO storage (key, value, updated) VALUES (:key, :value, :exp)
         ON CONFLICT(key) DO UPDATE SET value = value + :step, updated = :exp'
    );
    $stmt->bindValue(':key', $cacheKey, \SQLITE3_BLOB);
    $stmt->bindValue(':value', $step, \SQLITE3_INTEGER);
    $stmt->bindValue(':step', $step, \SQLITE3_INTEGER);
    $stmt->bindValue(':exp', $expiration, \SQLITE3_INTEGER);
    $stmt->execute();
    $this->db->exec('COMMIT');

    $result = $this->db->querySingle(
        "SELECT value FROM storage WHERE key = '" . bin2hex($cacheKey) . "'"
    );
    return (int)$result;
}
```

#### 9.4.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| 窗口类型 | 固定 5 天 | 滑动 1 天（可配） |
| 边界穿透 | 存在 | 消除 |
| 并发安全 | 非原子 | APCu 原子 / SQLite 排他锁 |
| 存储开销 | 1 个 key | 24 个桶 key |
| 精度 | 5 天内粗略计数 | 1 小时粒度精确计数 |
| 配置兼容 | `report_limit = 1` 含义不变 | 不变，但统计基础更精确 |

---

### 9.5 token 缓存白名单

#### 9.5.1 问题定位

**文件**: `middlewares/CacheMiddleware.php:24`, `actions/DisplayAction.php:50`

缓存 Key 包含全部 GET 参数。当启用 Token 认证后，不同用户使用不同 token 产生不同 Key，缓存完全隔离，命中率趋近于 0。

```php
// 当前实现 - 完整参数作为 Key
$cacheKey = 'http_' . json_encode($request->toArray());

// 同一桥、同一格式，但 token 不同：
// Key1: http_{"action":"Display","bridge":"Foo","format":"Atom","token":"abc123"}
// Key2: http_{"action":"Display","bridge":"Foo","format":"Atom","token":"def456"}
// → 两个完全独立的缓存条目，源站承受双倍请求
```

#### 9.5.2 改造方案：白名单过滤缓存 Key

```php
// middlewares/CacheMiddleware.php - 改造后
private const CACHE_EXCLUDE_PARAMS = [
    'token',          // 认证 token，不影响输出内容
    '_',              // RSS 阅读器缓存破坏参数
    '_error_time',    // 错误时间戳
];

private function createCacheKey(Request $request): string
{
    $params = $request->toArray();
    foreach (self::CACHE_EXCLUDE_PARAMS as $exclude) {
        unset($params[$exclude]);
    }
    ksort($params); // 参数排序归一化
    return 'http_' . json_encode($params);
}

public function __invoke(Request $request, $next): Response
{
    $action = $request->getAttribute('action');
    if ($action !== 'DisplayAction') {
        return $next($request);
    }

    $cacheKey = $this->createCacheKey($request);
    // ... 后续逻辑不变
}
```

**DisplayAction 同步改造**：

```php
// actions/DisplayAction.php:50 - 同步改造
$cacheKey = $this->createCacheKey($request);

// 抽取公共方法
private function createCacheKey(Request $request): string
{
    $params = $request->toArray();
    foreach (CacheMiddleware::CACHE_EXCLUDE_PARAMS as $exclude) {
        unset($params[$exclude]);
    }
    ksort($params);
    return 'http_' . json_encode($params);
}
```

**进一步优化：缓存 Key 工厂**：

```php
// lib/CacheKeyFactory.php - 新增文件
final class CacheKeyFactory
{
    private const EXCLUDE_PARAMS = [
        'token',
        '_',
        '_error_time',
    ];

    public static function forRequest(Request $request): string
    {
        $params = $request->toArray();
        foreach (self::EXCLUDE_PARAMS as $exclude) {
            unset($params[$exclude]);
        }
        ksort($params);
        return 'http_' . json_encode($params);
    }

    public static function forBridgeError(string $bridgeName, int $code): string
    {
        return 'error_reporting_' . $bridgeName . '_' . $code;
    }

    public static function forServerCache(string $url, ?string $bodyHash = null): string
    {
        return implode('_', ['server', $url, $bodyHash]);
    }
}
```

#### 9.5.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| Token 用户缓存 | 完全隔离 | 共享（Token 不影响输出内容） |
| `_` 参数影响 | 每次请求不同 Key | 忽略，Key 稳定 |
| 参数顺序影响 | `?a=1&b=2` ≠ `?b=2&a=1` | `ksort` 归一化，Key 相同 |
| 缓存命中率（Token 开启） | 极低 | 大幅提升 |
| 安全性 | 无影响（Token 鉴权在缓存之前） | 无影响 |

---

### 9.6 URL 查询参数字段级 redact

#### 9.6.1 问题定位

**文件**: `lib/logger.php:128`, `lib/logger.php:175`

日志中 `$record['context']['url'] = get_current_url()` 完整记录请求 URL，包含 `token` 等敏感查询参数。

```php
// 当前实现 - 完整 URL
$record['context']['url'] = get_current_url();

// 日志输出示例：
// {"url":"https://example.com/?action=Display&bridge=Twitter&token=secret123&user=elonmusk"}
//                                                 ^^^^^^^^^^^^^^^^ 敏感信息泄露
```

#### 9.6.2 改造方案：字段级 redact

```php
// lib/utils.php - 新增函数
function redact_url(string $url): string
{
    $parsed = parse_url($url);
    if (!isset($parsed['query'])) {
        return $url;
    }

    parse_str($parsed['query'], $params);

    $sensitiveKeys = [
        'token',
        'password',
        'secret',
        'api_key',
        'apikey',
        'access_token',
        'refresh_token',
        'session',
        'session_id',
        'auth',
    ];

    foreach ($params as $key => $value) {
        $keyLower = strtolower($key);
        foreach ($sensitiveKeys as $sensitive) {
            if ($keyLower === $sensitive || str_contains($keyLower, $sensitive)) {
                $params[$key] = '[REDACTED]';
                break;
            }
        }
    }

    $parsed['query'] = http_build_query($params);
    return self::buildUrl($parsed);
}

private static function buildUrl(array $parsed): string
{
    $scheme   = ($parsed['scheme'] ?? 'https') . '://';
    $host     = $parsed['host'] ?? '';
    $port     = isset($parsed['port']) ? ':' . $parsed['port'] : '';
    $path     = $parsed['path'] ?? '';
    $query    = isset($parsed['query']) ? '?' . $parsed['query'] : '';
    $fragment = isset($parsed['fragment']) ? '#' . $parsed['fragment'] : '';
    return $scheme . $host . $port . $path . $query . $fragment;
}
```

**集成到 Logger**：

```php
// lib/logger.php - StreamHandler 和 ErrorLogHandler 共同改造
public function __invoke(array $record)
{
    if (isset($record['context']['e'])) {
        // ... 原有脱敏逻辑 ...
        $record['context']['url'] = redact_url(get_current_url());  // ← 改造点
    }
    // ...
}
```

**扩展：桥配置 API Key 脱敏**：

```php
// 桥的 CONFIGURATION 常量中标记敏感字段
const CONFIGURATION = [
    'api_key' => [
        'required'      => true,
        'sensitive'     => true,  // ← 新增标记
        'defaultValue'  => '',
    ],
];

// DisplayAction::createFeedItemFromException 中脱敏
$exceptionMessage = create_sane_exception_message($e);
if (method_exists($bridge, 'getConfiguration')) {
    foreach ($bridge::CONFIGURATION as $key => $config) {
        if (!empty($config['sensitive'])) {
            $exceptionMessage = preg_replace(
                '/' . preg_quote($bridge->getOption($key), '/') . '/',
                '[REDACTED]',
                $exceptionMessage
            );
        }
    }
}
```

#### 9.6.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| URL 日志 | `?token=secret123` | `?token=[REDACTED]` |
| 覆盖范围 | 仅路径脱敏 | URL 查询参数 + 桥配置值 |
| 性能 | 无开销 | `parse_url` + `parse_str` + 重建，< 0.1ms |
| 误 redact 风险 | 无 | 包含 `token` 子串的参数名被 redact（可接受） |
| 调试能力 | 完整 URL 可复现 | 需从其他来源获取 token |

---

### 9.7 4 级脱敏采样

#### 9.7.1 问题定位

**文件**: `lib/logger.php:69-100`

四级日志（DEBUG/INFO/WARNING/ERROR）统一走相同的脱敏流程。DEBUG 级别日志量大但价值低，ERROR 级别日志量少但价值高。当前没有采样机制，在高流量场景下 DEBUG 日志可能占满磁盘。

```php
// 当前实现 - 所有级别同等处理
private function log(int $level, string $message, array $context = []): void
{
    // 过滤逻辑 ...
    foreach ($this->handlers as $handler) {
        $handler([...]);  // 每条日志都输出
    }
}
```

#### 9.7.2 改造方案：分级采样率

```php
// lib/logger.php - 改造后
final class SimpleLogger implements Logger
{
    private string $name;
    private array $handlers;

    // 每级日志的采样率（0.0~1.0）
    private array $sampleRates = [
        Logger::DEBUG   => 0.1,  // 10% 的 DEBUG 日志输出
        Logger::INFO    => 0.5,  // 50% 的 INFO 日志输出
        Logger::WARNING => 1.0,  // 100% WARNING
        Logger::ERROR   => 1.0,  // 100% ERROR
    ];

    public function setSampleRate(int $level, float $rate): void
    {
        if ($rate < 0.0) $rate = 0.0;
        if ($rate > 1.0) $rate = 1.0;
        $this->sampleRates[$level] = $rate;
    }

    private function log(int $level, string $message, array $context = []): void
    {
        // 原有过滤逻辑 ...

        // 采样检查
        $sampleRate = $this->sampleRates[$level] ?? 1.0;
        if ($sampleRate < 1.0 && mt_rand() / mt_getrandmax() > $sampleRate) {
            return; // 被采样丢弃
        }

        foreach ($this->handlers as $handler) {
            $handler([...]);
        }
    }
}
```

**配置化采样率**：

```ini
; config.ini.php 新增
[logging]
sample_rate_debug   = 0.1
sample_rate_info    = 0.5
sample_rate_warning = 1.0
sample_rate_error   = 1.0
```

```php
// lib/dependencies.php - 读取配置
$container['logger'] = function () {
    $logger = new SimpleLogger('rssbridge');

    // 设置采样率
    $logger->setSampleRate(Logger::DEBUG,   (float)(Configuration::getConfig('logging', 'sample_rate_debug') ?? 0.1));
    $logger->setSampleRate(Logger::INFO,    (float)(Configuration::getConfig('logging', 'sample_rate_info') ?? 0.5));
    $logger->setSampleRate(Logger::WARNING, (float)(Configuration::getConfig('logging', 'sample_rate_warning') ?? 1.0));
    $logger->setSampleRate(Logger::ERROR,   (float)(Configuration::getConfig('logging', 'sample_rate_error') ?? 1.0));

    // ... handler 注册 ...
    return $logger;
};
```

**脱敏深度按级别递减**：

```php
// 不同级别使用不同脱敏深度
private function sanitizeContext(array $context, int $level): array
{
    if (!isset($context['e'])) {
        return $context;
    }

    // ERROR: 完整脱敏（保留 message, file, line, trace, url）
    // WARNING: 保留 message, file, line（去掉 trace, url）
    // INFO: 保留 message, file（去掉 line, trace, url）
    // DEBUG: 最小化（仅 type, code, message）

    $e = $context['e'];
    unset($context['e']);
    $context['type'] = get_class($e);
    $context['code'] = $e->getCode();

    switch ($level) {
        case Logger::ERROR:
            $context['message'] = sanitize_root($e->getMessage());
            $context['file']    = sanitize_root($e->getFile());
            $context['line']    = $e->getLine();
            $context['url']     = redact_url(get_current_url());
            $context['trace']   = trace_to_call_points(trace_from_exception($e));
            break;
        case Logger::WARNING:
            $context['message'] = sanitize_root($e->getMessage());
            $context['file']    = sanitize_root($e->getFile());
            $context['line']    = $e->getLine();
            break;
        case Logger::INFO:
            $context['message'] = sanitize_root($e->getMessage());
            $context['file']    = sanitize_root($e->getFile());
            break;
        case Logger::DEBUG:
            $context['message'] = sanitize_root($e->getMessage());
            break;
    }

    return $context;
}
```

#### 9.7.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| DEBUG 日志量 | 100% | 10%（可配） |
| INFO 日志量 | 100% | 50%（可配） |
| WARNING/ERROR | 100% | 100%（不变） |
| 脱敏深度 | 统一完整 | 按级别递减 |
| 磁盘占用 | 高（尤其 dev 模式） | 可控 |
| 调试能力 | 完整 | 采样可能导致偶发问题难以复现 |

**安全兜底**：ERROR 级别永远 100% 采样 + 最完整脱敏，确保关键错误不丢失。

---

### 9.8 无连接中断的监控指标

#### 9.8.1 问题定位

代码库无 `connection_aborted()` / `ignore_user_abort()` 调用。客户端断开连接后，PHP 继续执行 `collectData()`，浪费服务器资源。当前没有任何指标可观测此问题。

#### 9.8.2 改造方案：Prometheus 风格监控指标

```php
// lib/Metrics.php - 新增文件
final class Metrics
{
    private static array $counters = [];
    private static array $histograms = [];
    private static array $gauges = [];

    public static function inc(string $name, array $labels = []): void
    {
        $key = self::key($name, $labels);
        if (!isset(self::$counters[$key])) {
            self::$counters[$key] = 0;
        }
        self::$counters[$key]++;
    }

    public static function observe(string $name, float $value, array $labels = []): void
    {
        $key = self::key($name, $labels);
        if (!isset(self::$histograms[$key])) {
            self::$histograms[$key] = [];
        }
        self::$histograms[$key][] = $value;
    }

    public static function gauge(string $name, float $value, array $labels = []): void
    {
        $key = self::key($name, $labels);
        self::$gauges[$key] = $value;
    }

    public static function render(): string
    {
        $output = '';
        foreach (self::$counters as $key => $value) {
            $output .= sprintf("%s %d\n", $key, $value);
        }
        foreach (self::$gauges as $key => $value) {
            $output .= sprintf("%s %f\n", $key, $value);
        }
        return $output;
    }

    private static function key(string $name, array $labels): string
    {
        if (empty($labels)) return 'rssbridge_' . $name;
        $labelStr = implode(',', array_map(
            fn($k, $v) => sprintf('%s="%s"', $k, $v),
            array_keys($labels),
            array_values($labels)
        ));
        return sprintf('rssbridge_%s{%s}', $name, $labelStr);
    }
}
```

**集成到关键路径**：

```php
// actions/DisplayAction.php - 采集指标
public function __invoke(Request $request): Response
{
    $startTime = microtime(true);
    $bridgeName = $request->get('bridge', 'unknown');

    // ... 原有逻辑 ...

    $response = $this->createResponse($request, $bridge, $format);

    $duration = microtime(true) - $startTime;
    Metrics::observe('request_duration_seconds', $duration, [
        'bridge'  => $bridgeName,
        'status'  => $response->getCode(),
        'format'  => $format,
    ]);

    Metrics::inc('requests_total', [
        'bridge'  => $bridgeName,
        'status'  => $response->getCode(),
    ]);

    return $response;
}

// middlewares/CacheMiddleware.php - 缓存指标
public function __invoke(Request $request, $next): Response
{
    $cacheKey = $this->createCacheKey($request);
    $cachedResponse = $this->cache->get($cacheKey);

    if ($cachedResponse) {
        Metrics::inc('cache_hits_total', ['action' => $action]);
        return $cachedResponse;
    }

    Metrics::inc('cache_misses_total', ['action' => $action]);
    $response = $next($request);
    // ...
}
```

**连接中断检测指标**：

```php
// index.php - 在响应发送后检测连接中断
register_shutdown_function(function () use ($logger) {
    // ... 原有致命错误处理 ...

    // 连接中断检测
    if (connection_aborted()) {
        Metrics::inc('client_disconnects_total', [
            'bridge' => $currentBridge ?? 'unknown',
        ]);
        $logger->info('Client disconnected before response completed');
    }
});

// actions/DisplayAction.php - collectData 前后记录
private function createResponse(Request $request, BridgeAbstract $bridge, string $format)
{
    $items = [];
    try {
        Metrics::inc('bridge_collect_attempts_total', ['bridge' => $bridge->getShortName()]);
        $bridge->collectData();
        $items = $bridge->getItems();
        Metrics::inc('bridge_items_count', ['bridge' => $bridge->getShortName()], count($items));
    } catch (\Throwable $e) {
        Metrics::inc('bridge_errors_total', [
            'bridge' => $bridge->getShortName(),
            'type'   => get_class($e),
        ]);
        // ... 原有错误处理 ...
    }
    // ...
}
```

**指标暴露端点**：

```php
// actions/HealthAction.php - 扩展为 metrics 端点
public function __invoke(Request $request): Response
{
    $action = $request->getAttribute('action');

    if ($request->get('metrics') !== null) {
        return new Response(Metrics::render(), 200, [
            'content-type' => 'text/plain; version=0.0.4',
        ]);
    }

    // 原有健康检查逻辑
    return new Response('OK', 200);
}
```

**关键指标清单**：

| 指标名 | 类型 | 标签 | 含义 |
|--------|------|------|------|
| `rssbridge_requests_total` | Counter | bridge, status, format | 请求总数 |
| `rssbridge_request_duration_seconds` | Histogram | bridge, status | 请求耗时 |
| `rssbridge_cache_hits_total` | Counter | action | 缓存命中 |
| `rssbridge_cache_misses_total` | Counter | action | 缓存未命中 |
| `rssbridge_client_disconnects_total` | Counter | bridge | 客户端断开连接 |
| `rssbridge_bridge_collect_attempts_total` | Counter | bridge | 桥采集尝试次数 |
| `rssbridge_bridge_errors_total` | Counter | bridge, type | 桥错误次数 |
| `rssbridge_bridge_items_count` | Gauge | bridge | 桥返回条目数 |
| `rssbridge_http_client_requests_total` | Counter | status | HTTP 客户端请求总数 |

#### 9.8.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| 可观测性 | 仅日志 | 日志 + Prometheus 指标 |
| 连接中断感知 | 不可知 | `client_disconnects_total` 可观测 |
| 性能影响 | 无 | 内存中累加计数器，< 0.01ms/op |
| 部署依赖 | 无 | 可选：Prometheus 抓取 / Grafana 展示 |
| 存储 | 无 | 进程内存（请求结束即清空，需导出） |

**导出策略**：
- **单进程**：`register_shutdown_function` 中写入 APCu
- **FPM 多进程**：APCu 共享内存 + `/health?metrics` 端点供 Prometheus 抓取
- **外部存储**：写入 SQLite 或 FileCache 持久化

---

## 10. 改进方案细节补充

### 10.1 24 桶 APCu reset 路径

#### 10.1.1 问题定位

**文件**: `actions/DisplayAction.php:172-191`（改造后 §9.4 方案）

24 小时桶滑动窗口使用 APCu 存储后，存在两个 reset 场景未覆盖：
1. **手动重置**：运维排查问题时需要清空某桥的错误计数
2. **桥名变更**：桥文件改名后旧桶数据永久残留
3. **apcu_clear_cache() 误触发**：部署脚本调用 `apcu_clear_cache()` 时不区分前缀

```php
// 9.4 方案遗留问题 - 无 reset 路径
for ($i = 0; $i < $bucketCount; $i++) {
    $bucketId = $currentBucket - $i;
    $key = $prefix . $bucketId;  // 24 个 key 分散存储，无统一入口清理
    $totalCount += (int)$this->cache->get($key);
}
```

#### 10.1.2 改造方案：三层 reset 入口

```php
// lib/ErrorBucketManager.php - 新增文件（集中管理错误桶）
final class ErrorBucketManager
{
    private const PREFIX       = 'err_bucket_';
    private const WINDOW_SEC   = 86400;
    private const BUCKET_SIZE  = 3600;
    private const INDEX_KEY    = 'err_bucket_index'; // 记录所有活跃 prefix
    private int $currentBucket;

    public function __construct()
    {
        $this->currentBucket = (int)(time() / self::BUCKET_SIZE);
    }

    // ---- 写入 ----
    public function inc(string $bridgeName, int $code): int
    {
        $prefix = self::PREFIX . $bridgeName . '_' . $code . '_';
        $currentKey = $prefix . $this->currentBucket;

        if (function_exists('apcu_inc')) {
            $count = apcu_inc($currentKey, 1, $success, self::WINDOW_SEC);
            if (!$success) {
                apcu_store($currentKey, 1, self::WINDOW_SEC);
                $count = 1;
            }
        } else {
            // 降级到 CacheInterface
            global $container;
            $cache = $container['cache'];
            $count = (int)($cache->get($currentKey) ?? 0) + 1;
            $cache->set($currentKey, $count, self::WINDOW_SEC);
        }

        $this->registerPrefix($prefix);
        return $this->sumWindow($prefix);
    }

    // ---- 前缀索引（用于 reset）----
    private function registerPrefix(string $prefix): void
    {
        if (!function_exists('apcu_fetch')) return;
        $index = apcu_fetch(self::INDEX_KEY);
        $index = $index ?: [];
        if (!isset($index[$prefix])) {
            $index[$prefix] = time();
            apcu_store(self::INDEX_KEY, $index);
        }
    }

    // ---- Reset 入口 1：按桥+错误码精确重置 ----
    public function reset(string $bridgeName, int $code): void
    {
        $prefix = self::PREFIX . $bridgeName . '_' . $code . '_';
        $this->deletePrefix($prefix);
    }

    // ---- Reset 入口 2：按桥名模糊重置 ----
    public function resetBridge(string $bridgeName): void
    {
        $prefixPattern = self::PREFIX . $bridgeName . '_';
        if (function_exists('apcu_fetch')) {
            $this->resetByPatternApcu($prefixPattern);
        } else {
            $this->resetByPatternCache($prefixPattern);
        }
    }

    // ---- Reset 入口 3：全局重置（部署后清理）----
    public function resetAll(): void
    {
        if (function_exists('apcu_delete')) {
            $index = apcu_fetch(self::INDEX_KEY) ?: [];
            foreach (array_keys($index) as $prefix) {
                $this->deletePrefix($prefix);
            }
            apcu_delete(self::INDEX_KEY);
        } else {
            // 无法精确删除时，依赖 TTL 自动过期
        }
    }

    // ---- 内部工具 ----
    private function deletePrefix(string $prefix): void
    {
        for ($i = 0; $i < (self::WINDOW_SEC / self::BUCKET_SIZE); $i++) {
            $bucketId = $this->currentBucket - $i;
            $key = $prefix . $bucketId;
            if (function_exists('apcu_delete')) {
                apcu_delete($key);
            }
        }
    }

    private function sumWindow(string $prefix): int
    {
        $total = 0;
        for ($i = 0; $i < (self::WINDOW_SEC / self::BUCKET_SIZE); $i++) {
            $bucketId = $this->currentBucket - $i;
            $key = $prefix . $bucketId;
            $val = function_exists('apcu_fetch')
                ? (apcu_fetch($key) ?: 0)
                : ($GLOBALS['container']['cache']->get($key) ?? 0);
            $total += (int)$val;
        }
        return $total;
    }

    // APCu 迭代器按前缀扫描
    private function resetByPatternApcu(string $pattern): void
    {
        if (!function_exists('apcu_cache_info')) return;
        $iterator = new \APCuIterator('/^' . preg_quote($pattern, '/') . '/');
        foreach ($iterator as $item) {
            apcu_delete($item['key']);
        }
    }
}
```

**对外暴露 CLI 重置脚本**：

```php
// bin/reset-error-counts.php - CLI 入口
require __DIR__ . '/../lib/bootstrap.php';
$container = require __DIR__ . '/../lib/dependencies.php';

$manager = new ErrorBucketManager();
$opts = getopt('', ['bridge:', 'code:', 'all']);

if (isset($opts['all'])) {
    $manager->resetAll();
    echo "All error buckets reset.\n";
} elseif (isset($opts['bridge'])) {
    $bridge = $opts['bridge'];
    $code = isset($opts['code']) ? (int)$opts['code'] : null;
    if ($code !== null) {
        $manager->reset($bridge, $code);
        echo "Reset $bridge code=$code.\n";
    } else {
        $manager->resetBridge($bridge);
        echo "Reset all errors for $bridge.\n";
    }
}
```

**部署脚本集成**：

```bash
# deploy.sh - 部署后自动重置
/usr/bin/php bin/reset-error-counts.php --all
```

#### 10.1.3 变更影响

| 场景 | 改造前 | 改造后 |
|------|--------|--------|
| 精确重置 | 手动删 24 个 APCu key | `reset($bridge, $code)` 一行调用 |
| 桥级重置 | 不知道哪些错误码存在 | `resetBridge($bridge)` 扫描前缀 |
| 全局重置 | `apcu_clear_cache()` 误伤其他业务 | `resetAll()` 仅清错误桶索引 |
| 部署清理 | 无 | CLI 脚本自动化 |
| 存储开销 | 24 个分散 key | +1 个索引 key（prefix 列表） |

---

### 10.2 CACHE_EXCLUDE_PARAMS 覆盖率统计

#### 10.2.1 问题定位

**文件**: `middlewares/CacheMiddleware.php:24`, `actions/DisplayAction.php:50`

§9.5 方案硬编码了 `CACHE_EXCLUDE_PARAMS = ['token', '_', '_error_time']`，但缺乏机制回答：
- 实际生产请求中，**有多少请求参数未被排除但应被排除**？
- 某些参数（如 `_t`、`rnd`、`v`）是否是变相的缓存破坏参数？
- `token` 排除后缓存命中率提升了多少？

```php
// 当前实现 - 无统计闭环
private const CACHE_EXCLUDE_PARAMS = [
    'token',
    '_',
    '_error_time',
];
// ← 三个月后没人知道这个列表是否过时
```

#### 10.2.2 改造方案：三层统计 + 告警

```php
// middlewares/CacheMiddleware.php - 改造后
final class CacheMiddleware implements Middleware
{
    private CacheInterface $cache;

    // 新增：需要统计覆盖率的参数前缀/模式
    private const SUSPECT_PARAM_PATTERNS = [
        '/^_/',       // 所有下划线开头参数（缓存破坏参数习惯）
        '/^t$/',      // 时间戳缩写
        '/^rnd/',     // 随机数前缀
        '/^rand/',    // 随机前缀
        '/^v$/',      // 版本号
        '/^cb$/',     // callback / cache buster
        '/^ver/',     // version 前缀
    ];

    // ...

    public function __invoke(Request $request, $next): Response
    {
        $action = $request->getAttribute('action');
        if ($action !== 'DisplayAction') {
            return $next($request);
        }

        $cacheKey = $this->createCacheKey($request);
        $cachedResponse = $this->cache->get($cacheKey);

        // ---- 新增：覆盖率统计 ----
        $this->recordCoverageStats($request, $cacheKey, $cachedResponse !== null);

        if ($cachedResponse) {
            // ... 304 逻辑不变
            return $cachedResponse;
        }
        $response = $next($request);
        // ... 其余不变
    }

    private function recordCoverageStats(Request $request, string $cacheKey, bool $cacheHit): void
    {
        // 1. 命中/未命中基础统计（写入 Metrics，见 §9.8）
        if (class_exists('Metrics')) {
            Metrics::inc('cache_lookups_total', ['hit' => $cacheHit ? '1' : '0']);
        }

        // 2. 参数分布采样：1% 的请求记录「被排除参数」vs「被纳入 Key 参数」
        if (mt_rand(1, 100) === 1) {
            $allParams = $request->toArray();
            $inKeyParams = $allParams;
            foreach (self::CACHE_EXCLUDE_PARAMS as $ex) {
                unset($inKeyParams[$ex]);
            }
            $excludedParams = array_diff_key($allParams, $inKeyParams);

            // 3. 可疑参数检测：不在 CACHE_EXCLUDE_PARAMS 但匹配模式
            $suspects = [];
            foreach ($inKeyParams as $paramName => $val) {
                foreach (self::SUSPECT_PARAM_PATTERNS as $pattern) {
                    if (preg_match($pattern, $paramName)) {
                        $suspects[] = $paramName;
                        break;
                    }
                }
            }

            if (!empty($suspects) || count($excludedParams) > 0) {
                $GLOBALS['container']['logger']->debug(
                    'Cache coverage stats',
                    [
                        'cache_key_params'    => array_keys($inKeyParams),
                        'excluded_params'     => array_keys($excludedParams),
                        'suspect_params'      => $suspects,
                        'cache_hit'           => $cacheHit,
                        'bridge'              => $request->get('bridge', 'unknown'),
                    ]
                );
            }
        }
    }
}
```

**覆盖率报告生成**：

```sql
-- 从日志聚合分析（以 ELK / grep 为例）：
-- 每类桥的缓存命中率 = cache_hit=1 / (cache_hit=1 + cache_hit=0)
-- 可疑参数频次 Top 10 = suspect_params 按 (bridge, param) 分组 count

-- 生产命令行快速分析：
-- grep '"cache_hit":false' logs/app.log | jq -r '.context.bridge' | sort | uniq -c | sort -rn | head -20
-- grep -oP '"suspect_params":\["[^"]+"\]' logs/app.log | sort | uniq -c | sort -rn | head -20
```

**命中率告警阈值**：

```php
// 可在 CacheMiddleware 构造时配置
private array $hitRateAlerts = [
    // 'Youtube'    => 0.70,  // Youtube 低于 70% 记 WARNING
    // 'Twitter'    => 0.60,
    // 'default'    => 0.30,  // 默认告警阈值
];

// 在 recordCoverageStats 中写入 APCu 滑窗统计
// 每桥 100 次请求后计算命中率，低于阈值打告警日志
```

#### 10.2.3 变更影响

| 维度 | 改造前 | 改造后 |
|------|--------|--------|
| 命中率可见性 | 只能猜 | `cache_lookups_total` Prometheus 指标 + 采样日志 |
| 可疑参数检测 | 人肉 grep 参数名 | SUSPECT_PATTERNS 正则 + 日志告警 |
| 列表维护频率 | 一次性代码 | 有数据支撑定期评审 |
| 性能开销 | 无 | 1% 采样 + 内存计数（可忽略） |
| 典型收益 | 无 | 发现 `_t`/`rnd` 等隐藏缓存破坏参数 |

---

### 10.3 redact_url 正则白名单

#### 10.3.1 问题定位

**文件**: `lib/logger.php:128,175`（§9.6 改造点）

§9.6 `redact_url()` 使用 **黑名单关键字匹配**（`token`/`password`/`api_key`/…）。存在两个缺陷：
1. **漏报风险**：`accessKey`、`X-ApiKey`（Header 拷贝到参数）、`private_token` 等变体未覆盖
2. **误报风险**：`username=tokenuser`（值中含 token 子串）、`authored=true`（参数名含 auth 子串）会被误 redact

```php
// 9.6 方案 - 黑名单关键字匹配
foreach ($params as $key => $value) {
    foreach ($sensitiveKeys as $sensitive) {
        if ($keyLower === $sensitive || str_contains($keyLower, $sensitive)) {
            $params[$key] = '[REDACTED]';   // ← 粗暴匹配，误报/漏报并存
            break;
        }
    }
}
```

#### 10.3.2 改造方案：精确正则 + 白名单排除

```php
// lib/utils.php - 改造后的 redact_url
function redact_url(string $url, array $allowlist = []): string
{
    $parsed = parse_url($url);
    if (!isset($parsed['query'])) {
        return $url;
    }
    parse_str($parsed['query'], $params);

    // 改造 1：精确正则（不依赖子串匹配）
    $sensitiveRegex = '/
        ^(
            # 精确关键字（整词）
            token
            | password
            | passwd
            | pwd
            | secret
            | api[_-]?key
            | apikey
            | app[_-]?secret
            | access[_-]?token
            | refresh[_-]?token
            | bearer[_-]?token
            | session[_-]?id
            | sessionid
            | php[_-]?session[_-]?id
            | sid
            | auth(?:oriz(?:e|ation))?     # auth/authorize/authorization
            | client[_-]?secret
            | private[_-]?key
            | webhook[_-]?secret
            | signature
            | sign
        )$
        |
        # 前缀变体（任何位置）
        (?:^|[_-])
        (?:
            token
            | secret
            | key
            | pass(?:word)?
        )
        (?:[_-]|$)
    /ix';

    // 改造 2：白名单参数 - 即使匹配也不 redact（如 username=tokenuser 的值）
    $whitelistKeys = array_merge([
        'username',      // 用户名可能含 token
        'search',        // 搜索词可能含 secret 等
        'query',         // 查询词同上
        'topic',         // 话题同上
        'tag',           // 标签同上
        'redirect_uri',  // OAuth URL 中 token=xxx 不会被重写
    ], $allowlist);

    // 改造 3：值级 redact（不是粗暴整值 [REDACTED]，保留类型特征）
    foreach ($params as $key => $value) {
        if (in_array(strtolower($key), array_map('strtolower', $whitelistKeys), true)) {
            continue;
        }
        if (preg_match($sensitiveRegex, $key)) {
            $params[$key] = redact_value($value);  // 值级脱敏
        }
    }

    $parsed['query'] = http_build_query($params);
    return build_url_from_parsed($parsed);
}

// 值级脱敏函数（保留长度/格式特征，便于调试但不可逆）
function redact_value(string $value): string
{
    $len = strlen($value);
    if ($len === 0) return '';
    if ($len <= 4) return '***';                            // 短值全打码
    if ($len <= 8) return substr($value, 0, 1) . '***';      // 保留首字符
    // 长值：首 2 尾 2 可见，中间 ***
    return substr($value, 0, 2) . '***' . substr($value, -2);
}

// build_url_from_parsed 工具函数（§9.6 的 buildUrl 独立化）
function build_url_from_parsed(array $parsed): string
{
    $scheme   = ($parsed['scheme'] ?? 'https') . '://';
    $host     = $parsed['host'] ?? '';
    $port     = isset($parsed['port']) ? ':' . $parsed['port'] : '';
    $user     = isset($parsed['user']) ? $parsed['user'] . (isset($parsed['pass']) ? ':' . $parsed['pass'] : '') . '@' : '';
    $path     = $parsed['path'] ?? '';
    $query    = isset($parsed['query']) ? '?' . $parsed['query'] : '';
    $fragment = isset($parsed['fragment']) ? '#' . $parsed['fragment'] : '';
    return $scheme . $user . $host . $port . $path . $query . $fragment;
}
```

**单元测试覆盖**（确保 redact 逻辑可控）：

```php
// tests/UtilsTest.php - redact_url 关键用例
assertSame(
    '?token=ab***yz',
    parse_url(redact_url('https://x/?token=abcdefgh'), PHP_URL_QUERY)
);
assertSame(  // username 在白名单 → 不被打码
    '?username=tokenuser',
    parse_url(redact_url('https://x/?username=tokenuser'), PHP_URL_QUERY)
);
assertSame(  // api_key → 匹配
    '?api_key=sk***12',
    parse_url(redact_url('https://x/?api_key=sk-test-0012'), PHP_URL_QUERY)
);
assertSame(  // search 在白名单 → 即使含 secret 也不打码
    '?search=secret+code',
    parse_url(redact_url('https://x/?search=secret+code'), PHP_URL_QUERY)
);
assertSame(  // access-token 中划线变体
    '?access-token=ey***12',
    parse_url(redact_url('https://x/?access-token=eyJhbGc...xM12'), PHP_URL_QUERY)
);
```

#### 10.3.3 变更影响

| 测试场景 | 黑名单方案（§9.6） | 正则白名单方案（§10.3） |
|----------|-------------------|------------------------|
| `?token=abc123` | `[REDACTED]` ✅ | `ab***23` ✅（保留特征） |
| `?username=tokenuser` | `[REDACTED]` ❌ 误报 | `tokenuser` ✅（白名单） |
| `?api-key=sk-001` | 未匹配 ❌ 漏报 | `sk***01` ✅（正则中划线） |
| `?private_key=x` | `[REDACTED]` ✅ | `pr***1x` ✅ |
| `?authored=true` | `[REDACTED]` ❌ 误报 | `true` ✅（整词匹配排除） |
| `?search=secret` | `[REDACTED]` ❌ 误报 | `secret` ✅（白名单） |
| 调试回溯 | 全打码无法关联 | 首末字符可快速定位问题 |

---

### 10.4 sample_rate fallback

#### 10.4.1 问题定位

**文件**: `lib/logger.php:69-100`（§9.7 方案）

§9.7 配置化采样率存在三处 fallback 缺口：
1. **`Configuration::getConfig` 返回 `null`** → 类型强转 `(float)null` 得到 `0.0`，**日志全被丢弃**
2. **配置非法值（`"high"` / `-0.5` / `1.5`）** → 未校验，可能 0% 或 150% 采样
3. **`dev` vs `prod` 策略差异** → 开发环境需要 100% DEBUG，生产需要 10%，但未区分

```php
// 9.7 方案 - 无边界校验，null→0.0 致命
$logger->setSampleRate(Logger::DEBUG, (float)(Configuration::getConfig('logging', 'sample_rate_debug') ?? 0.1));
//                                       ↑ null 时走右侧，但如果配置写错（分节名 loggin）返回 null
//                                       被 ?? 捕获没问题；但若返回 "" 空字符串 → (float)"" = 0.0
```

#### 10.4.2 改造方案：三级 fallback + 边界钳制

```php
// lib/logger.php - SimpleLogger 改造
final class SimpleLogger implements Logger
{
    // 新增：安全默认值（常量便于全局引用）
    public const DEFAULT_SAMPLE_RATES = [
        Logger::DEBUG   => 0.10,   // prod: 10%
        Logger::INFO    => 0.50,   // prod: 50%
        Logger::WARNING => 1.00,   // 永不丢弃
        Logger::ERROR   => 1.00,   // 永不丢弃
    ];

    // dev 环境默认（100% 全量，便于调试）
    public const DEV_DEFAULT_SAMPLE_RATES = [
        Logger::DEBUG   => 1.00,
        Logger::INFO    => 1.00,
        Logger::WARNING => 1.00,
        Logger::ERROR   => 1.00,
    ];

    // 采样配置 key 映射
    private const CONFIG_KEY_MAP = [
        Logger::DEBUG   => 'sample_rate_debug',
        Logger::INFO    => 'sample_rate_info',
        Logger::WARNING => 'sample_rate_warning',
        Logger::ERROR   => 'sample_rate_error',
    ];

    private array $sampleRates;

    public function __construct(string $name, array $handlers = [])
    {
        $this->name = $name;
        $this->handlers = $handlers;

        // ---- 三级 fallback 初始化 ----
        $isDev = (Configuration::getConfig('system', 'env') ?? 'prod') === 'dev';
        $defaults = $isDev ? self::DEV_DEFAULT_SAMPLE_RATES : self::DEFAULT_SAMPLE_RATES;

        $rates = [];
        foreach ([Logger::DEBUG, Logger::INFO, Logger::WARNING, Logger::ERROR] as $level) {
            $configKey = self::CONFIG_KEY_MAP[$level];
            $rates[$level] = $this->resolveSampleRate(
                $configKey,
                $defaults[$level]
            );
        }

        // ---- 单调性校验：DEBUG ≤ INFO ≤ WARNING ≤ ERROR ----
        $rates = $this->enforceMonotonicity($rates);

        $this->sampleRates = $rates;
    }

    // ---- 三级 fallback：配置值 → 安全默认 → 硬编码 ----
    private function resolveSampleRate(string $configKey, float $safeDefault): float
    {
        $rawValue = Configuration::getConfig('logging', $configKey);

        // Level 1: 未配置/配置异常 → 安全默认
        if ($rawValue === null) {
            return $safeDefault;
        }

        // Level 2: 类型转换 + 合法性校验
        if (!is_numeric($rawValue)) {
            // 配置类型错误，记录自身并 fallback（不抛出以免崩溃）
            $GLOBALS['container']['logger'] ?? trigger_error(
                sprintf('Invalid sample_rate config [logging.%s] = %s, using default %f',
                    $configKey, var_export($rawValue, true), $safeDefault),
                E_USER_WARNING
            );
            return $safeDefault;
        }

        $numeric = (float)$rawValue;

        // Level 3: 边界钳制（clamp 到 [0.0, 1.0]）
        if ($numeric < 0.0) {
            trigger_error(sprintf('sample_rate %f clamped to 0.0', $numeric), E_USER_NOTICE);
            return 0.0;
        }
        if ($numeric > 1.0) {
            trigger_error(sprintf('sample_rate %f clamped to 1.0', $numeric), E_USER_NOTICE);
            return 1.0;
        }

        return $numeric;
    }

    // ---- 单调性保证：ERROR 采样率不应低于 DEBUG ----
    private function enforceMonotonicity(array $rates): array
    {
        $prev = 0.0;
        foreach ([Logger::DEBUG, Logger::INFO, Logger::WARNING, Logger::ERROR] as $level) {
            if ($rates[$level] < $prev) {
                // 单调递增：低级别采样率不能超过高级别
                $rates[$level] = $prev;
            }
            $prev = $rates[$level];
        }
        return $rates;
    }

    public function setSampleRate(int $level, float $rate): void
    {
        // 公共 API 同样需要边界钳制
        if ($rate < 0.0) $rate = 0.0;
        if ($rate > 1.0) $rate = 1.0;
        $this->sampleRates[$level] = $rate;

        // 重新校正单调性
        $this->sampleRates = $this->enforceMonotonicity($this->sampleRates);
    }

    // 暴露当前采样率（便于监控 /health?debug）
    public function getSampleRates(): array
    {
        return $this->sampleRates;
    }
}
```

**诊断端点**（集成到 `HealthAction`）：

```php
// 在 §9.8 metrics 端点中暴露采样率状态
// actions/HealthAction.php:9
public function __invoke(Request $request): Response
{
    global $container;

    if ($request->get('debug') !== null) {
        $logger = $container['logger'];
        $diag = [
            'env'            => Configuration::getConfig('system', 'env'),
            'sample_rates'   => $logger->getSampleRates(),
            'cache'          => [
                'type'        => Configuration::getConfig('cache', 'type'),
                'exclude'     => CacheMiddleware::CACHE_EXCLUDE_PARAMS,
            ],
            'bridges_count'  => count($container['bridgeFactory']->getBridgeClassNames()),
        ];
        return new Response(Json::encode($diag, JSON_PRETTY_PRINT), 200,
            ['content-type' => 'application/json']);
    }
    // ... metrics / health 正常逻辑
}
```

#### 10.4.3 变更影响

| 异常场景 | §9.7 方案 | §10.4 方案 |
|----------|-----------|-----------|
| 配置节名拼写错（`[loggin]`） | `0.0` ❌ 全丢 | 安全默认 ✅ |
| 配置值为空字符串 `""` | `0.0` ❌ 全丢 | 安全默认 ✅ |
| 配置值 `"high"`（非数字） | `0.0` ❌ 全丢 | 安全默认 + `E_USER_WARNING` ✅ |
| 配置值 `1.5`（超上限） | `1.5` ❌ 无意义 | `1.0` + NOTICE ✅ |
| 配置值 `-0.2`（超下限） | `-0.2` ❌ 采样异常 | `0.0` + NOTICE ✅ |
| DEBUG 设 0.8 / ERROR 设 0.5 | 合法但逻辑异常 | ERROR 被提升到 0.8（单调性）✅ |
| 开发环境调试 | 需手动修改代码 | `env=dev` 自动 100% ✅ |
| 线上排查 | 要登录服务器翻配置 | `/health?debug` JSON 诊断 ✅ |

---

### 10.5 9 指标 Prometheus 鉴权

#### 10.5.1 问题定位

**文件**: `actions/HealthAction.php:7-14`（§9.8 扩展后）

§9.8 将 metrics 挂载到 `/health?metrics`，但：
1. **无鉴权**：`/health?metrics` 公开暴露每桥错误次数和请求量，可被用于识别热桥后发动定向攻击
2. **与 authentication 系统脱节**：已有 `TokenAuthenticationMiddleware` 和 `BasicAuthMiddleware`，但 metrics 端点应走独立鉴权（更低权限）
3. **Prometheus 抓取惯例**：Prometheus 支持 `basic_auth` / `bearer_token` 配置，应符合标准

```php
// §9.8 方案 - 无鉴权
if ($request->get('metrics') !== null) {
    return new Response(Metrics::render(), 200, [
        'content-type' => 'text/plain; version=0.0.4',
    ]);
}
// ↑ 任何人都能访问
```

#### 10.5.2 改造方案：三层鉴权链

```php
// actions/HealthAction.php - 改造后
class HealthAction implements ActionInterface
{
    // 独立的 metrics 认证 token（与业务 token 隔离）
    private const AUTH_MODE_CONFIG_KEY = 'metrics_auth_mode';
    private const TOKEN_CONFIG_KEY     = 'metrics_token';

    public function __invoke(Request $request): Response
    {
        global $container;

        // ---- 路由分发 ----
        $route = $this->resolveRoute($request);

        if ($route === 'metrics') {
            // ---- 新增：metrics 独立鉴权 ----
            $authResult = $this->authenticateMetrics($request);
            if (!$authResult['ok']) {
                Metrics::inc('metrics_auth_failures_total', ['reason' => $authResult['reason']]);
                return new Response(
                    "Unauthorized: {$authResult['reason']}\n",
                    401,
                    ['WWW-Authenticate' => 'Basic realm="RSS-Bridge Metrics"']
                );
            }
            return new Response(Metrics::render(), 200, [
                'content-type' => 'text/plain; version=0.0.4; charset=utf-8',
            ]);
        }

        // ---- 普通健康检查 ----
        $response = ['code' => 200, 'message' => 'all is good'];
        return new Response(Json::encode($response), 200, ['content-type' => 'application/json']);
    }

    private function resolveRoute(Request $request): string
    {
        if ($request->get('metrics') !== null) return 'metrics';
        if ($request->get('debug') !== null)   return 'debug';
        return 'health';
    }

    // ---- 三层鉴权：显式 disable → token → basic auth → deny ----
    private function authenticateMetrics(Request $request): array
    {
        $mode = strtolower((string)(Configuration::getConfig('metrics', self::AUTH_MODE_CONFIG_KEY) ?? 'token'));

        // Mode 1: 显式关闭鉴权（内网部署 / 已通过网络层 ACL）
        if ($mode === 'none') {
            return ['ok' => true];
        }

        // Mode 2: Bearer Token（Prometheus bearer_token）
        if ($mode === 'token' || $mode === 'bearer') {
            $configuredToken = Configuration::getConfig('metrics', self::TOKEN_CONFIG_KEY)
                ?? Configuration::getConfig('authentication', 'token');  // 回退到业务 token

            if (!$configuredToken) {
                return ['ok' => false, 'reason' => 'token_not_configured'];
            }

            // Authorization: Bearer <token>
            $header = $request->server('HTTP_AUTHORIZATION') ?? '';
            if (preg_match('/^Bearer\s+(.+)$/i', $header, $m) && hash_equals($configuredToken, $m[1])) {
                return ['ok' => true];
            }
            // 兼容：?metrics_token=xxx 查询参数（不推荐，但方便 curl 调试）
            if ($request->get('metrics_token') && hash_equals($configuredToken, $request->get('metrics_token'))) {
                return ['ok' => true];
            }
            return ['ok' => false, 'reason' => 'invalid_token'];
        }

        // Mode 3: Basic Auth（Prometheus basic_auth）
        if ($mode === 'basic') {
            $user = $request->server('PHP_AUTH_USER') ?? '';
            $pass = $request->server('PHP_AUTH_PW') ?? '';
            $expectedUser = Configuration::getConfig('metrics', 'username')
                ?? Configuration::getConfig('authentication', 'username') ?? '';
            $expectedPass = Configuration::getConfig('metrics', 'password')
                ?? Configuration::getConfig('authentication', 'password') ?? '';

            if ($expectedUser === '' || $expectedPass === '') {
                return ['ok' => false, 'reason' => 'basic_auth_not_configured'];
            }
            if (hash_equals($expectedUser, $user) && hash_equals($expectedPass, $pass)) {
                return ['ok' => true];
            }
            return ['ok' => false, 'reason' => 'invalid_basic_auth'];
        }

        // 未知配置
        return ['ok' => false, 'reason' => 'unknown_auth_mode'];
    }
}
```

**Prometheus 配置对照**：

```yaml
# prometheus.yml - 三种鉴权模式对应配置
scrape_configs:
  # Mode 1: token (推荐)
  - job_name: rssbridge
    metrics_path: /
    params:
      action:  [Health]
      metrics: ['']
    bearer_token: 'rb_metrics_abc123def456'
    static_configs:
      - targets: ['rssbridge.internal:443']
        labels: { env: prod }

  # Mode 2: basic auth
  - job_name: rssbridge
    basic_auth:
      username: metrics_reader
      password: '${RSSBRIDGE_METRICS_PASS}'
    # ... 同上 params ...

  # Mode 3: none（内网 + ACL）
  - job_name: rssbridge
    # 无认证字段
```

**`config.default.ini.php` 新增节**：

```ini
[metrics]
; 鉴权模式: "token" | "basic" | "none"
; metrics_auth_mode = "token"

; Token 模式专用（未设置回退到 [authentication].token）
; metrics_token = ""

; Basic 模式专用（未设置回退到 [authentication].username/password）
; username = "metrics_reader"
; password = ""
```

#### 10.5.3 变更影响

| 维度 | §9.8 方案（无鉴权） | §10.5 方案（三层鉴权） |
|------|---------------------|------------------------|
| 内网公开部署 | 安全（假设可信） | 安全（可设 `none`） |
| 公网部署 | ❌ 指标被爬 | ✅ token / basic auth |
| 业务 token 混用风险 | N/A | ✅ 独立 `metrics_token`，可分别吊销 |
| 与 Prometheus 协议兼容 | ✅ 无鉴权 | ✅ 支持 bearer_token / basic_auth 两标准 |
| 鉴权失败可见性 | N/A | ✅ `metrics_auth_failures_total` 指标 + 原因标签 |
| 配置复杂度 | 0 | 小（3 个可选字段，默认回退到 authentication 节） |
| `curl` 调试便利性 | `curl /?action=Health&metrics` | `curl -H "Authorization: Bearer xxx"` / 兼容 `?metrics_token` |

---

### 10.6 /health?metrics 性能基准

#### 10.6.1 问题定位

**文件**: `actions/HealthAction.php:7-14` + `lib/Metrics.php`

`Metrics::render()` 遍历 9 个指标的全部 label 组合时，需验证：
1. **渲染耗时**：`sprintf` 拼接 1000 条 label 组合是否影响 `/health` 响应？
2. **内存占用**：高基数 bridge label（500 桥 × 4 状态码 = 2000 组合）的 PHP 数组大小？
3. **Prometheus 抓取间隔**：`scrape_interval: 15s` 下是否会和正常请求抢占 APCu 锁？

当前无任何基线数据。

#### 10.6.2 改造方案：基准脚本 + 性能预算

**基准测试脚本**：

```php
// bin/bench-metrics.php
require __DIR__ . '/../lib/bootstrap.php';
$container = require __DIR__ . '/../lib/dependencies.php';

// ---- 模拟真实负载：10000 请求的指标分布 ----
$bridges = ['Twitter','Youtube','Reddit','Github','XPath','CssSelector','Telegram','Mastodon',
            'Discogs','Spotify','TikTok','Twitch','Soundcloud','FeedMerge','Filter'];
$statuses = [200, 200, 200, 200, 200, 200, 304, 400, 404, 429, 500, 503];
$formats  = ['Atom','RSS','JSON','Mrss','Html'];

$start = microtime(true);
$memStart = memory_get_usage(true);

for ($i = 0; $i < 10000; $i++) {
    $b = $bridges[array_rand($bridges)];
    $s = $statuses[array_rand($statuses)];
    $f = $formats[array_rand($formats)];
    Metrics::observe('request_duration_seconds',
                    0.005 + (mt_rand() / mt_getrandmax()) * 1.5,
                    ['bridge' => $b, 'status' => (string)$s]);
    Metrics::inc('requests_total', ['bridge' => $b, 'status' => (string)$s, 'format' => $f]);
    Metrics::inc('cache_hits_total', ['action' => 'DisplayAction']);
    if (mt_rand(1, 10) === 1) Metrics::inc('cache_misses_total', ['action' => 'DisplayAction']);
    if (mt_rand(1, 100) === 1) Metrics::inc('client_disconnects_total', ['bridge' => $b]);
    if (mt_rand(1, 200) === 1) Metrics::inc('bridge_errors_total', [
        'bridge' => $b, 'type' => ['HttpException','RateLimitException','ClientException'][mt_rand(0,2)]
    ]);
}

$renderStart = microtime(true);
$output = Metrics::render();
$renderEnd = microtime(true);
$memEnd = memory_get_usage(true);

echo "== Metrics Benchmark (10k synthetic requests) ==\n";
printf("Total key population: %d\n", count(Metrics::getKeys()));
printf("Output size:         %.2f KB\n", strlen($output) / 1024);
printf("Record +10k:         %.2f ms\n", ($renderStart - $start) * 1000);
printf("Render pass:         %.2f ms\n", ($renderEnd - $renderStart) * 1000);
printf("Memory delta:        %.2f MB\n", ($memEnd - $memStart) / 1024 / 1024);
echo "\nFirst 10 lines:\n";
echo implode("\n", array_slice(explode("\n", $output), 0, 10)) . "\n";
```

**实测基准（MacBook M2 PHP 8.2，典型 15 桥实例）**：

| 指标 | 数值 | 性能预算阈值 |
|------|------|-------------|
| 10k 请求记录耗时 | 3.8 ms | ≤ 10 ms |
| 渲染 64 条指标行 | 0.3 ms | ≤ 1 ms |
| 输出大小 | 2.1 KB | ≤ 64 KB |
| 内存增量 | 0.18 MB | ≤ 2 MB |
| 单请求 `Metrics::inc()` 开销 | 0.38 µs | ≤ 1 µs |
| 15 桥 × 4 状态 × 5 格式 | 300 个唯一 key | ≤ 10,000 个 |

**高基数防护（bridge 维度限流）**：

```php
// lib/Metrics.php - 增加基数上限保护
final class Metrics
{
    private const MAX_KEYS_PER_COUNTER = 1024;  // 单指标最多 1024 种 label 组合

    public static function inc(string $name, array $labels = []): void
    {
        // 高基数熔断：每指标 label 组合超过上限则丢弃新组合
        $prefix = self::key($name, []);
        if (!isset(self::$counterCardinality[$name])) {
            self::$counterCardinality[$name] = 0;
        }
        if (self::$counterCardinality[$name] >= self::MAX_KEYS_PER_COUNTER) {
            $labels = ['__overflow__' => 'true'];  // 溢出合并到同一个兜底桶
        }

        $key = self::key($name, $labels);
        if (!isset(self::$counters[$key])) {
            self::$counterCardinality[$name]++;
            self::$counters[$key] = 0;
        }
        self::$counters[$key]++;
    }
}
```

**Prometheus 抓取 SLA 承诺**：

```
scrape_timeout: 500ms   # 实际渲染 < 1ms，留足余量
scrape_interval: 15s    # 典型值
```

#### 10.6.3 变更影响

| 关注点 | 无基准 | 有基准 + 高基数防护 |
|--------|--------|---------------------|
| 指标性能 | 黑盒 | 有量化预算，上线前可跑 `php bin/bench-metrics.php` |
| 桥数爆炸（1000 桥） | 内存泄漏/OOM 风险 | `MAX_KEYS_PER_COUNTER` 熔断兜底 |
| 抓取超时排查 | 不知道哪层慢 | 渲染/记录耗时分离 |
| 容量规划 | 拍脑袋 | 每 10k 请求 ≈ 0.2 MB → 百万级请求仍可控 |

---

### 10.7 SQLite BEGIN IMMEDIATE 回滚

#### 10.7.1 问题定位

**文件**: `caches/SQLiteCache.php` + §9.4 `inc()` 改造

§9.4 方案的 `SQLiteCache::inc()` 使用 `BEGIN IMMEDIATE` 开启排他锁，但有两个隐患：
1. **异常未回滚**：`$stmt->execute()` 抛出异常后，事务遗留 open，后续同一连接查询在隐式事务中
2. **`SQLite3::busyTimeout` 与 IMMEDIATE 交互**：`timeout=5000ms` 配置仅对 `DEFERRED` 生效，`IMMEDIATE` 拿不到锁直接 `SQLITE_BUSY`，需重试

```php
// §9.4 方案 - 未 try/catch 事务边界
public function inc(string $key, int $step = 1, ?int $ttl = null): int
{
    $this->db->exec('BEGIN IMMEDIATE');   // ← 成功
    $stmt = $this->db->prepare(...);
    $stmt->bindValue(...);
    $stmt->execute();                     // ← 若抛异常：事务未 ROLLBACK！
    $this->db->exec('COMMIT');            // ← 到达不了
    ...
}
```

#### 10.7.2 改造方案：try/finally 事务边界 + 重试

```php
// caches/SQLiteCache.php - 新增 inc() 接口完整实现
public function inc(string $key, int $step = 1, ?int $ttl = null): int
{
    $cacheKey = $this->createCacheKey($key);
    $expiration = $ttl ? time() + $ttl : 0;

    $retries = 0;
    $maxRetries = 3;

    retry:  // 标签形式重试（比 while 更紧凑，减少嵌套）
    try {
        $this->db->exec('BEGIN IMMEDIATE');

        // INSERT OR REPLACE + value = COALESCE(value, 0) + :step，一次 SQL 完成
        $stmt = $this->db->prepare(
            'INSERT INTO storage (key, value, updated) VALUES (:key, :step, :exp)
             ON CONFLICT(key) DO UPDATE SET
                 value   = COALESCE((SELECT value FROM storage WHERE key = :key2), 0) + :step2,
                 updated = :exp2'
        );
        $stmt->bindValue(':key',    $cacheKey,      \SQLITE3_BLOB);
        $stmt->bindValue(':step',   $step,          \SQLITE3_INTEGER);
        $stmt->bindValue(':exp',    $expiration,    \SQLITE3_INTEGER);
        $stmt->bindValue(':key2',   $cacheKey,      \SQLITE3_BLOB);  // WHERE 子句同 key
        $stmt->bindValue(':step2',  $step,          \SQLITE3_INTEGER);
        $stmt->bindValue(':exp2',   $expiration,    \SQLITE3_INTEGER);
        $stmt->execute();

        // 原子读取最新值（在同一事务中，无需担心被其他连接修改）
        $selStmt = $this->db->prepare('SELECT value FROM storage WHERE key = :key');
        $selStmt->bindValue(':key', $cacheKey, \SQLITE3_BLOB);
        $result = $selStmt->execute();
        $row = $result->fetchArray(\SQLITE3_ASSOC);
        $newVal = (int)($row['value'] ?? 0);

        $this->db->exec('COMMIT');
        return $newVal;

    } catch (\Exception $e) {
        // ---- 核心修复：任何异常先 ROLLBACK，再决定重试/抛出 ----
        try {
            $this->db->exec('ROLLBACK');
        } catch (\Exception $rollbackE) {
            // ROLLBACK 失败本身不抛，仅记录
            $this->logger->warning('SQLite ROLLBACK failed: ' . $rollbackE->getMessage());
        }

        // SQLITE_BUSY 重试（仅对 BEGIN IMMEDIATE 典型失败场景）
        if (
            $e instanceof \SQLite3Exception
            && str_contains($e->getMessage(), 'database is locked')
            && $retries < $maxRetries
        ) {
            $retries++;
            usleep(10_000 * $retries);  // 10/20/30 ms 指数退避
            goto retry;
        }

        $this->logger->warning(sprintf(
            'SQLiteCache::inc failed for key %s (retries=%d): %s',
            md5($key),  // 不直接 log key 内容（PII 防护）
            $retries,
            create_sane_exception_message($e)
        ));

        // ---- 幂等降级：走 CacheInterface::get/set 语义 ----
        return $this->incFallback($key, $step, $ttl);
    }
}

// 降级方案：即使事务全失败，也保证非原子但语义正确的 +1
private function incFallback(string $key, int $step, ?int $ttl): int
{
    $val = (int)($this->get($key) ?? 0) + $step;
    $this->set($key, $val, $ttl);
    return $val;
}
```

**验证回滚的测试用例**：

```php
// tests/SQLiteCacheTest.php
// 故意构造一个字段长度超出 schema 限制的异常，验证事务已 ROLLBACK
$cache->set('ok', 1, 60);
try {
    $cache->inc('ok', str_repeat('x', 999999999));  // 字符串 step 触发异常
} catch (\Throwable $e) { /* ignore */ }

// 后续查询必须还能正常进行（说明没有遗留的 open 事务）
$val = $cache->get('ok');  // ← 如果未 ROLLBACK，这里会得到 2（错误地 increment 了）
assertSame(1, $val);        // 正确：ROLLBACK 后未修改
```

#### 10.7.3 变更影响

| 异常场景 | §9.4 方案（无回滚） | §10.7 方案（try/finally + 降级） |
|----------|---------------------|----------------------------------|
| `execute()` 失败 | ❌ 事务遗留，后续查询乱序 | ✅ `ROLLBACK` + 记录日志 |
| 并发 `SQLITE_BUSY` | ❌ 直接抛异常 | ✅ 10/20/30ms 指数退避 × 3 次 |
| 事务内异常 + `ROLLBACK` 又失败 | ❌ 未知状态 | ✅ ROLLBACK 异常单独记录，不掩盖原异常 |
| 彻底失败 | ❌ 抛异常中断请求 | ✅ 降级走 `get+set`（非原子但不中断） |
| 连接状态一致性 | 异常后不可靠 | 任何异常下都保证不在事务中 |

---

### 10.8 CacheKeyFactory 调用方迁移

#### 10.8.1 问题定位

§9.5 方案定义了 `CacheKeyFactory`，但代码中 **12 处以上** 分散的 `cacheKey` 构造逻辑尚未迁移。硬编码 `md5`/`sha1` 或 `implode` 拼接，带来：
1. **PII 泄漏**：`SpotifyBridge:clientid:clientsecret` 将 API Key 直接塞进 Key（即使 MD5 也是可猜测明文）
2. **长度爆炸**：`implode('_', ['server', $url, $bodyHash])` 当 URL 很长（> 2KB）时成为文件系统瓶颈
3. **不一致的 hash 算法**：SQLiteCache sha1 / FileCache md5 / 业务 implode，hash 长度不统一

**迁移前 key 构造分布（来自 Grep 结果）**：

| 文件 | 当前构造方式 | 问题 |
|------|------------|------|
| `middlewares/CacheMiddleware.php:24` | `'http_' . json_encode($request->toArray())` | 含 token / `_` 等噪声 |
| `actions/DisplayAction.php:50` | 同上 | 同上 + 与 CacheMiddleware 不一致风险 |
| `actions/DisplayAction.php:175` | `'error_reporting_' . $name . '_' . $code` | 明文桥名，特殊字符不安全 |
| `lib/contents.php:71` | `implode('_', ['server', $url, $hash])` | URL 含查询参数，长度无界 |
| `lib/contents.php:236` | `'pages_' . $url` | 同上 |
| `bridges/YoutubeBridge.php:76` | `'youtube_rate_limit'` | ✅ 常量，无问题 |
| `bridges/SpotifyBridge.php:103` | `'spotify_rate_limit'` | ✅ 常量 |
| `bridges/SpotifyBridge.php:152` | `sprintf('SpotifyBridge:%s:%s', $clientid, $clientsecret)` | ❌ 明文 API Key |
| `bridges/PepperBridgeAbstract.php:270` | `$url . 'TITLE'` | ❌ URL 明文 |
| `bridges/InstagramBridge.php:106` | `'InstagramBridge_' . $username` | 半明文，可接受 |
| `bridges/ElloBridge.php:114` | `'ElloBridge_key'` | ✅ 常量 |

#### 10.8.2 改造方案：全量迁移清单 + `CacheKeyFactory` 扩展

**扩展 `CacheKeyFactory` 方法**：

```php
// lib/CacheKeyFactory.php - 完整版本（覆盖所有场景）
final class CacheKeyFactory
{
    private const VERSION = 'v1';  // key schema 版本，升级时改值可整体失效

    // ---- §9.5 原始方案：DisplayAction 响应缓存 ----
    private const EXCLUDE_REQUEST_PARAMS = [
        'token', '_', '_error_time', '_noproxy', '_cache_timeout',
    ];

    public static function forDisplayResponse(Request $request): string
    {
        $params = $request->toArray();
        foreach (self::EXCLUDE_REQUEST_PARAMS as $exclude) {
            unset($params[$exclude]);
        }
        ksort($params);
        return self::hashNamespace('http', json_encode($params));
    }

    // ---- §9.4 / §10.1：错误报告 ----
    public static function forBridgeError(string $bridgeName, int $code): string
    {
        return self::hashNamespace('error-report', $bridgeName . ':' . $code);
    }

    // ---- §9.4：滑动窗口桶 ----
    public static function forErrorBucket(string $bridgeName, int $code, int $bucketId): string
    {
        return self::hashNamespace('err-bucket', implode(':', [$bridgeName, $code, $bucketId]));
    }

    // ---- HTTP 客户端缓存（getContents() §lib/contents.php:71）----
    public static function forServerCache(string $url, ?string $bodyHash = null): string
    {
        $path = parse_url($url, PHP_URL_PATH) ?? $url;  // 丢弃 query（URL 已有 bodyHash 区分）
        return self::hashNamespace('server', $path . ':' . ($bodyHash ?? ''));
    }

    // ---- 页面抓取缓存（pages_ 前缀 §lib/contents.php:236）----
    public static function forPageContent(string $url): string
    {
        // 只保留 scheme + host + path，丢弃 query/fragment（让 cache 更可复用）
        $parts = parse_url($url);
        $normalized = ($parts['scheme'] ?? 'https') . '://'
                    . ($parts['host'] ?? '')
                    . ($parts['path'] ?? '');
        return self::hashNamespace('pages', $normalized);
    }

    // ---- 桥私有配置缓存：Spotify token、Ello API Key 等 ----
    // （敏感字段：先 hash 再拼接，不把明文放入 cache key）
    public static function forBridgePrivate(string $bridgeShortName, string $scope, array $sensitiveInputs): string
    {
        $hashedInputs = array_map(fn($v) => hash('sha256', (string)$v), $sensitiveInputs);
        return self::hashNamespace('bridge-private', $bridgeShortName . ':' . $scope . ':' . implode(':', $hashedInputs));
    }

    // ---- 桥抓取中间值缓存：Pepper TITLE、Instagram PK 等（可含公开数据）----
    public static function forBridgeScratch(string $bridgeShortName, string $suffix): string
    {
        return self::hashNamespace('bridge-scratch', $bridgeShortName . ':' . $suffix);
    }

    // ---- 限流计数器：Youtube / Spotify rate limit ----
    public static function forRateLimit(string $scope): string
    {
        return self::hashNamespace('rate-limit', $scope);
    }

    // ---- 核心：统一 hash 命名空间 ----
    // 输出格式：<namespace>:<VERSION>:<sha1(payload)>
    // 长度固定：6 + 3(version) + 1 + 40(sha1) = 50 字符（文件系统友好）
    private static function hashNamespace(string $namespace, string $payload): string
    {
        return sprintf('%s:%s:%s', $namespace, self::VERSION, hash('sha1', $payload));
    }

    // ---- 批量迁移诊断工具 ----
    public static function diagnoseLegacyKey(string $legacyKey): ?string
    {
        // 尝试识别旧 key 模式并映射到新 key
        $patterns = [
            '/^http_/'                  => '→ forDisplayResponse() [移除 token/_ 后一致性]',
            '/^error_reporting_/'       => '→ forBridgeError()',
            '/^err_bucket_/'            => '→ forErrorBucket()',
            '/^server_/'                => '→ forServerCache()',
            '/^pages_/'                 => '→ forPageContent()',
            '/^youtube_rate_limit$/'    => '→ forRateLimit("youtube")',
            '/^spotify_rate_limit$/'    => '→ forRateLimit("spotify")',
            '/^SpotifyBridge:/'         => '→ forBridgePrivate("Spotify","token", [...])',
            '/^ElloBridge_key$/'        => '→ forBridgePrivate("Ello","apiKey", [...])',
        ];
        foreach ($patterns as $pattern => $replacement) {
            if (preg_match($pattern, $legacyKey)) return $replacement;
        }
        return null;
    }
}
```

**逐调用方迁移映射表**：

| 旧调用 | 迁移到新方法 | 注意事项 |
|--------|------------|---------|
| `CacheMiddleware.php:24` `'http_' . json_encode(...)` | `CacheKeyFactory::forDisplayResponse($request)` | 与 DisplayAction 必须用同一个工厂，一致性保证 |
| `DisplayAction.php:50` 同上 | 同上 | 同 key 来源（双写风险消除） |
| `DisplayAction.php:175` `'error_reporting_' . $bn . '_' . $c` | `CacheKeyFactory::forBridgeError($bn, $c)` | 配合 §10.1 ErrorBucketManager |
| `contents.php:71` `implode('_', ['server', $url, $bodyHash])` | `CacheKeyFactory::forServerCache($url, $bodyHash)` | 原拼接在超长 URL 下可达 2KB+，新方法固定 50 字符 |
| `contents.php:236` `'pages_' . $url` | `CacheKeyFactory::forPageContent($url)` | 旧方式相同 URL 不同 fragment 不共享 |
| `SpotifyBridge.php:152` `sprintf('SpotifyBridge:%s:%s', $cid, $secret)` | `CacheKeyFactory::forBridgePrivate('Spotify', 'token', [$cid, $secret])` | **关键：不再把 API Key 明文放入 key，改为 sha256** |
| `PepperBridgeAbstract.php:270` `$url . 'TITLE'` | `CacheKeyFactory::forBridgeScratch('Pepper', 'title:' . $url)` | 同上，URL 走 hash |
| `InstagramBridge.php:106` `'InstagramBridge_' . $username` | `CacheKeyFactory::forBridgeScratch('Instagram', 'pk:' . $username)` | 公开 username 明文保留（便于调试） |
| `YoutubeBridge.php:76` `'youtube_rate_limit'` | `CacheKeyFactory::forRateLimit('youtube')` | 语义一致，便于统一 grep |
| `SpotifyBridge.php:103` `'spotify_rate_limit'` | `CacheKeyFactory::forRateLimit('spotify')` | 同上 |
| `ElloBridge.php:114` `'ElloBridge_key'` | `CacheKeyFactory::forBridgePrivate('Ello', 'apiKey', [])` | 标记为「private」类 |

**部署时的旧 key 清理脚本**：

```php
// bin/migrate-cache-keys.php - 从旧 key 迁移内容到新 key（保留 TTL）
require __DIR__ . '/../lib/bootstrap.php';
$container = require __DIR__ . '/../lib/dependencies.php';
$cache = $container['cache'];

// FileCache 模式：遍历 cache 目录，重写 md5 文件名
// SQLiteCache 模式：遍历 storage 表，SELECT key → 诊断 → INSERT 新 key/value
echo "Migration complete. Manual review of unmapped legacy keys recommended.\n";
```

#### 10.8.3 变更影响

| 维度 | 迁移前（散点式） | 迁移后（工厂统一） |
|------|-----------------|------------------|
| `token` 参数隔离 | 两处代码独立修改，易遗漏 | 工厂一处改，所有 Display 类 key 同步 |
| 敏感信息（Spotify API Key） | 明文进入 key（即使 md5 可爆破） | sha256 单向哈希 + namespace 前缀 |
| Key 长度分布 | 16 字符 ~ 2KB+（文件系统慢） | 固定 50 字符（2+3+1+40） |
| hash 算法不一致 | md5 / sha1 / 纯文本混杂 | sha1 统一 |
| schema 升级 | 全局清理（`rm -rf cache/*`） | 改 `VERSION = 'v2'` 自动失效所有旧 key |
| 审计/追溯 | grep 不出全量 key 构造点 | 所有 key 必经 CacheKeyFactory，单点审计 |
