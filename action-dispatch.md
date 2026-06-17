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
    
    // 格式化输出
    $formatFactory = new FormatFactory();
    $format = $formatFactory->create($format);      // 创建格式器
    
    $format->setItems($items);                      // 注入数据
    $format->setFeed($bridge->getFeed());           // 注入 Feed 元数据
    $format->setLastModified(time());               // 设置更新时间
    
    $headers = [
        'last-modified' => gmdate('D, d M Y H:i:s ', $now) . 'GMT',
        'content-type'  => $format->getMimeType() . '; charset=UTF-8',
    ];
    $body = $format->render();                      // 渲染输出
    
    return new Response($body, 200, $headers);
}
```

### 5.3 BridgeAbstract 桥基类
**文件**: `lib/BridgeAbstract.php:1-200+`

**核心抽象方法**:
```php
abstract public function collectData();  // 各桥必须实现的抓取逻辑
```

**桥实现示例** (`bridges/DemoBridge.php`):
```php
class DemoBridge extends BridgeAbstract
{
    const NAME = 'DemoBridge';
    const URI = 'https://github.com/rss-bridge/rss-bridge';
    const CACHE_TIMEOUT = 15;  // 缓存 15 秒
    
    public function collectData()
    {
        $item = [];
        $item['author'] = 'Me!';
        $item['title'] = 'Test';
        $item['content'] = 'Awesome content !';
        $item['uri'] = 'http://example.com/test';
        
        $this->items[] = $item;  // 写入 $this->items 数组
    }
}
```

### 5.4 FeedItem 数据封装
**文件**: `lib/FeedItem.php:1-150+`

```php
// FormatAbstract.php:32-37
public function setItems(array $items): void
{
    foreach ($items as $item) {
        $this->items[] = FeedItem::fromArray($item);  // 数组转 FeedItem 对象
    }
}
```

**标准化字段**:
- `uri` - 条目链接
- `title` - 标题
- `timestamp` - 时间戳
- `author` - 作者
- `content` - 内容
- `enclosures` - 附件
- `categories` - 分类
- `uid` - 唯一标识
- 其他字段存入 `$misc` 数组

### 5.5 格式化输出链路

#### FormatFactory
**文件**: `lib/FormatFactory.php:1-52`

```php
public function create(string $name): FormatAbstract
{
    $sanitizedName = $this->sanitizeName($name);
    $className = '\\' . $sanitizedName . 'Format';
    return new $className();
}
```

**可用格式**:
| 格式类 | 文件 | MIME 类型 |
|--------|------|-----------|
| `AtomFormat` | `formats/AtomFormat.php` | `application/atom+xml` |
| `JsonFormat` | `formats/JsonFormat.php` | `application/json` |
| `MrssFormat` | `formats/MrssFormat.php` | `application/rss+xml` |
| `HtmlFormat` | `formats/HtmlFormat.php` | `text/html` |
| `PlaintextFormat` | `formats/PlaintextFormat.php` | `text/plain` |
| `SfeedFormat` | `formats/SfeedFormat.php` | `text/plain` |

#### Atom 输出示例
**文件**: `formats/AtomFormat.php:17-100+`

```php
public function render(): string
{
    $document = new \DomDocument('1.0', 'UTF-8');
    $feed = $document->createElementNS(self::ATOM_NS, 'feed');
    
    // Feed 元数据: name, uri, icon, updated, author
    foreach ($this->getItems() as $item) {
        $entry = $document->createElement('entry');
        // 转换为 Atom <entry> 结构
        // id, title, updated, content, link, author, etc.
    }
    
    return $document->saveXML();
}
```

#### JSON 输出示例
**文件**: `formats/JsonFormat.php:26-100+`

```php
public function render(): string
{
    $data = [
        'version'       => 'https://jsonfeed.org/version/1',
        'title'         => $feedArray['name'],
        'home_page_url' => $feedArray['uri'],
        'items'         => [],
    ];
    
    foreach ($this->getItems() as $item) {
        $entry = [
            'id'            => $item->getUid(),
            'title'         => $item->getTitle(),
            'url'           => $item->getURI(),
            'content_html'  => $item->getContent(),
            'date_modified' => gmdate(\DATE_ATOM, $item->getTimestamp()),
        ];
        $data['items'][] = $entry;
    }
    
    return Json::encode($data, \JSON_PRETTY_PRINT);
}
```

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

### 6.2 错误分类处理策略

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

### 6.4 错误计数与报告阈值
**配置项**: `error.report_limit`

```php
// DisplayAction.php:108-113
$reportLimit = Configuration::getConfig('error', 'report_limit');
$errorCount = 1;
if ($reportLimit > 1) {
    $errorCount = $this->logBridgeError($bridge->getName(), $e->getCode());
}
if ($errorCount >= $reportLimit) {
    // 达到阈值才向客户端暴露错误
}
```

**错误计数缓存**:
```php
// DisplayAction.php:172-191
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
    $this->cache->set($cacheKey, Json::encode($report), 86400 * 5);  // 5 天 TTL
    return $report['count'];
}
```

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
    
    if ($cachedResponse) {
        // 检查 If-Modified-Since，可能返回 304
        return $cachedResponse;
    }
    
    $response = $next($request);
    
    // 错误响应缓存策略
    if ($response->getCode() === 200) {
        // DisplayAction 内部已缓存
    } elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
        $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));  // 5~15 分钟
    }
    
    // 1% 概率触发缓存清理
    if (rand(1, 100) === 1) {
        $this->cache->prune();
    }
    
    return $response;
}
```

### 7.2 DisplayAction 内部缓存 (成功响应)
**文件**: `actions/DisplayAction.php:50-64`

```php
$cacheKey = 'http_' . json_encode($request->toArray());

// ... 执行桥 ...

if ($response->getCode() === 200) {
    $ttl = $request->get('_cache_timeout');
    if (Configuration::getConfig('cache', 'custom_timeout') && isset($ttl)) {
        $ttl = (int) $ttl;
    } else {
        $ttl = $bridge->getCacheTimeout();  // 各桥自定义，默认 3600s
    }
    $this->cache->set($cacheKey, $response, $ttl);
}
```

### 7.3 304 Not Modified 支持
```php
// CacheMiddleware.php:28-38
$ifModifiedSince = $request->server('HTTP_IF_MODIFIED_SINCE');
$lastModified = $cachedResponse->getHeader('last-modified');
if ($ifModifiedSince && $lastModified) {
    $lastModifiedTimestamp = (new \DateTimeImmutable($lastModified))->getTimestamp();
    $modifiedSince = strtotime($ifModifiedSince);
    if ($lastModifiedTimestamp <= $modifiedSince) {
        return new Response('', 304, ['last-modified' => $modificationTimeGMT . 'GMT']);
    }
}
```

---

## 8. 日志系统

### 8.1 Logger 初始化
**文件**: `lib/dependencies.php:48-64`

```php
$container['logger'] = function () {
    $logger = new SimpleLogger('rssbridge');
    if (Configuration::getConfig('system', 'env') === 'dev') {
        $logger->addHandler(new ErrorLogHandler(Logger::DEBUG));
    } else {
        $logger->addHandler(new ErrorLogHandler(Logger::INFO));
    }
    
    // 可选文件日志
    $file_path  = Configuration::getConfig('logging', 'file_path');
    $file_level = Configuration::getConfig('logging', 'file_level');
    if ($file_path && $file_level) {
        $level = array_flip(Logger::LEVEL_NAMES)[strtoupper($file_level)];
        $logger->addHandler(new StreamHandler($file_path, $level));
    }
    
    return $logger;
};
```

### 8.2 日志级别
| 级别 | 值 | 场景 |
|------|----|------|
| DEBUG | 10 | 客户端错误、限流、预期内的 HTTP 错误 |
| INFO | 20 | 桥缺失、一般信息 |
| WARNING | 30 | PHP 非致命错误 |
| ERROR | 40 | 未捕获异常、桥执行失败 |

### 8.3 日志格式化
**文件**: `lib/logger.php:103-149` (StreamHandler)

```php
public function __invoke(array $record)
{
    if ($record['level'] < $this->level) return;
    
    // 异常对象提取
    if (isset($record['context']['e'])) {
        $e = $record['context']['e'];
        $record['context']['type'] = get_class($e);
        $record['context']['code'] = $e->getCode();
        $record['context']['message'] = sanitize_root($e->getMessage());
        $record['context']['file'] = sanitize_root($e->getFile());
        $record['context']['line'] = $e->getLine();
        $record['context']['trace'] = trace_to_call_points(trace_from_exception($e));
    }
    
    // 输出格式: [时间] rssbridge.级别 消息 JSON上下文
    $text = sprintf("[%s] %s.%s %s %s\n", ...);
    file_put_contents($this->stream, $text, FILE_APPEND);
}
```

### 8.4 日志过滤
```php
// logger.php:69-88
private function log(int $level, string $message, array $context = []): void
{
    if (isset($context['e'])) {
        $e = $context['e'];
        if ($e instanceof RateLimitException) return;  // 跳过限流日志
        
        // 跳过已知无害错误
        $ignoredMessages = ['Format name invalid', 'Unknown format given', 'Unable to find'];
        foreach ($ignoredMessages as $ignoredMessage) {
            if (str_starts_with($e->getMessage(), $ignoredMessage)) return;
        }
    }
    // ... 输出到 handlers
}
```

---

## 9. 关键类关系图

```
┌─────────────────────────────────────────────────────────┐
│                      index.php                          │
│  - set_exception_handler                                │
│  - set_error_handler                                    │
│  - register_shutdown_function                           │
│  - Request::fromGlobals()                               │
│  - $rssBridge->main($request)                           │
│  - $response->send()                                    │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│                     RssBridge                           │
│  main(Request): Response                                │
│  ├─ Action 名称解析                                     │
│  ├─ 从 Container 获取 Action 实例                       │
│  └─ 中间件链包装与执行                                  │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│                   Middleware Chain                      │
│  TokenAuthenticationMiddleware → MaintenanceMiddleware  │
│    → SecurityMiddleware → ExceptionMiddleware           │
│      → CacheMiddleware → BasicAuthMiddleware            │
│        → DisplayAction                                  │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│                   DisplayAction                         │
│  __invoke(Request): Response                            │
│  ├─ 参数校验 (bridge, format)                           │
│  ├─ BridgeFactory::createBridgeClassName()              │
│  ├─ BridgeFactory::isEnabled()                          │
│  ├─ BridgeFactory::create() → BridgeAbstract            │
│  ├─ createResponse()                                    │
│  │  ├─ try-catch 错误处理                               │
│  │  ├─ BridgeAbstract::loadConfiguration()              │
│  │  ├─ BridgeAbstract::setInput()                       │
│  │  ├─ BridgeAbstract::collectData()                    │
│  │  ├─ BridgeAbstract::getItems()                       │
│  │  ├─ FormatFactory::create() → FormatAbstract         │
│  │  ├─ FormatAbstract::setItems()                       │
│  │  ├─ FormatAbstract::setFeed()                        │
│  │  └─ FormatAbstract::render()                         │
│  └─ 成功响应缓存写入                                    │
└─────────────────────────────┬───────────────────────────┘
                              │
               ┌──────────────┴──────────────┐
               ▼                             ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│     BridgeAbstract       │   │     FormatAbstract       │
│  - collectData()         │   │  - render(): string      │
│  - getItems(): array     │   │  - setItems(array)       │
│  - setInput(array)       │   │  - setFeed(array)        │
│  - getCacheTimeout()     │   │  - getMimeType()         │
└──────────────────────────┘   └──────────────────────────┘
               ▲                             ▲
               │                             │
┌──────────────────────────┐   ┌──────────────────────────┐
│    GitHubTrendingBridge  │   │     AtomFormat           │
│    DemoBridge            │   │     JsonFormat           │
│    ...                   │   │     MrssFormat           │
│                          │   │     ...                  │
└──────────────────────────┘   └──────────────────────────┘
```

---

## 10. 完整执行时序 (成功路径)

```
1.  GET /?action=Display&bridge=GitHubTrending&format=Atom
    │
    ▼
2.  index.php 引导，创建 Request 对象
    │
    ▼
3.  RssBridge::main() 解析 action=Display → DisplayAction
    │
    ▼
4.  中间件链执行：
    ├─ TokenAuthenticationMiddleware
    ├─ MaintenanceMiddleware
    ├─ SecurityMiddleware
    ├─ ExceptionMiddleware
    ├─ CacheMiddleware → 检查缓存，未命中继续
    └─ BasicAuthMiddleware
        │
        ▼
5.  DisplayAction::__invoke()
    ├─ 校验 bridge=GitHubTrending, format=Atom
    ├─ BridgeFactory::createBridgeClassName("GitHubTrending") → "GitHubTrendingBridge"
    ├─ 白名单检查通过
    ├─ new GitHubTrendingBridge($cache, $logger)
    │
    ▼
6.  createResponse() 执行
    ├─ $bridge->loadConfiguration()
    ├─ 过滤参数，移除系统参数
    ├─ $bridge->setInput($input) → 参数校验
    ├─ $bridge->collectData() → 调用桥的抓取逻辑
    │  └─ $bridge->items[] = [...];  // 原始数据存入
    ├─ $items = $bridge->getItems()
    │
    ▼
7.  格式化输出
    ├─ FormatFactory::create("Atom") → new AtomFormat()
    ├─ $format->setItems($items) → 转为 FeedItem[]
    ├─ $format->setFeed($bridge->getFeed())
    ├─ $format->setLastModified(time())
    ├─ $body = $format->render() → 生成 Atom XML
    │
    ▼
8.  返回 Response($body, 200, ['Content-Type' => 'application/atom+xml'])
    │
    ▼
9.  DisplayAction 写入缓存 (TLL = GitHubTrendingBridge::CACHE_TIMEOUT)
    │
    ▼
10. 中间件链返回，CacheMiddleware 不重复缓存
    │
    ▼
11. Response::send() 输出 HTTP 响应
```

---

## 11. 关键文件速查表

| 模块 | 文件路径 | 核心职责 |
|------|----------|----------|
| 入口 | `index.php` | 请求引导、全局错误处理 |
| 调度器 | `lib/RssBridge.php` | Action 解析、中间件组装 |
| 请求 | `lib/http.php` | Request/Response/HttpClient 定义 |
| 桥工厂 | `lib/BridgeFactory.php` | 桥扫描、名称解析、实例化 |
| 桥基类 | `lib/BridgeAbstract.php` | 桥抽象基类、参数处理 |
| 格式工厂 | `lib/FormatFactory.php` | 输出格式创建 |
| 格式基类 | `lib/FormatAbstract.php` | 格式抽象基类 |
| 数据项 | `lib/FeedItem.php` | Feed 条目数据封装 |
| 容器 | `lib/Container.php` | 依赖注入容器 |
| 依赖配置 | `lib/dependencies.php` | DIC 服务注册 |
| 日志 | `lib/logger.php` | Logger 实现 |
| 配置 | `lib/Configuration.php` | 配置读取 |
| DisplayAction | `actions/DisplayAction.php` | 核心业务：桥执行+格式化+缓存 |
| 缓存中间件 | `middlewares/CacheMiddleware.php` | 响应缓存 |
| 异常中间件 | `middlewares/ExceptionMiddleware.php` | 异常兜底 |
| Atom 格式 | `formats/AtomFormat.php` | Atom XML 渲染 |
| JSON 格式 | `formats/JsonFormat.php` | JSON Feed 渲染 |
