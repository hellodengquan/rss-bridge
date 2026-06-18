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

### 2.4 Invalid HTML 下的 DOMDocument 容错路径

RSS-Bridge 采用双层解析器架构应对互联网上广泛存在的不规范 HTML：`simple_html_dom`（默认）和 `DOMDocument`（XPathAbstract 使用）。

#### 2.4.1 容错路径总览

```
Invalid HTML 输入
     ↓
┌─────────────────────────────────────────────────────┐
│  路径 1: simple_html_dom + forceTagsClosed          │
│  ├─ 自动闭合未闭合标签                               │
│  ├─ 预解析噪音移除（<script>, <style>, <!-- -->）    │
│  └─ 字符集规范化                                    │
├─────────────────────────────────────────────────────┤
│  路径 2: DOMDocument + libxml 错误抑制              │
│  ├─ libxml_use_internal_errors(true)                │
│  ├─ loadHTML() 自动修复模式                         │
│  ├─ libxml_clear_errors()                           │
│  └─ libxml_use_internal_errors(false)               │
└─────────────────────────────────────────────────────┘
     ↓
可用的 DOM 对象
```

#### 2.4.2 simple_html_dom 的 `forceTagsClosed` 机制

**位置**: `lib/simplehtmldom/simple_html_dom.php:1488-1492`

```php
// Forcing tags to be closed implies that we don't trust the html, but
// it can lead to parsing errors if we SHOULD trust the html.
if (!$forceTagsClosed) {
    $this->optional_closing_array = array();
}
```

**`optional_closing_array` 预定义标签** (`simple_html_dom.php`):
```php
// 这些标签在 HTML 规范中允许不闭合
protected $optional_closing_array = [
    'li' => 1, 'dt' => 1, 'dd' => 1, 'p' => 1,
    'rt' => 1, 'rp' => 1, 'optgroup' => 1, 'option' => 1,
    'colgroup' => 1, 'thead' => 1, 'tfoot' => 1,
    'tr' => 1, 'th' => 1, 'td' => 1,
];
```

**`forceTagsClosed = true` 时的行为**:
- 清空 `optional_closing_array`，强制所有标签必须闭合
- 解析器遇到未闭合标签时会自动插入闭合标签
- 适用场景：混乱的 HTML（论坛、博客、新闻网站）

**`forceTagsClosed = false` 时的行为**:
- 保留 HTML 规范中的可选闭合标签
- 解析更准确但对不规范 HTML 容忍度低
- 适用场景：规范的 HTML（API 响应、结构化数据）

**预解析噪音移除阶段** (`simple_html_dom.php:1517-1544`):
```php
// strip out <script> tags
$this->remove_noise("'<\s*script[^>]*[^/]>(.*?)<\s*/\s*script\s*>'is");
$this->remove_noise("'<\s*script\s*>(.*?)<\s*/\s*script\s*>'is");
// strip out cdata
$this->remove_noise("'<!\[CDATA\[(.*?)\]\]>'is", true);
// strip out comments
$this->remove_noise("'<!--(.*?)-->'is");
// strip out <style> tags
$this->remove_noise("'<\s*style[^>]*[^/]>(.*?)<\s*/\s*style\s*>'is");
// strip out preformatted tags
$this->remove_noise("'<\s*(?:code)[^>]*>(.*?)<\s*/\s*(?:code)\s*>'is");
// strip out server side scripts
$this->remove_noise("'(<\?)(.*?)(\?>)'s", true);
```

> **注意**: `remove_noise()` 在 `simple_html_dom` 内部完成，`sanitize()` 是更高层次的安全清洗，两者是互补关系而非重复。

#### 2.4.3 DOMDocument + libxml 错误抑制（XPathAbstract 路径）

**位置**: `lib/XPathAbstract.php:404-408`

```php
public function collectData()
{
    $this->feedUri = $this->getParam('url');

    $webPageHtml = new \DOMDocument();
    libxml_use_internal_errors(true);      // 1. 开启错误抑制
    $webPageHtml->loadHTML($this->provideWebsiteContent());  // 2. 加载（自动修复）
    libxml_clear_errors();                 // 3. 清除错误记录
    libxml_use_internal_errors(false);     // 4. 恢复错误报告

    // fix relative links
    defaultLinkTo($webPageHtml, $webPageHtml->baseURI ?? $this->feedUri);

    $xpath = new \DOMXPath($webPageHtml);
    // ... 后续 XPath 查询
}
```

**loadHTML() 自动修复能力** (libxml 内置):
1. 自动添加缺失的 `<html>`, `<head>`, `<body>` 标签
2. 自动闭合未闭合的标签（如 `<p>`, `<li>`, `<td>`）
3. 自动转义非法字符
4. 尝试修复嵌套错误（如 `<div><p></div></p>` → `<div><p></p></div>`）

**同类模式**:
- `FeedParser.php:18-21` - XML 解析时的错误抑制
- `bridges/LWNprevBridge.php:58-61` - 独立 Bridge 的 DOMDocument 使用

#### 2.4.4 两种解析器的兼容层

**位置**: `lib/html.php:259-270`

```php
// Use long method names for compatibility with simple_html_dom and DOMDocument

// Work around bug in simple_html_dom->getElementsByTagName
if ($dom instanceof simple_html_dom) {
    $findByTag = function ($name) use ($dom) {
        return $dom->getElementsByTagName($name, null);  // simple_html_dom 需要第二个参数
    };
} else {
    $findByTag = function ($name) use ($dom) {
        return $dom->getElementsByTagName($name);         // DOMDocument 标准签名
    };
}
```

**兼容处理边界**:
| 特性 | simple_html_dom | DOMDocument |
|------|-----------------|-------------|
| `getElementsByTagName($name)` | 需要第二个参数 `null` | 标准单参数 |
| `outertext` 属性 | 支持，返回 HTML 字符串 | 不支持，需用 `saveHTML()` |
| `find()` CSS 选择器 | 原生支持 | 需用 DOMXPath |
| `getAttribute('src')` | 支持 | 支持（标准方法） |
| `setAttribute('src', $val)` | 支持 | 支持（标准方法） |

#### 2.4.5 容错失效场景与边界

| 场景 | simple_html_dom 行为 | DOMDocument 行为 | 可能后果 |
|------|---------------------|------------------|----------|
| 标签深度嵌套 > 1000 | 栈溢出，解析失败 | libxml 限制，截断 | 内容丢失 |
| 自闭合标签写错（`<div />`） | 可能错误解析 | 自动修复 | 结构错乱 |
| 未闭合引号（`<a href="url>`） | 属性解析错误 | 自动修复 | 链接失效 |
| 二进制字符嵌入 | `stripRN` 可能处理不完全 | 忽略或转义 | 解析中断 |
| `</body></html>` 缺失 | 自动补全 | 自动补全 | 正常 |
| `<` 字符未转义在文本中 | 可能误识别为标签开始 | 自动转义 | 内容截断 |

---

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

### 2.5 大文档下三种解析器的内存占用对比

RSS-Bridge 在不同场景使用三种 XML/HTML 解析器，它们的内存模型和占用差异巨大，在处理大文档时需要特别关注。

#### 2.5.1 三种解析器内存模型对比

| 解析器 | 使用场景 | 实现语言 | 内存模型 | 内存效率 |
|--------|----------|----------|----------|----------|
| **SimpleXML** | FeedParser 解析 RSS/Atom Feed | C（PHP 扩展） | 整个文档加载到内存，形成对象树 | ⭐⭐⭐⭐⭐ 最高 |
| **DOMDocument** | XPathAbstract 解析 HTML | C（libxml） | 整个文档加载到内存，DOM 树 | ⭐⭐⭐⭐ 高 |
| **simple_html_dom** | 大部分 Bridge 解析 HTML | 纯 PHP | 节点数组 + 循环引用 + 文本复制 | ⭐⭐ 低 |

#### 2.5.2 simple_html_dom 内存结构深度分析

**位置**: `lib/simplehtmldom/simple_html_dom.php:130-164`

```php
class simple_html_dom_node
{
    public $nodetype = HDOM_TYPE_TEXT;    // 节点类型
    public $tag = 'text';                  // 标签名
    public $attr = array();                // 属性数组
    public $children = array();            // 直接子节点数组
    public $nodes = array();               // 所有子节点（含文本节点）
    public $parent = null;                 // 父节点引用（循环引用！）
    public $_ = array();                   // 位置信息数组（BEGIN, END, TEXT 等 8 个）
    public $tag_start = 0;                 // 标签起始位置
    private $dom = null;                   // DOM 根引用
}
```

**内存开销构成**：
1. **节点对象本身**: 每个 `simple_html_dom_node` 对象约 1-2KB 基础开销
2. **`$_` 数组**: 存储 8 个位置索引（BEGIN, END, QUOTE, SPACE, TEXT, INNER, OUTER, ENDSPACE）
3. **`children` + `nodes` 数组**: 子节点引用，造成重复存储
4. **循环引用**: `parent` <-> `children` 形成引用环，GC 无法自动回收
5. **文本复制**: `innertext()`, `outertext()` 等方法通过字符串截取生成新字符串

**文档级内存结构** (`simple_html_dom` 类):
```php
class simple_html_dom
{
    public $nodes = array();      // 所有节点的平面数组（引用所有节点）
    public $doc = '';             // 原始 HTML 文档字符串
    public $noise = array();      // 被移除的噪音（script/style/comment 等）
    // ...
}
```

**内存放大系数**: 对于普通 HTML 文档，simple_html_dom 的内存占用约为原始 HTML 大小的 **8-15 倍**。

#### 2.5.3 SimpleXML 内存结构分析

**位置**: `lib/FeedParser.php:19`

```php
$xml = simplexml_load_string(trim($xmlString));
```

**内存特性**:
- **C 级实现**: SimpleXML 是 PHP 扩展，底层用 C 实现，内存效率接近原生 C 代码
- **按需加载**: 属性和子节点在首次访问时才创建 PHP 对象代理
- **引用计数**: 多个 SimpleXMLElement 对象共享底层 libxml 节点
- **无循环引用**: 父子关系通过内部指针管理，不形成 PHP 层面的循环引用

**内存放大系数**: 约为原始 XML 大小的 **1.5-3 倍**。

#### 2.5.4 DOMDocument 内存结构分析

**位置**: `lib/XPathAbstract.php:404-406`

```php
$webPageHtml = new \DOMDocument();
libxml_use_internal_errors(true);
$webPageHtml->loadHTML($this->provideWebsiteContent());
```

**内存特性**:
- **libxml 原生 DOM**: 底层由 libxml2 C 库实现，内存效率高
- **DOM 树结构**: 完整的 W3C DOM 实现，节点类型丰富
- **PHP 代理对象**: `DOMNode`, `DOMElement` 等是对底层 C 节点的 PHP 包装
- **XPath 查询高效**: 底层原生 XPath 引擎，查询速度快

**内存放大系数**: 约为原始 HTML 大小的 **2-4 倍**。

#### 2.5.5 内存泄漏与释放机制

**simple_html_dom 的内存泄漏问题**:

**位置**: `lib/simplehtmldom/simple_html_dom.php:1598-1620`

```php
// This add next line is documented in the sourceforge repository.
// 2977248 as a fix for ongoing memory leaks that occur even with the
// use of clear.
if (isset($this->children)) {
    foreach ($this->children as $n) {
        $n->clear();
        $n = null;
    }
}

if (isset($this->parent)) {
    $this->parent->clear();
    unset($this->parent);  // 手动断开循环引用
}

if (isset($this->root)) {
    $this->root->clear();
    unset($this->root);
}

unset($this->doc);
unset($this->noise);
```

**问题根源**:
1. **循环引用**: `parent` 指向父节点，`children` 包含子节点 → 形成引用环
2. **PHP GC 延迟**: 循环引用需要 PHP 周期收集器运行才能回收，有延迟
3. **clear() 不彻底**: SourceForge #2977248 号 bug 显示即使调用 `clear()` 仍有内存泄漏

**RSS-Bridge 中的内存风险点**:

| 场景 | 代码位置 | 风险等级 | 说明 |
|------|----------|----------|------|
| FeedExpander 批量解析 | `lib/FeedExpander.php:36-42` | 中 | 循环解析每个 item，SimpleXML 对象循环结束后自动释放 |
| getSimpleHTMLDOMCached 大页面 | `lib/contents.php:219-251` | 高 | 大页面 DOM 对象可能占用数十 MB |
| FeedMergeBridge 多 Feed 合并 | `bridges/FeedMergeBridge.php:62-87` | 中 | 连续调用 collectExpandableDatas，上一个 Feed 的 DOM 在下一次调用前未释放 |
| convertLazyLoading 遍历所有 img | `lib/html.php:362-424` | 中 | 遍历 DOM 树可能创建大量临时对象 |

#### 2.5.6 MAX_FILE_SIZE 限制

**位置**: `lib/simplehtmldom/simple_html_dom.php:45`

```php
defined('MAX_FILE_SIZE') || define('MAX_FILE_SIZE', 600000);  // 600 KB
```

**作用**:
- `file_get_html()` 等函数使用此常量限制最大文件大小
- 防止超大文档导致内存耗尽
- RSS-Bridge 中主要通过 `getContents()` 层面的 `max_filesize` 配置控制

**配置位置**: `config.default.ini.php` → `http.max_filesize`

#### 2.5.7 内存优化建议

1. **及时释放 DOM 对象**:
   ```php
   $dom = getSimpleHTMLDOMCached($url);
   $content = $dom->find('.article', 0)->innertext;
   $dom->clear();  // 手动释放
   unset($dom);
   ```

2. **优先使用 SimpleXML 处理 XML**:
   - Feed 解析用 SimpleXML，不用 simple_html_dom
   - 结构化数据优先用 JSON + `json_decode()`，内存效率最高

3. **大文档分页处理**:
   - 不要一次性加载整个大文档
   - 能用 API 接口就不用爬取完整 HTML

4. **利用缓存减少解析次数**:
   - 解析结果缓存（`getSimpleHTMLDOMCached`）
   - 避免重复解析相同内容

### 2.6 Stream-Based (Chunked) 解析的代码挂载点

RSS-Bridge 在 HTTP 传输层支持 chunked transfer encoding，但在解析层采用"完整接收→整体解析"的两步模型，没有真正的流式解析器。

#### 2.6.1 数据流架构总览

```
远端服务器 (可能使用 Transfer-Encoding: chunked)
     ↓
[cURL 层] CurlHttpClient::request()
     ├─ CURLOPT_HEADERFUNCTION  ← 逐块接收响应头
     ├─ CURLOPT_PROGRESSFUNCTION ← 逐块监控下载进度/大小
     └─ CURLOPT_RETURNTRANSFER   ← 完整 body 存入内存字符串
     ↓
[内存] 完整 HTML/XML 字符串
     ↓
[解析层] 整体解析 (非流式)
     ├─ str_get_html()         ← simple_html_dom: 一次性 DOM 树构建
     ├─ simplexml_load_string() ← SimpleXML: 一次性对象树构建
     ├─ DOMDocument::loadHTML() ← DOMDocument: 一次性 DOM 树构建
     └─ json_decode()          ← JSON: 一次性数组/对象构建
     ↓
可用的解析结果
```

**关键结论**: chunked encoding 只在传输阶段由 cURL 处理，解析层始终接收完整字符串。

#### 2.6.2 传输层流式处理挂载点

**挂载点 1: `CURLOPT_HEADERFUNCTION` - 响应头逐块回调**

**位置**: `lib/http.php:146-168`

```php
$responseStatusLines = [];
$responseHeaders = [];
curl_setopt($ch, CURLOPT_HEADERFUNCTION, function ($ch, $rawHeader) use (&$responseHeaders, &$responseStatusLines) {
    $len = strlen($rawHeader);
    if ($rawHeader === "\r\n") {
        // 空行 = header 结束标记，直接跳过
        return $len;
    }
    if (preg_match('#^HTTP/(2|1.1|1.0)#', $rawHeader)) {
        // 状态行 (可能有多条，如 100 Continue 之后是 200 OK)
        $responseStatusLines[] = trim($rawHeader);
        return $len;
    }
    $header = explode(':', $rawHeader);
    if (count($header) === 1) {
        return $len;
    }
    $name = mb_strtolower(trim($header[0]));
    $value = trim(implode(':', array_slice($header, 1)));
    if (!isset($responseHeaders[$name])) {
        $responseHeaders[$name] = [];
    }
    $responseHeaders[$name][] = $value;
    return $len;  // ⚠️ 必须返回读取的字节数，否则 cURL 会中止传输
});
```

**处理特征**:
- cURL 每收到一行 header 就调用一次回调
- 重定向时会收到多组 header（先 302，再 200）
- `Transfer-Encoding: chunked` 在此阶段已被 cURL 解码，回调收到的是完整 header 行
- 返回值错误（非 `$len`）会导致传输立即中断

**挂载点 2: `CURLOPT_PROGRESSFUNCTION` - 下载进度回调**

**位置**: `lib/http.php:122-131`

```php
if ($config['max_filesize']) {
    curl_setopt($ch, CURLOPT_MAXFILESIZE, $config['max_filesize']);
    curl_setopt($ch, CURLOPT_NOPROGRESS, false);
    curl_setopt($ch, CURLOPT_PROGRESSFUNCTION, function ($ch, $downloadSize, $downloaded, $uploadSize, $uploaded) use ($config) {
        // 回调被 cURL 频繁调用（每个 chunk 至少一次）
        if ($downloaded > $config['max_filesize']) {
            // 返回非零值 → cURL 立即中止下载
            return -1;
        }
        return 0;
    });
}
```

**双保险机制**:
1. **`CURLOPT_MAXFILESIZE`**: 仅检查 `Content-Length` 头，对 chunked 响应无效（无 Content-Length）
2. **进度回调**: 检查实际已下载字节数，对 chunked 响应也有效

**chunked 传输下的行为**:
- `$downloadSize` 在 chunked 模式下通常为 0（未知总大小）
- `$downloaded` 实时更新为已接收的字节数
- 超过限制时返回 `-1`，cURL 返回 `CURLE_ABORTED_BY_CALLBACK` 错误

#### 2.6.3 传输重试机制（透明容错）

**位置**: `lib/http.php:170-192`

```php
// This retry logic is a bit hard to understand, but it works
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        // 网络调用成功，跳出循环
        break;
    }
    if ($tries <= $config['retries']) {  // 默认 retries = 2
        // 失败，重试（不重新创建 cURL handle）
        continue;
    }
    // 达到最大重试次数，抛出异常
    $curl_error = curl_error($ch);
    $curl_errno = curl_errno($ch);
    throw new HttpException(sprintf(
        'cURL error %s: %s (%s) for %s',
        $curl_error,
        $curl_errno,
        'https://curl.haxx.se/libcurl/c/libcurl-errors.html',
        $url
    ));
}
```

**重试触发条件**（`curl_exec() === false`）:
- 网络超时
- DNS 解析失败
- TCP 连接断开
- chunked 传输中连接中断
- SSL 握手失败
- 进度回调返回非零（主动中止不算失败，因为 body 已接收部分）

**重试不触发条件**:
- HTTP 4xx / 5xx 状态码（此时 `curl_exec()` 返回 body，不是 `false`）
- `max_filesize` 通过 `Content-Length` 头拒绝（请求根本没发出去）

#### 2.6.4 解析层：无流式解析的设计权衡

**所有解析器均为整体加载**：

| 解析器 | 入口函数 | 是否流式 | 内存行为 |
|--------|----------|---------|----------|
| simple_html_dom | `str_get_html($html)` | ❌ 否 | 完整 DOM 树驻留内存 |
| SimpleXML | `simplexml_load_string($xml)` | ❌ 否 | 对象树代理，底层 libxml 节点 |
| DOMDocument | `loadHTML($html)` | ❌ 否 | 完整 DOM 树，libxml 管理 |
| JSON | `json_decode($json)` | ❌ 否 | 完整数组/对象树 |

**为何不使用流式解析器**：
1. **RSS-Bridge 处理规模**: 文章列表页通常 < 1MB，详情页 < 5MB，整体加载无压力
2. **`MAX_FILE_SIZE` 限制**: simple_html_dom 默认 600KB 上限，防止超大文档
3. **`http.max_filesize` 配置**: 传输层硬限制，超限即中止
4. **解析 API 便捷性**: DOM 树可任意 `find()` 查询，流式解析需要手动维护状态机

**替代的流式处理方案**:
- `XMLReader` (PHP 扩展) - 真正的流式 XML 解析，但 RSS-Bridge 未使用
- `fopen()` + stream wrappers - 用于 MIME 类型检测等小文件处理

#### 2.6.5 MIME 类型检测的逐行流式处理

**位置**: `lib/utils.php:183-217`

```php
// 读取 /etc/mime.types 时使用 fgets() 逐行处理，避免一次性加载大文件
if (file_exists('/etc/mime.types')) {
    $file = fopen('/etc/mime.types', 'r');
    while (($line = fgets($file)) !== false) {
        $line = trim(preg_replace('/#.*/', '', $line));
        if (!$line) {
            continue;
        }
        $parts = preg_split('/\s+/', $line);
        if (count($parts) > 1) {
            $type = array_shift($parts);
            foreach ($parts as $part) {
                $typeMaps[$part] = $type;
            }
        }
    }
    fclose($file);
}
```

**这是项目中少数真正使用流式读取的场景**，但目的是读取系统配置文件而非网络响应。

#### 2.6.6 Chunked Transfer Encoding 的完整处理链

```
服务器发送 chunked 响应:
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/html

5\r\n        ← chunk 1: 5 字节
Hello\r\n
6\r\n        ← chunk 2: 6 字节
World!\r\n
0\r\n        ← 结束 chunk
\r\n

     ↓ cURL 自动解码 (CURLOPT_RETURNTRANSFER)
     ↓ HeaderFunction 被调用 4 次 (HTTP/1.1, Transfer-Encoding, Content-Type, 空行)
     ↓ ProgressFunction 被调用 N 次 (每个 chunk 更新 $downloaded)

存储到 $body 变量: "HelloWorld!"  (已去除 chunk size 和 \r\n)
     ↓
FeedParser / simple_html_dom / json_decode 整体解析
```

**cURL 自动处理的协议细节**:
- Chunk size 解析和去除
- `\r\n` 分隔符去除
- 最后 zero-size chunk 识别
- gzip/deflate 解压（`CURLOPT_ENCODING = ''` 自动协商）

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

### 4.5 跨时区与夏令时的时间处理代码挂载点

RSS-Bridge 的时间处理采用分层挂载设计，从系统入口到具体 Bridge 形成完整的时区处理链。

#### 4.5.1 时区挂载点总览

```
系统启动入口
     ↓ 挂载点 1: index.php:61
date_default_timezone_set(Configuration::getConfig('system', 'timezone'))
     ↓ 挂载点 2: Configuration::verifyInstallation()
时区合法性校验（timezone_identifiers_list 白名单）
     ↓
┌─────────────────────────────────────────────────────┐
│  挂载点 3: FeedParser 内部 strtotime()              │
│  使用全局时区解析 Feed 中的 <updated>, <pubDate>     │
├─────────────────────────────────────────────────────┤
│  挂载点 4: Bridge 自定义时间处理                     │
│  ├─ TestFaktaBridge: DateTimeZone('Europe/Stockholm')│
│  ├─ MastodonBridge: setTimezone(new DateTimeZone('GMT')) │
│  ├─ VkBridge: 手动年份/月份推断 + date() 默认时区    │
│  └─ XenForoBridge: 直接使用 data-time Unix 时间戳    │
├─────────────────────────────────────────────────────┤
│  挂载点 5: getContents() 缓存时间处理                │
│  ├─ Last-Modified 头解析 → DateTimeImmutable        │
│  └─ If-Modified-Since 协商缓存 → strtotime()        │
└─────────────────────────────────────────────────────┘
     ↓
FeedItem['timestamp'] (Unix 时间戳，无时区概念)
```

#### 4.5.2 挂载点 1: 系统时区入口

**位置**: `index.php:61`

```php
date_default_timezone_set(Configuration::getConfig('system', 'timezone'));
```

**作用**: 设置 PHP 运行时全局时区，影响所有 `strtotime()`, `date()`, `new DateTime()` 等函数的默认行为。

**默认值**: `config.default.ini.php:32` → `timezone = "UTC"`

#### 4.5.3 挂载点 2: 时区合法性校验

**位置**: `lib/Configuration.php:92-97`

```php
if (
    !is_string(self::getConfig('system', 'timezone'))
    || !in_array(self::getConfig('system', 'timezone'), timezone_identifiers_list(DateTimeZone::ALL_WITH_BC))
) {
    self::throwConfigError('system', 'timezone');
}
```

**白名单机制**: 使用 `timezone_identifiers_list(DateTimeZone::ALL_WITH_BC)` 获取 PHP 支持的所有时区标识符（包括历史时区），确保配置的时区合法。

**可配置方式**:
- 配置文件: `config.ini.php` 中 `[system] timezone = "Asia/Shanghai"`
- 环境变量: `RSSBRIDGE_system_timezone=Europe/Berlin`
- 默认值: `UTC`

#### 4.5.4 挂载点 3: FeedParser 隐式时区使用

**位置**: `lib/FeedParser.php:98, 203, 242`

```php
// Atom
$item['timestamp'] = strtotime((string)$feedItem->updated);

// RSS 2.0
$item['timestamp'] = strtotime((string) $item['timestamp']);

// RDF
$item['timestamp'] = strtotime((string)$dc->date);
```

**关键特性**:
- `strtotime()` 会自动识别时间字符串中的时区信息（如 `+0000`, `Z`, `Europe/London`）
- 如果时间字符串无时区信息，使用 `date_default_timezone_set()` 设置的全局时区
- 标准 Feed 格式（Atom/RSS）通常包含时区偏移，因此解析结果是准确的

#### 4.5.5 挂载点 4: Bridge 级自定义时区处理

**模式 A: 明确指定源时区**

**位置**: `bridges/TestFaktaBridge.php:44-47`

```php
$dateValue = DateTime::createFromFormat(
    'd M, Y',
    trim($dateString),
    new DateTimeZone('Europe/Stockholm')  // 源数据时区
);
```

**模式 B: 输出时强制转换时区**

**位置**: `bridges/MastodonBridge.php:256-259`

```php
$d = new DateTime();
$d->setTimezone(new DateTimeZone('GMT'));  // 强制 GMT 时区
$date = $d->format('D, d M Y H:i:s e');
```

**模式 C: 未知时区的保守处理**

多个 Bridge 采用此模式 (`XenForoBridge.php:372`, `NasestrechaBridge.php:88`, `JustETFBridge.php:124`):

```php
/**
 * We don't know the timezone, so just assume +00:00 (or whatever
 * DateTime chooses)
 */
```

**模式 D: 相对时间的时区依赖**

**位置**: `bridges/VkBridge.php:490-504`

```php
// 处理 "12:34" (今天/昨天的时间)
if ($date['day'] && !$date['month']) {
    // date() 使用全局时区获取当前日期
    $strdate = date('d-m-Y') . ' ' . $strdate;
}
// 处理 "5 мар" (今年的日期，需判断是否跨年)
elseif ($date['month'] && !$date['year']) {
    // 如果当前月份 < 帖子月份，说明是去年
    if (intval(date('m')) < $date['month']) {
        $strdate = $strdate . ' ' . (date('Y') - 1);
    } else {
        $strdate = $strdate . ' ' . date('Y');
    }
}
```

> **风险点**: `date('m')` 和 `date('Y')` 依赖全局时区。如果网站服务器时区与用户设置的 RSS-Bridge 时区不一致，跨年判断可能出错。

**模式 E: 直接使用 Unix 时间戳（无时区问题）**

**位置**: `bridges/XenForoBridge.php:235-237`

```php
if ($timestamp = $post->find('abbr.DateTime', 0)) {
    // <abbr class="DateTime" data-time="1660569120" ...>
    $item['timestamp'] = $timestamp->getAttribute('data-time');
}
```

#### 4.5.6 挂载点 5: 缓存协商中的时间处理

**位置**: `middlewares/CacheMiddleware.php:28-38`

```php
$ifModifiedSince = $request->server('HTTP_IF_MODIFIED_SINCE');
$lastModified = $cachedResponse->getHeader('last-modified');
if ($ifModifiedSince && $lastModified) {
    $lastModified = new \DateTimeImmutable($lastModified);
    $lastModifiedTimestamp = $lastModified->getTimestamp();
    $modifiedSince = strtotime($ifModifiedSince);
    // 比较时间戳（已转换为 UTC，无时区问题）
    if ($lastModifiedTimestamp <= $modifiedSince) {
        return new Response('', 304, ['last-modified' => gmdate('D, d M Y H:i:s ', $lastModifiedTimestamp) . 'GMT']);
    }
}
```

**关键点**:
- HTTP 日期头 (`Last-Modified`, `If-Modified-Since`) 按 RFC 7231 规定总是 GMT
- 使用 `gmdate()` 输出时强制 GMT 时区，避免时区问题

#### 4.5.7 夏令时处理边界

**PHP 夏令时自动处理**:
- `strtotime("2023-03-26 02:30:00", "Europe/London")` - 时钟向前拨 1 小时，这个时间不存在，PHP 会自动调整为 03:30
- `strtotime("2023-10-29 01:30:00", "Europe/London")` - 时钟向后拨 1 小时，这个时间出现两次，PHP 会取第一个

**夏令时相关 Bug 挂载点**:
1. **`createFromFormat()` 不带时区** (`WebfailBridge.php:93`):
   ```php
   $dt = DateTime::createFromFormat('!d.m.Y', $matches[1]);
   // '!' 前缀会将时间部分设为 00:00:00
   // 若当天是夏令时切换日且正好跳过 00:00，会解析失败
   ```

2. **VkBridge 跨年判断** (`bridges/VkBridge.php:504`):
   ```php
   return strtotime($date['day'] . '-' . $date['month'] . '-' . $date['year'] . ' ' . $strdate);
   // 若拼接出的时间正好在夏令时切换间隙，结果可能偏差 1 小时
   ```

3. **`format()` 输出时的时区转换**:
   ```php
   // 时间戳是 UTC，但输出时会转换为全局时区
   date('Y-m-d H:i:s', $timestamp);
   ```

#### 4.5.8 时区与夏令时处理最佳实践

| 场景 | 推荐做法 | 反模式 |
|------|----------|--------|
| 解析含时区的时间 | `strtotime()` 自动处理 | 手动截取时区偏移 |
| 解析已知源时区的时间 | `DateTime::createFromFormat($format, $time, new DateTimeZone($sourceTz))` | 假设源时区 = 全局时区 |
| 输出 HTTP 头 | `gmdate('D, d M Y H:i:s', $ts) . 'GMT'` | `date()` 输出本地时间 |
| 相对时间计算 | 使用 `DateTime::modify()` | 手动加减秒数 |
| 存储时间 | 存 Unix 时间戳（整数） | 存带时区的字符串 |

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

## 六、缓存对解析结果的影响边界

RSS-Bridge 采用四层缓存架构，每层缓存都可能影响解析结果，理解各层的边界条件对于调试和优化至关重要。

### 6.1 四层缓存架构总览

```
用户请求
     ↓
┌─────────────────────────────────────────────────────────┐
│  缓存层 1: CacheMiddleware (HTTP 响应缓存)              │
│  Key: 'http_' + json_encode($request->toArray())        │
│  TTL: 5分钟 + 随机抖动(错误响应) / 由 DisplayAction 管理 │
│  影响: 直接返回完整响应，跳过所有解析逻辑               │
├─────────────────────────────────────────────────────────┤
│  缓存层 2: getContents() (HTTP 响应缓存)                │
│  Key: 'server_' + $url + md5($requestBody)              │
│  TTL: 10天 (86400 * 10)                                 │
│  影响: 影响 HTML 原始内容，进而影响所有下游解析         │
├─────────────────────────────────────────────────────────┤
│  缓存层 3: getSimpleHTMLDOMCached() (页面内容缓存)      │
│  Key: 'pages_' + $url                                   │
│  TTL: 默认 24小时 (86400)，可自定义                    │
│  影响: DOM 解析结果缓存，跳过 HTTP 请求和 str_get_html() │
├─────────────────────────────────────────────────────────┤
│  缓存层 4: BridgeAbstract::saveCacheValue() (自定义缓存)│
│  Key: $bridgeShortName + '_' + $key                     │
│  TTL: 默认 1天 (86400)，可自定义                        │
│  影响: 由具体 Bridge 控制，如 Twitter API token 等      │
└─────────────────────────────────────────────────────────┘
     ↓
解析结果输出
```

### 6.2 缓存层 1: CacheMiddleware - HTTP 响应缓存

**位置**: `middlewares/CacheMiddleware.php:14-64`

```php
public function __invoke(Request $request, $next): Response
{
    $action = $request->getAttribute('action');
    if ($action !== 'DisplayAction') {
        return $next($request);  // 只缓存 DisplayAction
    }

    $cacheKey = 'http_' . json_encode($request->toArray());
    $cachedResponse = $this->cache->get($cacheKey);

    if ($cachedResponse) {
        // 304 协商缓存判断
        $ifModifiedSince = $request->server('HTTP_IF_MODIFIED_SINCE');
        $lastModified = $cachedResponse->getHeader('last-modified');
        if ($ifModifiedSince && $lastModified) {
            $lastModifiedTimestamp = strtotime($lastModified);
            $modifiedSince = strtotime($ifModifiedSince);
            if ($lastModifiedTimestamp <= $modifiedSince) {
                return new Response('', 304, ['last-modified' => gmdate('D, d M Y H:i:s ', $lastModifiedTimestamp) . 'GMT']);
            }
        }
        return $cachedResponse;  // ⚠️ 直接返回缓存，跳过所有解析！
    }

    $response = $next($request);

    // 错误响应缓存策略
    if (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
        // 5分钟 + 1~600秒随机抖动，防止缓存击穿
        $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
    }
    // 1% 概率触发缓存清理
    if (rand(1, 100) === 1) {
        $this->cache->prune();
    }

    return $response;
}
```

**影响边界**:
- ✅ **缓存命中时**: 完全跳过 Bridge 的 `collectData()`、HTML 解析、路径修正、时间处理等所有逻辑
- ❌ **缓存未命中时**: 正常执行完整解析流程
- ⚠️ **缓存键包含**: 完整请求参数（action, bridge, 所有查询参数）
- ⚠️ **缓存排除**: 非 DisplayAction（如 frontpage, list, detect）不缓存

**解析结果影响场景**:
| 场景 | 缓存命中行为 | 对用户的影响 |
|------|-------------|-------------|
| Bridge 代码已更新但缓存未过期 | 返回旧解析结果 | 用户看到旧内容，最长 5 分钟 |
| 源网站内容已更新但缓存未过期 | 返回旧解析结果 | 用户看到旧内容，最长取决于各层 TTL |
| 源网站返回错误被缓存 | 继续返回错误 | 服务暂时不可用，5+分钟 |
| 请求参数有微小差异（如大小写） | 视为不同缓存键 | 重复解析，浪费资源 |

### 6.3 缓存层 2: getContents() - HTTP 响应缓存

**位置**: `lib/contents.php:36-138`

```php
function getContents(string $url, ...) {
    // 缓存键包含请求体哈希，支持 POST 请求
    $requestBodyHash = isset($curlOptions[CURLOPT_POSTFIELDS]) 
        ? md5(Json::encode($curlOptions[CURLOPT_POSTFIELDS], false)) 
        : null;
    $cacheKey = implode('_', ['server', $url, $requestBodyHash]);

    $cachedResponse = $cache->get($cacheKey);
    if ($cachedResponse) {
        // 🔄 协商缓存：附加 If-Modified-Since 和 If-None-Match
        $lastModified = $cachedResponse->getHeader('last-modified');
        if ($lastModified) {
            try {
                $lastModified = new \DateTimeImmutable(
                    (is_numeric($lastModified) ? '@' : '') . $lastModified
                );
                $config['if_not_modified_since'] = $lastModified->getTimestamp();
            } catch (Exception $e) { /* 忽略 */ }
        }
        $etag = $cachedResponse->getHeader('etag');
        if ($etag) {
            $httpHeadersNormalized['if-none-match'] = $etag;
        }
    }

    $response = $httpClient->request($url, $config);

    switch ($response->getCode()) {
        case 200:
        case 201:
        case 202:
            // 除非服务器明确禁止缓存，否则缓存 10 天
            $cacheControl = $response->getHeader('cache-control');
            if ($cacheControl) {
                $directives = explode(',', $cacheControl);
                $directives = array_map('trim', $directives);
                if (in_array('no-cache', $directives) || in_array('no-store', $directives)) {
                    break;  // 不缓存
                }
            }
            $cache->set($cacheKey, $response, 86400 * 10);  // 10 天
            break;
        case 304:
            // Not Modified - 使用缓存的 body
            $response = $response->withBody($cachedResponse->getBody());
            break;
    }

    return $returnFull ? $response : $response->getBody();
}
```

**关键机制**:
1. **协商缓存**: 有缓存时发条件请求，304 时复用缓存体
2. **缓存键**: `server_ + URL + 请求体哈希`，区分 POST 请求
3. **TTL**: 10 天，但受 `Cache-Control` 头约束
4. **服务器 Cache-Control 优先级**: `no-cache` / `no-store` 指令会阻止缓存

**解析结果影响场景**:
| 场景 | 行为 | 对解析的影响 |
|------|------|-------------|
| 源站内容 10 天内更新但未发 304 | 返回缓存内容 | HTML 解析基于旧内容，结果过时 |
| 源站修复了 HTML 错误但缓存未过期 | 继续使用有问题的 HTML | DOM 解析仍然出错 |
| 源站返回 304 Not Modified | 使用缓存的 body | 解析结果不变，但节省带宽 |
| 源站 Cache-Control: no-store | 每次都请求新内容 | 总能获取最新，但性能下降 |

### 6.4 缓存层 3: getSimpleHTMLDOMCached() - 页面内容缓存

**位置**: `lib/contents.php:219-251`

```php
function getSimpleHTMLDOMCached(
    $url,
    $ttl = 86400,  // 默认 24 小时
    $header = [],
    ...
): \simple_html_dom {
    global $container;
    $cache = $container['cache'];

    $cacheKey = 'pages_' . $url;
    $content = $cache->get($cacheKey);
    if (!$content) {
        // ⚠️ 缓存未命中时调用 getContents()，这会触发缓存层 2
        $content = getContents($url, $header ?? [], $opts ?? []);
        $cache->set($cacheKey, $content, $ttl);
    }
    // 🔍 每次都重新解析 DOM！缓存的是原始 HTML 字符串
    return str_get_html($content, $lowercase, $forceTagsClosed, ...);
}
```

**重要特性**:
- **缓存的是 HTML 字符串，不是 DOM 对象** - 每次调用都会重新 `str_get_html()`
- **TTL 可自定义** - 不同 Bridge 可根据内容更新频率设置
- **与缓存层 2 的关系**: 缓存未命中时会调用 `getContents()`，可能命中层 2 缓存

**解析结果影响场景**:
| 场景 | 行为 | 对解析的影响 |
|------|------|-------------|
| `forceTagsClosed` 参数改变 | 用缓存的 HTML 重新解析 | 可能得到不同的 DOM 结构 |
| simple_html_dom 库升级 | 用缓存的 HTML 重新解析 | 解析结果可能变化 |
| 相同 URL，不同的解析参数 | 共享缓存，参数各自生效 | TTL 内多次调用，HTML 相同但 DOM 解析方式可不同 |
| 解析依赖当前时间（如相对日期） | HTML 不变但时间解析变 | 每次调用时间字段可能不同 |

### 6.5 缓存层 4: Bridge 自定义缓存

**位置**: `lib/BridgeAbstract.php:325-333`

```php
protected function loadCacheValue(string $key, $default = null)
{
    return $this->cache->get($this->getShortName() . '_' . $key, $default);
}

protected function saveCacheValue(string $key, $value, int $ttl = 86400)
{
    $this->cache->set($this->getShortName() . '_' . $key, $value, $ttl);
}
```

**使用示例**:

```php
// TwitterClient.php - 缓存 API Token
$data = $this->cache->get('twitter') ?? [];
// ...
$this->cache->set('twitter', $this->data);

// YoutubeBridge.php - 缓存搜索结果
if ($this->cache->get($cacheKey)) {
    // 缓存命中，跳过 API 请求
}
$this->cache->set($cacheKey, true, 60 * 16);  // 缓存 16 分钟
```

**解析结果影响场景**:
- 缓存的 API Token 过期 → 解析失败
- 缓存的元数据（如分类映射）过时 → 分类信息错误
- 缓存的增量同步标记 → 漏掉新内容

### 6.6 缓存一致性边界与失效条件

#### 6.6.1 缓存穿透（Cache Miss Storm）

**场景**: 缓存过期瞬间大量请求涌入，都绕过缓存直接请求源站。

**缓解措施**:
- 随机 TTL 抖动（CacheMiddleware 错误响应 +1~600 秒）
- 四层缓存架构形成渐变失效（层 1: 5分钟 → 层 3: 24小时 → 层 2: 10天）
- 1% 概率 `prune()` 主动清理过期缓存

#### 6.6.2 缓存击穿（Hot Key Invalid）

**场景**: 热门 Bridge 的缓存同时失效，导致源站压力骤增。

**代码特征**:
- 各层 TTL 差异设计（5分钟 vs 24小时 vs 10天）避免同时失效
- 协商缓存（304 Not Modified）降低回源压力

#### 6.6.3 缓存污染（Cache Poisoning）

**场景**: 源站返回错误或异常内容被缓存。

**风险点**:
```php
// getContents() 缓存所有 2xx 响应，包括内容错误的页面
case 200:
case 201:
case 202:
    $cache->set($cacheKey, $response, 86400 * 10);  // 缓存 10 天！
    break;
```

**影响**: 如果源站返回 200 OK 但内容是错误页面（如 Cloudflare 拦截页），会被缓存 10 天。

#### 6.6.4 缓存键冲突边界

**CacheMiddleware 键**: `'http_' + json_encode($request->toArray())`
- 包含所有查询参数 → 参数顺序不同会生成不同键
- 参数值大小写敏感 → `?u=user` 和 `?u=User` 是不同键

**getContents 键**: `'server_' + $url + $requestBodyHash`
- URL 大小写敏感 → 大多数 HTTP 服务器不区分大小写，但这里区分
- 锚点 (`#section`) 会被包含 → 实际对 HTTP 请求无影响

**getSimpleHTMLDOMCached 键**: `'pages_' + $url`
- 不包含 header 和 opts 参数 → 相同 URL 不同 header 共享缓存

### 6.7 解析结果可重现性边界

| 条件 | 相同输入是否得到相同输出 | 原因 |
|------|------------------------|------|
| 所有缓存都命中 | ✅ 是 | 完全相同的输入，跳过所有可变逻辑 |
| 缓存层 1 未命中，其他命中 | ⚠️ 大概率是 | 除非解析依赖当前时间或随机数 |
| 缓存层 3 未命中 | ⚠️ 可能不同 | 重新解析 DOM，`forceTagsClosed` 等参数可能影响 |
| 所有缓存都未命中 | ❌ 可能不同 | 源站内容可能已变，时间戳基于当前时间 |
| 解析依赖 `now()` 或 `date()` | ❌ 否 | 每次调用时间不同 |
| 解析依赖随机数 (`rand()`) | ❌ 否 | 随机值不同 |

### 6.8 Multi-Bridge 执行时的缓存键冲突分析

RSS-Bridge 中存在多种多 Bridge 执行场景，不同场景下缓存键冲突的风险和影响各不相同。

#### 6.8.1 Multi-Bridge 执行场景总览

| 场景 | 执行方式 | 并发/顺序 | 缓存冲突风险 |
|------|----------|----------|-------------|
| **FeedMergeBridge** | 单进程内顺序执行 1-10 个 Feed | 顺序 | ⚠️ 中 |
| **DetectAction** | 遍历所有 Bridge 检测 URL 匹配 | 顺序 | ✅ 低 |
| **批量请求** | 用户发起多次独立请求 | 并发（多进程） | ⚠️ 中 |
| **同一 Bridge 多上下文** | 单 Bridge 多个 context 参数 | 同一进程内 | ❌ 高 |

#### 6.8.2 FeedMergeBridge 缓存键冲突路径

**位置**: `bridges/FeedMergeBridge.php:62-87`

```php
foreach ($feeds as $feed) {
    if (count($feeds) > 1) {
        try {
            $this->collectExpandableDatas($feed, 10);
        } catch (HttpException $e) {
            // 容错：单个 feed 失败不影响整体
            continue;
        }
    } else {
        $this->collectExpandableDatas($feed, 10);
    }
}
```

**执行流程**:
```
FeedMergeBridge::collectData()
     ↓
循环 10 个 feed URL
     ↓
collectExpandableDatas(feed_url, 10)
     ↓ 每层都可能命中缓存
getContents(feed_url)         → 缓存键: server_ + feed_url
     ↓
FeedParser::parseFeed()       → SimpleXML 对象，无缓存
     ↓
parseItem($item)              → 具体 Bridge 自定义逻辑
     ↓ 若 Bridge 内部抓取详情页
getSimpleHTMLDOMCached(item_url) → 缓存键: pages_ + item_url
     ↓
$this->items[] = $item        → 累加到同一数组
```

**缓存冲突分析**:

| 缓存层 | 是否冲突 | 原因 | 影响 |
|--------|---------|------|------|
| **CacheMiddleware** | 不冲突 | FeedMerge 是单个请求，只有一个缓存键 | 无 |
| **getContents** | 不冲突 | 不同 feed 有不同 URL → 不同缓存键 | 无 |
| **getSimpleHTMLDOMCached** | ⚠️ 可能冲突 | 不同 feed 可能有相同的文章 URL（转载） | 共享缓存，节省资源 |
| **Bridge 自定义缓存** | ⚠️ 可能冲突 | 缓存键包含 `getShortName()`，但同一 Bridge 相同 key 会冲突 | 不同 feed 共享同一份缓存数据 |

**隐藏风险点**: `FeedMergeBridge` 本身继承自 `FeedExpander`，如果合并的是两个 WordPressBridge 的 feed，那在 parseItem 时... 等等，不对，FeedMergeBridge 直接用 FeedExpander 的 parseItem（默认返回原 item），不会触发 WordPressBridge 的 parseItem。

**真实风险**: 如果两个 feed 中包含相同的文章 URL，`getSimpleHTMLDOMCached` 会共享缓存。这通常是好事，但如果两个 feed 的文章内容虽然 URL 相同但实际内容不同（如 A/B 测试），就会出现内容不一致。

#### 6.8.3 DetectAction 遍历所有 Bridge

**位置**: `actions/DetectAction.php:25-45`

```php
foreach ($this->bridgeFactory->getBridgeClassNames() as $bridgeClassName) {
    if (!$this->bridgeFactory->isEnabled($bridgeClassName)) {
        continue;
    }

    $bridge = $this->bridgeFactory->create($bridgeClassName);
    $bridgeParams = $bridge->detectParameters($url);

    if (!$bridgeParams) {
        continue;
    }
    // 找到第一个匹配的就重定向
    $query = ['action' => 'display', 'bridge' => $bridgeClassName, 'format' => $format];
    $query = array_merge($query, $bridgeParams);
    return new Response('', 301, ['location' => '?' . http_build_query($query)]);
}
```

**缓存冲突分析**:
- **CacheMiddleware**: DetectAction 不走 DisplayAction 缓存（`$action !== 'DisplayAction'` 时跳过）
- **getContents**: 大多数 `detectParameters()` 实现不发起 HTTP 请求，只做 URL 正则匹配 → 无缓存
- **结论**: 缓存冲突风险极低

#### 6.8.4 同一 Bridge 多上下文的缓存键冲突

**位置**: `lib/BridgeAbstract.php:145, 325-333`

```php
// 输入参数处理
unset($input['context']);  // context 从参数中移除

// 自定义缓存键
protected function loadCacheValue(string $key, $default = null)
{
    return $this->cache->get($this->getShortName() . '_' . $key, $default);
}

protected function saveCacheValue(string $key, $value, int $ttl = 86400)
{
    $this->cache->set($this->getShortName() . '_' . $key, $value, $ttl);
}
```

**风险点**:
- `saveCacheValue()` 的缓存键只包含 `getShortName()` + `$key`，**不包含 context**
- 如果同一 Bridge 有多个 context，且使用相同的 key 保存缓存，会发生**缓存互相覆盖**

**实际案例**: `PepperBridgeAbstract.php:270-277`

```php
$cacheKey = $this->getInput('url') . 'TITLE';  // ⚠️ 用 URL 做缓存键，不包含 context
$title = $this->loadCacheValue($cacheKey);
// ...
$this->saveCacheValue($cacheKey, $title, 86400 * 15);
```

> **注意**: PepperBridgeAbstract 用 `getInput('url')` 做 key 的一部分，实际上避免了 context 冲突。但如果 Bridge 只用固定字符串做 key，就会有问题。

#### 6.8.5 多进程并发请求的缓存键冲突

**场景**: 多个用户同时请求同一个 Bridge，或同一个用户刷新页面

**四层缓存的并发行为**:

| 缓存层 | 并发安全性 | 冲突表现 |
|--------|-----------|----------|
| **CacheMiddleware** | 安全 | 相同请求得到相同缓存，无冲突 |
| **getContents** | 基本安全 | 缓存击穿：缓存失效瞬间大量请求穿透到源站 |
| **getSimpleHTMLDOMCached** | 基本安全 | 同上，缓存击穿问题 |
| **Bridge 自定义缓存** | ⚠️ 视实现而定 | 如果 saveCacheValue 非原子操作，可能出现竞态条件 |

**缓存击穿路径** (`getContents` 为例):
```
T=0: 缓存键 server_example.com 即将过期
T=0.1: 请求 A 到达，缓存 miss → 发起 HTTP 请求
T=0.2: 请求 B 到达，缓存 miss → 发起 HTTP 请求
T=0.3: 请求 C 到达，缓存 miss → 发起 HTTP 请求
...
T=1.5: 请求 A 收到响应，写入缓存
T=1.6: 请求 B 收到响应，覆盖缓存
T=1.7: 请求 C 收到响应，覆盖缓存
```

**结果**: 缓存失效瞬间，N 个并发请求会穿透到源站，造成瞬时压力。

#### 6.8.6 缓存键冲突的真实 Bug 挂载点

1. **getSimpleHTMLDOMCached 相同 URL 不同 Header**

   **位置**: `lib/contents.php:236`
   ```php
   $cacheKey = 'pages_' . $url;  // ⚠️ 只包含 URL，不包含 header
   ```
   **场景**: 同一 URL 用不同的 `Accept` header 请求（如 HTML vs JSON）
   **后果**: 第一个请求缓存了 HTML，第二个请求会拿到 HTML 而不是期望的 JSON

2. **SpotifyBridge 用 clientid 做缓存键**

   **位置**: `bridges/SpotifyBridge.php:152-154`
   ```php
   $cacheKey = sprintf('SpotifyBridge:%s:%s', $this->getInput('clientid'), $this->getInput('clientsecret'));
   $token = $this->cache->get($cacheKey);
   ```
   **问题**: clientsecret 不应出现在缓存键中（安全考虑），且缓存键可能过长

3. **YoutubeBridge rate_limit 缓存**

   **位置**: `bridges/YoutubeBridge.php:76-84`
   ```php
   $cacheKey = 'youtube_rate_limit';  // ⚠️ 全局共享，不分用户/IP
   if ($this->cache->get($cacheKey)) {
       // 触发限流
   }
   $this->cache->set($cacheKey, true, 60 * 16);
   ```
   **问题**: 一个用户触发限流后，所有用户都被限流

4. **TwitterClient 缓存键全局共享**

   **位置**: `lib/TwitterClient.php:15, 274`
   ```php
   $data = $this->cache->get('twitter') ?? [];  // ⚠️ 所有用户共享
   // ...
   $this->cache->set('twitter', $this->data);
   ```
   **问题**: guest token 全局共享，一个 token 失效影响所有用户

#### 6.8.7 缓存键设计最佳实践

| 场景 | 推荐做法 | 反模式 |
|------|----------|--------|
| 按用户隔离的数据 | 缓存键包含用户标识 | 全局共享一个缓存键 |
| 不同参数不同结果 | 缓存键包含所有影响结果的参数 | 只包含部分参数 |
| 敏感信息 | 不出现在缓存键中 | 把 token/secret 放键里 |
| 防止缓存击穿 | 加锁或预热缓存 | 完全依赖自动过期 |
| 多上下文 Bridge | 缓存键包含 context | 只使用固定 key 名称 |

### 6.9 Bridge 失败时的 Fallback 内容代码路径

RSS-Bridge 构建了从传输层到应用层的多层 fallback 体系，确保在各种失败场景下尽可能返回有意义的内容而非空白页面。

#### 6.9.1 Fallback 层级总览

```
HTTP 请求发起
     ↓
┌─────────────────────────────────────────────────────┐
│  层级 1: cURL 重试 (lib/http.php:170-192)           │
│  网络失败自动重试 2 次                               │
├─────────────────────────────────────────────────────┤
│  层级 2: 源数据 URL/格式 fallback                    │
│  ├─ WordPressBridge: /feed/atom/ 失败 → /?feed=atom │
│  ├─ FeedExpander: XML 解析异常 → 抛出异常           │
│  └─ YoutubeBridge: API 方式失败 → try/catch         │
├─────────────────────────────────────────────────────┤
│  层级 3: CSS 选择器链 fallback                       │
│  ├─ ?? 运算符链式降级 (5+ 个常见选择器依次尝试)      │
│  ├─ 多级嵌套属性访问 ?? null                        │
│  └─ find($selector, 0) ?? null → 异常处理           │
├─────────────────────────────────────────────────────┤
│  层级 4: FeedMergeBridge 单条失败跳过                │
│  catch HttpException → continue 下一条 feed         │
├─────────────────────────────────────────────────────┤
│  层级 5: DisplayAction 全局异常处理                  │
│  ├─ ClientException → 只记录 debug 日志             │
│  ├─ RateLimitException → 返回 429 错误页             │
│  ├─ HttpException 429/503 → 立即返回错误页           │
│  ├─ 其他异常 → 记录 error 日志                      │
│  └─ error.output = 'feed' → 渲染错误为 feed item    │
├─────────────────────────────────────────────────────┤
│  层级 6: ExceptionMiddleware 兜底                    │
│  Throwable → 渲染 500 错误页模板                     │
└─────────────────────────────────────────────────────┘
     ↓
最终响应 (正常内容 / 错误 feed item / 错误 HTML 页)
```

#### 6.9.2 层级 1: cURL 传输层自动重试

**位置**: `lib/http.php:170-192`

```php
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        break;  // 成功，跳出
    }
    if ($tries <= $config['retries']) {  // 默认 retries = 2
        continue;  // 失败，重试
    }
    // 2 次重试全部失败，抛出异常
    throw new HttpException(sprintf('cURL error %s: %s ...', $curl_error, $curl_errno));
}
```

**重试触发场景**:
- 网络超时、DNS 失败、TCP 断开、SSL 失败等连接层错误
- chunked 传输中途连接断开

**不重试场景**:
- HTTP 4xx/5xx（`curl_exec()` 返回 body，不是 `false`）
- `max_filesize` 超限被 `Content-Length` 头拦截

#### 6.9.3 层级 2: 源数据 URL/格式 Fallback

**模式 A: 备用 Feed URL**

**位置**: `bridges/WordPressBridge.php:30-34`

```php
try {
    $this->collectExpandableDatas($this->getURI() . '/feed/atom/', $limit);
} catch (Exception $e) {
    // 标准 Atom Feed 路径失败，尝试查询参数形式
    $this->collectExpandableDatas($this->getURI() . '/?feed=atom', $limit);
}
```

**模式 B: Cloudflare 识别与特殊异常**

**位置**: `lib/http.php:23-35`

```php
public static function fromResponse(Response $response, string $url): HttpException
{
    $message = sprintf('%s resulted in %s %s', $url, $response->getCode(), $response->getStatusLine());
    if (CloudFlareException::isCloudFlareResponse($response)) {
        // 识别出 Cloudflare 拦截页，抛专用子类
        return new CloudFlareException($message, $response->getCode(), $response);
    }
    return new HttpException(trim($message), $response->getCode(), $response);
}
```

**Cloudflare 检测特征** (`lib/http.php:40-55`):
- `<title>Just a moment...`
- `<title>Please Wait...`
- `<title>Attention Required!`
- `<title>Access denied</title>`
- 匹配到即标记为 `CloudFlareException`，上层可针对性处理

**模式 C: XML 预处理降级**

**位置**: `lib/FeedExpander.php:61-70`

```php
protected function prepareXml(string $xmlString): string
{
    $problematicStrings = [
        '&nbsp;',   // XML 中不是合法实体
        '&raquo;',
        '&rsquo;',
    ];
    return str_replace($problematicStrings, '', $xmlString);
}
```

> 这是一种"移除而不是修复"的降级策略：宁可丢失一些 HTML 实体字符，也要保证 XML 能被解析。

#### 6.9.4 层级 3: CSS 选择器链 Fallback

**模式 A: `??` 运算符链式降级**

**位置**: `bridges/WordPressBridge.php:43-61`

```php
$article = null;
switch (true) {
    case !empty($this->getInput('content-selector')):
        $article = $dom->find($this->getInput('content-selector'), 0);
        break;
    case !is_null($dom->find('[itemprop=articleBody]', 0)):
        $article = $dom->find('[itemprop=articleBody]', 0);
        break;
    case !is_null($dom->find('.article-content', 0)):
        $article = $dom->find('.article-content', 0);
        break;
    case !is_null($dom->find('article', 0)):
        $article = $dom->find('article', 0);
        break;
    // ... 共 6 级降级
}
```

**位置**: `bridges/YoutubeBridge.php:463-467`

```php
// JSON 属性访问多级 fallback
$title = $wrapper->title->runs[0]->text
      ?? $wrapper->title->accessibility->accessibilityData->label
      ?? null;

$publishedTimeText = $wrapper->publishedTimeText->simpleText
                  ?? $wrapper->videoInfo->runs[2]->text
                  ?? null;
```

**模式 B: `or throw` 断言式失败**

**位置**: `bridges/XenForoBridge.php:130,142,169,255`

```php
$titleBar = $postsBar->find('.titleBar', 0)
    or throwServerException('Error finding title bar!');
```

这利用了 PHP 的短路求值：`find()` 返回 `null` 时触发 `or` 后面的 `throwServerException()`。

#### 6.9.5 层级 4: FeedMergeBridge 单条失败跳过

**位置**: `bridges/FeedMergeBridge.php:62-87`

```php
foreach ($feeds as $feed) {
    if (count($feeds) > 1) {
        try {
            $this->collectExpandableDatas($feed, 10);
        } catch (HttpException $e) {
            // ⭐ 单条 feed 失败不影响其他，静默跳过
            continue;
        }
    } else {
        $this->collectExpandableDatas($feed, 10);
    }
}
```

**设计意图**: 多 feed 合并时局部容错，整体可用性优先。

#### 6.9.6 层级 5: DisplayAction 全局异常处理

**位置**: `actions/DisplayAction.php:73-124`

```php
try {
    $bridge->loadConfiguration();
    $bridge->setInput($input);
    $bridge->collectData();
    $items = $bridge->getItems();
} catch (\Throwable $e) {
    // 异常分类处理
    if ($e instanceof ClientException) {
        $this->logger->debug(...);          // 用户输入错误：只记 debug
    } elseif ($e instanceof RateLimitException) {
        return new Response(..., 429);      // 限流：返回 429
    } elseif ($e instanceof HttpException) {
        if (in_array($e->getCode(), [429, 503])) {
            return new Response(..., $e->getCode());  // 服务不可用：立即返回
        }
        // 其他 HTTP 错误（404 等）：正常走错误报告流程
    } else {
        $this->logger->error(...);          // 未知错误：记 error 日志
    }

    // 错误报告频率限制
    $errorCount = $this->logBridgeError($bridge->getName(), $e->getCode());
    if ($errorCount >= $reportLimit) {
        // 超过阈值，根据配置返回不同形式的错误
        if ($errorOutput === 'feed') {
            // ⭐ Fallback 内容：把异常渲染成一个 feed item
            $items = [$this->createFeedItemFromException($e, $bridge)];
        } elseif ($errorOutput === 'http') {
            return new Response(render(...), 500);  // 返回 HTML 错误页
        } elseif ($errorOutput === 'none') {
            // 静默：产生一个空 feed
        }
    }
}
```

**错误 feed item 构造** (`DisplayAction.php:146-170`):
```php
private function createFeedItemFromException($e, $bridge): array
{
    // 每 24 小时一个唯一标识符，避免 feed 阅读器识别为重复条目
    $uniqueIdentifier = urlencode((int)(time() / 86400));
    $title = sprintf('Bridge returned error %s! (%s)', $e->getCode(), $uniqueIdentifier);
    $item['title'] = $title;
    $item['uri'] = get_current_url();
    $item['timestamp'] = time();
    $item['uid'] = $bridge->getName() . '_' . $uniqueIdentifier;

    // 内容包含：异常栈 + GitHub 搜索链接 + Issue 创建链接 + 维护者
    $content = render_template('bridge-error.html.php', [
        'error' => render_template('exception.html.php', ['e' => $e]),
        'searchUrl' => self::createGithubSearchUrl($bridge),
        'issueUrl' => self::createGithubIssueUrl($bridge, $e),
        'maintainer' => $bridge->getMaintainer(),
    ]);
    $item['content'] = $content;
    return $item;
}
```

**三种错误输出模式**:
| `error.output` 配置 | 行为 | 适用场景 |
|---------------------|------|----------|
| `feed` | 异常渲染为 feed item，HTTP 200 | RSS 阅读器友好，用户能看到错误信息 |
| `http` | 返回 500 HTML 错误页 | 浏览器访问，用户能看到完整栈跟踪 |
| `none` | 返回空 feed（HTTP 200） | 静默失败，不打扰用户 |

#### 6.9.7 层级 6: ExceptionMiddleware 兜底

**位置**: `middlewares/ExceptionMiddleware.php:14-23`

```php
public function __invoke(Request $request, $next): Response
{
    try {
        return $next($request);
    } catch (\Throwable $e) {
        $this->logger->error('Exception in ExceptionMiddleware', ['e' => $e]);
        // 所有未被 DisplayAction 捕获的异常都在这里兜底
        return new Response(render(__DIR__ . '/../templates/exception.html.php', ['e' => $e]), 500);
    }
}
```

**触发场景**: DisplayAction 之外的 Action（FrontpageAction、ListAction、DetectAction 等）抛出的异常。

#### 6.9.8 Fallback 策略的设计权衡

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 自动重试 | 对瞬时网络抖动透明 | 增加延迟，可能放大源站压力 | cURL 传输层 |
| URL 降级 | 兼容不同源站配置 | 可能试错浪费资源 | WordPress 等多路径 Feed |
| 选择器链降级 | 兼容网站模板改版 | 可能选到错误内容，静默产生脏数据 | CSS 选择器定位文章 |
| 单条失败跳过 | 整体可用性优先 | 用户可能丢失部分内容而不自知 | FeedMergeBridge |
| 错误转 Feed Item | RSS 阅读器友好，有 GitHub 链接 | 非 RSS 格式输出场景无效 | `error.output = 'feed'` |
| 错误转 HTTP 500 页 | 调试信息完整 | RSS 阅读器可能忽略 | 浏览器访问 + 开发模式 |

---

## 七、关键设计模式总结

### 7.1 模板方法模式
- `BridgeAbstract::collectData()` 是抽象方法，由子类实现具体采集逻辑
- `FeedExpander::collectExpandableDatas()` 定义了 Feed 扩展的骨架流程，`parseItem()` 由子类自定义

### 7.2 工具函数门面
`html.php` 中的函数对 simplehtmldom 库进行了封装，提供更高层次的操作：
- `convertLazyLoading()` 封装了懒加载属性的查找、解析、转换
- `defaultLinkTo()` 封装了 DOM 遍历和 `urljoin()` 调用

### 7.3 不可变对象
`Url` 类使用 `with*()` 方法返回新实例，确保线程安全和可预测性。

### 7.4 防御式编程
- 多层参数校验 (`ParameterValidator`, `Url::validate()`)
- 类型检查和容错处理 (`is_string()`, `is_object()`, `??` 运算符)
- 输入净化 (`sanitize()`, `stripWithDelimiters()`)

### 7.5 缓存策略
- 内容缓存: `getSimpleHTMLDOMCached()` TTL 24 小时
- HTTP 缓存: `getContents()` 支持 `Last-Modified` / `ETag` 协商缓存
- 服务器缓存: 不同 URL 分开缓存，缓存键包含请求体哈希

---

## 八、代码优化建议

### 8.1 `defaultLinkTo()` 可扩展支持 srcset

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

### 8.2 时间解析统一封装

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

### 8.3 HTML 处理链式调用

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

### 8.4 缓存污染防护

针对 getContents() 缓存 10 天可能导致的污染问题，建议增加内容校验：

```php
// 在缓存前增加内容合法性检查
case 200:
case 201:
case 202:
    // 检查是否为有效 HTML（非错误页）
    if (strlen($response->getBody()) > 1024  // 内容足够长
        && !str_contains($response->getBody(), 'error') // 非错误页
        && !str_contains($response->getBody(), 'blocked')) { // 非拦截页
        $cache->set($cacheKey, $response, 86400 * 10);
    }
    break;
```

---

## 九、参考文件索引

| 功能 | 文件 | 关键行号 |
|------|------|----------|
| Bridge 抽象基类 | `lib/BridgeAbstract.php` | 1-339 |
| Feed 扩展器 | `lib/FeedExpander.php` | 1-86 |
| HTML 处理函数 | `lib/html.php` | 1-609 |
| URL 合并算法 | `lib/php-urljoin/src/urljoin.php` | 12-143 |
| URL 封装类 | `lib/url.php` | 14-151 |
| 工具函数 | `lib/utils.php` | 1-283 |
| 内容获取 | `lib/contents.php` | 1-251 |
| Feed 解析器 | `lib/FeedParser.php` | SimpleXML 19, 时间 98/202/242 |
| WordPressBridge 示例 | `bridges/WordPressBridge.php` | 1-129 |
| simple_html_dom 解析器 | `lib/simplehtmldom/simple_html_dom.php` | 节点结构 130-164, clear() 1598-1620, MAX_FILE_SIZE 45 |
| DOMDocument 容错 | `lib/XPathAbstract.php` | libxml 404-408 |
| 缓存中间件 | `middlewares/CacheMiddleware.php` | 1-64 |
| 时区配置入口 | `index.php` | date_default_timezone_set 61 |
| 时区合法性校验 | `lib/Configuration.php` | 92-97 |
| 时区处理示例 | `bridges/TestFaktaBridge.php` | DateTimeZone 44-47 |
| 时区处理示例 | `bridges/VkBridge.php` | 相对时间 490-504 |
| 时区处理示例 | `bridges/XenForoBridge.php` | data-time 235-237, 时区注释 372 |
| FeedMerge 多 Feed 合并 | `bridges/FeedMergeBridge.php` | 循环解析 62-87, 单条失败跳过 62-87 |
| URL 自动检测 | `actions/DetectAction.php` | 遍历 Bridge 25-45 |
| 缓存键冲突示例 | `bridges/YoutubeBridge.php` | rate_limit 全局缓存 76-84 |
| 缓存键冲突示例 | `bridges/SpotifyBridge.php` | clientid 缓存键 152-154 |
| 缓存键冲突示例 | `lib/TwitterClient.php` | 全局 twitter 缓存 15,274 |
| HTTP 客户端与流式处理 | `lib/http.php` | CurlHttpClient 63-198, 头部回调 146-168, 进度回调 122-131, 重试 170-192 |
| Cloudflare 识别 | `lib/http.php` | CloudFlareException 38-56 |
| DisplayAction 全局异常处理 | `actions/DisplayAction.php` | createResponse 69-144, 错误 Feed Item 146-170, 错误计数 172-191 |
| ExceptionMiddleware 兜底 | `middlewares/ExceptionMiddleware.php` | 1-24 |
| MIME 类型流式读取 | `lib/utils.php` | fgets 逐行 183-217 |
| 选择器链 Fallback 示例 | `bridges/YoutubeBridge.php` | JSON ?? 链 463-467 |
| 选择器链 Fallback 示例 | `bridges/XenForoBridge.php` | or throw 130,142,169,255 |
| 默认配置 | `config.default.ini.php` | timezone 32, max_filesize, error output |
