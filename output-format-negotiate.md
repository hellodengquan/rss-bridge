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

## 四、Format 子类 render 内部字段对齐差异逐行对比

### 4.1 整体循环结构对比

三种主流格式（Atom / Mrss / Json）的 `render()` 都遵循相同的两段式结构：

```
render() {
  1. 构建 Feed 级元数据（channel/feed 对象）
       └─ foreach ($this->getFeed() as $feedKey => $feedValue) ...
  2. 遍历条目并逐项渲染
       └─ foreach ($this->getItems() as $item) {
             预提取字段 + 降级补全
             构建条目容器
             按各字段分支追加
          }
  3. 序列化为最终字符串
}
```

但在**字段预提取顺序、降级补全位置、条目字段输出顺序、条件分支**上存在显著差异。

### 4.2 Feed 级元数据的分支差异

Atom 和 Mrss 都用 `foreach ($feedArray as $feedKey => $feedValue)` 遍历 feed 数组，但 switch/elseif 分支和 skip 规则不同：

| feedKey | Atom (`formats/AtomFormat.php:29-69`) | Mrss (`formats/MrssFormat.php:53-109`) |
|---|---|---|
| `donationUri` | `continue` 跳过 | `continue` 跳过 |
| `atom` | 无专门分支，走 else 生成 `<atom>` | `continue` 跳过 |
| `name` | 生成 `<title type="text">` | 生成 `<title>` + `<description>`（同值复用） |
| `icon` | 生成 `<icon>` + `<logo>`（同一 URL） | 生成完整 `<image>` 子树：`<url>`+`<title>`+`<link>` |
| `uri` | 生成 `<link rel="alternate">` + `<link rel="self">`（type=atom） | 生成 `<link>` + `<atom:link rel="alternate">` + `<atom:link rel="self">`（type=atom） |
| `itunes` | `// todo: skip?` 实际跳过 | 遍历生成 `<itunes:*>` 命名空间元素，并在 `<rss>` 根上声明 xmlns:itunes |
| 其他 key | 生成同名标签 | 生成同名标签 |

JsonFormat 完全不用 foreach，而是**硬编码固定字段**（`formats/JsonFormat.php:30-40`），仅读取 `name/uri/icon` 三个键：

```php
$data = [
    'version'       => 'https://jsonfeed.org/version/1',
    'title'         => $feedArray['name'],
    'home_page_url' => $feedArray['uri'],
    'feed_url'      => get_current_url(),
];
if ($feedArray['icon']) {
    $data['icon'] = $feedArray['icon'];
    $data['favicon'] = $feedArray['icon'];
}
```

扩展 feed 字段（如 `itunes`、`donationUri`）在 Json 中**完全丢失**，因为 Json 没有对应的 else 分支。

### 4.3 条目级字段预提取与降级顺序

三种格式都在 foreach 开头做字段预提取，但顺序和降级补全的位置不同：

**Atom (`formats/AtomFormat.php:88-121`)：**
```
第 1 步：$itemArray = $item->toArray()          // 取完整数组（用于 itunes 分支）
第 2 步：$entryTimestamp / $entryTitle / $entryContent / $entryUri  // 逐个 getXxx()
第 3 步：$entryID = ''
第 4 步：UID 三级降级（getUid → entryUri → sha1(title+content)）
第 5 步：标题空值降级（从 content strip_tags 截取 140 字）
第 6 步：内容空值降级（→ ' ' 单空格，Atom 规范不允许空 content）
第 7 步：创建 <entry> 容器
第 8 步：按固定顺序追加子元素
```

**Mrss (`formats/MrssFormat.php:111-192`)：**
```
第 1 步：$itemArray = $item->toArray()
第 2 步：$itemTimestamp / $itemTitle / $itemUri / $itemContent / $itemUid
第 3 步：$isPermaLink = 'false'
第 4 步：UID 两级降级（先 itemUid → itemUri，此时 isPermaLink='true'；再 sha1(title+content)）
         └─ 注意：Mrss 的 UID 降级与 Atom 代码**不是同一套**——Mrss 在 fallback 到 uri 时标记 isPermaLink=true
第 5 步：创建 <item> 容器
第 6 步：按固定顺序追加子元素
         └─ 注意：Mrss 不做标题空值降级（空标题直接不输出 <title>），也不做内容空值降级
```

**Json (`formats/JsonFormat.php:43-109`)：**
```
第 1 步：$entry = []
第 2 步：$entryAuthor / $entryTitle / $entryUri / $entryTimestamp / $entryContent / $entryEnclosures / $entryCategories
第 3 步：$vendorFields = $item->toArray() + VENDOR_EXCLUDES 过滤
第 4 步：$entry['id'] = getUid()
第 5 步：UID 第一级降级（empty → entryUri）
第 6 步：逐字段 if (!empty) 判断写入
第 7 步：内容分支：is_html() ? content_html : content_text
第 8 步：写入 _rssbridge（如有 vendorFields）
第 9 步：UID 第二级降级（仍 empty → sha1(title+content)）——位置在最后！
```

> 关键差异：Json 的 UID 最终降级放在**所有字段写入之后**，与 Atom/Mrss 的"先算 ID 再写字段"顺序相反。这意味着 `_rssbridge` 中的扩展字段不会影响 fallback ID（因为只用 title+content），但如果未来修改降级策略，位置差异可能引入 bug。

### 4.4 条目字段输出顺序对齐表

同一 FeedItem，三种格式输出 XML/JSON 字段的**物理顺序**不同：

| 顺序 | Atom | Mrss | Json |
|---|---|---|---|
| 1 | `<title type="html">` | `<title>`（可选） | `id` |
| 2 | `<published>` + `<updated>`（可选） | `<itunes:*>`（可选） | `title`（可选） |
| 3 | `<id>` | `<link>`（可选） | `author`（可选） |
| 4 | `<itunes:*>` 或 `<link rel="alternate">` | `<guid isPermaLink>` | `date_modified`（可选） |
| 5 | `<author>`（可选） | `<pubDate>`（可选） | `url`（可选） |
| 6 | `<content type="html">` | `<description>`（可选） | `content_html` 或 `content_text`（可选） |
| 7 | `<link rel="enclosure">` × N | `<media:content>` × N | `attachments[]`（可选） |
| 8 | `<category term>` × N | `<category>` × N | `tags[]`（可选） |
| 9 | `<media:thumbnail>`（可选） | — | `_rssbridge`（可选） |

### 4.5 itunes + enclosure 的互斥分支

Atom 和 Mrss 都有一个关键的 `if (isset($itemArray['itunes'])) ... else if (!empty($entryUri))` 互斥结构：

**Atom (`formats/AtomFormat.php:146-166`)：**
```php
if (isset($itemArray['itunes'])) {
    // 声明 xmlns:itunes + 输出 itunes 元素 + 输出 <enclosure>（带 length/type）
} elseif (!empty($entryUri)) {
    // 输出 <link rel="alternate" type="text/html">
}
```

**Mrss (`formats/MrssFormat.php:140-161`)：**
```php
if (isset($itemArray['itunes'])) {
    // 声明 xmlns:itunes + 输出 itunes 元素 + 输出 <enclosure>（带 length/type）
}
// itunes 分支后，无条件额外输出 <link>（如果有 itemUri）
```

> 差异：Atom 中 itunes 和 alternate link 是**互斥**的——含 itunes 的播客条目不会输出 `<link rel="alternate">`。而 Mrss 中两者可以**并存**。

### 4.6 enclosure 输出的两套路径

三种格式对 enclosure 的处理分为"播客模式"和"普通模式"：

**播客模式（itunes + enclosure 数组）：**
- Atom/Mrss：从 `$itemArray['enclosure']` 读取带 `url/length/type` 的关联数组，输出标准 `<enclosure url= length= type=>`
- Json：走普通模式（因为 Json 不特别处理 itunes）

**普通模式（enclosures URL 数组）：**
- Atom：`<link rel="enclosure" type="{parse_mime_type()}" href="{url}">`
- Mrss：`<media:content url="{url}" type="{parse_mime_type()}">`（Media RSS 命名空间）
- Json：`attachments: [{url, mime_type}]`

---

## 五、parse_mime_type：扩展名优先级与 MIME 判定链路

`parse_mime_type()` 在 `lib/utils.php:164-218` 定义，被 Atom/Mrss/Json 三个 Format 的 enclosure 分支调用。它不读取文件内容，完全基于 URL 的文件扩展名推断 MIME。

### 5.1 扩展名映射表的两级加载

函数使用 `static $mime = null` 做进程内单例缓存，首次调用时加载：

**第一级：硬编码默认表**（`lib/utils.php:170-177`）
```php
$mime = [
    'jpg'   => 'image/jpeg',
    'gif'   => 'image/gif',
    'png'   => 'image/png',
    'webp'  => 'image/webp',
    'image' => 'image/*',       // 特殊：用于 URL 锚点提示 #.image
    'mp3'   => 'audio/mpeg',
];
```

**第二级：系统 `/etc/mime.types` 覆盖/扩展**（`lib/utils.php:179-200`）
```php
if (!ini_get('open_basedir')) {
    if (@is_readable('/etc/mime.types')) {
        // 逐行解析：跳过注释和空行
        // 每行格式：mime/type  ext1  ext2  ext3 ...
        // 后面的扩展名覆盖前面的同名 key
    }
}
```

> 优先级：`/etc/mime.types` > 硬编码默认表。因为第二级用 `$mime[$part] = $type` 直接赋值覆盖。`/etc/mime.types` 中同一 MIME 类型的多个扩展名按从左到右顺序写入，不会互相覆盖。

### 5.2 URL 解析与扩展名提取流程

对传入 URL 的处理步骤（`lib/utils.php:203-217`）：

```
输入 URL: https://example.com/foo.mp3?x=1#.image
          │
          ▼
第 1 步：剥离 query string（保留 anchor）
          https://example.com/foo.mp3#.image
          │
          ▼
第 2 步：pathinfo($url, PATHINFO_EXTENSION) 取扩展名
          ext = 'image'   （锚点伪装的扩展名优先级最高！）
          │
          ▼
第 3 步：strtolower(ext) 查表
          mime['image'] = 'image/*'
          │
          ▼
第 4 步：命中 → 返回；未命中 → 返回 'application/octet-stream'
```

### 5.3 锚点提示 `#.ext` 的特殊优先级

代码注释明确说明：*"A caller can hint for a MIME type by appending `#.ext` to the URL"*。

因为第 1 步的特殊处理逻辑——**先剥离 query，再把 anchor 拼接回去**，导致 `pathinfo` 会把 `#.jpg` 中的 `jpg` 当作扩展名。这实际上给了调用方一种"强制 MIME"的手段，优先级高于真实文件扩展名。

示例：
| URL | 提取的 ext | 返回 MIME |
|---|---|---|
| `https://x.com/foo.png` | `png` | `image/png` |
| `https://x.com/foo.png?size=lg` | `png` | `image/png` |
| `https://x.com/foo.png#.jpg` | `jpg` | `image/jpeg` |
| `https://x.com/foo.png?size=lg#.webp` | `webp` | `image/webp` |
| `https://x.com/unknown.xyz` | `xyz` | `application/octet-stream` |

### 5.4 缺失的保护

- 不检查 URL 真实性（纯字符串处理）
- 不处理多重扩展名（`tar.gz` 只取最后一段 `gz`）
- 大小写不敏感（strtolower）
- 无法识别无扩展名的 URL（pathinfo 返回空 → fallback octet-stream）

---

## 六、缓存协商：Last-Modified 与 304 分支的完整链路

RSS-Bridge 实现了基于 Last-Modified 的 HTTP 缓存协商，**未实现 ETag**。整条链路涉及两个缓存层：服务端缓存（CacheInterface）和客户端缓存（304 Not Modified）。

### 6.1 中间件栈中的位置

`lib/RssBridge.php:25-39` 注册的中间件执行顺序（注意 `array_reverse` 洋葱模型）：

```
Request → SecurityMiddleware
        → MaintenanceMiddleware
        → ExceptionMiddleware
        → CacheMiddleware    ← 缓存协商在此
        → TokenAuthenticationMiddleware
        → BasicAuthMiddleware
        → DisplayAction
```

CacheMiddleware 只拦截 DisplayAction（`middlewares/CacheMiddleware.php:17-21`）：
```php
if ($action !== 'DisplayAction') {
    return $next($request);
}
```

### 6.2 完整流程分支图

```
                    Request 到达 CacheMiddleware
                              │
                              ▼
               计算 cacheKey = 'http_' + json_encode($_GET)
                              │
                              ▼
                cache.get(cacheKey) 是否命中？
                    │                    │
                   是                    否
                    │                    │
                    ▼                    ▼
          ┌──────────────────┐   调用 $next() → DisplayAction 执行业务
          │  服务端缓存命中  │        │
          └──────────────────┘        ▼
                    │           response.getCode() == ?
                    ▼           ┌──────┬──────┬──────┐
     ┌──────────────────────┐   200    4xx/5xx  其他
     │ 检查 HTTP_IF_MODIFIED │   │      │       │
     │   _SINCE 头是否存在   │   ▼      ▼       ▼
     └──────────────────────┘  (不缓存) cache.set  cache.set
                    │          DisplayAction     5min
                    │          内部已自行缓存
                    ▼
          If-Modified-Since 与
          cached.last-modified 比较
              │            │
         未修改(<=)       已修改
              │            │
              ▼            ▼
      返回 304 空 body   返回 cached
      仅带 Last-Modified  Response（完整 body）
```

### 6.3 服务端缓存的双写机制

**写缓存有两处**，分别对应不同状态码：

**第一处：DisplayAction 内部（`actions/DisplayAction.php:56-64`）**
```php
if ($response->getCode() === 200) {
    $ttl = $request->get('_cache_timeout');
    if (Configuration::getConfig('cache', 'custom_timeout') && isset($ttl)) {
        $ttl = (int) $ttl;
    } else {
        $ttl = $bridge->getCacheTimeout();  // 默认为各 Bridge 的 CACHE_TIMEOUT（通常 3600s）
    }
    $this->cache->set($cacheKey, $response, $ttl);
}
```
仅缓存 200 响应，TTL 取 Bridge 配置或请求覆盖参数 `_cache_timeout`。

**第二处：CacheMiddleware 尾部（`middlewares/CacheMiddleware.php:46-54`）**
```php
if ($response->getCode() === 200) {
    // Do nothing because DisplayAction has already cached this on $cacheKey
} elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
    // Cache these responses for about ~10 mins on average
    $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
}
```
仅缓存错误响应，TTL 为 5 分钟 + 0~10 分钟随机抖动（防止缓存雪崩）。

### 6.4 304 协商的具体代码（`middlewares/CacheMiddleware.php:28-39`）

```php
$ifModifiedSince = $request->server('HTTP_IF_MODIFIED_SINCE');
$lastModified = $cachedResponse->getHeader('last-modified');
if ($ifModifiedSince && $lastModified) {
    $lastModified = new \DateTimeImmutable($lastModified);
    $lastModifiedTimestamp = $lastModified->getTimestamp();
    $modifiedSince = strtotime($ifModifiedSince);
    if ($lastModifiedTimestamp <= $modifiedSince) {
        $modificationTimeGMT = gmdate('D, d M Y H:i:s ', $lastModifiedTimestamp);
        return new Response('', 304, ['last-modified' => $modificationTimeGMT . 'GMT']);
    }
}
```

关键点：
1. **只有服务端缓存命中才会进入 304 分支**——没有缓存时直接执行业务并返回 200。
2. 比较使用 `<=`（小于等于）——即如果客户端时间 >= 服务端时间，视为未修改。
3. 304 响应返回**空 body**，只携带 `Last-Modified` 头（Content-Type 不携带，符合 RFC 7232）。
4. **没有 ETag 支持**：代码中完全不处理 `If-None-Match`，也不生成 `ETag` 头。

### 6.5 Last-Modified 值的来源

Last-Modified 头由 `actions/DisplayAction.php:131-135` 设置：

```php
$now = time();
$format->setLastModified($now);
$headers = [
    'last-modified' => gmdate('D, d M Y H:i:s ', $now) . 'GMT',
    ...
];
```

值是**当前请求的执行时间**，而非 Bridge 数据的真实更新时间。这意味着：
- 每次缓存 miss 重新生成时，Last-Modified 都会更新为当前时间
- 客户端下次请求带 `If-Modified-Since` 会触发 304（只要缓存还在有效期内）
- 缓存过期后重新抓取，即使内容没变，Last-Modified 也会变，客户端收到新的 200

### 6.6 cacheKey 的构造

两处使用同一规则（`actions/DisplayAction.php:50` 和 `middlewares/CacheMiddleware.php:24`）：
```php
$cacheKey = 'http_' . json_encode($request->toArray());
```
即 `json_encode($_GET)`。所有查询参数（包括 `bridge`、`format`、各 Bridge 自定义参数、`_cache_timeout`、`_noproxy` 等）都参与 key 计算。

这意味着：
- 不同 `format` 参数天然隔离缓存（Atom 和 Mrss 各有各的缓存条目）
- 随机参数（如某些 RSS 阅读器加的 `_=<timestamp>`）会导致缓存完全失效——但 DisplayAction 在第 84 行把 `_` 列入 `$remove` 数组不传给 Bridge，**但它仍在 `$request->toArray()` 里参与 cacheKey 计算**，这是一个已知的问题注释。

---

## 七、关键文件索引

| 文件 | 职责 |
|---|---|
| `actions/DisplayAction.php` | 格式协商入口、编排数据流向、编码清洗、200 响应写入服务端缓存 |
| `lib/FormatFactory.php` | 格式自动发现、名称规范化、类实例化 |
| `lib/FormatAbstract.php` | 格式抽象基类，定义 `setItems/setFeed/setLastModified/getMimeType`，完成 array→FeedItem 转换 |
| `lib/FeedItem.php` | 中间数据模型，字段规范化与校验 |
| `lib/utils.php` | `parse_mime_type()` 基于扩展名推断 MIME，含锚点提示机制 |
| `lib/RssBridge.php` | 中间件栈注册与洋葱模型编排 |
| `middlewares/CacheMiddleware.php` | 服务端缓存读取、304 Not Modified 协商、错误响应写入缓存 |
| `lib/CacheInterface.php` | 缓存后端抽象接口 |
| `formats/AtomFormat.php` | Atom 1.0 渲染（RFC 4287） |
| `formats/MrssFormat.php` | RSS 2.0 + Media RSS 渲染 |
| `formats/JsonFormat.php` | JSON Feed 1.0 渲染 |
| `formats/HtmlFormat.php` | HTML 预览页（含其他格式的跳转链接） |
| `formats/PlaintextFormat.php` | PHP print_r 调试输出 |
| `formats/SfeedFormat.php` | sfeed TSV 格式输出 |

---

## 八、设计特点总结

1. **显式格式，零内容协商**：不依赖 HTTP Accept Header，用查询参数显式指定，简单可预测、便于缓存。
2. **基于文件系统的格式注册**：新增格式只需在 `formats/` 下添加 `*Format.php`，无需修改任何注册表。
3. **FeedItem 作为防腐层**：Bridge 的原始数组与各格式渲染逻辑解耦，字段规范化在中间层统一完成。
4. **格式间策略独立但约定一致**：UID 三级降级、MIME 类型声明、UTF-8 清洗等在各格式独立实现，逻辑基本一致但存在微妙差异（如 Json 的 UID 降级位置后置、Atom 对 itunes/alternate 的互斥处理）。
5. **错误输出保持格式承诺**：即使桥接失败，也按请求格式返回合法 Feed 文档，而不是 HTTP 错误页，保护 RSS 阅读器的解析链路。
6. **MIME 推断的锚点覆盖**：通过 `#.ext` 伪扩展名机制，允许 Bridge 强制指定 enclosure 的 MIME 类型，优先级高于真实文件扩展名。
7. **两级缓存双写**：服务端缓存分两处写入——DisplayAction 管 200 正常响应（Bridge TTL），CacheMiddleware 管错误响应（带随机抖动的短 TTL），避免错误风暴。
8. **无 ETag 的简化协商**：仅实现 Last-Modified / If-Modified-Since，未实现 ETag / If-None-Match，且 Last-Modified 值取请求执行时间而非数据真实变更时间。
