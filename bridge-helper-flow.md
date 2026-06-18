# Bridge 抽象工具集工作流程分析

## 一、整体架构概览

RSS-Bridge 是一个将各种网站内容转换为 RSS 订阅的工具集。Bridge 抽象工具集是其核心，为所有 Bridge 实现提供统一的基础能力。

### 1.1 核心类层次结构

```
BridgeAbstract (抽象基类)
    ├── FeedExpander (Feed 扩展器)
    │   ├── WordPressBridge
    │   ├── FeedExpanderExampleBridge
    │   └── ... (其他 400+ 个 Bridge)
    └── XPathAbstract
        └── ...
```

**BridgeAbstract** (`lib/BridgeAbstract.php`) 定义了所有 Bridge 的公共接口：
- `collectData()` - 抽象方法，由具体 Bridge 实现数据采集逻辑
- `getItems()` - 获取采集到的条目列表
- `getInput()` - 获取用户输入参数
- `loadCacheValue()` / `saveCacheValue()` - 缓存操作

### 1.2 工具函数模块分布

| 模块 | 文件 | 主要功能 |
|------|------|----------|
| HTML 解析 | `lib/html.php` | DOM 操作、内容清洗、懒加载转换 |
| 路径修正 | `lib/url.php`, `lib/php-urljoin/src/urljoin.php` | URL 解析、相对路径转绝对路径 |
| 时间处理 | `lib/utils.php`, `lib/FeedParser.php` | 时间戳解析、格式化 |
| 内容获取 | `lib/contents.php` | HTTP 请求、DOM 加载、缓存 |

---

## 二、HTML 解析工作流程

### 2.1 HTML 解析核心工具链

```
HTTP 响应字符串
     ↓
str_get_html()  [simple_html_dom]
     ↓
simple_html_dom 对象
     ↓
┌─────────────────────────────────────┐
│  sanitize()          - 标签清理     │
│  convertLazyLoading() - 懒加载转换  │
│  defaultLinkTo()     - 路径修正     │
│  backgroundToImg()   - 背景图转换   │
└─────────────────────────────────────┘
     ↓
处理后的 HTML 内容
```

### 2.2 核心函数详解

#### 2.2.1 `getSimpleHTMLDOM()` - 获取并解析 HTML

**位置**: `lib/contents.php:165-190`

```php
function getSimpleHTMLDOM(
    $url,
    $header = [],
    $opts = [],
    $lowercase = true,
    $forceTagsClosed = true,
    $target_charset = DEFAULT_TARGET_CHARSET,
    $stripRN = true,
    $defaultBRText = DEFAULT_BR_TEXT,
    $defaultSpanText = DEFAULT_SPAN_TEXT
): \simple_html_dom {
    $html = getContents($url, $header ?? [], $opts ?? []);
    return str_get_html($html, $lowercase, $forceTagsClosed, ...);
}
```

**工作流程**:
1. 调用 `getContents()` 发送 HTTP 请求获取原始 HTML
2. 使用 `str_get_html()` (来自 simplehtmldom 库) 解析为 DOM 对象
3. 支持字符集转换、换行符处理、标签自动闭合等选项

**缓存版本**: `getSimpleHTMLDOMCached()` - 相同 URL 24 小时内直接返回缓存

#### 2.2.2 `sanitize()` - HTML 内容安全清洗

**位置**: `lib/html.php:162-185`

```php
function sanitize(
    $html,
    $tags_to_remove = ['script', 'iframe', 'input', 'form'],
    $attributes_to_keep = ['title', 'href', 'src'],
    $text_to_keep = []
) {
    $htmlContent = str_get_html($html);
    foreach ($htmlContent->find('*') as $element) {
        if (in_array($element->tag, $text_to_keep)) {
            $element->outertext = $element->plaintext;  // 标签替换为纯文本
        } elseif (in_array($element->tag, $tags_to_remove)) {
            $element->outertext = '';                    // 移除危险标签
        } else {
            foreach ($element->getAllAttributes() as $attributeName => $attribute) {
                if (!in_array($attributeName, $attributes_to_keep)) {
                    $element->removeAttribute($attributeName);  // 清理多余属性
                }
            }
        }
    }
    return $htmlContent;
}
```

**清理策略**:
- **危险标签移除**: `<script>`, `<iframe>`, `<input>`, `<form>` 等直接删除
- **白名单属性**: 只保留 `title`, `href`, `src` 三个属性
- **文本化标签**: 对 `text_to_keep` 中的标签，用内部纯文本替换整个标签

#### 2.2.3 `convertLazyLoading()` - 懒加载图片转换

**位置**: `lib/html.php:362-424`

**解决的问题**: 现代网站常用 `data-src` 等属性实现图片懒加载，但 RSS 阅读器不支持 JavaScript，导致图片无法显示。

**转换流程**:

```
原始 HTML:
<img data-src="image.jpg" data-srcset="img320.jpg 320w, img640.jpg 640w" />

     ↓ 第一步：查找懒加载属性
data-src → src
data-srcset → 解析后取最大尺寸 → src
data-lazy-src → src
data-orig-file → src
srcset → 解析后取最大尺寸 → src

     ↓ 第二步：清理属性
移除所有 data-* 属性
移除 loading, decoding, srcset 属性

     ↓ 第三步：<picture> 转 <img>
<picture>
  <source srcset="..." />
  <img src="..." />
</picture>
     ↓
<img src="..." />

最终 HTML:
<img src="img640.jpg" />
```

**关键子函数 `parseSrcset()`** (`lib/html.php:306-329`):

```php
function parseSrcset(string $srcset)
{
    // 处理复杂格式: image.png?resize=640,640 640w,image.png?resize=960,960 960w
    $preg_status = preg_match_all('/[\s]*,?[\s]*([^\s]+)\s+([0-9]+[wxh])/', $srcset, $matches);
    $entries = [];
    if ($preg_status !== false && $preg_status > 0) {
        foreach ($matches[1] as $index => $url) {
            if (array_key_exists($index, $matches[2])) {
                $size = $matches[2][$index];
                $entries[$size] = html_entity_decode($url);
            }
        }
    }
    return $entries;
}
```

**解析算法**:
1. 用正则表达式匹配 `[空格]*,?[空格]* URL [空格]+ 尺寸` 格式
2. URL 部分允许包含逗号（URL 参数中常见）
3. 返回 `['640w' => 'url1', '960w' => 'url2']` 的映射
4. `parseSrcsetLargestImageUrl()` 从中选择尺寸最大的 URL

#### 2.2.4 `backgroundToImg()` - 背景图转 img 标签

**位置**: `lib/html.php:222-234`

```php
function backgroundToImg($htmlContent)
{
    $regex = '/background-image[ ]{0,}:[ ]{0,}url\([\'"]{0,}(.*?)[\'"]{0,}\)/';
    $htmlContent = str_get_html($htmlContent);
    foreach ($htmlContent->find('*') as $element) {
        if (preg_match($regex, $element->style, $matches) > 0) {
            $element->outertext = '<img style="display:block;" src="' . $matches[1] . '" />';
        }
    }
    return $htmlContent;
}
```

**用途**: 某些网站用 `background-image: url('bg.jpg')` CSS 样式展示图片，RSS 阅读器无法直接显示，需要转换为 `<img>` 标签。

#### 2.2.5 其他 HTML 处理工具

| 函数 | 位置 | 功能 |
|------|------|------|
| `extractFromDelimiters()` | `html.php:435-443` | 从字符串中提取指定分隔符之间的内容 |
| `stripWithDelimiters()` | `html.php:453-461` | 移除指定分隔符包裹的内容（如 `<script>...</script>`） |
| `stripRecursiveHTMLSection()` | `html.php:482-516` | 递归移除嵌套 HTML 标签（如广告 div） |
| `markdownToHtml()` | `html.php:527-542` | Markdown 转 HTML |
| `handleYoutube()` | `html.php:552-609` | YouTube 链接转嵌入 iframe 或缩略图 |

**`stripRecursiveHTMLSection()` 算法** (`html.php:482-516`):

```
移除嵌套标签原理：
<div class="ads"><div>ads</div>ads</div>
     ↑        ↑        ↑       ↑
     1        2        1       0  (open_tag_count)

1. 定位开始标签 <div class="ads">
2. 向后查找 </div>，统计中间的开标签和闭标签数量
3. 当 open_tag_count == close_tag_count 时，找到正确的闭合位置
4. 移除整个标签块
```

### 2.3 实际应用示例（WordPressBridge）

**位置**: `bridges/WordPressBridge.php:37-111`

```php
protected function parseItem(array $item)
{
    // 1. 获取文章页 DOM
    $dom = getSimpleHTMLDOMCached($item['uri']);

    // 2. 查找文章正文（多种选择器降级）
    $article = $dom->find('[itemprop=articleBody]', 0)
        ?? $dom->find('.article-content', 0)
        ?? $dom->find('article', 0)
        ?? $dom->find('.single-content', 0)
        ?? $dom->find('.post-content', 0)
        ?? $dom->find('.post', 0);

    // 3. 转换懒加载图片
    $article = convertLazyLoading($article);

    // 4. 提取并设置文章首图
    $article_image = $dom->find('img.wp-post-image', 0);
    if (is_object($article_image) && !empty($article_image->src)) {
        $item['enclosures'] = [$article_image->src];
    }

    // 5. 清理不需要的标签
    $content = stripWithDelimiters($article->innertext, '<script', '</script>');
    $content = preg_replace('/<div class="wpa".*/', '', $content);
    $content = preg_replace('/<form.*\/form>/', '', $content);

    // 6. 修正相对路径
    $item['content'] = defaultLinkTo($content, $item['uri']);

    return $item;
}
```

---

## 三、路径修正工作流程

### 3.1 路径修正核心工具链

```
相对路径 URL
     ↓
urljoin($base, $rel)  [php-urljoin 库]
     ↓
┌─────────────────────────────────────┐
│  解析 base URL 各部分               │
│  解析 rel URL 各部分                │
│  处理相对路径 (./, ../, 无斜杠)     │
│  路径规范化 (移除 ., .., 空片段)    │
│  合并 URL 各部分                    │
└─────────────────────────────────────┘
     ↓
绝对路径 URL
     ↓
defaultLinkTo($dom, $url)  [html.php]
     ↓
遍历 <img src> 和 <a href>，批量修正
```

### 3.2 核心函数详解

#### 3.2.1 `urljoin()` - URL 合并（核心算法）

**位置**: `lib/php-urljoin/src/urljoin.php:12-143`

这是 Python `urlparse.urljoin()` 的 PHP 移植版本，处理各种复杂的相对路径情况。

**核心处理流程**:

```php
function urljoin($base, $rel) {
    // 1. 边界情况处理
    if (!$base) return $rel;
    if (!$rel) return $base;

    // 2. 解析 URL
    $pbase = parse_url($base);
    $prel = parse_url($rel);

    // 3. 解析失败容错（rel 可能是纯路径）
    if ($prel === false || preg_match('/^[a-z0-9\-.]*[^a-z0-9\-.:][a-z0-9\-.]*:/i', $rel)) {
        $prel = array('path' => $rel);
    }

    // 4. rel 有独立 scheme 且与 base 不同 → 直接返回 rel
    if (isset($prel['scheme'])) {
        if ($prel['scheme'] != ($pbase['scheme'] ?? null)
            || in_array($prel['scheme'], $uses_relative) == false) {
            return $rel;
        }
    }

    // 5. 合并 base 和 rel 的组件
    $merged = array_merge($pbase, $prel);

    // 6. 处理相对路径 (path 不以 / 开头)
    if (array_key_exists('path', $prel) && substr($prel['path'], 0, 1) != '/') {
        // 移除开头的 ./
        if (substr($prel['path'], 0, 2) === './') {
            $prel['path'] = substr($prel['path'], 2);
        }

        if (array_key_exists('path', $pbase)) {
            // 去掉 base path 的文件名部分，保留目录
            $dir = preg_replace('@/[^/]*$@', '', $pbase['path']);
            $merged['path'] = $dir . '/' . $prel['path'];
        } else {
            $merged['path'] = '/' . $prel['path'];
        }
    }

    // 7. 路径规范化 (处理 . 和 ..)
    if (array_key_exists('path', $merged)) {
        $pathParts = explode('/', $merged['path']);
        array_shift($pathParts);  // 移除开头空字符串

        $path = [];
        $prevPart = '';
        foreach ($pathParts as $part) {
            if ($part == '..' && count($path) > 0) {
                // .. → 弹出上一级目录
                $parent = array_pop($path);
                if ($parent == '..') {
                    // 上一级也是 ..，保留两个 ..
                    array_push($path, $parent);
                    array_push($path, $part);
                }
            } else if ($prevPart != '' || ($part != '.' && $part != '')) {
                // . → 跳过，空片段 → 跳过（连续斜杠）
                if ($part == '.') {
                    $part = '';
                }
                array_push($path, $part);
            }
            $prevPart = $part;
        }
        $merged['path'] = '/' . implode('/', $path);
    }

    // 8. 重新组装 URL
    $ret = '';
    if (isset($merged['scheme'])) $ret .= $merged['scheme'] . ':';
    if (isset($merged['scheme']) || isset($merged['host'])) $ret .= '//';
    // ... host, user, pass, port, path, query, fragment
    return $ret;
}
```

**路径规范化示例**:

| 输入路径 | 处理过程 | 输出路径 |
|----------|----------|----------|
| `/a/./b` | 移除 `.` | `/a/b` |
| `/a/b/../c` | `..` 弹出 `b` | `/a/c` |
| `/a/b/../../c` | 两次 `..` 弹出 `b`, `a` | `/c` |
| `/a//b` | 跳过空片段 | `/a/b` |
| `../a/b` | 无上级可弹，保留 `..` | `/../a/b` |

#### 3.2.2 `Url` 类 - 严格 URL 验证和封装

**位置**: `lib/url.php:14-151`

```php
final class Url
{
    private string $scheme;
    private string $host;
    private int $port;
    private string $path;
    private ?string $queryString;

    public static function fromString(string $url): self
    {
        if (!self::validate($url)) {
            throw new UrlException(sprintf('Illegal url: "%s"', $url));
        }
        $parts = parse_url($url);
        return (new self())
            ->withScheme($parts['scheme'] ?? '')
            ->withHost($parts['host'])
            ->withPort($parts['port'] ?? 80)
            ->withPath($parts['path'] ?? '/')
            ->withQueryString($parts['query'] ?? null);
    }

    public static function validate(string $url): bool
    {
        if (strlen($url) > 1500) return false;
        $pattern = '#^https?://'        // scheme (仅 http/https)
            . '([a-z0-9-]+\.?)+'        // 域名部分
            . '(\.[a-z]{1,24})?'        // TLD
            . '(:\d+)?'                 // 可选端口
            . '($|/|\?)#i';             // 结束或路径开始
        return preg_match($pattern, $url) === 1;
    }
}
```

**设计特点**:
- **不可变对象**: 使用 `with*()` 方法返回新实例，避免副作用
- **严格验证**: 只允许 http/https 协议，防止 SSRF 等攻击
- **值对象**: `__toString()` 方法输出标准化 URL

#### 3.2.3 `defaultLinkTo()` - 批量修正 HTML 中的相对路径

**位置**: `lib/html.php:247-286`

```php
function defaultLinkTo($dom, $url)
{
    if ($dom === '') return $url;

    // 支持字符串和 DOM 对象两种输入
    $string_convert = false;
    if (is_string($dom)) {
        $string_convert = true;
        $dom = str_get_html($dom);
    }

    // 兼容性处理：simple_html_dom 和 DOMDocument 的 getElementsByTagName 行为不同
    if ($dom instanceof simple_html_dom) {
        $findByTag = function ($name) use ($dom) {
            return $dom->getElementsByTagName($name, null);
        };
    } else {
        $findByTag = function ($name) use ($dom) {
            return $dom->getElementsByTagName($name);
        };
    }

    // 修正所有图片 src
    foreach ($findByTag('img') as $image) {
        $image->setAttribute('src', urljoin($url, $image->getAttribute('src')));
    }

    // 修正所有链接 href
    foreach ($findByTag('a') as $anchor) {
        $anchor->setAttribute('href', urljoin($url, $anchor->getAttribute('href')));
    }

    // 返回与输入相同的类型
    if ($string_convert) {
        $dom = $dom->outertext;
    }
    return $dom;
}
```

**使用示例**:

```php
// 输入 HTML (字符串形式)
$html = '<a href="article.html">文章</a><img src="images/pic.jpg">';
$baseUrl = 'https://example.com/news/index.html';

// 调用
$result = defaultLinkTo($html, $baseUrl);

// 输出
// <a href="https://example.com/news/article.html">文章</a>
// <img src="https://example.com/news/images/pic.jpg">
```

**相对路径合并示例表**:

| Base URL | Relative URL | 合并结果 | 说明 |
|----------|--------------|----------|------|
| `https://example.com/a/b.html` | `c.html` | `https://example.com/a/c.html` | 同目录 |
| `https://example.com/a/b.html` | `../c.html` | `https://example.com/c.html` | 上级目录 |
| `https://example.com/a/b/` | `c.html` | `https://example.com/a/b/c.html` | 目录基址 |
| `https://example.com/a/b.html` | `/c.html` | `https://example.com/c.html` | 绝对路径 |
| `https://example.com/a/b.html` | `https://other.com/x.html` | `https://other.com/x.html` | 完整 URL |
| `https://example.com/a/b.html` | `//other.com/x.html` | `https://other.com/x.html` | 协议相对 |

### 3.3 实际应用场景

**WordPressBridge** (`bridges/WordPressBridge.php:107`):
```php
$item['content'] = defaultLinkTo($item['content'], $item['uri']);
```

**XenForoBridge** (`bridges/XenForoBridge.php:116,312,353`):
```php
$html = defaultLinkTo($html, $this->threadurl);  // 帖子内容
$html = defaultLinkTo($html, $hosturl);          // 列表内容
```

**注意**: `defaultLinkTo()` 只处理 `<img src>` 和 `<a href>`，不处理 `srcset` 属性。如 TheBellBridge 中提到的：
```php
// handle relative URL's in srcset (not supported in defaultLinkTo()
```

---

## 四、时间处理工作流程

### 4.1 时间处理核心工具链

```
各种时间格式字符串
     ↓
┌─────────────────────────────────────┐
│  strtotime()          - 通用解析    │
│  DateTime::createFromFormat() - 指定格式 │
│  new DateTimeImmutable() - 当前时间 │
└─────────────────────────────────────┘
     ↓
Unix 时间戳 (int)
     ↓
FeedItem['timestamp']
```

### 4.2 核心函数详解

#### 4.2.1 `now()` - 获取当前时间

**位置**: `lib/utils.php:235-238`

```php
function now(): \DateTimeImmutable
{
    return new \DateTimeImmutable();
}
```

**设计意图**: 统一当前时间获取方式，便于单元测试时 mock。

#### 4.2.2 Feed 解析中的时间处理

**位置**: `lib/FeedParser.php`

**Atom Feed 解析** (`FeedParser.php:97-98`):
```php
if (isset($feedItem->updated)) {
    $item['timestamp'] = strtotime((string)$feedItem->updated);
}
```

**RSS Feed 解析** (`FeedParser.php:202-203`):
```php
$item['timestamp'] = $feedItem->pubDate ?? $dc->date ?? '';
$item['timestamp'] = strtotime((string) $item['timestamp']);
```

**RDF Feed 解析** (`FeedParser.php:241-242`):
```php
if (isset($dc->date)) {
    $item['timestamp'] = strtotime((string)$dc->date);
}
```

**标准时间格式示例**:
- RFC 2822: `Mon, 15 Aug 2022 15:52:01 +0000`
- ISO 8601: `2022-08-15T15:52:01+00:00`
- RFC 3339: `2022-08-15T15:52:01Z`

#### 4.2.3 Bridge 中的时间处理模式

**模式 1: 直接使用 `strtotime()` 解析**

```php
// WikiLeaksBridge.php:98
$item['timestamp'] = strtotime($timestamp->plaintext);

// TwitterEngineeringBridge.php:40
$item['timestamp'] = strtotime($dom->find('span.b02-blog-post-no-masthead__date', 0)->innertext);
```

**模式 2: 使用 `DateTime::createFromFormat()` 指定格式**

```php
// WebfailBridge.php:93
$dt = DateTime::createFromFormat('!d.m.Y', $matches[1]);
```

**模式 3: 复杂日期字符串手动拼接后解析**

```php
// VkBridge.php:490-504
// 处理 "12:34"、"5 мар в 12:34"、"5 мар 2022 в 12:34" 等格式
if ($date['day'] && !$date['month']) {
    // 今天/昨天的时间
    $strdate = date('d-m-Y') . ' ' . $strdate;
} elseif ($date['month'] && !$date['year']) {
    // 今年的日期
    if (intval(date('m')) < $date['month']) {
        $strdate = $strdate . ' ' . (date('Y') - 1);  // 去年
    } else {
        $strdate = $strdate . ' ' . date('Y');        // 今年
    }
}
return strtotime($date['day'] . '-' . $date['month'] . '-' . $date['year'] . ' ' . $strdate);
```

**模式 4: 从 HTML 属性中提取时间戳**

```php
// XenForoBridge.php:235-237
if ($timestamp = $post->find('abbr.DateTime', 0)) {
    // <abbr class="DateTime" data-time="1660569120" title="2022-08-15T15:52:00+0000">
    $item['timestamp'] = $timestamp->getAttribute('data-time');
}
```

**模式 5: 从 JSON-LD 结构化数据中提取**

```php
// YouTubeBridge.php:471
$publicationDate = new \DateTimeImmutable($publishedTimeText);
```

#### 4.2.4 `getContents()` 中的时间处理（缓存）

**位置**: `lib/contents.php:76-85`

```php
$cachedResponse = $cache->get($cacheKey);
if ($cachedResponse) {
    $lastModified = $cachedResponse->getHeader('last-modified');
    if ($lastModified) {
        try {
            // 兼容 Unix 时间戳和 RFC7231 日期格式
            $lastModified = new \DateTimeImmutable(
                (is_numeric($lastModified) ? '@' : '') . $lastModified
            );
            $config['if_not_modified_since'] = $lastModified->getTimestamp();
        } catch (Exception $e) {
            // 解析失败，跳过
        }
    }
}
```

**巧妙之处**: 通过 `is_numeric()` 判断是否为 Unix 时间戳，是则前置 `@` 符号让 `DateTime` 正确解析。

### 4.3 时间格式兼容性处理

| 输入格式 | 处理方式 | 示例 |
|----------|----------|------|
| 标准 RFC 格式 | `strtotime()` 直接解析 | `Mon, 15 Aug 2022 15:52:01 +0000` |
| ISO 8601 | `strtotime()` 直接解析 | `2022-08-15T15:52:01+00:00` |
| Unix 时间戳 | 前置 `@` 后用 `DateTime` | `@1660569120` |
| 自定义格式 | `DateTime::createFromFormat()` | `!d.m.Y` 解析 `15.08.2022` |
| 相对日期 | `strtotime()` 自然语言解析 | `"-6 hours"`, `"yesterday"` |
| 俄文月份 | 手动映射后拼接 | `"5 мар 2022"` → `05-03-2022` |

### 4.4 时间处理注意事项

1. **时区问题**: `strtotime()` 会使用 PHP 配置的默认时区，确保 `date.timezone` 设置正确
2. **解析失败**: `strtotime()` 失败时返回 `false`，需要做容错处理
3. **夏令时**: 某些日期时间在夏令时切换时可能不存在或重复
4. **年份推断**: 处理不带年份的日期时，需要正确判断是今年还是去年
5. **数据类型**: FeedItem 中 `timestamp` 字段应为 Unix 时间戳（整数）

---

## 五、完整数据流示例

以 **WordPressBridge** 为例，展示三大工具如何协同工作：

```
用户请求: bridge=WordPressBridge&url=https://example.com/blog
     ↓
1. BridgeAbstract::setInput() 验证参数
     ↓
2. WordPressBridge::collectData()
   → collectExpandableDatas('https://example.com/blog/feed/atom/', 10)
     ↓
3. FeedExpander::collectExpandableDatas()
   → getContents() 抓取 Atom Feed
   → FeedParser::parseFeed() 解析 XML
     ├─ 解析 <entry><updated> → strtotime() → timestamp
     └─ 解析出 10 个 feed item
     ↓
4. 循环调用 WordPressBridge::parseItem($item)
   ├─ getSimpleHTMLDOMCached($item['uri']) 抓取文章详情
   ├─ $dom->find() CSS 选择器定位正文
   ├─ convertLazyLoading() 转换懒加载图片
   │   ├─ 查找 data-src, data-srcset 等属性
   │   ├─ parseSrcset() 解析 srcset → 取最大图
   │   └─ 替换为标准 src 属性
   ├─ stripWithDelimiters() 移除 <script> 等
   ├─ defaultLinkTo($content, $item['uri']) 修正相对路径
   │   ├─ 遍历 <img> → urljoin() 修正 src
   │   └─ 遍历 <a> → urljoin() 修正 href
   └─ return $item
     ↓
5. 收集所有 items → 格式化为 RSS/JSON/Atom 输出
```

---

## 六、关键设计模式总结

### 6.1 模板方法模式
- `BridgeAbstract::collectData()` 是抽象方法，由子类实现具体采集逻辑
- `FeedExpander::collectExpandableDatas()` 定义了 Feed 扩展的骨架流程，`parseItem()` 由子类自定义

### 6.2 工具函数门面
`html.php` 中的函数对 simplehtmldom 库进行了封装，提供更高层次的操作：
- `convertLazyLoading()` 封装了懒加载属性的查找、解析、转换
- `defaultLinkTo()` 封装了 DOM 遍历和 `urljoin()` 调用

### 6.3 不可变对象
`Url` 类使用 `with*()` 方法返回新实例，确保线程安全和可预测性。

### 6.4 防御式编程
- 多层参数校验 (`ParameterValidator`, `Url::validate()`)
- 类型检查和容错处理 (`is_string()`, `is_object()`, `??` 运算符)
- 输入净化 (`sanitize()`, `stripWithDelimiters()`)

### 6.5 缓存策略
- 内容缓存: `getSimpleHTMLDOMCached()` TTL 24 小时
- HTTP 缓存: `getContents()` 支持 `Last-Modified` / `ETag` 协商缓存
- 服务器缓存: 不同 URL 分开缓存，缓存键包含请求体哈希

---

## 七、代码优化建议

### 7.1 `defaultLinkTo()` 可扩展支持 srcset

当前只处理 `src` 和 `href`，可增加 `srcset` 处理：

```php
// 在 defaultLinkTo() 中增加
foreach ($findByTag('img, source') as $element) {
    $srcset = $element->getAttribute('srcset');
    if ($srcset) {
        $entries = parseSrcset($srcset);
        $newEntries = [];
        foreach ($entries as $size => $imgUrl) {
            $newEntries[$size] = urljoin($url, $imgUrl);
        }
        // 重建 srcset
        $newSrcset = [];
        foreach ($newEntries as $size => $imgUrl) {
            $newSrcset[] = "$imgUrl $size";
        }
        $element->setAttribute('srcset', implode(', ', $newSrcset));
    }
}
```

### 7.2 时间解析统一封装

可在 `utils.php` 中增加统一的时间解析函数：

```php
function parse_timestamp($dateString, $format = null): ?int
{
    if ($format) {
        $dt = DateTime::createFromFormat($format, $dateString);
        return $dt ? $dt->getTimestamp() : null;
    }
    
    if (is_numeric($dateString)) {
        return (int)$dateString;
    }
    
    $timestamp = strtotime($dateString);
    return $timestamp === false ? null : $timestamp;
}
```

### 7.3 HTML 处理链式调用

可考虑提供流式 API 提高可读性：

```php
// 当前
$article = convertLazyLoading($article);
$article = defaultLinkTo($article, $uri);
$article = sanitize($article);

// 优化后
$article = HtmlProcessor::from($article)
    ->convertLazyLoading()
    ->defaultLinkTo($uri)
    ->sanitize()
    ->get();
```

---

## 八、参考文件索引

| 功能 | 文件 | 关键行号 |
|------|------|----------|
| Bridge 抽象基类 | `lib/BridgeAbstract.php` | 1-339 |
| Feed 扩展器 | `lib/FeedExpander.php` | 1-86 |
| HTML 处理函数 | `lib/html.php` | 1-609 |
| URL 合并算法 | `lib/php-urljoin/src/urljoin.php` | 12-143 |
| URL 封装类 | `lib/url.php` | 14-151 |
| 工具函数 | `lib/utils.php` | 1-283 |
| 内容获取 | `lib/contents.php` | 1-251 |
| Feed 解析器 | `lib/FeedParser.php` | 时间相关行 98,202,242 |
| WordPressBridge 示例 | `bridges/WordPressBridge.php` | 1-129 |
