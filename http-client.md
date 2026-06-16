# RSS-Bridge HTTP 客户端架构分析

## 1. 整体架构概览

RSS-Bridge 的 HTTP 请求体系采用**三层架构**设计：

```
┌─────────────────────────────────────────────────┐
│  Bridge 层（各桥接器 collectData）                │
│  · 自定义 UA / Headers / Curl Options            │
│  · 调用 getContents() / getSimpleHTMLDOM() 等     │
├─────────────────────────────────────────────────┤
│  Helper 层（lib/contents.php）                    │
│  · 读取全局配置（UA、超时、代理、重试）            │
│  · 缓存协商（If-Modified-Since / ETag）          │
│  · 响应缓存写入 / 304 命中回填                    │
│  · HTTP 状态码分发处理                            │
├─────────────────────────────────────────────────┤
│  Client 层（lib/http.php）                        │
│  · HttpClient 接口 + CurlHttpClient 实现          │
│  · 默认浏览器指纹 Headers                         │
│  · BoringSSL 检测 → curl-impersonate 分支         │
│  · cURL 重试循环                                  │
│  · 文件大小限制 / 重定向控制                       │
└─────────────────────────────────────────────────┘
```

核心文件：
| 文件 | 职责 |
|------|------|
| `lib/http.php` | HTTP 客户端接口与 cURL 实现，异常类定义 |
| `lib/contents.php` | 全局辅助函数 `getContents()`，配置聚合与缓存协商 |
| `lib/Configuration.php` | INI 配置加载器，环境变量覆盖 |
| `config.default.ini.php` | 默认配置模板（http / proxy / cache 段） |
| `lib/dependencies.php` | DI 容器注册，`http_client` 绑定到 `CurlHttpClient` |
| `lib/WebDriverAbstract.php` | Selenium WebDriver 抽象类（JS 渲染场景） |
| `Dockerfile` | curl-impersonate 安装与 LD_PRELOAD 注入 |

---

## 2. HttpClient 接口与 CurlHttpClient 实现

### 2.1 接口定义

```php
interface HttpClient
{
    public function request(string $url, array $config = []): Response;
}
```

仅一个方法，通过 `$config` 数组传递全部可选项，不暴露 cURL 句柄。

### 2.2 默认配置项

`CurlHttpClient::request()` 内部定义的默认值（`lib/http.php:69-79`）：

```php
$defaultConfig = [
    'useragent'           => null,       // null = 不设置，交给 curl-impersonate
    'timeout'             => 5,          // 秒
    'headers'             => [],
    'proxy'               => null,
    'curl_options'        => [],
    'if_not_modified_since' => null,
    'retries'             => 2,          // 失败后最多重试 2 次
    'max_filesize'        => null,       // 字节
    'max_redirections'    => 5,
];
```

> **注意**：`CurlHttpClient` 内部的 `retries` 默认值是 `2`，但 `getContents()` 从配置文件读取的默认值是 `1`（见 `config.default.ini.php:49`），实际生效以配置文件为准。

### 2.3 默认浏览器指纹 Headers

`lib/http.php:82-91` 中硬编码了 Firefox 102 的请求头指纹，摘自 curl-impersonate 项目：

```php
$defaultHeaders = [
    'Accept'                    => 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
    'Accept-Language'           => 'en-US,en;q=0.5',
    'Upgrade-Insecure-Requests' => '1',
    'Sec-Fetch-Dest'            => 'document',
    'Sec-Fetch-Mode'            => 'navigate',
    'Sec-Fetch-Site'            => 'none',
    'Sec-Fetch-User'            => '?1',
    'TE'                        => 'trailers',
];
```

这些头部模拟了真实的 Firefox 浏览器导航请求，其中 `Sec-Fetch-*` 系列是现代浏览器的安全元数据头，缺少这些头部是服务端识别爬虫的常见信号。

---

## 3. 用户代理（UA）切换机制

### 3.1 三级 UA 优先级

UA 的生效遵循以下优先级链：

```
Bridge 自定义 UA  >  全局配置 UA  >  curl-impersonate 自动设置
```

#### 第一级：curl-impersonate 自动设置（最高隐式优先级）

`lib/http.php:93-101` 中的关键分支：

```php
if (curl_version()['ssl_version'] == 'BoringSSL') {
    // curl-impersonate 环境：不设置任何 UA，由库自动处理
    $config = array_merge($defaultConfig, $config);
} else {
    // 原生 cURL 环境：手动设置 Firefox 102 UA + 浏览器指纹 Headers
    $defaultConfig['useragent'] = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0) Gecko/20100101 Firefox/102.0';
    curl_setopt($ch, CURLOPT_HEADER, false);
    $headers = array_merge($defaultHeaders, $config['headers']);
    $config = array_merge($defaultConfig, $config);
    $config['headers'] = $headers;
}
```

**检测逻辑**：通过 `curl_version()['ssl_version']` 判断是否为 BoringSSL（curl-impersonate 的特征）。如果是：
- 不显式设置 UA，让 curl-impersonate 根据 `CURL_IMPERSONATE=chrome142` 环境变量自动匹配 Chrome 142 的完整 TLS 指纹 + UA
- 不强制合并默认 Headers，保留 curl-impersonate 提供的原生 Chrome 头部

如果不是（即原生 libcurl）：
- 强制设置 Firefox 102 UA
- 合并硬编码的 Firefox 浏览器指纹 Headers

#### 第二级：全局配置 UA

在 `config.default.ini.php` 的 `[http]` 段中：

```ini
;useragent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0) Gecko/20100101 Firefox/102.0"
```

默认被注释掉（为 null），通过 `getContents()` 传入：

```php
$config = [
    'useragent' => Configuration::getConfig('http', 'useragent'),  // null by default
    ...
];
```

在 `CurlHttpClient` 中，只有 `$config['useragent']` 非空时才会调用 `curl_setopt($ch, CURLOPT_USERAGENT, ...)`（`lib/http.php:109-111`），否则 UA 由底层库决定。

#### 第三级：Bridge 自定义 UA

各桥接器通过以下方式覆盖 UA：

**方式 A：通过 `$httpHeaders` 参数（推荐）**

```php
// InstagramBridge.php:90
$headers[] = 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/62.0.3202.94 Safari/537.36';
return getContents($uri, $headers);
```

这种方式将 `User-Agent` 作为 HTTP Header 数组传入，在 `getContents()` 中被解析为 `$httpHeadersNormalized['User-Agent']`，最终通过 `CURLOPT_HTTPHEADER` 设置。**注意**：这种方式可能与 `CURLOPT_USERAGENT` 产生冲突——如果全局配置也设置了 UA，cURL 会同时发送两个 UA 头。

**方式 B：通过 `$curlOptions` 参数**

```php
// ComickBridge.php:116
$opts = [
    CURLOPT_USERAGENT => 'rss-bridge (https://github.com/RSS-Bridge/rss-bridge)'
];
$content = getContents("$API/$url", [], $opts);
```

这种方式直接使用 `CURLOPT_USERAGENT` 常量，通过 `curl_setopt_array()` 生效，会覆盖任何先前的 UA 设置。

### 3.2 Bridge UA 定制策略分类

| 策略 | 示例 Bridge | UA 内容 | 目的 |
|------|-------------|---------|------|
| 伪装桌面浏览器 | Instagram, GoComics, Subito | Chrome / Firefox 桌面版 UA | 绕过移动端/桌面端检测 |
| 伪装移动应用 | LeBonCoin | `LBC;Android;10;SAMSUNG;...` | 模拟官方 App 访问 API |
| 伪装搜索引擎 | GatesNotes | `Googlebot/2.1` | 获取无 Canvas 懒加载的干净内容 |
| 自报身份 | Reddit, Modrinth, Comick, AO3 | `rss-bridge v0.0.2 (...)` | 遵守 API 使用规范，避免被封 |
| 伪装特定版本 | Fab, Idealo, ARMCommunity | Firefox 139/140 | 绕过版本检测 |

---

## 4. 超时配置

### 4.1 配置来源

```ini
; config.default.ini.php
[http]
timeout = 5    ; 秒
```

### 4.2 生效路径

```
config.ini.php → Configuration::getConfig('http', 'timeout')
    → getContents() 中 $config['timeout']
    → CurlHttpClient::request() 中 curl_setopt($ch, CURLOPT_TIMEOUT, $config['timeout'])
```

`CURLOPT_TIMEOUT` 是整体请求超时（含连接 + 传输），不是仅连接超时。默认 5 秒对大多数场景偏紧，生产环境通常需要适当调大。

---

## 5. 代理配置

### 5.1 全局代理

```ini
; config.default.ini.php
[proxy]
url = ""                     ; 代理地址，如 socks5://127.0.0.1:1080
name = "Hidden proxy name"   ; 前台显示的代理名称
by_bridge = false            ; 是否允许用户按请求禁用代理
```

### 5.2 代理生效流程

```
1. getContents() 读取配置：
   if (Configuration::getConfig('proxy', 'url') && !defined('NOPROXY')) {
       $config['proxy'] = Configuration::getConfig('proxy', 'url');
   }

2. CurlHttpClient::request() 设置：
   if ($config['proxy']) {
       curl_setopt($ch, CURLOPT_PROXY, $config['proxy']);
   }
```

### 5.3 按请求禁用代理（NOPROXY）

`DisplayAction.php:41-48` 实现了代理按请求禁用逻辑：

```php
if (
    Configuration::getConfig('proxy', 'url')           // 全局代理已配置
    && Configuration::getConfig('proxy', 'by_bridge')  // 允许按请求禁用
    && $noproxy                                        // 用户传了 _noproxy 参数
) {
    define('NOPROXY', true);  // 定义常量，getContents() 中检查
}
```

用户在请求 URL 中添加 `&_noproxy=1` 即可跳过代理。`NOPROXY` 是全局常量，一旦定义不可撤销，所以同一进程内后续请求也会受影响。

### 5.4 Bridge 级别的代理

PixivBridge 实现了独特的图像代理配置（`bridges/PixivBridge.php:18-22`）：

```php
const CONFIGURATION = [
    'proxy_url' => [
        'required' => false,
        'defaultValue' => null
    ]
];
```

这不是 HTTP 代理，而是**图片 URL 替换代理**——将 `https://i.pximg.net/` 替换为用户配置的镜像地址，用于绕过 Pixiv 的防盗链：

```php
if ($proxy_url) {
    $img_url = preg_replace('/https:\/\/i\.pximg\.net/', $proxy_url, $imagejson['body']['urls']['original']);
}
```

---

## 6. 重试机制

### 6.1 实现位置

`lib/http.php:171-192`，在 `CurlHttpClient::request()` 内部：

```php
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        break;  // 网络请求成功，跳出循环
    }
    if ($tries <= $config['retries']) {
        continue;  // 未达重试上限，继续
    }
    // 超过重试上限，抛出异常
    $curl_error = curl_error($ch);
    $curl_errno = curl_errno($ch);
    throw new HttpException(sprintf(
        'cURL error %s: %s (%s) for %s',
        $curl_error, $curl_errno,
        'https://curl.haxx.se/libcurl/c/libcurl-errors.html',
        $url
    ));
}
```

### 6.2 重试行为分析

| 维度 | 行为 |
|------|------|
| 触发条件 | 仅 `curl_exec()` 返回 `false` 时重试（网络层失败） |
| 不触发场景 | HTTP 4xx/5xx 状态码**不重试**（cURL 层面请求已成功） |
| 重试次数 | 默认 1 次（配置文件），客户端默认 2 次 |
| 重试间隔 | **无退避等待**，立即重试 |
| cURL 句柄 | 整个循环复用同一个 `$ch`，不重新初始化 |

### 6.3 重试次数配置链

```
config.default.ini.php: retries = 1
  → Configuration::getConfig('http', 'retries')
    → getContents() 中 $config['retries']
      → CurlHttpClient::request() 中 $config['retries']
```

---

## 7. 指纹伪装与反检测策略

### 7.1 curl-impersonate（核心指纹伪装）

这是 RSS-Bridge 最重要的一层反检测能力，通过替换底层 libcurl 实现完整的浏览器 TLS 指纹模拟。

**Dockerfile 安装过程（`Dockerfile:31-59`）：**

```dockerfile
curlimpersonate_version=1.2.5
# 根据架构选择对应的预编译包（aarch64 / armv7l / x86_64）
curl -LO "https://github.com/lexiforest/curl-impersonate/releases/download/v${curlimpersonate_version}/${archive}"
tar xaf "$archive" -C /usr/local/lib/curl-impersonate
# 修改 SO 名使其替代系统 libcurl
patchelf --set-soname libcurl.so.4 /usr/local/lib/curl-impersonate/libcurl-impersonate.so

ENV LD_PRELOAD=/usr/local/lib/curl-impersonate/libcurl-impersonate.so
ENV CURL_IMPERSONATE=chrome142
```

**工作原理**：
1. `LD_PRELOAD` 让 PHP 的 `curl_*` 函数实际调用 curl-impersonate 的库
2. `CURL_IMPERSONATE=chrome142` 让库自动模拟 Chrome 142 的完整 TLS 指纹（包括 JA3/JA4、HTTP/2 设置、加密套件顺序等）
3. PHP 代码无需任何修改即可获得浏览器级别的 TLS 指纹

**代码中的适配**（`lib/http.php:93`）：

```php
if (curl_version()['ssl_version'] == 'BoringSSL') {
    // curl-impersonate 使用 BoringSSL（Chrome 的 SSL 库）
    // 不强制设置 UA 和 Headers，由库自动完成
} else {
    // 普通环境使用 OpenSSL，手动设置 Firefox 102 指纹作为降级方案
}
```

### 7.2 HTTP Header 指纹

除了 TLS 指纹外，RSS-Bridge 还关注 HTTP Header 指纹的一致性：

- **curl-impersonate 环境**：库自动设置与 Chrome 142 一致的完整 Header 集
- **非 impersonate 环境**：代码硬编码 Firefox 102 的 Header 集（`Sec-Fetch-*`、`Accept` 等），确保与 UA 声明一致

### 7.3 CloudFlare 检测识别

`lib/http.php:38-56` 定义了 `CloudFlareException`，专门识别 CloudFlare 防护页：

```php
final class CloudFlareException extends HttpException
{
    public static function isCloudFlareResponse(Response $response): bool
    {
        $cloudflareTitles = [
            '<title>Just a moment...',        // 经典 CF 挑战页
            '<title>Please Wait...',           // CF 等待页
            '<title>Attention Required!',      // CF 验证页
            '<title>Security | Glassdoor',     // Glassdoor 定制 CF 页
            '<title>Access denied</title>',    // Patreon 等 CF 拦截页
        ];
        foreach ($cloudflareTitles as $cloudflareTitle) {
            if (str_contains($response->getBody(), $cloudflareTitle)) {
                return true;
            }
        }
        return false;
    }
}
```

当 HTTP 状态码非 2xx/304 时，`HttpException::fromResponse()` 会自动检查是否为 CloudFlare 拦截，并抛出 `CloudFlareException`。在 `DisplayAction` 中，429/503 状态码会直接返回给客户端而不记录为错误日志。

### 7.4 WebDriver 方案（终极渲染）

对于需要 JavaScript 渲染才能获取内容的站点，RSS-Bridge 提供了 `WebDriverAbstract`（`lib/WebDriverAbstract.php`）：

```php
abstract class WebDriverAbstract extends BridgeAbstract
{
    protected function prepareWebDriver()
    {
        $server = Configuration::getConfig('webdriver', 'selenium_server_url');
        $this->driver = RemoteWebDriver::create($server, $this->getDesiredCapabilities());
    }

    protected function getBrowserOption()
    {
        $chromeOptions = new ChromeOptions();
        if (Configuration::getConfig('webdriver', 'headless')) {
            $chromeOptions->addArguments(['--headless']);
        }
        return $chromeOptions;
    }
}
```

配置项：
```ini
[webdriver]
selenium_server_url = "http://localhost:4444"
headless = false
```

使用 WebDriver 的 Bridge（如 ScalableCapitalBlogBridge、GULPProjekteBridge）启动真实浏览器实例，完全绕过 TLS/JS 指纹检测，但代价是极高的资源消耗。

### 7.5 反检测策略总结

| 层级 | 策略 | 绕过能力 | 资源消耗 |
|------|------|----------|----------|
| L1 | 默认浏览器 Headers（Sec-Fetch-* 等） | 低 | 极低 |
| L2 | 自定义 UA 伪装（Chrome/Firefox/Mobile/Bot） | 中 | 极低 |
| L3 | curl-impersonate（完整 TLS + HTTP/2 指纹模拟） | 高 | 低 |
| L4 | WebDriver（真实浏览器渲染） | 极高 | 高 |

---

## 8. HTTP 缓存与条件请求

`getContents()` 实现了 HTTP 条件请求机制，减少不必要的数据传输：

### 8.1 缓存写入

HTTP 200/201/202 响应会被缓存（`lib/contents.php:119`）：

```php
$cache->set($cacheKey, $response, 86400 * 10);  // 缓存 10 天
```

但受 `Cache-Control` 约束：如果响应包含 `no-cache` 或 `no-store` 指令，则不缓存。

### 8.2 条件请求（缓存命中时）

当缓存存在时，`getContents()` 会自动附加条件请求头：

```php
// If-Modified-Since：通过 curl 的 CURLOPT_TIMEVALUE 实现
if ($lastModified) {
    $config['if_not_modified_since'] = $lastModified->getTimestamp();
}

// If-None-Match：通过自定义 Header 实现
if ($etag) {
    $httpHeadersNormalized['if-none-match'] = $etag;
}
```

### 8.3 304 响应处理

```php
case 304:
    $response = $response->withBody($cachedResponse->getBody());
    break;
```

304 响应体为空，用缓存的 Body 填充后返回给调用者，对上层透明。

---

## 9. 完整请求生命周期

```
Bridge::collectData()
  │
  ├─ 自定义 headers / curlOptions
  │
  ▼
getContents($url, $httpHeaders, $curlOptions)
  │
  ├─ 1. 读取全局配置（UA、timeout、retries、max_filesize）
  ├─ 2. 解析 headers 为关联数组
  ├─ 3. 检查缓存 → 注入 If-Modified-Since / ETag
  ├─ 4. 读取代理配置 → 检查 NOPROXY
  ├─ 5. 调用 $httpClient->request($url, $config)
  │     │
  │     ▼
  │   CurlHttpClient::request()
  │     ├─ 检测 BoringSSL → 分支处理 UA/Headers
  │     ├─ 合并默认配置与传入配置
  │     ├─ 设置 cURL 选项（UA、timeout、proxy、headers...）
  │     ├─ 重试循环（curl_exec 失败时重试）
  │     └─ 返回 Response
  │
  ├─ 6. 处理响应状态码
  │     ├─ 200/201/202 → 写入缓存
  │     ├─ 301/302/303 → 跟随重定向（cURL 层面处理）
  │     ├─ 304 → 回填缓存 Body
  │     └─ 其他 → 抛出 HttpException / CloudFlareException
  │
  └─ 7. 返回 Body 字符串或完整 Response 对象
```

---

## 10. 配置速查表

| 配置段 | 键 | 默认值 | 说明 |
|--------|-----|--------|------|
| `[http]` | `timeout` | `5` | 请求超时（秒） |
| `[http]` | `retries` | `1` | cURL 错误重试次数 |
| `[http]` | `useragent` | `null` | 自定义 UA（null=由 curl-impersonate 决定） |
| `[http]` | `max_filesize` | `20` | 最大响应体积（MB） |
| `[proxy]` | `url` | `""` | HTTP 代理地址 |
| `[proxy]` | `name` | `"Hidden proxy name"` | 前台代理显示名 |
| `[proxy]` | `by_bridge` | `false` | 允许用户通过 `_noproxy` 参数禁用代理 |
| `[cache]` | `custom_timeout` | `false` | 允许用户自定义缓存超时 |
| `[webdriver]` | `selenium_server_url` | `"http://localhost:4444"` | Selenium 服务器地址 |
| `[webdriver]` | `headless` | `false` | 浏览器无头模式 |

环境变量覆盖规则：`RSSBRIDGE_{SECTION}_{KEY}`，例如 `RSSBRIDGE_HTTP_TIMEOUT=10`。
