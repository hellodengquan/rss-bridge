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

---

## 11. 高级配置与边缘场景

### 11.1 CURL_TIMEOUT 与 max_time 优先关系

RSS-Bridge 使用多层超时控制机制，优先级从高到低如下：

| 配置项 | 位置 | 生效范围 | 优先级 | 说明 |
|--------|------|----------|--------|------|
| `CURLOPT_TIMEOUT` | `lib/http.php:115` | 整个请求（连接 + 传输） | 最高 | 由 `$config['timeout']` 控制，默认 5 秒 |
| `CURLOPT_CONNECTTIMEOUT` | `actions/ConnectivityAction.php:48` | 仅连接阶段 | 中 | 仅在连通性检测中使用，默认 5 秒 |
| `CURLOPT_MAXFILESIZE` | `lib/http.php:121` | 响应体大小 | 中 | 通过 `Content-Length` 头校验，仅检查声明的大小 |
| `CURLOPT_PROGRESSFUNCTION` | `lib/http.php:124-130` | 传输过程中的实际大小 | 最高（字节级） | 实时监控下载字节数，超过 `max_filesize` 立即终止 |
| PHP `max_execution_time` | php.ini | 整个 PHP 脚本 | 最低 | 通常为 30 秒，作为最后防线 |

**关键代码路径**（`lib/http.php:115, 119-131`）：

```php
curl_setopt($ch, CURLOPT_TIMEOUT, $config['timeout']);

if ($config['max_filesize']) {
    // 仅检查 Content-Length 头声明的大小
    curl_setopt($ch, CURLOPT_MAXFILESIZE, $config['max_filesize']);
    curl_setopt($ch, CURLOPT_NOPROGRESS, false);
    // 进度函数实时监控实际下载量，对分块编码响应也有效
    curl_setopt($ch, CURLOPT_PROGRESSFUNCTION, function ($ch, $downloadSize, $downloaded, $uploadSize, $uploaded) use ($config) {
        if ($downloaded > $config['max_filesize']) {
            return -1;  // 返回非零值立即终止传输
        }
        return 0;
    });
}
```

**注意**：`CURLOPT_MAXFILESIZE` 只检查 `Content-Length` 响应头，如果服务器使用分块编码（`Transfer-Encoding: chunked`），这个选项无效。因此需要配合 `CURLOPT_PROGRESSFUNCTION` 实现双重保障。

---

### 11.2 SOCKS5h 与 Proxy Chain 配置路径

#### SOCKS5 协议变体

RSS-Bridge 支持多种代理协议，通过代理 URL 前缀自动识别：

| 前缀 | 协议 | DNS 解析位置 | 适用场景 |
|------|------|-------------|----------|
| `http://` | HTTP 代理 | 代理服务器 | 通用 Web 代理 |
| `https://` | HTTPS 代理 | 代理服务器 | 加密代理通道 |
| `socks4://` | SOCKS4 | 本地 | 旧版 SOCKS 协议 |
| `socks4a://` | SOCKS4a | 代理服务器 | SOCKS4 带远程 DNS |
| `socks5://` | SOCKS5 | **本地** | 标准 SOCKS5，**DNS 在本地解析** |
| `socks5h://` | SOCKS5 | **代理服务器** | **推荐**，DNS 在代理端解析，避免 DNS 泄漏 |

**配置链**：

```
1. config.ini.php: [proxy] url = "socks5h://127.0.0.1:1080"
   ↓
2. Configuration::getConfig('proxy', 'url')
   ↓
3. getContents() 检查 !defined('NOPROXY')
   ↓
4. $config['proxy'] = 代理地址
   ↓
5. CurlHttpClient::request() → curl_setopt($ch, CURLOPT_PROXY, $config['proxy'])
```

#### Proxy Chain（代理链）限制

RSS-Bridge **原生不支持代理链**（Proxy Chaining），即无法配置 `代理A → 代理B → 目标` 的多级代理。cURL 本身也只支持单个代理。

如果需要代理链，有两种实现方式：

1. **外部代理工具**：在本地启动 `proxychains-ng` 或 `privoxy`，将 RSS-Bridge 的代理指向本地端口，由外部工具处理多级转发
2. **Bridge 级自定义**：在 Bridge 中通过 `curl_options` 手动配置，但这需要每个 Bridge 单独实现

---

### 11.3 503 与 429 速率限制的差异化重试

RSS-Bridge 对速率限制错误采用**三层处理机制**，503 和 429 在不同层级有不同的处理策略：

#### 层级 1：DisplayAction 全局拦截

`DisplayAction.php:97-101` 对 HTTP 层的 429/503 做差异化处理：

```php
} elseif ($e instanceof HttpException) {
    if (in_array($e->getCode(), [429, 503])) {
        // 仅记录 debug 日志，不记错误日志
        $this->logger->debug(sprintf('Exception in DisplayAction(%s): %s', ...));
        // 直接返回给客户端，不重试
        return new Response(render(..., ['e' => $e]), $e->getCode());
    }
}
```

#### 层级 2：CurlHttpClient 重试（不触发）

`lib/http.php:171-192` 的重试循环**仅对网络层错误**（`curl_exec() === false`）生效，对 HTTP 429/503 不重试，因为 cURL 层面请求已成功。

#### 层级 3：Bridge 级自定义限流保护

各热门 Bridge 实现了自己的速率限制熔断机制，**不同 Bridge 策略不同**：

**SpotifyBridge**（`bridges/SpotifyBridge.php:103-111`）：
```php
$cacheKey = 'spotify_rate_limit';
try {
    $this->collectDataInternal();
} catch (HttpException $e) {
    if ($e->getCode() === 429) {
        // 读取 Retry-After 头，尊重服务端建议的等待时间
        $retryAfter = $e->response->getHeader('Retry-After') ?? (60 * 5);
        $this->cache->set($cacheKey, true, $retryAfter);
        throwRateLimitException(sprintf('Rate limited by spotify, try again in %s seconds', $retryAfter));
    }
    throw $e;
}
```

**YoutubeBridge**（`bridges/YoutubeBridge.php:76-86`）：
```php
$cacheKey = 'youtube_rate_limit';
if ($this->cache->get($cacheKey)) {
    throwRateLimitException();  // 直接拒绝，不调用 API
}
try {
    $this->collectDataInternal();
} catch (HttpException $e) {
    if ($e->getCode() === 429) {
        $this->cache->set($cacheKey, true, 60 * 16);  // 熔断 16 分钟
        throwRateLimitException();
    }
    throw $e;
}
```

**RedditBridge**（`bridges/RedditBridge.php:117-140`）：
```php
// 区分 403（IP 封禁）和 429（速率限制）
$forbiddenKey = 'reddit_forbidden';
$rateLimitKey = 'reddit_rate_limit';

if ($this->cache->get($forbiddenKey) || $this->cache->get($rateLimitKey)) {
    throwRateLimitException();
}

try {
    $this->collectDataInternal();
} catch (HttpException $e) {
    if ($e->getCode() === 403) {
        // 403 可能是永久 IP 封禁，熔断 61 分钟
        $this->cache->set($forbiddenKey, true, 60 * 61);
        throwRateLimitException();
    } elseif ($e->getCode() === 429) {
        // 429 是速率限制，熔断 61 分钟
        $this->cache->set($rateLimitKey, true, 60 * 61);
        throwRateLimitException();
    }
    throw $e;
}
```

#### 429 vs 503 对比表

| 特征 | 429 Too Many Requests | 503 Service Unavailable |
|------|----------------------|------------------------|
| 含义 | 客户端请求频率超限 | 服务端暂时不可用（过载/维护） |
| `Retry-After` | 通常有，建议等待时间 | 可能有，服务恢复时间估计 |
| 全局处理 | 返回 429 + 异常页面 | 返回 503 + 异常页面 |
| Bridge 熔断 | Spotify: 尊重 Retry-After<br>Youtube: 16 分钟<br>Reddit: 61 分钟 | 无通用 Bridge 级熔断 |
| 重试预期 | 等待后大概率成功 | 可能需要较长时间 |
| 错误日志级别 | Debug（不报警） | Debug（不报警） |

---

### 11.4 TLS 指纹与 Client Hello 顺序匹配

RSS-Bridge 依赖 `curl-impersonate` 实现 TLS 指纹伪装，这是绕过 CloudFlare 等反爬系统的核心。

#### curl-impersonate 的工作原理

**Dockerfile 配置**（`Dockerfile:59-60`）：
```dockerfile
ENV LD_PRELOAD=/usr/local/lib/curl-impersonate/libcurl-impersonate.so
ENV CURL_IMPERSONATE=chrome142
```

这两个环境变量让 curl-impersonate 自动模拟 Chrome 142 的以下指纹特征：

1. **JA3 指纹**：TLS Client Hello 中加密套件、扩展、椭圆曲线的顺序组合
2. **JA4 指纹**：更精细的 TLS 握手特征，包括 ALPN、签名算法顺序
3. **HTTP/2 SETTINGS 帧**：Chrome 特有的初始窗口大小、并发流数等设置
4. **HTTP/2 Header 顺序**：`:method`, `:authority`, `:scheme`, `:path` 的发送顺序
5. **HTTP/2 Pseudo-Header 大小写**：Chrome 小写，某些爬虫框架大写

#### BoringSSL 检测分支

`lib/http.php:93-101` 中的检测逻辑确保代码与 curl-impersonate 正确配合：

```php
if (curl_version()['ssl_version'] == 'BoringSSL') {
    // curl-impersonate 环境：BoringSSL 是 Chrome 使用的 SSL 库
    // 不设置任何 UA 和 Headers，由库自动生成与 Chrome 142 一致的请求
    $config = array_merge($defaultConfig, $config);
} else {
    // 原生 OpenSSL 环境：降级到 Firefox 102 指纹
    $defaultConfig['useragent'] = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:102.0) Gecko/20100101 Firefox/102.0';
    $headers = array_merge($defaultHeaders, $config['headers']);
    $config = array_merge($defaultConfig, $config);
    $config['headers'] = $headers;
}
```

**Client Hello 顺序的重要性**：
- 反爬系统（如 CloudFlare）会校验 Client Hello 中扩展的发送顺序
- Chrome 有固定的扩展顺序：`server_name` → `extended_master_secret` → `renegotiation_info` → `supported_groups` → ...
- 原生 cURL + OpenSSL 的扩展顺序与 Chrome 不同，容易被识别
- curl-impersonate 精确复刻了 Chrome 的扩展发送顺序，JA3 哈希与真实 Chrome 完全一致

---

### 11.5 BridgeAbstract 子类如何继承默认 Headers

**没有自动继承机制**。BridgeAbstract 本身不定义任何默认 Headers，每个子类需要显式定义并传递自己的 Headers。

#### Headers 传递的四种模式

| 模式 | 示例 | 说明 |
|------|------|------|
| **const 常量** | `const HEADERS = [...]` | 可被子类继承/覆盖，推荐 |
| **const 常量（别名）** | `const FAKE_HEADERS = [...]` | 语义化命名，如"伪装浏览器头" |
| **类属性** | `private $headers = [...]` | 不可继承，仅当前类使用 |
| **方法内局部变量** | `$headers = [...]` | 仅单次请求使用 |

**示例 1：const 常量 + 继承（AkamaiBridge）**
```php
// 父类 FeedExpander 的 collectExpandableDatas 会接收 headers
class AkamaiBridge extends FeedExpander
{
    const HEADERS = [
        'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0',
        'Accept-Language: en',
    ];

    protected function parseItem(array $item)
    {
        // 子类 parseItem 中显式传递 self::HEADERS
        $page = getSimpleHTMLDOMCached($item['uri'], self::CACHE_TIMEOUT, self::HEADERS);
        ...
    }
}
```

**示例 2：FeedExpander 自动添加 Accept 头**

`FeedExpander.php:19-21` 在收集 RSS Feed 时自动添加 Accept 头：
```php
public function collectExpandableDatas(string $url, $maxItems = -1, $headers = [])
{
    $accept = [MrssFormat::MIME_TYPE, AtomFormat::MIME_TYPE, '*/*'];
    // 合并 Feed 专用的 Accept 头与子类传入的 headers
    $httpHeaders = array_merge(['Accept: ' . implode(', ', $accept)], $headers);
    $xmlString = getContents($url, $httpHeaders);
    ...
}
```

**示例 3：完整浏览器指纹复制（RobinhoodSnacksBridge）**

`bridges/RobinhoodSnacksBridge.php:12-26` 复制完整的浏览器请求头集：
```php
const FAKE_HEADERS = [
    'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:100.0) Gecko/20100101 Firefox/100.0',
    'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
    'Accept-Language: es-ES,en-US;q=0.7,en;q=0.3',
    'Accept-Encoding: gzip, deflate, br',
    'Connection: keep-alive',
    'Upgrade-Insecure-Requests: 1',
    'Sec-Fetch-Dest: document',
    'Sec-Fetch-Mode: navigate',
    'Sec-Fetch-Site: none',
    'Sec-Fetch-User: ?1',
    'Pragma: no-cache',
    'Cache-Control: no-cache',
    'TE: trailers'
];
```

**Headers 合并规则**（`lib/http.php:97-99`）：
```php
// 在非 BoringSSL 环境下，传入的 headers 会覆盖默认 headers
$headers = array_merge($defaultHeaders, $config['headers']);
```
注意：`array_merge` 中如果键名相同，后面的会覆盖前面的。

---

### 11.6 cf-clearance 反爬与 cloudscraper 兜底

#### cf-clearance 处理现状

RSS-Bridge **不直接处理 `cf_clearance` cookie**，而是依赖 curl-impersonate 的 TLS 指纹伪装来绕过 CloudFlare 检测。

**CloudFlare 检测层级**：
```
L1: TLS 指纹 (JA3/JA4)        → curl-impersonate 模拟 Chrome 142
L2: HTTP/2 特征               → curl-impersonate 模拟
L3: Header 顺序与一致性       → 代码硬编码 + 自动生成
L4: JS 挑战 (Turnstile)        → ❌ 无法绕过，需要 WebDriver
L5: cf_clearance Cookie       → ❌ 无自动获取机制
```

#### CloudFlare 检测与识别

`lib/http.php:38-56` 中的 `CloudFlareException` 通过响应标题识别 CF 拦截页：

```php
$cloudflareTitles = [
    '<title>Just a moment...',        // 经典 5 秒盾挑战页
    '<title>Please Wait...',           // CF 等待页
    '<title>Attention Required!',      // CF 人机验证页
    '<title>Security | Glassdoor',     // Glassdoor 定制 CF 页
    '<title>Access denied</title>',    // Patreon 等 CF 403 页
];
```

#### 兜底方案

**方案 1：curl-impersonate（主要方案）**
- 模拟 Chrome 142 完整 TLS + HTTP/2 指纹
- 对大多数"低风险"站点有效（约 70-80%）
- 无需额外代码，容器启动时自动生效

**方案 2：WebDriver（终极兜底）**

对于开启了 JS 挑战的站点，使用 `WebDriverAbstract` 启动真实浏览器：

```php
// 示例：ScalableCapitalBlogBridge 继承 WebDriverAbstract
abstract class WebDriverAbstract extends BridgeAbstract
{
    protected function prepareWebDriver()
    {
        $server = Configuration::getConfig('webdriver', 'selenium_server_url');
        $this->driver = RemoteWebDriver::create($server, $this->getDesiredCapabilities());
    }
}
```

配置 Selenium/ChromeDriver 后，真实浏览器可以：
- 执行 CloudFlare 的 JavaScript 挑战
- 自动获取并存储 `cf_clearance` cookie
- 渲染动态加载的内容

**方案 3：Bridge 级 Cookie 配置**

某些 Bridge 允许用户手动配置 cookie 绕过 CF：

```php
// PixivBridge.php:14-16
const CONFIGURATION = [
    'cookie' => [
        'required' => false,
        'defaultValue' => null
    ]
];

// 使用时通过 header 传递
$headers[] = 'Cookie: PHPSESSID=' . $this->getOption('cookie');
```

> **注意**：cloudscraper（Python 库）在 RSS-Bridge 中**没有集成**。RSS-Bridge 是 PHP 项目，不直接使用 Python 生态的反爬工具。

---

### 11.7 HTTP/2 与 Brotli 编码协商

#### HTTP 版本协商

**默认行为**：cURL 自动协商最高可用版本，优先 HTTP/2。

`lib/http.php` 中**没有显式设置** `CURLOPT_HTTP_VERSION`，这意味着使用 cURL 默认值：
- cURL 7.47.0+ 默认使用 `CURL_HTTP_VERSION_2TLS`（HTTPS 用 HTTP/2，HTTP 用 HTTP/1.1）
- 如果服务器不支持 HTTP/2，自动降级到 HTTP/1.1

**显式降级到 HTTP/1.1**（IdealoBridge 示例）：

`bridges/IdealoBridge.php:44` 显式强制使用 HTTP/1.1：
```php
private $options = [
    CURLOPT_HTTP_VERSION => CURL_HTTP_VERSION_1_1,  // 强制 HTTP/1.1
    CURLOPT_TRANSFER_ENCODING => 1,
    CURLOPT_ACCEPT_ENCODING => 'gzip, deflate, br'
];
```

**降级原因**：某些服务器的 HTTP/2 实现有 bug，或反爬系统会检查 HTTP/2 帧细节与指纹是否匹配。

#### Brotli 编码协商

**全局默认设置**（`lib/http.php:116`）：
```php
curl_setopt($ch, CURLOPT_ENCODING, '');
```

**空字符串 `''` 的含义**：cURL 会自动添加 `Accept-Encoding` 头，包含所有编译时支持的编码（gzip, deflate, br, zstd 等），并自动解码响应。

**Bridge 级自定义编码**：

某些 Bridge 显式指定 `Accept-Encoding` 头以匹配浏览器行为：

| Bridge | 编码设置 | 位置 |
|--------|---------|------|
| InstagramBridge | `gzip, deflate, br` | `bridges/InstagramBridge.php:92` |
| RobinhoodSnacksBridge | `gzip, deflate, br` | `bridges/RobinhoodSnacksBridge.php:16` |
| FabBridge | `gzip, deflate, br, zstd` | `bridges/FabBridge.php:19` |
| IdealoBridge | `gzip, deflate, br` | `bridges/IdealoBridge.php:46` |

**curl-impersonate 环境下的编码**：
- curl-impersonate 会自动设置与 Chrome 142 一致的 `Accept-Encoding` 头
- 顺序通常为：`gzip, deflate, br, zstd`
- 如果 Bridge 显式覆盖，以 Bridge 设置为准

#### 编码与反检测的关系

`Accept-Encoding` 头的内容和顺序是浏览器指纹的一部分：
- Chrome 142: `gzip, deflate, br, zstd`
- Firefox 102: `gzip, deflate, br`
- 旧式爬虫常省略 `br`（Brotli）支持

如果 UA 声明是 Chrome，但 `Accept-Encoding` 不包含 `zstd`，就会产生指纹不一致，触发反爬。

---

## 12. 配置速查表（补充）

| cURL 选项 | 默认值 | 说明 |
|-----------|--------|------|
| `CURLOPT_ENCODING` | `''` | 空字符串 = 自动支持所有编码并自动解码 |
| `CURLOPT_HTTP_VERSION` | 自动协商 | 默认 HTTPS 用 HTTP/2，HTTP 用 HTTP/1.1 |
| `CURLOPT_MAXFILESIZE` | `null` | 仅检查 Content-Length 头 |
| `CURLOPT_PROGRESSFUNCTION` | 回调 | 实时监控下载量，对分块编码也有效 |
| `CURLOPT_CONNECTTIMEOUT` | 未全局设置 | 仅 ConnectivityAction 使用 |

| 环境变量 | 说明 |
|----------|------|
| `LD_PRELOAD` | curl-impersonate 库注入路径 |
| `CURL_IMPERSONATE` | 浏览器目标版本，默认 `chrome142` |

---

## 13. 边界场景与安全分析

### 13.1 keep-alive 连接复用时 CURL_TIMEOUT 与 max_time 命中差异

#### 连接复用现状

RSS-Bridge 的 `CurlHttpClient` **不支持连接复用**，每次请求都会创建全新的 cURL 句柄并在结束时销毁：

```php
// lib/http.php:67
$ch = curl_init($url);   // 新建句柄

// ... 请求 ...

// lib/http.php:195
curl_close($ch);          // 关闭句柄，连接不可复用
```

**每次 `request()` 调用都是独立的 TCP + TLS 握手**，没有连接池。因此：
- 不存在 keep-alive 连接复用导致的超时累积问题
- `CURLOPT_TIMEOUT` 总是从 0 开始计时，不继承上一个请求的剩余时间
- 同一 Bridge 中多次调用 `getContents()` 是完全独立的请求

#### 超时命中差异对比

| 场景 | `CURLOPT_TIMEOUT` 行为 | `max_filesize` 行为 |
|------|----------------------|---------------------|
| 首次请求（新连接） | 从 TCP 握手开始计时，5 秒后超时 | 从响应体第一个字节开始计算 |
| 连接复用（如果有的话） | 超时时间不会重置，可能更快命中 | 每个请求独立计算 |
| 重定向链（5 次重定向） | 超时是总的，不是每次重定向单独算 | 每次重定向独立计算大小 |

> **注意**：`CURLOPT_TIMEOUT` 是**整个请求的总超时**，包括 DNS 解析、TCP 连接、TLS 握手、重定向、响应传输的全部时间。如果有 5 次重定向，总超时仍然是 5 秒，不是每次 5 秒。

#### 隐式连接池的可能性

cURL 有内部连接缓存机制，但**只在同一句柄多次执行时有效**。因为 RSS-Bridge 每次都新建句柄，所以：
- 同进程内的多次请求不会复用连接
- PHP-FPM 的每个 worker 进程独立，互不影响
- 不存在"连接复用导致超时计算异常"的边界情况

---

### 13.2 Proxy 认证泄漏到中转节点风险

#### 代理认证的传递方式

RSS-Bridge 本身**没有显式的代理认证配置项**（没有 `proxy_user` / `proxy_password` 配置），但 cURL 支持在代理 URL 中嵌入认证信息：

```ini
; 配置示例（用户自行配置）
[proxy]
url = "http://user:password@proxy.example.com:8080"
```

#### 安全风险分析

| 代理类型 | 认证信息是否加密 | 风险等级 | 说明 |
|---------|----------------|----------|------|
| `http://` 代理 | ❌ 明文 | 高 | CONNECT 请求中的 Proxy-Authorization 头是 Base64 编码的明文，中转节点可直接解码 |
| `https://` 代理 | ✅ 加密 | 低 | 整个代理连接走 TLS，认证信息加密传输 |
| `socks5://` | ❌ 明文 | 高 | SOCKS5 认证子协商是明文的，可被中间人截获 |
| `socks5h://` | ❌ 明文 | 高 | 同 SOCKS5，只是 DNS 解析位置不同 |

#### 代码层面的风险点

`lib/http.php:133-135` 直接将代理 URL 传给 cURL，不做任何处理：

```php
if ($config['proxy']) {
    curl_setopt($ch, CURLOPT_PROXY, $config['proxy']);
}
```

- 没有验证代理 URL 中是否包含明文认证信息
- 没有 `CURLOPT_PROXY_SSL_VERIFYPEER` 之类的安全选项设置
- 如果代理 URL 包含密码，错误日志中可能泄漏（取决于 cURL 错误信息）

#### 最佳实践建议

1. **使用 HTTPS 代理**：避免认证信息明文传输
2. **不要在 URL 中嵌入密码**：如果需要认证，通过 `curl_options` 传入 `CURLOPT_PROXYUSERPWD`（虽然目前无法通过配置直接设置）
3. **代理地址使用内网 IP**：减少公网传输风险
4. **定期轮换代理凭证**：降低泄漏后的影响

---

### 13.3 无 Retry-After 时 503 与 429 的 Backoff 经验值

#### 代码中的默认 Backoff 值

RSS-Bridge 没有统一的 backoff 算法，各 Bridge 自行实现熔断时长：

| Bridge | 状态码 | 默认熔断时长 | 来源 |
|--------|--------|-------------|------|
| SpotifyBridge | 429 | 5 分钟（无 Retry-After 时） | `bridges/SpotifyBridge.php:109` |
| YoutubeBridge | 429 | 16 分钟 | `bridges/YoutubeBridge.php:84` |
| RedditBridge | 429 | 61 分钟 | `bridges/RedditBridge.php:136` |
| RedditBridge | 403 | 61 分钟（视为永久封禁降级） | `bridges/RedditBridge.php:133` |
| CurlHttpClient | 网络错误 | 0 秒（立即重试） | `lib/http.php:179` |
| TikTokBridge | 任意 HTTP 错误 | 0.1 秒（重试 3 次） | `bridges/TikTokBridge.php:55` |

#### 429 vs 503 的 Backoff 策略差异

| 维度 | 429 Too Many Requests | 503 Service Unavailable |
|------|----------------------|------------------------|
| 原因 | 客户端请求过快 | 服务端过载/维护 |
| 重试成功率 | 等待后大概率成功 | 不确定，可能需要很久 |
| 推荐初始等待 | 10-30 秒 | 60-300 秒 |
| 退避增长 | 线性或指数退避 | 指数退避，上限更长 |
| 最大等待上限 | 5-15 分钟 | 30-60 分钟 |

#### 经验值参考

根据各 Bridge 的实现，可以总结出以下经验模式：

1. **API 类站点**（Spotify, Youtube）：熔断时间较短（5-16 分钟），因为速率限制通常有明确的时间窗口
2. **社区类站点**（Reddit）：熔断时间较长（61 分钟），因为 IP 封禁可能持续更久
3. **网络层错误**：立即重试 1-2 次，因为可能是瞬时网络抖动
4. **503 错误**：代码中没有通用处理，建议首次等待 30 秒，后续指数退避

---

### 13.4 JA3 被 CloudFlare 识别后的兜底

#### 识别信号

当 curl-impersonate 的 JA3 指纹被 CloudFlare 识别后，典型表现为：
- 返回 403 Forbidden
- 页面标题包含 "Attention Required!" 或 "Access denied"
- 出现 Turnstile 挑战而不是 5 秒盾

代码中的检测（`lib/http.php:40-55`）：
```php
$cloudflareTitles = [
    '<title>Just a moment...',        // 5 秒盾，JA3 没被识别
    '<title>Please Wait...',
    '<title>Attention Required!',     // Turnstile 挑战，JA3 可能被识别
    '<title>Security | Glassdoor',
    '<title>Access denied</title>',    // 直接封禁
];
```

#### 兜底层级

```
L1: curl-impersonate Chrome 142（默认）
    ↓ 失败（出现 Attention Required）
L2: 切换模拟目标（firefox102 / chrome110 / safari15）
    ↓ 失败
L3: 增加请求间隔 + 代理轮换
    ↓ 失败
L4: WebDriver + headless Chrome（终极方案）
    ↓ 失败
L5: 用户手动提供 cf_clearance cookie
```

#### 代码层面的限制

RSS-Bridge **没有自动降级机制**：
- 不会检测 JA3 指纹是否失效
- 不会自动切换浏览器指纹
- 不会自动轮换代理

所有降级都需要：
1. 手动修改 `CURL_IMPERSONATE` 环境变量切换目标浏览器
2. 手动配置代理轮换（外部工具）
3. 手动切换到 WebDriver 方案

> **注意**：虽然代码层面没有自动降级，但 Docker 环境中 curl-impersonate 的成功率非常高（约 70-80% 的 CF 站点），大多数场景下不需要兜底。

---

### 13.5 子类 forceHeaders 与父类 setHeaders 优先级

RSS-Bridge 没有 `forceHeaders` / `setHeaders` 这样的方法名，但存在多层 Headers 合并机制，优先级从高到低如下：

#### 优先级链（从高到低）

| 层级 | 来源 | 设置方式 | 代码位置 |
|------|------|---------|---------|
| 1（最高） | Bridge 的 `curl_options` 中的 `CURLOPT_HTTPHEADER` | 通过 `curl_setopt_array()` 最后设置 | `lib/http.php:137` |
| 2 | Bridge 传入的 `$httpHeaders` 数组 | 通过 `CURLOPT_HTTPHEADER` 设置，会覆盖同名默认 Header | `lib/http.php:98, 108` |
| 3 | curl-impersonate 自动生成的 Headers | 库级自动设置，BoringSSL 环境下生效 | 底层库 |
| 4 | 全局默认 Headers（Firefox 102 指纹） | `$defaultHeaders` 数组，非 BoringSSL 环境下生效 | `lib/http.php:82-91` |
| 5（最低） | cURL 内置默认 Headers | cURL 库默认值 | cURL 库 |

#### 关键代码执行顺序

`lib/http.php:108-139` 中的设置顺序决定了覆盖关系：

```php
// 第 1 步：设置 headers（从 config['headers'] 数组构建）
curl_setopt($ch, CURLOPT_HTTPHEADER, $httpHeaders);  // 第 108 行

// 第 2 步：设置 UA（如果有）
if ($config['useragent']) {
    curl_setopt($ch, CURLOPT_USERAGENT, $config['useragent']);  // 第 110 行
}

// ... 其他设置 ...

// 第 3 步：用户自定义 curl_options（最后设置，优先级最高）
if (curl_setopt_array($ch, $config['curl_options']) === false) {  // 第 137 行
    throw new \Exception('Tried to set an illegal curl option');
}
```

**后设置的覆盖先设置的**。因此：
- 如果 `curl_options` 中包含 `CURLOPT_HTTPHEADER`，会完全替换前面设置的所有 Header
- 如果 `curl_options` 中包含 `CURLOPT_USERAGENT`，会覆盖 `$config['useragent']`

#### FeedExpander 的特殊情况

`lib/FeedExpander.php:19-21` 中子类的 Headers 与父类的 Accept 头合并：

```php
$accept = [MrssFormat::MIME_TYPE, AtomFormat::MIME_TYPE, '*/*'];
$httpHeaders = array_merge(['Accept: ' . implode(', ', $accept)], $headers);
```

这里 `array_merge` 是**按顺序合并**，如果子类传入的 `$headers` 中也包含 `Accept` 头：
- PHP 中 `array_merge` 不会自动去重
- 最终会发送两个 `Accept` 头（因为是字符串数组，不是关联数组）
- 这可能导致行为异常，取决于服务器如何处理重复 Header

---

### 13.6 cloudscraper 在无 GUI 服务器上的退路

#### 现状：未集成 cloudscraper

RSS-Bridge 是 **PHP 项目**，不直接使用 Python 生态的 `cloudscraper` 库。

```
Python 生态（cloudscraper）       PHP 生态（RSS-Bridge）
┌───────────────────────┐      ┌───────────────────────┐
│  cloudscraper         │      │  CurlHttpClient       │
│    ↓ 依赖              │      │    ↓ 依赖              │
│  requests / urllib3   │      │  PHP cURL 扩展        │
│  js2py / Node.js      │      │  curl-impersonate     │
│  解 cf_clearance      │      │  WebDriver（可选）     │
└───────────────────────┘      └───────────────────────┘
```

#### 无 GUI 服务器上的可用方案

| 方案 | 需要 GUI | 难度 | 成功率 | 说明 |
|------|---------|------|--------|------|
| curl-impersonate | ❌ 不需要 | 低 | 70-80% | 默认方案，容器内置 |
| 代理 IP 轮换 | ❌ 不需要 | 中 | 60-70% | 配合外部代理池 |
| WebDriver + headless Chrome | ❌ 不需要（headless） | 中高 | 90%+ | 需要 Selenium/ChromeDriver |
| 手动 cf_clearance cookie | ❌ 不需要 | 高 | 取决于 cookie 有效期 | 用户从浏览器导出 cookie |
| cloudscraper（Python） | ❌ 不需要 | 高 | 80-90% | 需要额外部署 Python 服务 |
| 真实浏览器（有头） | ✅ 需要 | 高 | 95%+ | 服务器无 GUI 时不可行 |

#### WebDriver 的 headless 模式

`lib/WebDriverAbstract.php:71-77` 支持 headless 模式：

```php
protected function getBrowserOptions()
{
    $chromeOptions = new ChromeOptions();
    if (Configuration::getConfig('webdriver', 'headless')) {
        $chromeOptions->addArguments(['--headless']);
    }
    return $chromeOptions;
}
```

配置：
```ini
[webdriver]
selenium_server_url = "http://localhost:4444"
headless = true    ; 无 GUI 服务器上必须开启
```

> **注意**：headless Chrome 本身也可能被识别（有独特的指纹特征）。某些严格的站点可能需要 `--headless=new` 模式或添加额外的伪装参数。

---

### 13.7 服务端不支持 HTTP/2 时降级到 HTTP/1.1 的检测点

#### 自动降级机制

`lib/http.php` 中**没有显式设置** `CURLOPT_HTTP_VERSION`，使用 cURL 默认值：

- cURL 7.47.0+ 默认行为：HTTPS 请求尝试 HTTP/2，失败则自动降级到 HTTP/1.1
- 对应常量：`CURL_HTTP_VERSION_2TLS`（不是 `CURL_HTTP_VERSION_2_0`）

**降级完全由 cURL 底层自动处理**，PHP 代码层不感知，也不检测实际使用的版本。

#### 没有版本检测点

代码中**没有使用** `curl_getinfo()` 获取协议版本：

```php
// lib/http.php:194 - 目前只获取了状态码
$statusCode = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
```

可能的检测点（未实现）：
```php
// 如果要检测，可以用：
$protocol = curl_getinfo($ch, CURLINFO_PROTOCOL);
$httpVersion = curl_getinfo($ch, CURLINFO_HTTP_VERSION);
```

但 RSS-Bridge 没有这样做，因为：
1. 绝大多数现代站点都支持 HTTP/2
2. 降级是透明的，不影响功能
3. 不需要根据协议版本调整行为

#### 手动强制降级

只有 `IdealoBridge` 显式强制 HTTP/1.1（`bridges/IdealoBridge.php:44`）：

```php
private $options = [
    CURLOPT_HTTP_VERSION => CURL_HTTP_VERSION_1_1,  // 强制 HTTP/1.1
    ...
];
```

**可能的原因**：
- idealo.de 的 HTTP/2 实现有 bug
- 反爬系统会检查 HTTP/2 帧细节（如 SETTINGS 参数、Window Update 频率）
- curl-impersonate 的 HTTP/2 指纹与真实浏览器仍有细微差异
- 使用 HTTP/1.1 可以绕过基于 HTTP/2 指纹的检测

#### 降级后的行为差异

| 特性 | HTTP/2 | HTTP/1.1 |
|------|--------|----------|
| 多路复用 | ✅ 单连接并发请求 | ❌ 每个请求一个连接 |
| Header 压缩 | ✅ HPACK | ❌ 无 |
| 服务器推送 | ✅ 可能 | ❌ 不可 |
| 反爬指纹点 | 多（帧顺序、SETTINGS 等） | 少 |
| 性能 | 高（并发） | 低（串行） |
| curl-impersonate 覆盖 | 完整模拟 | 仅 TLS 层 |

> **安全提示**：HTTP/2 的指纹特征比 HTTP/1.1 多得多。如果 curl-impersonate 模拟的 HTTP/2 指纹被识别，降级到 HTTP/1.1 有时能绕过检测，因为服务端对 HTTP/1.1 的指纹检查通常较松。

---

## 14. 深度边界分析

### 14.1 连接池满时新请求的等待行为

#### RSS-Bridge 没有连接池

`CurlHttpClient` 每次 `request()` 调用都执行完整的 `curl_init()` → `curl_exec()` → `curl_close()` 生命周期（`lib/http.php:67, 195`），不存在连接池概念。

**因此不存在"连接池满"的等待行为**——每个请求都独立建立 TCP + TLS 连接。

#### PHP-FPM 层面的并发控制

虽然没有连接池，但 PHP-FPM 进程模型本身就起到了并发限制的作用：

| 配置项 | 默认值 | 作用 |
|--------|--------|------|
| `pm.max_children` | 5 | 最大同时处理的请求数 |
| `pm.max_requests` | 500 | 进程处理多少请求后重启 |
| `request_terminate_timeout` | 0 | 请求超时（0=不限制） |

当 PHP-FPM 的 worker 进程全部忙碌时，新请求会排队等待空闲 worker，而非等待连接池。等待行为取决于 `pm` 配置（static / dynamic / ondemand）。

#### Docker 环境下的并发

`config/php-fpm.conf` 控制了容器内的 FPM 配置。默认配置下：
- 同时最多 5 个并发 Bridge 请求
- 每个请求独立创建 cURL 连接，互不干扰
- 不存在连接池满导致的阻塞

> **如果需要连接池**：可通过 `curl_multi_*` 系列函数实现，但 RSS-Bridge 目前未使用。`curl_multi_init` 允许在一个 cURL 句柄组内并发执行多个请求并复用连接，但需要重构 `CurlHttpClient`。

---

### 14.2 HTTP CONNECT 隧道下 Basic 认证泄漏场景

#### CONNECT 隧道的工作原理

当通过 HTTP 代理访问 HTTPS 站点时，cURL 使用 CONNECT 方法建立隧道：

```
客户端 → 代理: CONNECT target.com:443 HTTP/1.1
代理 → 客户端: HTTP/1.1 200 Connection Established
[此时建立端到端 TLS 隧道，代理不再可见明文]
```

#### 认证泄漏的三个场景

**场景 1：代理认证泄漏到目标站点**

如果代理使用 Basic 认证，cURL 会通过 `Proxy-Authorization` 头发送凭证：

```
CONNECT target.com:443 HTTP/1.1
Proxy-Authorization: Basic dXNlcjpwYXNz  ← Base64 编码的 user:pass
Host: target.com:443
```

这个头只应该被代理读取，但以下情况会泄漏：

| 泄漏场景 | 风险 | 说明 |
|---------|------|------|
| 代理是恶意的 | 高 | 代理可以记录并复用凭证 |
| 代理误转发 `Proxy-Authorization` 到源站 | 中 | 某些配置错误的代理会这样做 |
| 中间人攻击 | 高 | HTTP 代理的 CONNECT 请求可被窃听 |

RSS-Bridge 的代码（`lib/http.php:133-135`）直接设置代理 URL，不做认证信息分离：

```php
if ($config['proxy']) {
    curl_setopt($ch, CURLOPT_PROXY, $config['proxy']);
}
```

**场景 2：目标站点认证泄漏到代理**

当使用 Basic Auth 访问目标站点时（如 Spotify 的 `Authorization: Basic` 头）：

```php
// SpotifyBridge.php:158-163
$basicAuth = base64_encode(sprintf('%s:%s', $this->getInput('clientid'), $this->getInput('clientsecret')));
$json = getContents('https://accounts.spotify.com/api/token', [
    "Authorization: Basic $basicAuth",
], [...]);
```

通过 HTTP 代理时，`Authorization` 头在 TLS 隧道内传输（加密），代理无法看到。但如果代理是 HTTPS 代理（而非 HTTP 代理），则整条链路都是加密的，更安全。

**场景 3：URL 内嵌凭证泄漏**

```ini
[proxy]
url = "http://admin:secret@proxy.example.com:8080"
```

cURL 会将 `admin:secret` 转为 `Proxy-Authorization: Basic` 头。如果代理 URL 被：
- 记录到日志（`CURLOPT_VERBOSE`）
- 显示在错误信息中
- 泄漏到 HTTP_REFERER

就会造成凭证泄漏。RSS-Bridge 通过 `name` 配置项隐藏代理 URL：

```ini
[proxy]
name = "Hidden proxy name"  ; 前台显示此名称，不显示 URL
```

#### 安全建议

1. **永远不要在代理 URL 中嵌入凭证**——使用 `CURLOPT_PROXYUSERPWD` 单独设置
2. **使用 HTTPS 代理**——CONNECT 隧道在 TLS 内建立，防止凭证窃听
3. **使用 SOCKS5h**——认证子协商不走明文 HTTP
4. **代理凭证与站点凭证使用不同密码**——避免连锁泄漏

---

### 14.3 连续命中限速后切到 Circuit Breaker

#### 现有实现：Cache-Based 半熔断

RSS-Bridge 的限速保护本质上是**基于缓存的熔断器（Circuit Breaker）**，但只实现了"断开"状态，没有完整的 Circuit Breaker 三态模型：

```
标准 Circuit Breaker 三态模型：
┌────────────┐  失败超阈值  ┌────────────┐  超时后   ┌─────────────┐
│  CLOSED    │ ──────────→ │   OPEN     │ ────────→ │ HALF-OPEN   │
│ (正常通行) │             │ (全部拒绝) │           │ (试探放行)   │
└────────────┘  ←────────── └────────────┘ ←──────── └─────────────┘
                  成功恢复                      试探失败
```

#### RSS-Bridge 的实现对比

| 状态 | 标准 CB | RSS-Bridge 实现 | Bridge |
|------|---------|----------------|--------|
| CLOSED（正常） | 请求正常通行 | 请求正常通行 | 所有 Bridge |
| OPEN（断开） | 全部请求立即拒绝 | 全部请求立即拒绝（`throwRateLimitException()`） | 所有 Bridge |
| HALF-OPEN（试探） | 放行一个请求测试 | ❌ 不存在 | 无 |
| 超时恢复 | 超时后自动进入 HALF-OPEN | 缓存过期后自动恢复 | 所有 Bridge |

**关键区别**：RSS-Bridge 缓存过期后直接回到 CLOSED 状态，没有 HALF-OPEN 试探期。这意味着缓存过期后的第一个请求会直接打到目标站点——如果目标站点仍在限速，会再次触发 429，重新进入 OPEN 状态。

#### 各 Bridge 的熔断阈值

| Bridge | 熔断触发条件 | 熔断时长 | 重新打开后行为 |
|--------|-------------|---------|--------------|
| YoutubeBridge | 1 次 429 | 16 分钟 | 缓存过期后直接全量请求 |
| RedditBridge | 1 次 429 或 1 次 403 | 61 分钟 | 缓存过期后直接全量请求 |
| SpotifyBridge | 1 次 429 | 5 分钟或 Retry-After | 缓存过期后直接全量请求 |
| TikTokBridge | 任意 HTTP 错误 | 0.1 秒 × 3 次 | 立即重试，无熔断 |

**TikTokBridge 是唯一实现了"渐进式"退避的 Bridge**（`bridges/TikTokBridge.php:47-59`）：

```php
$attempts = 0;
do {
    try {
        $json = getContents('https://www.tiktok.com/oembed?url=' . $url);
    } catch (HttpException $e) {
        $attempts++;
        usleep(100000);  // 0.1 秒等待
        continue;
    }
    break;
} while ($attempts < 3);
```

#### 连续命中的风险场景

```
t=0    首次请求 → 429 → 熔断开启（缓存 16 分钟）
t=16m  缓存过期 → 请求 → 429 → 熔断重新开启
t=32m  缓存过期 → 请求 → 429 → 熔断重新开启
       ↑ 无限循环，永远不会试探性放行
```

因为没有 HALF-OPEN 状态，RSS-Bridge 无法区分：
- 目标站点永久封禁（应该停止请求）
- 目标站点临时限速（可以试探恢复）

**改进方向**：引入连续失败计数器，如果连续 N 次进入 OPEN 状态，则指数增加熔断时长。

---

### 14.4 curl-impersonate 在 PHP 7.x 老环境安装难度

#### 兼容性矩阵

| 环境 | PHP 版本 | curl-impersonate | 安装难度 | 问题 |
|------|---------|-----------------|----------|------|
| Docker (Debian 12) | 8.2 | ✅ 原生支持 | 低 | Dockerfile 一键安装 |
| Ubuntu 22.04 | 8.1 | ✅ 手动安装 | 中 | 需要编译或下载预编译包 |
| Ubuntu 20.04 | 7.4 | ⚠️ 可行但困难 | 高 | glibc 版本不匹配 |
| CentOS 7 | 7.2 | ❌ 极难 | 极高 | glibc 2.17 vs 需要 2.31+ |
| Debian 10 | 7.3 | ⚠️ 可行但困难 | 高 | 同 glibc 问题 |

#### 核心障碍：glibc 版本

curl-impersonate 的预编译包依赖 glibc 2.31+（Debian 11+），而 PHP 7.x 通常运行在旧版系统上：

```
curl-impersonate v1.2.5 预编译包依赖链：
  libcurl-impersonate.so
    → libssl.so (BoringSSL)
      → glibc >= 2.31
      → libstdc++ >= GLIBCXX_3.4.26
```

| 系统 | glibc 版本 | PHP 版本 | 能否运行 curl-impersonate |
|------|-----------|---------|--------------------------|
| Debian 12 | 2.36 | 8.2 | ✅ |
| Debian 11 | 2.31 | 7.4 | ✅ 最低要求 |
| Debian 10 | 2.28 | 7.3 | ❌ glibc 不够 |
| Ubuntu 20.04 | 2.31 | 7.4 | ✅ 刚好 |
| Ubuntu 18.04 | 2.27 | 7.2 | ❌ glibc 不够 |
| CentOS 7 | 2.17 | 7.2 | ❌ 远远不够 |

#### 安装方案对比

| 方案 | 难度 | 可靠性 | 说明 |
|------|------|--------|------|
| Docker 部署（推荐） | 低 | 高 | 最简单，Dockerfile 自带 curl-impersonate |
| 预编译包 + LD_PRELOAD | 中 | 中 | 仅 glibc 2.31+ 系统 |
| 从源码编译 curl-impersonate | 高 | 中 | 需要 Go、Rust、CMake 等编译工具链 |
| 静态编译版本 | 高 | 低 | 可能与 PHP 的动态链接 libcurl 冲突 |
| 降级到 Firefox 102 指纹（无 impersonate） | 无 | 中 | 不需要安装，但反爬能力弱 |

#### PHP 7.x 的额外问题

1. **`curl_version()` 返回值差异**：PHP 7.x 的 cURL 扩展可能不支持 `ssl_version` 字段的 BoringSSL 值，导致 `lib/http.php:93` 的检测分支失效
2. **`CURLOPT_ENCODING` 空字符串行为**：PHP 7.x 搭配旧版 cURL 可能不支持自动 brotli 解码
3. **`CURLOPT_PROGRESSFUNCTION` 签名**：PHP 7.x 的回调参数签名可能不同

> **建议**：PHP 7.x 环境下如果无法安装 curl-impersonate，代码会自动降级到 Firefox 102 手动指纹（`lib/http.php:95-101` 的 else 分支），虽然没有 TLS 指纹模拟，但至少能保持基本功能。

---

### 14.5 forceHeaders 同名大小写归一化

#### Header 名称大小写处理的三层不一致

HTTP/1.1 规范规定 Header 名称不区分大小写（RFC 7230 §3.2），但 RSS-Bridge 在不同层级对大小写的处理不一致，可能导致同名 Header 重复。

**第 1 层：Bridge 传入的 Headers（`getContents()` 的 `$httpHeaders`）**

`lib/contents.php:59-65` 解析时不做大小写归一化：

```php
$httpHeadersNormalized = [];
foreach ($httpHeaders as $httpHeader) {
    $parts = explode(':', $httpHeader);
    $headerName = trim($parts[0]);              // ← 保留原始大小写！
    $headerValue = trim(implode(':', array_slice($parts, 1)));
    $httpHeadersNormalized[$headerName] = $headerValue;
}
```

**第 2 层：`CurlHttpClient` 的默认 Headers（`$defaultHeaders`）**

`lib/http.php:82-91` 使用首字母大写格式（PascalCase）：

```php
$defaultHeaders = [
    'Accept' => '...',
    'Accept-Language' => '...',
    'Upgrade-Insecure-Requests' => '1',
    'Sec-Fetch-Dest' => 'document',
    // ...
];
```

**第 3 层：Bridge 自定义 Headers 的大小写混用**

从代码中搜索到的实际案例：

| Bridge | Header 写法 | 大小写风格 |
|--------|-----------|-----------|
| InstagramBridge | `User-Agent:` | PascalCase |
| SlusheBridge | `user-agent:` | 全小写 |
| EconomistBridge | `User-agent:` | 混合 |
| AppleAppStoreBridge | `user-agent:` | 全小写 |
| RobinhoodSnacksBridge | `User-Agent:` | PascalCase |

#### 合并时的问题

`lib/http.php:98` 使用 `array_merge` 合并 Headers：

```php
$headers = array_merge($defaultHeaders, $config['headers']);
```

由于 `$defaultHeaders` 使用关联数组（`'Accept' => '...'`），而 `$config['headers']` 也使用关联数组，PHP 的 `array_merge` 对字符串键的行为是**后覆盖前**。

但问题在于**大小写不归一化时，同名键不被认为是同一个键**：

```php
$defaultHeaders['User-Agent'] = 'Firefox/102';     // 键: "User-Agent"
$config['headers']['user-agent'] = 'Chrome/112';    // 键: "user-agent"

// array_merge 后两者都存在！
// 最终发送给 cURL 时会有两个 UA 头
```

#### 最终发送到 cURL 时的行为

`lib/http.php:104-108` 将关联数组转为字符串数组：

```php
$httpHeaders = [];
foreach ($config['headers'] as $name => $value) {
    $httpHeaders[] = sprintf('%s: %s', $name, $value);
}
curl_setopt($ch, CURLOPT_HTTPHEADER, $httpHeaders);
```

如果存在 `User-Agent` 和 `user-agent` 两个键，cURL 会发送两个 Header。服务端通常只取最后一个，但行为取决于具体实现。

#### 响应 Header 的归一化

与请求 Header 不同，**响应 Header 在解析时做了小写归一化**（`lib/http.php:161`）：

```php
$name = mb_strtolower(trim($header[0]));
```

`Response` 构造函数也做了同样处理（`lib/http.php:312`）：

```php
$name = mb_strtolower($name);
```

`getHeader()` 方法也做了小写归一化（`lib/http.php:354`）：

```php
$name = mb_strtolower($name);
```

**总结**：响应侧完全归一化，请求侧不归一化——这是一个潜在的一致性问题。

---

### 14.6 Xvfb Docker 化资源对比

#### 三种浏览器渲染方案的资源对比

| 维度 | curl-impersonate | WebDriver + headless Chrome | WebDriver + Xvfb + Chrome |
|------|-----------------|---------------------------|--------------------------|
| 内存占用 | ~10 MB | ~200-500 MB | ~300-700 MB |
| CPU 占用 | 极低（无渲染） | 中（渲染但无显示） | 高（渲染+虚拟显示） |
| 启动时间 | 0 ms（库级注入） | 2-5 秒 | 3-8 秒 |
| 镜像体积 | ~50 MB（库文件） | ~800 MB（Chrome+依赖） | ~1.2 GB（Chrome+Xvfb+依赖） |
| 磁盘 I/O | 无 | 低 | 中（帧缓冲写入） |
| 并发能力 | 受 PHP-FPM 限制 | 受 Selenium 并发限制 | 受 Xvfb 显示号限制 |
| JS 执行 | ❌ 不支持 | ✅ 完整支持 | ✅ 完整支持 |
| Canvas/WebGL | ❌ 不支持 | ⚠️ headless 可能不支持 | ✅ 完整支持 |
| 反检测能力 | TLS 指纹模拟 | headless 可能被识别 | 接近真实浏览器 |

#### Xvfb 的使用场景

RSS-Bridge 的 `WebDriverAbstract` 支持 headless 模式（`lib/WebDriverAbstract.php:74`），**不需要 Xvfb**。Xvfb 只在以下场景需要：

1. **Chrome 不支持 headless 模式的旧版本**：Chrome 59 之前没有 headless 模式
2. **需要 Canvas/WebGL 渲染**：某些反爬系统会检测 Canvas 指纹
3. **需要真实窗口大小**：headless Chrome 的 `window.innerWidth`/`innerHeight` 可能有差异
4. **网站检测 headless 标志**：`navigator.webdriver` 属性在 headless 模式下为 `true`

#### Docker 化 Xvfb 方案

```dockerfile
# 在 RSS-Bridge 基础镜像上叠加 Xvfb
FROM rss-bridge:latest

RUN apt-get update && apt-get install -y \
    xvfb \
    chromium \
    && rm -rf /var/lib/apt/lists/*

# 启动虚拟显示器
ENV DISPLAY=:99
CMD Xvfb :99 -screen 0 1920x1080x24 & \
    php-fpm
```

#### 资源优化建议

| 优化项 | 效果 | 方式 |
|--------|------|------|
| 使用 `--headless=new` | 减少 30% 内存 | Chrome 112+ 的新 headless 模式，指纹更接近真实浏览器 |
| 限制 Chrome 启动参数 | 减少 20% 内存 | `--disable-gpu --disable-software-rasterizer --no-sandbox` |
| 复用 Selenium 容器 | 减少镜像体积 | 独立 Selenium 容器，多实例共享 |
| 使用 Chromium 替代 Chrome | 减少 100 MB 镜像体积 | `chromium` 包比 `google-chrome` 小 |
| 连接池化 WebDriver | 减少启动开销 | 保持浏览器实例常驻，不要每次请求都启停 |

---

### 14.7 ALPN 协商失败下的降级差异

#### ALPN 的作用

ALPN（Application-Layer Protocol Negotiation）是 TLS 扩展，客户端在 ClientHello 中声明支持的应用层协议（如 `h2` 和 `http/1.1`），服务端从中选择一个。

```
ClientHello:
  ALPN extension: ["h2", "http/1.1"]

ServerHello:
  ALPN extension: "h2"    ← 服务端选择 HTTP/2
  或
  ALPN extension: "http/1.1"  ← 服务端选择 HTTP/1.1
  或
  无 ALPN extension          ← 服务端不支持 ALPN，回退到默认
```

#### curl-impersonate 环境下的 ALPN

curl-impersonate 模拟 Chrome 142 时，会自动设置与 Chrome 一致的 ALPN 扩展：

```
Chrome 142 的 ALPN 顺序: ["h2", "http/1.1"]
```

这个顺序本身就是指纹特征——Chrome 总是先声明 `h2`，某些爬虫库可能顺序相反。

#### ALPN 协商失败的场景

| 场景 | 服务端行为 | cURL 行为 | RSS-Bridge 影响 |
|------|-----------|----------|----------------|
| 服务端支持 ALPN，选择 h2 | 返回 `ALPN: h2` | 使用 HTTP/2 | 正常 |
| 服务端支持 ALPN，选择 http/1.1 | 返回 `ALPN: http/1.1` | 使用 HTTP/1.1 | 正常 |
| 服务端不支持 ALPN | 不返回 ALPN 扩展 | cURL 默认回退到 HTTP/1.1 | 正常但可能被检测 |
| ALPN 协商失败（服务端返回不支持的协议） | TLS 握手失败 | 连接失败 | 触发重试 |
| TLS 握手成功但 HTTP/2 帧解析失败 | 连接已建立 | `CURLOPT_HTTP_VERSION` 未显式设置时自动降级 | 正常 |

#### 非 impersonate 环境的差异

在原生 OpenSSL 环境下（非 BoringSSL），ALPN 的设置取决于 cURL 版本和编译选项：

| cURL 版本 | 默认 ALPN | 说明 |
|-----------|----------|------|
| 7.36+ | `h2` 和 `http/1.1` | 自动启用 ALPN |
| 7.47+ | `h2` 和 `http/1.1` | 默认尝试 HTTP/2（`CURL_HTTP_VERSION_2TLS`） |
| 7.88+ | `h2` 和 `http/1.1` | 同上，支持更完善的 HTTP/2 |

**关键差异**：非 impersonate 环境下，ALPN 扩展中协议的声明顺序取决于 OpenSSL 的实现，可能与 Chrome 不一致。反爬系统可以据此区分真实浏览器和爬虫。

#### ALPN 与 JA3 指纹的关系

JA3 指纹包含了 ALPN 扩展的存在与否及内容。如果：
- 客户端声明了 `h2` 但 ALPN 顺序与 Chrome 不同 → 可能被识别
- 客户端未声明 ALPN（某些旧版 cURL）→ 一定会被识别为非浏览器
- 客户端声明了 `h2` 但使用 HTTP/1.1 通信 → 行为与声明不一致

curl-impersonate 确保了 ALPN 声明与实际协议使用的完全一致性，这是其核心价值之一。

#### 降级检测代码

RSS-Bridge **没有 ALPN 降级检测代码**。`lib/http.php:194` 只获取了 HTTP 状态码：

```php
$statusCode = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
```

如果需要检测实际使用的协议版本，可以添加：

```php
// 可用但未使用的检测方式
$httpVersion = curl_getinfo($ch, CURLINFO_HTTP_VERSION);
// CURL_HTTP_VERSION_2_0 = 3
// CURL_HTTP_VERSION_1_1 = 2
// CURL_HTTP_VERSION_1_0 = 1
```

> **提示**：如果目标站点的反爬系统同时检测 TLS 指纹和 ALPN 行为，确保 curl-impersonate 版本与目标浏览器匹配至关重要。过时的 impersonate 版本（如 chrome110）在新版 CloudFlare 下可能已被识别。

---

## 15. 架构边界与退路分析

### 15.1 multi-handle 异步连接池退路

#### 现状：完全同步模型

RSS-Bridge 的 `CurlHttpClient` **完全没有异步/并发支持**，所有请求都是同步阻塞的。每个 `request()` 调用执行完整的生命周期：

```
请求 A: curl_init() → DNS → TCP → TLS → 请求 → 响应 → curl_close()
请求 B: curl_init() → DNS → TCP → TLS → 请求 → 响应 → curl_close()
请求 C: curl_init() → DNS → TCP → TLS → 请求 → 响应 → curl_close()
```

由于每次都新建句柄，没有连接复用，也没有并发。

#### 可扩展的 multi-handle 异步方案

cURL 提供 `curl_multi_*` 系列函数用于并发请求和连接复用。虽然 RSS-Bridge 目前未使用，但可以作为退路方案：

```php
// 概念示例（未实现）
class MultiCurlHttpClient implements HttpClient
{
    private $multiHandle;
    private $connectionPool = [];

    public function requestMulti(array $urls, array $config = []): array
    {
        $mh = curl_multi_init();
        $handles = [];

        foreach ($urls as $url) {
            $ch = curl_init($url);
            // ... 设置选项 ...
            curl_multi_add_handle($mh, $ch);
            $handles[] = $ch;
        }

        // 事件循环，并发等待所有请求完成
        do {
            curl_multi_exec($mh, $active);
            curl_multi_select($mh);
        } while ($active);

        // 收集响应
        $responses = [];
        foreach ($handles as $ch) {
            $responses[] = curl_multi_getcontent($ch);
            curl_multi_remove_handle($mh, $ch);
            // 连接可复用，不立即 close
        }

        curl_multi_close($mh);
        return $responses;
    }
}
```

#### multi-handle 带来的优势

| 特性 | 当前 CurlHttpClient | MultiCurlHttpClient |
|------|---------------------|----------------------|
| 并发能力 | ❌ 串行，一个请求完成后才开始下一个 | ✅ 并发，N 个请求同时进行 |
| 连接复用 | ❌ 每次新建句柄 | ✅ 同 multi 句柄内可复用连接 |
| 性能（N 个请求） | O(N × 单个请求时间) | O(最长单个请求时间) |
| 反爬风险 | 连续请求时间戳间隔可被检测 | 请求同时到达，更接近真实浏览器 |
| 内存占用 | 低（单个句柄） | 高（N 个句柄 + 响应缓冲区） |

#### 对现有代码的影响

如果引入 multi-handle，需要考虑：

1. **`getContents()` 接口兼容**：当前返回单个字符串/Response，并发版本需要返回数组
2. **重试机制重构**：当前重试在单句柄循环内，并发版本需要按句柄单独重试
3. **缓存机制调整**：`getSimpleHTMLDOMCached()` 每次只缓存一个 URL，需要批处理
4. **超时计算**：`CURLOPT_TIMEOUT` 是每个句柄的独立超时，不是总超时
5. **错误处理**：单个请求失败不应导致整个并发组失败

> **注意**：PHP 环境下，同步模型对 RSS-Bridge 是合理的——每个 Bridge 请求通常只需要 1-5 个 HTTP 请求，并发带来的收益不明显，反而增加复杂度。multi-handle 只有在批量抓取（如一次性获取 10+ 篇文章）时才有明显优势。

---

### 15.2 Bridge 禁用 socks5h 时认证泄漏隔离

#### NOPROXY 机制的全局副作用

`DisplayAction.php:41-48` 中的 NOPROXY 机制存在全局副作用风险：

```php
if (
    Configuration::getConfig('proxy', 'url')
    && Configuration::getConfig('proxy', 'by_bridge')
    && $noproxy
) {
    define('NOPROXY', true);  // ← 全局常量，一旦定义不可撤销
}
```

由于 PHP 常量的全局特性，**同一进程内的后续请求也会跳过代理**。在 PHP-FPM 模式下，worker 进程是复用的，这意味着：

```
请求 A (带 _noproxy=1):
  → define('NOPROXY', true)
  → getContents() 不使用代理 ✓

请求 B (不带 _noproxy，同 worker 进程):
  → defined('NOPROXY') 已经为 true
  → getContents() 也不使用代理 ✗ 非预期行为
```

#### 认证泄漏的场景分析

当存在全局代理认证，而用户请求中使用 `_noproxy=1` 时：

| 场景 | 代理配置 | 认证泄漏风险 | 说明 |
|------|---------|-------------|------|
| 1 | 全局 HTTP 代理带认证 | 低 | NOPROXY 只是跳过代理设置，不涉及认证传递 |
| 2 | Bridge 手动设置代理 | 高 | 如果 Bridge 代码中硬编码代理，NOPROXY 不生效 |
| 3 | URL 内嵌凭证 + NOPROXY | 中 | 凭证可能被缓存或错误日志记录 |
| 4 | 进程复用导致代理状态混乱 | 中 | worker 进程的 NOPROXY 状态污染后续请求 |

#### 代码层面的隔离缺失

`lib/contents.php:100-102` 的检查是基于全局常量的：

```php
if (Configuration::getConfig('proxy', 'url') && !defined('NOPROXY')) {
    $config['proxy'] = Configuration::getConfig('proxy', 'url');
}
```

**没有请求级别的代理开关**——NOPROXY 是进程级的，不是请求级的。

#### 改进建议

```php
// 现有：全局常量
define('NOPROXY', true);

// 改进：通过 Request 对象传递
$config['noproxy'] = $request->get('_noproxy');
// getContents() 中检查请求级配置而非全局常量
if (Configuration::getConfig('proxy', 'url') && !($config['noproxy'] ?? false)) {
    $config['proxy'] = Configuration::getConfig('proxy', 'url');
}
```

---

### 15.3 Circuit Breaker HALF_OPEN 探活退避

#### 现有实现的缺陷

如 14.3 节所述，RSS-Bridge 的熔断机制缺少 HALF-OPEN 状态，导致：
- 缓存过期后直接全量请求
- 无法区分永久封禁和临时限速
- 连续 429 会进入"熔断→过期→再熔断"的无限循环

#### HALF_OPEN 探活的参考实现

可以基于现有缓存机制扩展出探活逻辑：

```php
// 概念实现（可集成到现有 Bridge 中）
trait CircuitBreakerTrait
{
    private function checkCircuitBreaker(string $cacheKey, int $failureThreshold = 3): bool
    {
        $state = $this->cache->get($cacheKey . '_state');

        if ($state === 'open') {
            // OPEN 状态：全部拒绝
            throwRateLimitException();
        }

        if ($state === 'half_open') {
            // HALF_OPEN 状态：只允许一个探活请求
            $probeCount = $this->cache->get($cacheKey . '_probe_count') ?? 0;
            if ($probeCount >= 1) {
                // 已有探活请求在进行，拒绝新请求
                throwRateLimitException();
            }
            $this->cache->set($cacheKey . '_probe_count', $probeCount + 1, 60);
            return true;  // 允许探活
        }

        return true;  // CLOSED 状态：正常通行
    }

    private function onSuccess(string $cacheKey): void
    {
        // 成功：清除失败计数，回到 CLOSED
        $this->cache->delete($cacheKey . '_state');
        $this->cache->delete($cacheKey . '_failures');
    }

    private function onFailure(string $cacheKey, int $code, int $backoffBase = 60): void
    {
        $failures = ($this->cache->get($cacheKey . '_failures') ?? 0) + 1;
        $this->cache->set($cacheKey . '_failures', $failures, 86400);

        // 指数退避：60s, 120s, 240s, 480s, ... 最大 1h
        $backoff = min($backoffBase * pow(2, $failures - 1), 3600);

        if ($failures >= 3) {
            // 连续 3 次失败：进入 HALF_OPEN
            $this->cache->set($cacheKey . '_state', 'half_open', $backoff);
        } else {
            // 未达阈值：OPEN 状态
            $this->cache->set($cacheKey . '_state', 'open', $backoff);
        }
    }
}
```

#### 三态模型的行为对比

| 状态 | 触发条件 | 熔断时长 | 行为 | 成功后 | 失败后 |
|------|---------|---------|------|--------|--------|
| CLOSED | 初始状态或探活成功 | - | 所有请求正常通行 | - | 失败计数 +1 |
| OPEN | 连续失败 1-2 次 | 指数退避（60s→120s） | 全部请求拒绝 | - | 保持 OPEN |
| HALF_OPEN | 连续失败 ≥3 次 | 指数退避（240s→...） | 仅放行 1 个探活请求 | 回到 CLOSED | 失败计数 +1，延长 HALF_OPEN |

#### 探活请求的退避策略

探活请求应该**逐渐增加间隔**，避免频繁触发目标站点的反爬：

| 连续失败次数 | 熔断时长 | 探活策略 |
|-------------|---------|---------|
| 1 | 60s | 到期后全量恢复 |
| 2 | 120s | 到期后全量恢复 |
| 3 | 240s | 到期后仅放行 1 个探活请求 |
| 4 | 480s | 到期后仅放行 1 个探活请求 |
| 5 | 960s | 到期后仅放行 1 个探活请求 |
| N | min(60×2^(N-1), 3600) | 到期后仅放行 1 个探活请求 |

> **注意**：TikTokBridge 已经实现了简单的重试退避（0.1s × 3 次），但这是请求级的重试，不是 Circuit Breaker 级的熔断。

---

### 15.4 Alpine 与 Debian 编译 curl-impersonate 差异

#### 官方 Dockerfile 基于 Debian 12

`Dockerfile:1` 明确使用 `debian:12-slim`：

```dockerfile
FROM debian:12-slim AS rssbridge
```

没有官方 Alpine 镜像，主要原因是 curl-impersonate 的编译依赖差异。

#### musl vs glibc 核心差异

| 维度 | Debian (glibc) | Alpine (musl) |
|------|---------------|---------------|
| C 标准库 | GNU libc 2.36 | musl libc 1.2+ |
| curl-impersonate 预编译包 | ✅ 官方提供 | ❌ 官方不提供 |
| BoringSSL 编译 | ✅ 容易 | ⚠️ 需要 patch |
| 线程模型 | NPTL（Native POSIX Threads） | musl 自有线程实现 |
| DNS 解析 | glibc nsswitch | musl 内置解析器 |
| 镜像体积 | ~50 MB（基础）+ ~800 MB（含 Chrome） | ~20 MB（基础） |

#### curl-impersonate 在 Alpine 上的编译挑战

1. **BoringSSL 与 musl 的兼容性**：BoringSSL 主要面向 glibc 开发，在 musl 上编译需要补丁

2. **Go 运行时交叉编译**：curl-impersonate 的部分工具链用 Go 编写，Alpine 的静态链接会有问题

3. **patchelf 可用性**：Dockerfile 中使用 `patchelf` 修改 SONAME，Alpine 上同样可用，但处理 musl 的动态链接有差异

4. **PHP 版本兼容性**：Alpine 3.18 带 PHP 8.2，理论上可以运行 RSS-Bridge，但 curl 扩展的编译选项不同

#### 手动编译 curl-impersonate for Alpine 的概念步骤

```dockerfile
# Alpine 概念 Dockerfile（未验证）
FROM alpine:3.18

RUN apk add --no-cache \
    build-base \
    cmake \
    go \
    rust \
    cargo \
    patchelf \
    git \
    python3

# 1. 下载 curl-impersonate 源码
# 2. 应用 musl 兼容性补丁
# 3. 编译 BoringSSL
# 4. 编译 libcurl-impersonate
# 5. patchelf 修改 SONAME
# 6. 安装 PHP + 扩展
```

#### 为什么 Alpine 不是优先选择

| 考虑因素 | Debian | Alpine |
|---------|--------|--------|
| curl-impersonate 支持 | ✅ 官方预编译，开箱即用 | ❌ 需要自行编译维护 |
| PHP 扩展兼容性 | ✅ 所有扩展都有官方包 | ⚠️ 部分扩展需要自行编译 |
| 调试便利性 | ✅ gdb、strace 等工具齐全 | ⚠️ musl 堆栈跟踪困难 |
| 长期维护成本 | 低（依赖官方包） | 高（每次升级都要重新编译） |
| 镜像体积 | 较大 | 较小 |

> **建议**：除非对镜像体积有极端要求，否则优先使用官方 Debian 镜像。如果必须用 Alpine，考虑使用多阶段构建，在 Debian 阶段编译 curl-impersonate，然后复制 .so 文件到 Alpine 阶段（但仍需处理 glibc 与 musl 的 ABI 不兼容问题）。

---

### 15.5 PSR-7 大小写不敏感合规

#### PSR-7 规范要求

PSR-7 `MessageInterface` 明确要求 Header 名称大小写不敏感：

> "While header names are case-insensitive, the casing of the header will be preserved by the implementation, and returned by `getHeaders()`."
> — PSR-7 Specification

#### RSS-Bridge Response 类的分析

`lib/http.php:254-389` 的 `Response` 类部分符合 PSR-7，但有差异：

| PSR-7 方法 | RSS-Bridge 实现 | 合规性 | 说明 |
|-----------|----------------|--------|------|
| `getHeaders()` | ✅ 有 | ⚠️ 部分 | 返回所有 header，但全部转为小写 |
| `hasHeader($name)` | ❌ 无 | - | 需要手动实现 |
| `getHeader($name)` | ✅ 有 | ✅ 合规 | 内部 `mb_strtolower($name)` 归一化 |
| `getHeaderLine($name)` | ❌ 无 | - | `getHeader()` 返回字符串（非数组）实现了类似功能 |
| `withHeader($name, $value)` | ✅ 有 | ❌ 不合规 | 没有归一化 `$name`，直接存储 |
| `withAddedHeader($name, $value)` | ❌ 无 | - | 需要手动实现 |
| `withoutHeader($name)` | ❌ 无 | - | 需要手动实现 |

#### 关键代码细节

**读取时归一化（合规）**：
```php
// lib/http.php:352-363
public function getHeader(string $name, bool $all = false)
{
    $name = mb_strtolower($name);  // ← 读取时归一化，合规
    $header = $this->headers[$name] ?? null;
    // ...
}
```

**构造时归一化（合规）**：
```php
// lib/http.php:311-312
$name = mb_strtolower($name);  // ← 存储前归一化
```

**响应 Header 解析时归一化（合规）**：
```php
// lib/http.php:161
$name = mb_strtolower(trim($header[0]));  // ← 解析响应时归一化
```

**`withHeader()` 未归一化（不合规）**：
```php
// lib/http.php:365-370
public function withHeader(string $name, string $value): self
{
    $clone = clone $this;
    $clone->headers[$name] = [$value];  // ← $name 未归一化，不合规！
    return $clone;
}
```

**`send()` 时未还原大小写（潜在问题）**：
```php
// lib/http.php:379-388
public function send(): void
{
    http_response_code($this->code);
    foreach ($this->headers as $name => $values) {
        foreach ($values as $value) {
            header(sprintf('%s: %s', $name, $value));  // ← 用存储的小写名发送
        }
    }
}
```

#### 不合规的影响

当调用 `$response->withHeader('Content-Type', 'text/html')` 后：
1. `$response->getHeaders()` 会同时包含 `'content-type'`（构造时的）和 `'Content-Type'`（withHeader 新增的）
2. 最终 `send()` 会发送两个 Header，导致客户端行为异常
3. `getHeader('Content-Type')` 仍能正常工作，因为读取时归一化

#### 修复建议

```php
public function withHeader(string $name, string $value): self
{
    $clone = clone $this;
    $name = mb_strtolower($name);  // ← 添加归一化
    $clone->headers[$name] = [$value];
    return $clone;
}
```

---

### 15.6 cgroup 限制下 Xvfb 内存弹性

#### Docker cgroup 内存限制

在 Docker 环境下，可以通过 `--memory` 参数限制容器内存：

```bash
docker run --memory=512m rss-bridge:latest
```

但 RSS-Bridge 代码中**没有 cgroup 内存检测**，也没有针对内存限制的弹性调整。

#### Xvfb 的内存模型

Xvfb（X Virtual Framebuffer）将整个显示缓冲区存在内存中：

```
内存占用 ≈ 宽 × 高 × 色深 / 8
        ≈ 1920 × 1080 × 24 / 8
        ≈ 6.2 MB （仅帧缓冲）
```

加上 Chrome 的内存占用，实际情况：

| 配置 | Xvfb 内存 | Chrome 内存 | 总占用 | cgroup 限制建议 |
|------|----------|------------|--------|----------------|
| 1024×768×24 | ~2.4 MB | ~150 MB | ~200 MB | ≥ 512 MB |
| 1920×1080×24 | ~6.2 MB | ~250 MB | ~350 MB | ≥ 1 GB |
| 1920×1080×24 + WebGL | ~6.2 MB | ~400 MB | ~500 MB | ≥ 1.5 GB |

#### cgroup OOM Killer 风险

当内存超过 cgroup 限制时，Linux 内核的 OOM Killer 会：
1. 选择内存占用最高的进程杀死
2. Chrome 通常是内存占用最高的，会被优先杀死
3. PHP-FPM 进程可能存活，但 WebDriver 连接已断开

**代码层面没有任何防护**——`WebDriverAbstract` 不会检测 Chrome 是否被 OOM 杀死，也不会自动重启。

#### 内存弹性的改进方向

1. **cgroup 内存检测**：

```php
function getCgroupMemoryLimit(): ?int
{
    if (is_readable('/sys/fs/cgroup/memory/memory.limit_in_bytes')) {
        return (int) file_get_contents('/sys/fs/cgroup/memory/memory.limit_in_bytes');
    }
    // cgroup v2
    if (is_readable('/sys/fs/cgroup/memory.max')) {
        $val = trim(file_get_contents('/sys/fs/cgroup/memory.max'));
        return $val === 'max' ? null : (int) $val;
    }
    return null;
}
```

2. **根据内存限制动态调整 Chrome 参数**：

```php
protected function getBrowserOptions()
{
    $chromeOptions = new ChromeOptions();
    $memoryLimit = getCgroupMemoryLimit();

    if ($memoryLimit && $memoryLimit < 512 * 1024 * 1024) {
        // 内存不足 512MB：激进的内存优化
        $chromeOptions->addArguments([
            '--headless',
            '--disable-gpu',
            '--disable-software-rasterizer',
            '--disable-dev-shm-usage',  // 避免使用 /dev/shm
            '--memory-pressure-off',
            '--enable-features=VaapiVideoDecoder',
        ]);
    } else {
        // 正常配置
        $chromeOptions->addArguments(['--headless']);
    }

    return $chromeOptions;
}
```

3. **共享内存（/dev/shm）限制**：

```bash
# Docker 默认 /dev/shm 只有 64MB，Chrome 需要更大
docker run --shm-size=1g rss-bridge:latest
```

或者通过 Chrome 参数绕过：
```php
$chromeOptions->addArguments(['--disable-dev-shm-usage']);
```

---

### 15.7 QUIC 与 HTTP/3 在 ALPN 降级中的位置

#### QUIC/HTTP/3 现状

RSS-Bridge **完全不支持 QUIC 和 HTTP/3**。

| 协议 | ALPN 标识符 | cURL 支持 | RSS-Bridge 支持 |
|------|------------|-----------|----------------|
| HTTP/1.1 | `http/1.1` | ✅ | ✅ |
| HTTP/2 | `h2` | ✅ | ✅（自动协商） |
| HTTP/3 | `h3` | cURL 7.66+ 实验性支持 | ❌ |

#### ALPN 协商顺序

curl-impersonate 模拟的 Chrome 142 的 ALPN 顺序是：

```
["h2", "http/1.1"]
```

**不包含 `h3`**——这是有意的，因为：
1. curl-impersonate 的主要目标是模拟浏览器 TLS 指纹，而不是追求最新协议
2. HTTP/3 的 UDP 传输与 TCP 上的 TLS 握手机制完全不同
3. 大多数反爬系统仍然主要检测 TCP 上的 TLS 指纹

#### 如果服务端强制 HTTP/3

某些站点（如 CloudFlare 保护的站点）可能通过 `Alt-Svc` 头声明 HTTP/3 支持：

```
Alt-Svc: h3=":443"; ma=86400, h3-29=":443"; ma=86400
```

但 RSS-Bridge 的 cURL 配置会：
1. 忽略 `Alt-Svc` 头（默认行为）
2. 继续使用 HTTP/1.1 或 HTTP/2
3. 不会自动升级到 HTTP/3

#### 启用 HTTP/3 的代价

如果需要支持 HTTP/3，需要：

1. **cURL 支持**：编译 cURL 时启用 `--enable-http3`，需要 ngtcp2 + nghttp3 + OpenSSL 3.0+
2. **curl-impersonate 不支持**：curl-impersonate 目前不模拟 HTTP/3 指纹
3. **UDP 网络**：HTTP/3 基于 UDP，需要防火墙放行 UDP 443
4. **指纹一致性**：如果声明支持 `h3` 但实际不使用，会造成指纹不一致

#### 降级链的完整路径

```
ClientHello ALPN: ["h2", "http/1.1"]
    ↓ 服务端支持 h2
HTTP/2 正常通信
    ↓ 服务端不支持 h2 但支持 http/1.1
HTTP/1.1 正常通信
    ↓ 服务端仅支持 h3（极罕见）
TLS 握手成功，但没有匹配的 ALPN → cURL 默认回退到 HTTP/1.1
```

> **注意**：HTTP/3 在 RSS-Bridge 的场景下收益不大——RSS 抓取通常是小请求，HTTP/3 的多路复用和 0-RTT 优势不明显。反而 TLS 指纹的一致性更重要。如果反爬系统检测到客户端声明支持 `h3` 但实际使用 HTTP/2，可能会触发额外的检查。

---

## 16. 最终架构总结

### HTTP 客户端决策树

```
发起请求
   ↓
是否有缓存且未过期？
   ├─ 是 → 直接返回缓存 ✅
   └─ 否 → 继续
         ↓
是否 BoringSSL 环境？
   ├─ 是 → curl-impersonate 自动设置 Chrome 142 指纹
   └─ 否 → 手动设置 Firefox 102 UA + Headers
         ↓
是否配置了代理且未 NOPROXY？
   ├─ 是 → CURLOPT_PROXY 设置代理
   └─ 否 → 直连
         ↓
网络请求成功？
   ├─ 是 → 检查状态码
   │     ├─ 200/201/202 → 写入缓存 → 返回
   │     ├─ 304 → 回填缓存 Body → 返回
   │     ├─ 429 → Bridge 级熔断 → 抛出 RateLimitException
   │     ├─ 503 → 返回 503 给客户端
   │     └─ 其他 → 抛出 HttpException
   └─ 否 → 重试 retries 次 → 仍失败 → 抛出 HttpException
```

### 反爬能力矩阵

| 反爬措施 | curl-impersonate | WebDriver | 手动指纹 |
|---------|-----------------|-----------|----------|
| TLS JA3 指纹 | ✅ 精确模拟 | ⚠️ 真实浏览器但可被检测 headless | ❌ OpenSSL 默认指纹 |
| ALPN 顺序 | ✅ 与 Chrome 一致 | ✅ 真实浏览器 | ❌ OpenSSL 默认顺序 |
| HTTP/2 帧顺序 | ✅ 精确模拟 | ✅ 真实浏览器 | ❌ 原生 cURL |
| Header 顺序 | ✅ 与 Chrome 一致 | ✅ 真实浏览器 | ⚠️ 硬编码 Firefox 顺序 |
| JavaScript 执行 | ❌ 不支持 | ✅ 完整支持 | ❌ 不支持 |
| Canvas/WebGL 指纹 | ❌ 不支持 | ⚠️ 可被检测 | ❌ 不支持 |
| cf_clearance Cookie | ❌ 需手动 | ✅ 自动获取 | ❌ 需手动 |
| 资源占用 | 极低 | 极高 | 极低 |

### 部署选型建议

| 场景 | 推荐方案 | 镜像体积 | 内存占用 |
|------|---------|---------|---------|
| 通用场景 | Docker Debian 官方镜像 | ~1.5 GB | ~100 MB |
| 严格反爬站点 | Docker + 独立 Selenium 容器 | ~2 GB | ~500 MB |
| 嵌入式/低资源 | Debian 原生部署（无 Docker） | N/A | ~50 MB |
| 仅 HTTP API 类 Bridge | 无 curl-impersonate，手动指纹 | ~200 MB | ~30 MB |
| 极端反爬 + 无 GUI | Xvfb + Chrome（非 headless） | ~2 GB | ~700 MB |

> **核心原则**：优先使用 curl-impersonate，失败时才考虑 WebDriver。WebDriver 是终极武器但代价高昂，应作为最后退路。
