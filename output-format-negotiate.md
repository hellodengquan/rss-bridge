# RSS-Bridge 输出格式协商与编码路径分析

## 一、总览

RSS-Bridge 将同一份桥接数据（Bridge 收集的 FeedItem 列表）输出为多种 Feed 格式。整个系统分为三层：

```
┌─────────────────┐     ┌───────────────────┐     ┌─────────────────┐
│  Bridge 层      │ ──► │  FormatAbstract   │ ──► │  具体 Format    │
│  收集原始数据   │     │  FeedItem 中间层   │     │  渲染目标格式   │
└─────────────────┘     └───────────────────┘     └─────────────────┘
```

格式协商**不使用 HTTP Accept Header**，而是完全通过 URL 查询参数 `format` 指定。

---

## 二、格式协商规则

### 2.1 协商入口：DisplayAction

格式选择的起点在 `actions/DisplayAction.php:22`：

```php
$format = $request->get('format');
```

客户端必须通过查询字符串显式传入 `format` 参数，否则在第 33-35 行直接返回 400 错误：

```php
if (!$format) {
    return new Response(render(__DIR__ . '/../templates/error.html.php', ['message' => 'You must specify a format']), 400);
}
```

这意味着 RSS-Bridge 完全**不做 HTTP 内容协商**（Content Negotiation），不解析 `$_SERVER['HTTP_ACCEPT']`，格式选择是纯粹的查询参数驱动。

### 2.2 格式解析：FormatFactory

`format` 参数值交给 `lib/FormatFactory.php` 进行解析和实例化，流程如下：

**第一步：自动发现可用格式（构造函数）**

```php
// lib/FormatFactory.php:9-16
$iterator = new \FilesystemIterator(__DIR__ . '/../formats');
foreach ($iterator as $file) {
    if (preg_match('/^([^.]+)Format\.php$/U', $file->getFilename(), $m)) {
        $this->formatNames[] = $m[1];
    }
}
sort($this->formatNames);
```

扫描 `formats/` 目录，按文件名模式 `*Format.php` 提取格式名。当前支持 6 种格式：

| 格式类文件 | 格式名 | MIME 类型 | 说明 |
|---|---|---|---|
| `AtomFormat.php` | `Atom` | `application/atom+xml` | RFC 4287 Atom |
| `MrssFormat.php` | `Mrss` | `application/rss+xml` | RSS 2.0 + Media RSS |
| `JsonFormat.php` | `Json` | `application/json` | JSON Feed Version 1 |
| `HtmlFormat.php` | `Html` | `text/html` | 网页预览（含其他格式链接） |
| `PlaintextFormat.php` | `Plaintext` | `text/plain` | PHP print_r 调试输出 |
| `SfeedFormat.php` | `Sfeed` | `text/plain` | sfeed 工具的 TSV 格式 |

**第二步：名称规范化与校验**

```php
// lib/FormatFactory.php:18-29
public function create(string $name): FormatAbstract
{
    if (! preg_match('/^[a-zA-Z0-9-]*$/', $name)) {
        throw new \InvalidArgumentException('Format name invalid!');
    }
    $sanitizedName = $this->sanitizeName($name);
    // ...
    $className = '\\' . $sanitizedName . 'Format';
    return new $className();
}
```

`sanitizeName()` 做了三层归一化（`lib/FormatFactory.php:36-51`）：

1. `ucfirst(strtolower($name))` — 首字母大写，其余小写
2. 去除尾部 `.php`（兼容旧链接）
3. 去除尾部 `Format`（兼容旧链接）

这意味着 `?format=atom`、`?format=ATOM`、`?format=AtomFormat`、`?format=atom.php` 最终都解析为 `AtomFormat` 类。

规范化后与已发现的格式名单比对，未命中则抛出 `Unknown format given` 异常。

### 2.3 前端默认格式

在 `actions/FrontpageAction.php:235-240`，前端的 "Generate feed" 按钮默认提交 `format=Html`：

```php
$form .= html_tag('button', 'Generate feed', [
    'type'          => 'submit',
    'name'          => 'format',
    'value'         => 'Html',
    'formtarget'    => '_blank',
]);
```

而 `HtmlFormat` 在渲染时会额外生成其他 5 种格式的跳转链接供用户/客户端选择（`formats/HtmlFormat.php:16-33`）。

---

## 三、编码路径衔接

### 3.1 数据流全景

```
HTTP Request
     │
     ▼
DisplayAction::__invoke()
     │
     ├─► BridgeFactory::create()  ──► 具体 Bridge 类
     │                                   │
     │                                   ▼
     │                             collectData()
     │                                   │
     │                                   ▼
     │                             $items = getItems()      [原始数组]
     │
     ├─► FormatFactory::create($format)  ──► 具体 Format 类
     │
     ▼
DisplayAction::createResponse()
     │
     ├─ $format->setItems($items)         [数组 → FeedItem[]]
     ├─ $format->setFeed($bridge->getFeed())
     ├─ $format->setLastModified($now)
     │
     ├─ $body = $format->render()         [FeedItem[] → 目标格式字符串]
     │
     ├─ mb_convert_encoding($body, 'UTF-8', 'UTF-8')  [编码清洗]
     │
     ▼
new Response($body, 200, [
    'content-type' => $format->getMimeType() . '; charset=UTF-8',
    'last-modified' => gmdate('D, d M Y H:i:s ', $now) . 'GMT',
])
```

### 3.2 中间层：FeedItem

Bridge 返回的是普通 `array[]`，进入 Format 层时统一转换为 `FeedItem` 对象数组。这是编码路径衔接的关键解耦点。

**转换发生在 `lib/FormatAbstract.php:32-37`：**

```php
public function setItems(array $items): void
{
    foreach ($items as $item) {
        $this->items[] = FeedItem::fromArray($item);
    }
}
```

**FeedItem 的字段模型（`lib/FeedItem.php:5-13`）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `uri` | `?string` | 条目链接（自动过滤非 http(s) URL） |
| `title` | `?string` | 标题（自动 trim + truncate） |
| `timestamp` | `?int` | Unix 时间戳（支持数字或字符串自动 `strtotime`） |
| `author` | `?string` | 作者 |
| `content` | `?string` | 正文（可接受 HTML DOM 对象自动转字符串） |
| `enclosures` | `string[]` | 附件 URL 数组（自动去重、URL 校验） |
| `categories` | `string[]` | 分类标签数组 |
| `uid` | `?string` | 唯一 ID（非 SHA1 字符串自动做 SHA1） |
| `misc` | `array` | 其他扩展字段（如 `itunes`、`thumbnail`） |

`FeedItem::fromArray()` 通过魔术方法 `__set()` 分发字段，未知字段进入 `misc` 数组，保证前向兼容。

### 3.3 各格式的渲染差异

所有格式继承 `FormatAbstract`，实现各自的 `render(): string` 方法。以下对比核心字段在不同格式中的映射策略：

#### 3.3.1 字段映射总表

| FeedItem 字段 | Atom | Mrss | Json | Html | Plaintext | Sfeed |
|---|---|---|---|---|---|---|
| `title` | `<title type="html">` | `<title>` | `title` | `<h3>` 展示 | print_r | TAB 分隔第 2 列 |
| `uri` | `<link rel="alternate">` | `<link>` + `<guid isPermaLink="true">` | `url` | 超链接 | print_r | TAB 分隔第 3 列 |
| `timestamp` | `<published>` + `<updated>` (DATE_ATOM) | `<pubDate>` (DATE_RFC2822) | `date_modified` (DATE_ATOM) | 格式化显示 | print_r | TAB 分隔第 1 列 |
| `author` | `<author><name>` | 不输出（RSS 需邮箱） | `author: {name}` | 展示 | print_r | TAB 分隔第 7 列 |
| `content` | `<content type="html">` | `<description>` | `content_html` / `content_text` | HTML 渲染 | print_r | TAB 分隔第 4 列 |
| `enclosures` | `<link rel="enclosure">` + Media RSS `<media:thumbnail>` | `<media:content url>` | `attachments[]` | 列表展示 | print_r | 仅取第 1 个，第 8 列 |
| `categories` | `<category term>` | `<category>` | `tags[]` | 标签展示 | print_r | `\|` 连接，第 9 列 |
| `uid` | `<id>` (URN 前缀 `urn:sha1:`) | `<guid isPermaLink="false">` | `id` | 不展示 | print_r | 不展示 |
| `thumbnail` (misc) | `<media:thumbnail url>` | 不单独输出 | `_rssbridge.thumbnail` | 不展示 | print_r | 不展示 |
| `itunes` (misc) | iTunes 命名空间元素 | iTunes 命名空间元素 | `_rssbridge.itunes` | 不展示 | print_r | 不展示 |

#### 3.3.2 降级策略：UID 生成链

所有格式都遵循统一的 UID 三级降级（格式内各自实现，逻辑一致）：

```
优先级 1: item['uid']  ──► 已存在则直接使用
优先级 2: item['uri']  ──► 无 uid 时回退到链接
优先级 3: sha1(title + content)  ──► 两者都无时，基于内容哈希
```

Atom 中体现为（`formats/AtomFormat.php:96-108`）：

```php
if (!empty($item->getUid())) {
    $entryID = 'urn:sha1:' . $item->getUid();
}
if (empty($entryID)) {
    $entryID = $entryUri;
}
if (empty($entryID)) {
    $entryID = 'urn:sha1:' . hash('sha1', $entryTitle . $entryContent);
}
```

#### 3.3.3 扩展字段的处理：Json 格式的 `_rssbridge` 命名空间

`JsonFormat` 定义了 `VENDOR_EXCLUDES` 列表（`formats/JsonFormat.php:15-24`），将 FeedItem 中除标准字段外的所有 misc 字段统一收纳到 `_rssbridge` 前缀下：

```php
const VENDOR_EXCLUDES = [
    'author', 'title', 'uri', 'timestamp', 'content',
    'enclosures', 'categories', 'uid',
];
// ...
$vendorFields = $item->toArray();
foreach (self::VENDOR_EXCLUDES as $key) {
    unset($vendorFields[$key]);
}
if (!empty($vendorFields)) {
    $entry['_rssbridge'] = $vendorFields;
}
```

这保证了 JSON Feed 1.0 规范兼容性的同时，不丢失桥接产生的扩展数据。

### 3.4 响应编码与 MIME 类型

渲染完成后，在 `actions/DisplayAction.php:133-143` 做最终编码处理：

```php
$headers = [
    'last-modified' => gmdate('D, d M Y H:i:s ', $now) . 'GMT',
    'content-type'  => $format->getMimeType() . '; charset=UTF-8',
];
$body = $format->render();

ini_set('mbstring.substitute_character', 'none');
$body = mb_convert_encoding($body, 'UTF-8', 'UTF-8');
```

关键点：
1. **MIME 类型**由各 Format 类的 `const MIME_TYPE` 常量决定，`FormatAbstract::getMimeType()` 通过 `static::MIME_TYPE` 实现延迟静态绑定。
2. **强制 UTF-8**：无论原始数据和格式如何，都在 Content-Type 中声明 `charset=UTF-8`。
3. **编码清洗**：`mb_convert_encoding($body, 'UTF-8', 'UTF-8')` 是一个经典的 PHP 技巧——从 UTF-8 转到 UTF-8，配合 `mbstring.substitute_character=none`，会**静默丢弃所有非法 UTF-8 字节序列**，避免 Feed 解析器因为个别坏字节而整体解析失败。

### 3.5 错误时的格式降级

在 Bridge 抛出异常时（`actions/DisplayAction.php:107-123`），如果配置了 `error.output=feed`，系统会生成一个"错误条目"替代正常数据，并继续走正常的 Format 渲染路径：

```php
if ($errorOutput === 'feed') {
    $items = [$this->createFeedItemFromException($e, $bridge)];
}
```

`createFeedItemFromException()` 返回一个符合 FeedItem 约定的数组，保证下游所有格式都能正常渲染——客户端始终拿到它所请求格式的合法文档，而非 HTML 错误页。

---

## 四、关键文件索引

| 文件 | 职责 |
|---|---|
| `actions/DisplayAction.php` | 格式协商入口、编排数据流向、编码清洗 |
| `lib/FormatFactory.php` | 格式自动发现、名称规范化、类实例化 |
| `lib/FormatAbstract.php` | 格式抽象基类，定义 `setItems/setFeed/setLastModified/getMimeType`，完成 array→FeedItem 转换 |
| `lib/FeedItem.php` | 中间数据模型，字段规范化与校验 |
| `formats/AtomFormat.php` | Atom 1.0 渲染（RFC 4287） |
| `formats/MrssFormat.php` | RSS 2.0 + Media RSS 渲染 |
| `formats/JsonFormat.php` | JSON Feed 1.0 渲染 |
| `formats/HtmlFormat.php` | HTML 预览页（含其他格式的跳转链接） |
| `formats/PlaintextFormat.php` | PHP print_r 调试输出 |
| `formats/SfeedFormat.php` | sfeed TSV 格式输出 |

---

## 五、设计特点总结

1. **显式格式，零内容协商**：不依赖 HTTP Accept Header，用查询参数显式指定，简单可预测、便于缓存。
2. **基于文件系统的格式注册**：新增格式只需在 `formats/` 下添加 `*Format.php`，无需修改任何注册表。
3. **FeedItem 作为防腐层**：Bridge 的原始数组与各格式渲染逻辑解耦，字段规范化在中间层统一完成。
4. **格式间策略独立但约定一致**：UID 降级、MIME 类型声明、UTF-8 清洗等在各格式独立实现，但遵循同一组约定。
5. **错误输出保持格式承诺**：即使桥接失败，也按请求格式返回合法 Feed 文档，而不是 HTTP 错误页，保护 RSS 阅读器的解析链路。
