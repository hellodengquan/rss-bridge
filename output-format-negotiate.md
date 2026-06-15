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

## 七、Disk Cache 落盘：五种 Cache 后端与 TTL 过期机制

### 7.1 CacheFactory 与后端选择

缓存后端由 `lib/CacheFactory.php` 通过扫描 `caches/` 目录自动发现，配置项 `[cache] type` 决定使用哪一种（默认 `file`）：

| 类 | 配置名 | 存储介质 | TTL 精确性 | 适用场景 |
|---|---|---|---|---|
| `FileCache` | `file` | 本地文件系统（每 key 一个文件） | 读时比对（惰性过期） | 默认，单机部署 |
| `SQLiteCache` | `sqlite` | SQLite3 单文件数据库 | 读时比对（惰性过期），支持索引 | 中规模，减少文件数 |
| `MemcachedCache` | `memcached` | Memcached 服务端 | 服务端主动过期 | 分布式 / 多机部署 |
| `ArrayCache` | `array` | PHP 进程内存数组 | 读时比对 | DEBUG 模式 / 单次请求 |
| `NullCache` | `null` | 空实现，永不存储 | N/A | 禁用缓存 |

DEBUG 模式（存在 `DEBUG` 文件且为空）自动强制使用 `ArrayCache`（`lib/Configuration.php:38-43`）。

### 7.2 FileCache 落盘细节

**key → 路径映射**（`caches/FileCache.php:115-118`）：
```php
private function createCacheFile(string $key): string
{
    return $this->config['path'] . hash('md5', $key) . '.cache';
}
```
原始 key（可能含特殊字符、超长）先做 MD5，保证文件名合法且等长。路径默认 `./cache/`，可通过 `[FileCache] path` 覆盖。

**文件内容结构**（`caches/FileCache.php:54-60`）：
```php
$item = [
    'key'        => $key,           // 原始 key（便于调试，冗余存储）
    'expiration' => time() + $ttl,  // 过期时间戳；0 表示永不过期
    'value'      => $value,         // 任意可序列化 PHP 值（Response 对象、HTML 字符串等）
];
file_put_contents($cacheFile, serialize($item));
```
使用 PHP 原生 `serialize()`，因此可以缓存 Response 对象等复合结构。`ttl === 0` 时直接跳过不写（避免写一个永不过期的 0 TTL 条目）；`ttl === null` 时 `expiration = 0` 表示永久缓存。

**读取与惰性过期**（`caches/FileCache.php:27-46`）：
```php
$data = file_get_contents($cacheFile);
$item = unserialize($data);
$expiration = $item['expiration'] ?? time();
if ($expiration === 0 || $expiration > time()) {
    return $item['value'];
}
$this->delete($key);  // 过期即删（读时触发）
return $default;
```
反序列化失败（文件损坏）也会删除该文件。`prune()` 遍历整个目录删除所有过期文件，由 CacheMiddleware 以 1% 概率随机触发（`middlewares/CacheMiddleware.php:57-60`）。

### 7.3 SQLiteCache 落盘差异

SQLiteCache 用 `sha1(key, true)`（二进制 20 字节）作为 BLOB 主键，value 同样用 `serialize()` 序列化为 BLOB：

```sql
CREATE TABLE storage ('key' BLOB PRIMARY KEY, 'value' BLOB, 'updated' INTEGER)
CREATE INDEX idx_storage_updated ON storage (updated)
```

列名 `updated` 实际存的是过期时间戳（代码注释坦言命名错误）。过期用单条 SQL 批量删除：
```sql
DELETE FROM storage WHERE updated > 0 AND updated <= :now
```
比 FileCache 逐个文件扫描高效得多。开启 WAL 模式和 `synchronous = NORMAL`，在性能与安全间取平衡。

### 7.4 三种 key 命名空间

整个系统用 cache key 前缀区分三大缓存域，互不干扰：

| 前缀 | 生产方 | 典型 key | 典型 TTL |
|---|---|---|---|
| `http_` | CacheMiddleware + DisplayAction | `http_` + `json_encode($_GET)` | Bridge CACHE_TIMEOUT（默认 3600s）或错误响应 5~15min |
| `server_` | `getContents()` | `server_{url}_{md5(postBody)}` | 固定 864000s（10 天），受 no-cache/no-store 头抑制 |
| `pages_` | `getSimpleHTMLDOMCached()` | `pages_{url}` | 调用方指定，默认 86400s（1 天） |
| `error_reporting_` | DisplayAction | `error_reporting_{bridgeName}_{code}` | 固定 432000s（5 天），用于错误频率计数 |

---

## 八、URL 重写与 Sanitization：三级处理链路

URL 处理分为三个层次，各司其职：

### 8.1 第一层：Url 类 — 严格验证与规范化

`lib/Url.php` 是一个"故意做得非常严格"的 URL 解析器，只接受绝对的 http/https URL：

**正则校验**（`lib/Url.php:46-58`）：
```php
$pattern = '#^https?://'   // scheme
    . '([a-z0-9-]+\.?)+'   // 一个或多个域名段
    . '(\.[a-z]{1,24})?'   // 可选全局 TLD
    . '(:\d+)?'            // 可选端口
    . '($|/|\?)#i';        // 结束或 / 或 ?
```
额外限制：总长不超过 1500 字符。scheme 非 http/https 直接抛 `UrlException`。

**规范化输出**（`lib/Url.php:129-150`）：
- 端口 80 自动省略（`http://x:80/` → `http://x/`）
- path 不以 `/` 开头或含 `//` 前缀均抛异常
- 不处理 fragment（注释 `// todo: add fragment`）

注意：Url 类目前在代码中使用较少，主要用于 Bridge 内部严格校验；大多数路径仍使用宽松的 `parse_url()` + `urljoin()`。

### 8.2 第二层：urljoin() — 相对 URL 转绝对

`lib/php-urljoin/src/urljoin.php` 是 Python `urllib.parse.urljoin()` 的 PHP 移植，被 50+ Bridge 广泛调用。

**核心合并规则**：
```
输入: base = "https://example.com/a/b/page.html"
      rel  = "../images/photo.png?size=lg#thumb"
      │
      ▼
1. parse_url 拆分为 $pbase 和 $prel
2. 若 rel 含合法 scheme 且与 base 相同（或在白名单内），保留 rel scheme
3. 合并：array_merge($pbase, $prel)  →  rel 字段覆盖 base 同名字段
4. 相对 path 处理：
   - rel path 非 "/" 开头 → 取 base path 目录 + "/" + rel path
   - 消除 "./" 前缀
5. 路径规范化：按 "/" 拆分，逐段消解 ".." 和 "."
6. 重组：scheme://[user:pass@]host[:port][/path][?query][#fragment]
```

相对 scheme 的白名单（`$uses_relative`）包括 http/https/ftp/ws/wss 等 20 种，意味着 `javascript:` 等危险 scheme 不会被当作 relative 合并——但如果 rel 本身就是完整的 `javascript:` URL，第 42-48 行会直接返回原 rel（安全隐患，依赖调用方过滤）。

### 8.3 第三层：defaultLinkTo() — HTML 内容中的链接批量补全

`lib/html.php:247-286` 遍历 HTML DOM 中的 `<img src>` 和 `<a href>`，逐一用 `urljoin()` 把相对链接补全为绝对链接：

```php
foreach ($findByTag('img') as $image) {
    $image->setAttribute('src', urljoin($url, $image->getAttribute('src')));
}
foreach ($findByTag('a') as $anchor) {
    $anchor->setAttribute('href', urljoin($url, $anchor->getAttribute('href')));
}
```

只处理 img 和 a，不处理 `<iframe src>`、`<video src>`、`<source srcset>`、`<link href>` 等。`parseSrcset()`（`lib/html.php:306-329`）单独解析 `srcset` 属性，但不自动重写 URL——需要 Bridge 自行调用。

### 8.4 FeedItem 层的隐式 Sanitization

`lib/FeedItem.php:92-112` 在 `setURI()` 里做了一道隐式过滤：
```php
if (!preg_match('#^https?://#i', $uri)) {
    return;  // 非 http/https 直接丢弃，不存入
}
```
因此 enclosure 和其他字段中的 URL **不会**经过这个过滤——只有 `uri` 字段受保护。enclosure 由 `setEnclosures()` 用 `FILTER_VALIDATE_URL` 校验。

---

## 九、Fetch 层：重试机制、文件大小限流与条件请求

RSS-Bridge 没有实现并发请求调度或全局限流器，HTTP 抓取完全由 `getContents()` + `CurlHttpClient` 串行执行。保护措施体现在三个方面。

### 9.1 请求重试与超时

`CurlHttpClient::request()`（`lib/http.php:65-197`）的重试逻辑：

```php
$defaultConfig = [
    'timeout'   => 5,       // [http] timeout，默认 5s
    'retries'   => 2,       // [http] retries，默认 1（注意：$defaultConfig 写 2，被配置覆盖为 1）
    'max_redirections' => 5,
];
// ...
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) break;
    if ($tries <= $config['retries']) continue;
    throw new HttpException(...);
}
```
重试只针对 cURL 层错误（网络不通、DNS 失败、超时），HTTP 4xx/5xx 状态码**不会触发重试**——`curl_exec` 视为成功，后续由 `getContents()` 的 switch 处理。

### 9.2 文件大小限流

由 `[http] max_filesize`（默认 20MB，见 `config.default.ini.php:58`）控制，两种方式双重保险：

```php
// lib/http.php:119-131
if ($config['max_filesize']) {
    // 方式一：依赖服务器返回 Content-Length（可能被伪造或缺失）
    curl_setopt($ch, CURLOPT_MAXFILESIZE, $config['max_filesize']);
    // 方式二：回调函数实时监控下载字节数（即使无 Content-Length 也生效）
    curl_setopt($ch, CURLOPT_NOPROGRESS, false);
    curl_setopt($ch, CURLOPT_PROGRESSFUNCTION, function ($ch, $downloadSize, $downloaded, ...) {
        if ($downloaded > $config['max_filesize']) return -1;  // 非零返回中止传输
        return 0;
    });
}
```
单位换算在 `getContents()` 侧完成：配置值（MB）× 2²⁰ = 实际字节数。

### 9.3 条件请求与 304 复用

`getContents()`（`lib/contents.php:73-90`）在命中服务端缓存时，自动带上条件请求头让源服务器做 304 判断：

```php
$cachedResponse = $cache->get($cacheKey);
if ($cachedResponse) {
    $lastModified = $cachedResponse->getHeader('last-modified');
    if ($lastModified) {
        // 兼容服务器可能发送 Unix 时间戳（非 RFC 7231 格式）
        $lastModified = new \DateTimeImmutable((is_numeric($lastModified) ? '@' : '') . $lastModified);
        $config['if_not_modified_since'] = $lastModified->getTimestamp();
    }
    $etag = $cachedResponse->getHeader('etag');
    if ($etag) {
        $httpHeadersNormalized['if-none-match'] = $etag;
    }
}
```

收到 304 后用缓存 body 填充响应（`lib/contents.php:126-129`）：
```php
case 304:
    $response = $response->withBody($cachedResponse->getBody());
    break;
```

只有 200/201/202 会写入缓存（TTL 10 天），且受响应头 `Cache-Control: no-cache / no-store` 抑制。301/302/303 的缓存被注释为 `// todo: cache`，目前重定向响应不落盘。

### 9.4 "并发"与"限流"的真实情况

- **无并发**：PHP-FPM/mod_php 模型下每次请求单进程串行执行，Bridge 中多次 `getContents()` 按顺序发起。没有 curl_multi、没有异步任务、没有连接池。
- **无全局限流**：没有请求级令牌桶、没有按域名速率限制、没有并发数控制。防护手段仅为：单请求超时（默认 5s）、重试次数（默认 1 次）、响应大小上限（默认 20MB）、以及源站 429 被捕获后抛 `RateLimitException` 返回给客户端。
- **代理可选**：配置 `[proxy] url` 后所有请求走 HTTP 代理，支持按 Bridge 维度让用户通过 `_noproxy` 参数关闭。

---

## 十、Error 分类降级：异常类型分支与三段式输出策略

### 10.1 异常类继承体系

```
\Throwable
 ├─ \Exception
 │   ├─ HttpException              lib/http.php:13 — HTTP 状态码异常（含 Response）
 │   │   └─ CloudFlareException    lib/http.php:38 — 识别 CF 拦截页
 │   ├─ RateLimitException         lib/http.php:6 — 源站限流
 │   ├─ ClientException            lib/utils.php:250 — Bridge 判定的客户端参数错误
 │   ├─ UrlException               lib/url.php:5 — URL 校验失败
 │   └─ 其他 Bridge 抛出的 \Exception — 通用服务端错误
 └─ \Error (PHP 运行时错误，由异常处理器兜底)
```

### 10.2 DisplayAction 内的分类分支

`actions/DisplayAction.php:91-124` 按异常类型分四档处理：

```php
try {
    $bridge->collectData();
} catch (\Throwable $e) {
    if ($e instanceof ClientException) {
        // 第 1 档：客户端参数错误 —— 仅 debug 日志，不计数，不暴露给用户
        $this->logger->debug(...);
    } elseif ($e instanceof RateLimitException) {
        // 第 2 档：被源站限流 —— 直接返回 429 + 异常 HTML 页
        $this->logger->debug(...);
        return new Response(render(exception.html.php), 429);
    } elseif ($e instanceof HttpException) {
        if (in_array($e->getCode(), [429, 503])) {
            // 第 3 档：HTTP 429/503 —— 直接返回对应状态码 + 异常 HTML 页
            return new Response(render(exception.html.php), $e->getCode());
        }
        // 其他 HTTP 错误（404/500 等）：静默，走下面的错误计数逻辑
    } else {
        // 第 4 档：未知异常 —— error 日志 + 错误计数
        $this->logger->error(...);
    }
    // ========== 统一错误计数与输出 ==========
    $errorOutput = Configuration::getConfig('error', 'output');  // feed | http | none
    $reportLimit = Configuration::getConfig('error', 'report_limit');  // 默认 1
    $errorCount = 1;
    if ($reportLimit > 1) {
        $errorCount = $this->logBridgeError($bridge->getName(), $e->getCode());
    }
    if ($errorCount >= $reportLimit) {
        if ($errorOutput === 'feed') {
            // 输出为 Feed 条目（格式兼容！）
            $items = [$this->createFeedItemFromException($e, $bridge)];
        } elseif ($errorOutput === 'http') {
            // 输出为 HTTP 500 错误页
            return new Response(render(exception.html.php), 500);
        } elseif ($errorOutput === 'none') {
            // 静默：返回空 Feed
        }
    }
}
```

### 10.3 错误频率计数（report_limit 机制）

`logBridgeError()`（`actions/DisplayAction.php:172-191`）用独立缓存域 `error_reporting_` 做滑动窗口计数：

```php
$cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
$report = $this->cache->get($cacheKey);
if ($report) {
    $report = Json::decode($report);
    $report['time'] = time();
    $report['count']++;
} else {
    $report = ['error' => $code, 'time' => time(), 'count' => 1];
}
$ttl = 86400 * 5;  // 5 天
$this->cache->set($cacheKey, Json::encode($report), $ttl);
```
TTL 5 天，每次错误刷新过期时间。`report_limit` 默认为 1，意味着首次错误即暴露；设为 N 则需同一 Bridge 在 5 天内同一错误码累计 N 次才对外暴露——用于过滤偶发抖动。

### 10.4 中间件兜底：ExceptionMiddleware

`middlewares/ExceptionMiddleware.php:14-23` 是整个洋葱的最后一道防线，捕获所有未被 DisplayAction 处理的异常：

```php
try {
    return $next($request);
} catch (\Throwable $e) {
    $this->logger->error('Exception in ExceptionMiddleware', ['e' => $e]);
    return new Response(render(exception.html.php), 500);
}
```
所有漏网之鱼（包括 DisplayAction 外的 Action、中间件自身异常）统一返回 500 HTML 错误页。此外 `index.php:20-59` 还注册了全局 `set_exception_handler` + `set_error_handler` + `register_shutdown_function` 三层兜底，确保 Fatal Error 也不会暴露 PHP 原生堆栈。

### 10.5 三种错误输出模式对比

| `[error] output` | 正常渲染 | 错误时行为 | Feed 解析器感知 |
|---|---|---|---|
| `feed`（默认） | 正常条目 | 生成一条"错误条目"混入 Feed，title 含错误码 | 能继续解析，用户在阅读器里看到错误信息 |
| `http` | 正常条目 | 返回 HTTP 500 + HTML 异常页 | Feed 解析失败，阅读器标红 |
| `none` | 正常条目 | 返回空 Feed（0 条目） | 解析成功但无内容，可能被误判为"无更新" |

结合 `report_limit` 的节流效果：例如 `output=feed` + `report_limit=3`，同一 Bridge 同一错误码前两次完全静默（像没发生一样返回空 Feed？不——代码逻辑是 `errorCount < reportLimit` 时三个分支都不触发，$items 保持空数组，实际等价于 `output=none`），第三次起才以 Feed 条目形式对外暴露。

---

## 十一、Bridge 参数校验：PARAMETERS 常量 Schema + ParameterValidator 双环

### 11.1 无 YAML，基于 PHP 类常量的 Schema 定义

RSS-Bridge **没有 YAML 配置文件**定义 Bridge 参数，而是通过每个 Bridge 类的 `const PARAMETERS = []` 类常量声明参数模型，本质上是一个**以 Context 为第一层 key 的嵌套关联数组 Schema**：

```php
const PARAMETERS = [
    // context 名（前端按 context 渲染多组表单）
    'By user ID' => [
        'user_id' => [
            'name'      => 'User ID',          // 前端 label
            'type'      => 'text',             // text | number | list | checkbox
            'title'     => 'Enter numeric ID', // 前端 tooltip/占位符
            'required'  => true,               // 必填/选填
            'pattern'   => '\d+',              // 仅 text 类型：正则约束
            'exampleValue' => '1234567',       // 前端示例值按钮
            'defaultValue' => '',              // 默认值
        ],
        'limit' => self::LIMIT,                // 复用 BridgeAbstract 的 LIMIT 常量（number 类型）
    ],
    'global' => [
        // global context：跨所有 context 共享的参数
        'formatting' => [
            'name'  => 'Rich text',
            'type'  => 'checkbox',
        ],
    ],
];
```

**四种类型的属性约定**（由 `lib/ParameterValidator.php:24-41` 和 `tests/BridgeImplementationTest.php:69-128` 双重约束）：

| type | 必需属性 | 可选属性 | 校验方式 |
|---|---|---|---|
| `text`（默认） | `name` | `required`, `pattern`, `defaultValue`, `exampleValue`, `title` | `filter_var` 或 `FILTER_VALIDATE_REGEXP` 匹配 `^pattern$` |
| `number` | `name` | `required`, `defaultValue`, `exampleValue`, `title` | `FILTER_VALIDATE_INT` |
| `checkbox` | `name` | `title` | `FILTER_VALIDATE_BOOLEAN + FILTER_NULL_ON_FAILURE`；**不允许声明 `required`** |
| `list` | `name`, `values` (非空数组) | `title`, `defaultValue`, `exampleValue` | 必须在 `values` 顶层或子数组中存在；**不允许声明 `required`** |

`tests/BridgeImplementationTest.php:58-158` 对所有 Bridge 做强制静态检查：所有参数名非空、type 在四种内、list 必须有 values、checkbox/list 不得声明 required 等。

### 11.2 校验流程：BridgeAbstract::setInput() → ParameterValidator

调用链在 `actions/DisplayAction.php:79` 触发：

```php
$bridge->setInput($request->toArray());
```

`lib/BridgeAbstract.php:138-179` 的完整流程：

```
输入 $_GET（去掉 action/bridge/format/_cache_timeout/_noproxy/_ 等保留键）
      │
      ▼
PARAMETERS 是否为空？
 ├─ 是：还有输入 → throw ClientException("Unexpected parameters")
 └─ 否：继续
      │
      ▼
ParameterValidator::validateInput($input 引用传递, $parameters)
  → 双重循环：每个 input 名在所有 context 中查找是否注册
  → 未注册 → error: "Parameter is not registered!"
  → 已注册 → switch(type) 做类型过滤，失败则置 null
  → required 但被过滤为 null → error: "Parameter is invalid!"
  → 有错误 → throw ClientException("Invalid parameters value(s): ...")
      │
      ▼
ParameterValidator::getQueriedContext($input, $parameters)
  → 推理用户使用的是哪个 context：
     - 用户 input 的多余键（不在 context 也不在 global 中）→ 跳过
     - 任一必填参数缺失 → 标记为 false
     - 任一非 checkbox/list 参数有值 → 标记为 true
     - 0 个匹配 → 返回第一个 context（无参数默认）或 null
     - 1 个匹配 → 返回该 context 名
     - ≥2 个匹配 → 返回 false（"Mixed context parameters" → \Exception）
      │
      ▼
setInputWithContext() → 赋值 + 补默认值 + global 合并到选中 context
```

**关键细节：input 是引用传递**（`validateInput(array &$input, ...)`），校验过程同步完成了类型转换（如字符串 "123" → int 123，"true" → bool true）。Checkbox 默认值为 false，List 默认值为 values 数组的第一项（或 defaultValue）。

### 11.3 配置层校验：CONFIGURATION 常量 + loadConfiguration()

`const CONFIGURATION = []` 是管理员在 `config.ini.php` 中配置的密钥/凭证级参数（与用户通过 URL 传入的 PARAMETERS 不同层级）：

```php
const CONFIGURATION = [
    'api_key' => [
        'required'      => true,
    ],
    'cache_ttl' => [
        'defaultValue'  => 3600,
    ],
];
```

`lib/BridgeAbstract.php:119-136` 的 `loadConfiguration()` 在 Bridge 实例化后立即调用，按 section = BridgeShortName 从 `Configuration::getConfig()` 读取，缺省时：
- `required=true` → 抛 `\Exception`（启动时即失败）
- 有 `defaultValue` → 采用默认值
- 既不 required 也无 defaultValue → 不写入 `$this->configuration`

---

## 十二、CLI 模式入口：argv 平铺为 query + Request::fromCli

### 12.1 入口代码：index.php 第 63-69 行

```php
$argv = $argv ?? null;
if ($argv) {
    parse_str(implode('&', array_slice($argv, 1)), $cliArgs);
    $request = Request::fromCli($cliArgs);
} else {
    $request = Request::fromGlobals();
}
```

**SAPI 判定方式**：不调用 `php_sapi_name()`，而是通过 PHP 运行时是否自动注入 `$argv` 变量判断——当且仅当 CLI 模式且 `php.ini` 的 `register_argc_php` = On 时 `$argv` 才存在（这是一个经典简写，但依赖 php.ini 配置）。

### 12.2 参数语法：`key=value` 空格分隔

与 HTTP GET 的查询字符串语法一一对应：

```bash
# Web URL
# ?action=display&bridge=DansTonChat&format=Json

# CLI 等价
php index.php action=display bridge=DansTonChat format=Json
```

实现细节：`array_slice($argv, 1)` 跳过脚本名 `index.php`，`implode('&', ...)` 拼接为查询字符串，再用 `parse_str()` 解析为关联数组。

- 支持 `&` 在单个参数内部（`parse_str` 会二次拆，但 CLI 下的 `&` 需转义或单引号包裹，否则被 shell 解释为后台运行）
- 无显式布尔类型：`checkbox=true` 仍是字符串，由 ParameterValidator 转换
- 无文件上传（`$_FILES` 为空），但 RSS-Bridge 本身不依赖此
- 所有支持的 action（list/display/detect/findfeed 等）完全一致

### 12.3 Request::fromCli 与 fromGlobals 的差异

`lib/http.php:210-224` 定义两种工厂方法：

| 行为 | `fromGlobals()`（HTTP） | `fromCli()`（CLI） |
|---|---|---|
| `$this->get` | `$_GET` | `$cliArgs`（parse_str 结果） |
| `$this->server` | `$_SERVER` | **未初始化**（始终 null） |
| `toArray()` | 返回 `$_GET` | 返回 `$cliArgs` |

**重要影响**：
- CLI 下 `$request->server('HTTP_IF_MODIFIED_SINCE')`、`$request->server('HTTP_HOST')` 等永远返回 null → **CacheMiddleware 的 304 协商在 CLI 下失效**，始终返回完整 body。
- `get_current_url()` 等依赖 `$_SERVER` 的辅助函数在 CLI 下可能返回空值或异常。
- SecurityMiddleware、BasicAuthMiddleware、TokenAuthenticationMiddleware 等依赖 `$this->server` 的中间件走默认分支，行为基本安全但无鉴权。

### 12.4 CLI 响应输出

`Response::send()`（`lib/http.php:294-314`）不区分 SAPI：HTTP 模式下设置 header + echo body，CLI 模式下直接 echo body 到 stdout，`header()` 调用被 PHP 静默忽略。因此 JSON/Atom/RSS/Plaintext 输出直接显示在终端，HTML 输出会看到完整的 HTML 源码。

---

## 十三、Debug 开关与日志分级：DEBUG 文件 + 环境 + handler 链

### 13.1 DEBUG 文件触发三重切换

`lib/Configuration.php:38-43` 在配置加载阶段检查项目根目录下是否存在 `DEBUG` 文件：

```php
if (file_exists(__DIR__ . '/../DEBUG')) {
    $debug = trim(file_get_contents(__DIR__ . '/../DEBUG'));
    if ($debug === '') {
        self::setConfig('system', 'env', 'dev');
        self::setConfig('cache', 'type', 'array');
    }
}
```

**注意**：只当 `DEBUG` 文件**内容为空**时触发（有内容的 DEBUG 文件不做任何事）。

触发的三重效果：

| 效果 | 值 | 说明 |
|---|---|---|
| `system.env` | `dev` | 替换默认的 `prod` |
| `cache.type` | `array` | 替换默认的 `file`，每次请求后缓存随进程销毁 |
| PHP 错误转异常 | 见 `index.php:32-35` | `env=dev` 时 `set_error_handler` 把所有非屏蔽错误升级为 `ErrorException`，严格暴露隐患 |

### 13.2 日志分级与 handler 装配

`lib/dependencies.php:48-64` 创建 `SimpleLogger`，根据 `system.env` 装配不同 handler：

```
                SimpleLogger(name='rssbridge')
                 │
                 ├─ env=dev → ErrorLogHandler(level=DEBUG)
                 │              → syslog(error_log PHP 默认目的地)
                 │
                 ├─ env=prod → ErrorLogHandler(level=INFO)
                 │              → syslog(同上)
                 │
                 └─ [logging].file_path && [logging].file_level 同时配置
                                → StreamHandler(level=file_level, stream=file_path)
                                      → 追加写入文件 (FILE_APPEND)
```

四级日志常量（`lib/logger.php:7-10`）：

| level | 数值 | 触发场景 |
|---|---|---|
| `DEBUG` | 10 | ClientException 捕获、RateLimitException 捕获、调试追踪 |
| `INFO` | 20 | 启用的 Bridge 未找到、服务启动信息 |
| `WARNING` | 30 | 缓存反序列化失败、磁盘写失败、PHP 非致命错误（prod 模式下）、SQLite 执行失败 |
| `ERROR` | 40 | 未知异常（非 Client/RateLimit/指定 HttpException 类型）、PHP Fatal（shutdown）、未捕获异常 |

### 13.3 日志过滤与异常脱敏

`SimpleLogger::log()`（`lib/logger.php:69-100`）做了两级过滤：

**第一级：完全丢弃特定异常类型与消息**
```php
if ($e instanceof RateLimitException) return;  // 限流太吵，完全不记
foreach ($ignoredMessages as $ignored) {
    if (str_starts_with($e->getMessage(), $ignored)) return;
}
// 被忽略的消息前缀：'Format name invalid', 'Unknown format given', 'Unable to find'
```

**第二级：handler 阈值过滤**  
每个 handler 按 `$record['level'] < $this->level` 做数值比较，小则丢弃。因此 `StreamHandler(WARNING)` 只收 WARNING + ERROR，丢弃 DEBUG/INFO。

异常字段写入前经过 `sanitize_root()`（`lib/utils.php:xxx`）处理，去除项目根目录绝对路径，避免信息泄漏。

### 13.4 dev vs prod 的行为差异汇总

| 维度 | dev（DEBUG 文件空） | prod（默认） |
|---|---|---|
| 缓存后端 | ArrayCache（内存，请求间不持久） | FileCache（磁盘，持久） |
| PHP 错误升级 | 所有非屏蔽错误 → `ErrorException` 抛出 | WARNING 级别写入日志，继续执行 |
| ErrorLogHandler 阈值 | DEBUG（全量） | INFO（仅 INFO/WARNING/ERROR） |
| 日志量 | 大，含 ClientException 等调试信息 | 小，仅业务级别以上 |

---

## 十四、Trial / Test 模式：三类运行通路与自动化衔接

RSS-Bridge 没有单一 "trial mode" 开关，而是由**生产端 URL 试跑**、**参数自动探测**、**单元测试 BridgeImplementationTest** 三套机制组合实现。

### 14.1 路径一：前端 Generate feed（直接试跑）

`actions/FrontpageAction.php` 渲染每个 Bridge 的参数表单，提交后走标准 `action=display&format=Html` 通路——**即生产路径本身就是 trial 路径**，没有独立模式。前端通过 `static/rss-bridge.js:41-46` 提供 "Use example value" 按钮填充 `exampleValue`，便于开发者试跑。

### 14.2 路径二：DetectAction / FindfeedAction（URL 自动参数探测）

这是面向最终用户的"我给你一个网页地址，你帮我匹配 Bridge 并生成参数"的自动化 trial 流程。

**DetectAction**：遍历所有启用的 Bridge，调用 `detectParameters($url)`，首个命中即 301 跳转：
```
输入: ?action=detect&url=https://example.com/...&format=Atom
       │
       ▼
  for 每个 enabled Bridge:
     $params = $bridge->detectParameters($url)
     $params !== null → 301 redirect 到
       ?action=display&bridge=Xxx&format=Atom&{http_build_query($params)}
  全部失败 → "No bridge found" 错误页
```

**FindfeedAction**：前端 feed finder 搜索框的后端，返回 JSON 数组（所有命中的 Bridge 及其参数元数据）供前端多选展示，不做重定向。

`detectParameters()` 默认实现（`lib/BridgeAbstract.php:308-323`）仅针对**无参数 Bridge**（`PARAMETERS` 为空）做 URL 主域名匹配；有参数的 Bridge 需自行重载此方法，从 URL 中提取子路径/查询参数并重建为 `PARAMETERS` 对应的键值对数组，可选包含 `'context' => 'By user ID'` 指定 context 名。

`tests/BridgeImplementationTest.php:154-157` 对 `const TEST_DETECT_PARAMETERS = ['url1' => ['expected_param1' => 'v1'], ...]` 做断言，验证 `detectParameters($url)` 输出与预期一致。

### 14.3 路径三：PHPUnit 自动化测试（BridgeImplementationTest）

`tests/BridgeImplementationTest.php` 是 CI 级别的 trial/test，每次提交运行，覆盖：

| 测试方法 | 校验内容 | 示例 |
|---|---|---|
| `testClassName` | 命名规范（大写开头、无空格、以 Bridge 结尾） | - |
| `testClassType` | 必须继承 `BridgeAbstract` | - |
| `testConstants` | NAME/URI/DESCRIPTION/MAINTAINER 非空字符串、PARAMETERS 为数组、CACHE_TIMEOUT ≥ 0 且为 int | - |
| `testParameters` | PARAMETERS schema 有效性：type 四种内、list 必须有 values、checkbox/list 不得声明 required、pattern/exampleValue/defaultValue 格式合法、TEST_DETECT_PARAMETERS 映射正确 | 见 §11.1 表 |
| `testMethodValues` | getDescription/getMaintainer/getName/getURI/getIcon 返回非空字符串 | - |
| `testUri` | URI 常量和 `getURI()` 返回值必须通过 `FILTER_VALIDATE_URL` | - |

使用 NullCache + NullLogger 构造 Bridge（隔离 I/O），保证纯静态校验可离线运行。`dataBridgesProvider()` 用 `glob(bridges/*Bridge.php)` 遍历项目内所有 Bridge 类做参数化测试。

---

## 十五、Contrib 自定义桥：Docker 挂载注册 + 文件系统扫描发现

RSS-Bridge 没有独立的 `contrib/` 插件目录扫描机制（`contrib/` 目录仅含 `.gitkeep` 为空），自定义桥的注册与发现通过**两条路径**实现。

### 15.1 路径一：直接文件放置（所有部署方式通用）

与内置 Bridge 完全相同：将 `XxxBridge.php` 放入 `bridges/` 目录即可。三个地方会自动发现：

1. **BridgeFactory 构造函数**（`lib/BridgeFactory.php:19-23`）：
   ```php
   foreach (scandir(__DIR__ . '/../bridges/') as $file) {
       if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
           $this->bridgeClassNames[] = $m[1];
       }
   }
   ```
   正则只匹配 `*Bridge.php`，非 Bridge 命名的文件会被忽略。

2. **SPL 自动加载器**（`lib/bootstrap.php:29-44`）：
   ```php
   spl_autoload_register(function ($className) {
       $folders = [
           __DIR__ . '/../bridges/',  // ← 已注册
           __DIR__ . '/../actions/',
           ...
       ];
       foreach ($folders as $folder) {
           $file = $folder . $className . '.php';
           if (is_file($file)) require $file;
       }
   });
   ```

3. **enabled_bridges 白名单**（`config.default.ini.php:11-12`）：
   ```ini
   [system]
   enabled_bridges[] = "*"    ; 或逐一列出 enabled_bridges[] = "XxxBridge"
   ```
   `*` 表示全部启用，新增 Bridge 无需改配置。如果是白名单模式，需要在 `config.ini.php` 中追加。

### 15.2 路径二：Docker /config 目录挂载（容器部署专属）

`docker-entrypoint.sh:8-32` 在容器启动时执行一次性注册：

```bash
find /config/ -type f -name '*' -print0 2> /dev/null |
while IFS= read -r -d '' file; do
    file_name="$(basename "$file")"
    case "$file_name" in
    *Bridge.php)    yes | cp "$file" /app/bridges/ ;
                    chown www-data:www-data "/app/bridges/$file_name";
                    printf "Custom Bridge %s added.\n" $file_name;;
    *Format.php)    yes | cp "$file" /app/formats/ ;
                    chown www-data:www-data "/app/formats/$file_name";
                    printf "Custom Format %s added.\n" $file_name;;
    config.ini.php) yes | cp "$file" /app/ ;
                    ...
    whitelist.txt)  yes | cp "$file" /app/ ;
                    ...
    DEBUG)          yes | cp "$file" /app/ ;
                    ...
    esac
done
```

**关键约束**：
- 文件名不能含空格（含空格直接跳过）
- 只匹配 `*Bridge.php` 后缀
- 用 `yes | cp` 强制覆盖同名文件（允许自定义桥覆盖内置桥）
- `chown www-data` 确保 PHP 进程可读
- **仅在容器启动时执行一次**，运行期修改 `/config/` 不会自动同步，需重启容器

### 15.3 命名冲突与优先级

- 文件名匹配：`scandir` 按字典序，同名文件后出现的不会加入 `bridgeClassNames` 数组（但 Docker 拷贝会覆盖文件内容，实际以后拷贝的为准）
- 大小写不敏感匹配：`createBridgeClassName()` 用 `strtolower` 做 array_search，所以 `?bridge=githubrelease` 和 `?bridge=GithubRelease` 都能命中 `GithubReleaseBridge`
- 自动加载优先级：`spl_autoload_register` 按 `$folders` 数组顺序查找，`bridges/` 在最前，所以 `bridges/XxxBridge.php` 优先级高于其他目录同名文件（但其他目录不会放 Bridge）

---

## 十六、Bridge Factory 热加载：每次请求 scandir，无持久化缓存

### 16.1 无缓存的扫描机制

`BridgeFactory` 和 `FormatFactory`、`CacheFactory` 一样，**没有使用 CacheInterface 缓存扫描结果**，每次实例化都会重新 `scandir` 目录：

| 组件 | 构造函数扫描 | 扫描路径 | 缓存方式 |
|---|---|---|---|
| `BridgeFactory` | `scandir(__DIR__ . '/../bridges/')` | 硬编码相对路径 | 无 |
| `FormatFactory` | `scandir(__DIR__ . '/../formats/')` | 硬编码相对路径 | 无 |
| `CacheFactory` | `scandir(PATH_LIB_CACHES)` | 常量定义路径 | 无 |

每次扫描的开销：3 个目录 × 数百文件的 `scandir` + 正则匹配，在现代 SSD 上单请求增加 1~3ms。

### 16.2 与 DI 容器的生命周期衔接

`lib/dependencies.php:24-28` 在 Container 中注册为 lazy 工厂：

```php
$container['bridgeFactory'] = function ($c) {
    return new BridgeFactory($c['cache'], $c['logger']);
};
```

`Container::offsetGet()`（`lib/Container.php:21-24`）做单例缓存：

```php
if (!isset($this->resolved[$offset])) {
    $this->resolved[$offset] = $this->values[$offset]($this);
}
return $this->resolved[$offset];
```

**生命周期模型**：
```
HTTP 请求到达
    │
    ▼
index.php → 创建新 Container（每次请求全新实例）
    │
    ▼
RssBridge::run() → DisplayAction 构造参数需要 BridgeFactory
    │
    ▼
Container::offsetGet('bridgeFactory')
    → resolved['bridgeFactory'] 为空 → 调用工厂函数
    → new BridgeFactory → scandir bridges/ → 存 resolved 缓存
    │
    ▼
DisplayAction 执行 → 多次调用 bridgeFactory 方法
    → 全部复用同一个实例，不再 scandir
    │
    ▼
请求结束 → PHP 进程销毁 → Container 和 resolved 缓存全部销毁
```

### 16.3 "热加载"的真实含义

因为**每次请求都重新扫描**，所以：
- ✅ 新增 Bridge 文件后，**下一个请求自动发现**，无需重启服务器/清空缓存
- ✅ 删除 Bridge 文件后，下一个请求自动失效
- ⚠️ 同一请求过程中新增/删除文件**不会被感知**（因为实例已缓存）
- ⚠️ 没有 `reload()` / `refresh()` API 手动触发重扫描
- ⚠️ 没有基于文件 mtime 的增量扫描（每次全量扫描）

这与传统的"热加载"（进程内监听文件变化自动重载）不同——RSS-Bridge 靠** PHP shared-nothing 架构天然实现热加载**，每个请求是独立进程，不存在进程内缓存需要失效的问题。

### 16.4 enabled_bridges 白名单的静态性

`enabled_bridges` 配置在 `Configuration` 构造时（`lib/Configuration.php:27-35`）从 `config.ini.php` 读取，**不会随 BridgeFactory 扫描刷新**。因此：
- 白名单模式下列出的不存在的 Bridge 会在每次请求时被 `logger->info()` 记录一次（`lib/BridgeFactory.php:38-40`）
- 新增 Bridge 后如果用白名单模式，必须修改 `config.ini.php` 才会生效（Configuration 每次请求也重新读取，所以修改配置后下个请求生效，无需重启）

---

## 十七、Cron 调度：无内置调度器，依赖外部拉模式

**RSS-Bridge 代码中没有任何 cron 调度器、定时任务或后台进程**。整个系统基于**拉模式（Pull Model）**设计：

```
RSS 阅读器 ──定期 HTTP GET──▶ RSS-Bridge ──HTTP GET──▶ 源网站
             (间隔由阅读器配置)           (按需即时抓取)
```

### 17.1 代码中"cron"的真实含义

Grep 到的 6 个含 "cron" 的文件，全部是** Bridge 解析数据源中的 cron 字段**，不是系统调度：

| 文件 | cron 用途 |
|---|---|
| `bridges/CachetBridge.php` | 解析 Cachet 状态页数据中的 `cron` 字段（数据源本身的元数据） |
| `bridges/BAEBridge.php` | 同上 |
| `bridges/NHKWorldJapanShowBridge.php` | 同上 |
| `bridges/KilledbyMicrosoftBridge.php` | 同上 |
| `static/connectivity.js` | 前端连通性测试的调度描述文案 |
| `lib/parsedown/Parsedown.php` | Markdown 解析器的无关代码 |

### 17.2 调度触发的四种外部方式

实际部署中，cron 调度完全由外部系统实现：

**方式一：RSS 阅读器轮询（最常见）**
- 阅读器按用户配置的间隔（通常 15~60 分钟）定期请求 Feed URL
- RSS-Bridge 按需执行 Bridge::collectData() → 抓取源站 → 渲染格式
- 服务端缓存（`http_` 命名空间）降低源站压力

**方式二：Linux crontab + curl**
```bash
*/15 * * * * curl -s "https://rss-bridge.example.com/?action=display&bridge=Xxx&format=Atom" > /dev/null
```
- 主动预热缓存，让第一个真实用户请求命中缓存
- 适合热门 Bridge

**方式三：Docker HEALTHCHECK 间接触发**
`Dockerfile` 定义：
```dockerfile
HEALTHCHECK --interval=5m --timeout=10s --retries=3 \
    CMD curl -f http://localhost/ || exit 1
```
每 5 分钟访问首页，触发一次 PHP 进程，但不执行具体 Bridge。

**方式四：第三方调度服务（IFTTT / Zapier / n8n）**
- Webhook 定时触发 RSS-Bridge 的 DisplayAction
- 结果推送到通知渠道

### 17.3 间接的"调度效果"：缓存 TTL

虽然没有调度器，但 `http_` 缓存的 TTL 机制产生了类似调度的效果：
```
T0: 首次请求 → 抓取源站 → 渲染 → 缓存（TTL=3600s）→ 返回 200
T0+1800s: 第二次请求 → 缓存命中 → 返回缓存 → 不抓取源站
T0+3601s: 第三次请求 → 缓存过期 → 重新抓取源站 → 刷新缓存 → 返回新 200
```
这意味着**每小时最多抓取一次源站**（默认 `CACHE_TIMEOUT = 3600`），等效于 1 小时的最低调度间隔。如果阅读器轮询间隔短于 TTL，实际抓取频率由 TTL 决定。

### 17.4 设计取舍

不内置 cron 的原因：
1. PHP shared-nothing 模型不适合常驻后台进程
2. 拉模式与 Feed 生态天然契合（阅读器负责调度）
3. 降低部署复杂度（无需额外守护进程）
4. 源站频率限制由缓存 TTL 间接保证

---

## 十八、多 Bridge 并发 Fetch：无并发调度，单请求单 Bridge 串行

**RSS-Bridge 没有多 Bridge 并发抓取机制，也没有跨请求的全局限流。**

### 18.1 单请求的执行模型

每个 HTTP 请求对应 **一个 Bridge**，串行执行：

```
Request ?action=display&bridge=GithubRelease&u=rss-bridge
    │
    ▼
DisplayAction::execute()
    ├─ 1. BridgeFactory->create('GithubReleaseBridge')
    ├─ 2. $bridge->setInput($request->toArray())      # 参数校验
    ├─ 3. $bridge->collectData()                      # ⬅ 抓取发生在这里
    │     └─ getContents('https://github.com/rss-bridge/releases')  # 单次 HTTP GET
    │     └─ 解析 HTML → 组装 FeedItem[]
    ├─ 4. FormatFactory->create('Atom')
    └─ 5. $format->setItems($items)->render()
```

即使 Bridge 内部多次调用 `getContents()`，也是**串行发起**，没有 `curl_multi`、没有 `Guzzle Promise`、没有异步队列。

### 18.2 "多 Bridge"场景的实际处理

只有两个 Action 会遍历多个 Bridge，但**均不并发、也不抓取**：

**DetectAction**（`actions/DetectAction.php:25-38`）：
```php
foreach ($this->bridgeFactory->getBridgeClassNames() as $bridgeClassName) {
    if (!$this->bridgeFactory->isEnabled($bridgeClassName)) continue;
    $bridge = $this->bridgeFactory->create($bridgeClassName);
    $params = $bridge->detectParameters($url);  # ⬅ 默认实现不发 HTTP 请求
    if ($params !== null) {
        $format = $request->get('format');
        $uri = '?action=display&bridge=' . $bridgeClassName .
               '&format=' . $format . '&' . http_build_query($params);
        return new Response('', 301, ['Location' => $uri]);
    }
}
```
`detectParameters()` 默认实现（`lib/BridgeAbstract.php:308-323`）仅做 URL 字符串匹配，不发起 HTTP 请求。少数重载此方法的 Bridge 会发请求，但也是串行逐个。

**FindfeedAction**（`actions/FindfeedAction.php:31-46`）：
同上，但是收集所有命中结果返回 JSON 数组，不做 301 跳转。

### 18.3 限流的真实边界

保护措施完全在**单请求单次抓取**层面，没有全局维度：

| 限流维度 | 实现方式 | 作用范围 |
|---|---|---|
| 单请求超时 | `CURLOPT_TIMEOUT = 5s`（默认） | 单次 curl 请求 |
| 单请求大小 | `CURLOPT_MAXFILESIZE + CURLOPT_PROGRESSFUNCTION` 20MB（默认） | 单次响应 |
| 重试次数 | `retries = 1`（默认） | 单次 curl 错误 |
| 源站频率 | 缓存 TTL（默认 3600s） + 源站返回 429 透传 | 同 Bridge 同参数 |
| 按域名限流 | ❌ 无 | - |
| 总并发数 | ❌ 无（依赖 php-fpm pm.max_children） | - |
| 全局 QPS | ❌ 无 | - |

### 18.4 PHP-FPM 层面的隐式并发控制

真实的并发上限由 PHP-FPM 配置决定（`config/php-fpm.conf`）：
```ini
pm = dynamic
pm.max_children = 5        ; 最大同时处理 5 个请求
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```
这意味着 RSS-Bridge 实例**最多同时处理 5 个 Bridge 请求**，超出的请求排队。这是隐式的、非业务逻辑层面的并发限制。

### 18.5 没有分布式锁与重复请求抑制

同一 Bridge 同一参数在缓存过期瞬间，可能收到多个并发请求导致"惊群效应"——同时多次抓取源站。代码中没有：
- `LockInterface` 或类似分布式锁
- `fetch_in_progress` 标记让后续请求等待第一个完成
- 基于 Bridge + 参数的请求去重

这是一个已知的设计简化，在缓存命中率高的场景下影响较小。

---

## 十九、关键文件索引

| 文件 | 职责 |
|---|---|
| `actions/DisplayAction.php` | 格式协商入口、编排数据流向、编码清洗、200 响应写入服务端缓存、异常分类降级、错误频率计数 |
| `actions/DetectAction.php` | URL → Bridge 参数自动探测（单命中 301 跳转） |
| `actions/FindfeedAction.php` | URL → Bridge 参数自动探测（多命中 JSON 数组返回） |
| `lib/BridgeFactory.php` | Bridge 名称规范化、白名单过滤、自动发现 |
| `lib/BridgeAbstract.php` | 所有 Bridge 基类，定义 PARAMETERS/CONFIGURATION/TEST_DETECT_PARAMETERS 常量、setInput 参数校验流程、detectParameters 默认实现 |
| `lib/ParameterValidator.php` | PARAMETERS schema 校验引擎（validateInput 引用传值过滤 + getQueriedContext 上下文推断） |
| `lib/FormatFactory.php` | 格式自动发现、名称规范化、类实例化 |
| `lib/FormatAbstract.php` | 格式抽象基类，定义 `setItems/setFeed/setLastModified/getMimeType`，完成 array→FeedItem 转换 |
| `lib/FeedItem.php` | 中间数据模型，字段规范化与校验、URI/enclosure sanitization |
| `lib/utils.php` | `parse_mime_type()` 基于扩展名推断 MIME、`ClientException` |
| `lib/RssBridge.php` | Action 路由解析（dash→camelCase）、中间件栈注册与洋葱模型编排 |
| `lib/Configuration.php` | 三层配置加载（默认→自定义→环境变量 RSSBRIDGE_*）与校验、DEBUG 文件三重切换 |
| `lib/logger.php` | Logger 接口 + 四级常量、SimpleLogger（过滤 + 分发）、ErrorLogHandler、StreamHandler、NullLogger |
| `lib/dependencies.php` | DI 容器：logger/cache/bridge_factory/http_client 的 lazy 装配 |
| `lib/CacheFactory.php` | 缓存后端自动发现、实例化与配置校验 |
| `lib/CacheInterface.php` | 缓存后端抽象接口 |
| `lib/url.php` | 严格 URL 解析器与 `UrlException` |
| `lib/php-urljoin/src/urljoin.php` | 相对 URL 转绝对（Python urljoin 移植） |
| `lib/html.php` | `defaultLinkTo()` 批量重写 HTML 中 img/a 链接、`parseSrcset()` |
| `lib/http.php` | `CurlHttpClient`（重试、超时、大小限流、条件请求）、`HttpException` / `CloudFlareException` / `RateLimitException`、Request（fromCli/fromGlobals）、Response |
| `lib/contents.php` | `getContents()`（服务端缓存 + 条件请求 + 状态码分支）、`getSimpleHTMLDOMCached()` |
| `lib/bootstrap.php` | PATH_* 常量、自动加载器、helper 全局函数加载 |
| `lib/Container.php` | DI 容器实现（ArrayAccess + lazy 单例缓存 `$resolved`） |
| `middlewares/CacheMiddleware.php` | 服务端缓存读取、304 Not Modified 协商、错误响应写入缓存、1% 概率触发 prune |
| `middlewares/ExceptionMiddleware.php` | 全局异常兜底，统一返回 500 HTML |
| `middlewares/SecurityMiddleware.php` | 查询参数类型校验（仅允许字符串） |
| `caches/FileCache.php` | 磁盘文件缓存（md5 文件名 + serialize） |
| `caches/SQLiteCache.php` | SQLite 单文件缓存（sha1 BLOB 主键 + WAL 模式） |
| `caches/MemcachedCache.php` | Memcached 分布式缓存 |
| `caches/ArrayCache.php` | 进程内内存缓存（DEBUG 模式默认） |
| `caches/NullCache.php` | 空实现（禁用缓存） |
| `formats/AtomFormat.php` | Atom 1.0 渲染（RFC 4287） |
| `formats/MrssFormat.php` | RSS 2.0 + Media RSS 渲染 |
| `formats/JsonFormat.php` | JSON Feed 1.0 渲染 |
| `formats/HtmlFormat.php` | HTML 预览页（含其他格式的跳转链接） |
| `formats/PlaintextFormat.php` | PHP print_r 调试输出 |
| `formats/SfeedFormat.php` | sfeed TSV 格式输出 |
| `tests/BridgeImplementationTest.php` | CI 级别 trial：所有 Bridge 的命名、常量、PARAMETERS schema、TEST_DETECT_PARAMETERS 断言 |
| `tests/ParameterValidatorTest.php` | ParameterValidator 的单测 |
| `static/rss-bridge.js` | 前端：Bridge 搜索、Feed Finder AJAX 搜索、exampleValue 一键填充 |
| `index.php` | 全局异常/错误/关闭 三层 handler 兜底、CLI vs HTTP 入口分流 |
| `docker-entrypoint.sh` | Docker 容器启动脚本：从 /config 挂载目录复制自定义 Bridge/Format/配置 |
| `config.default.ini.php` | 默认配置（http 超时/重试/大小、cache 类型、error 输出模式、env、enabled_bridges 等） |

---

## 二十、设计特点总结

1. **显式格式，零内容协商**：不依赖 HTTP Accept Header，用查询参数显式指定，简单可预测、便于缓存。
2. **基于文件系统的格式/缓存/Bridge 注册**：新增任一类只需在对应目录下添加 `*Format.php` / `*Cache.php` / `*Bridge.php`，Factory 扫描自动发现。
3. **FeedItem 作为防腐层**：Bridge 的原始数组与各格式渲染逻辑解耦，字段规范化与 URL sanitization 在中间层统一完成。
4. **格式间策略独立但约定一致**：UID 三级降级、MIME 类型声明、UTF-8 清洗等在各格式独立实现，逻辑基本一致但存在微妙差异（如 Json 的 UID 降级位置后置、Atom 对 itunes/alternate 的互斥处理）。
5. **错误输出保持格式承诺**：即使桥接失败，也按请求格式返回合法 Feed 文档，而不是 HTTP 错误页，保护 RSS 阅读器的解析链路。
6. **MIME 推断的锚点覆盖**：通过 `#.ext` 伪扩展名机制，允许 Bridge 强制指定 enclosure 的 MIME 类型，优先级高于真实文件扩展名。
7. **两级缓存双写 + 四命名空间隔离**：服务端缓存分两处写入——DisplayAction 管 200 正常响应（Bridge TTL），CacheMiddleware 管错误响应（带随机抖动的短 TTL），用 `http_` / `server_` / `pages_` / `error_reporting_` 前缀隔离。
8. **无 ETag 的简化协商**：对外响应仅实现 Last-Modified / If-Modified-Since，未实现 ETag；但对源站发起 getContents 请求时同时携带 If-Modified-Since 和 If-None-Match。
9. **URL 三级处理链**：严格 `Url` 类验证 → `urljoin()` 相对转绝对 → `defaultLinkTo()` HTML 批量补全，三层各司其职但覆盖范围不同（仍有 iframe/srcset 等盲区）。
10. **串行 Fetch + 被动限流**：无并发、无全局限流，依赖单请求超时（5s）、大小上限（20MB）、重试次数（1 次）和源站 429 透传；重试仅针对 curl 层错误，HTTP 4xx/5xx 不重试。
11. **异常四档分类 + report_limit 节流**：ClientException（静默）、RateLimitException/Http429/503（立即透传）、其他 HttpException（计数后降级）、未知 Exception（error 日志 + 计数），结合 `error_reporting_` 缓存域的 5 天滑动窗口阈值过滤偶发抖动。
12. **多层兜底防御**：DisplayAction try/catch → ExceptionMiddleware → index.php 全局 exception/error/shutdown handler 三层兜底，确保任何异常都不会暴露 PHP 原生堆栈。
13. **PARAMETERS 类常量 Schema + 双环校验**：无 YAML，用 PHP `const PARAMETERS = [...]` 定义参数模型，ParameterValidator 在运行时做类型/值校验，BridgeImplementationTest 在 CI 时做 schema 格式校验，引用传值同步完成类型转换。
14. **CLI 与 HTTP 同一入口**：不调用 `php_sapi_name()`，以 `$argv` 变量是否存在为 CLI 判断依据，argv 平铺为 key=value 后经 `parse_str` 进同一 Request 管道，action 路由和响应与 HTTP 完全一致。
15. **DEBUG 文件触发环境切换**：根目录下空 `DEBUG` 文件同时触发 `system.env=dev` + `cache.type=array` + PHP 错误转 ErrorException 三重效果，避免开发模式下缓存脏数据和隐藏的 PHP Warning。
16. **Trial/Test 三条通路并行**：生产路径即试跑路径（前端表单→DisplayAction）+ URL 参数自动探测（DetectAction/FindfeedAction + detectParameters）+ PHPUnit 静态扫描（BridgeImplementationTest 遍历所有 Bridge），三层覆盖从开发者手动试跑到 CI 自动合规检查。
17. **Contrib 自定义桥双路径注册**：直接放 `bridges/` 目录（所有部署通用，每次请求 scandir 自动发现） + Docker `/config` 挂载（容器启动时复制到 `/app/bridges/`），无需修改注册表。
18. **Factory 热加载靠 shared-nothing 天然实现**：Bridge/Format/Cache Factory 每次实例化 scandir 无持久化缓存，Container 单例仅在单请求内有效，新增文件下一个请求自动发现，无需重启或清理缓存。
19. **拉模式架构，无内置 Cron**：依赖 RSS 阅读器/外部 crontab 定时触发，缓存 TTL 间接限制源站抓取频率（默认最低 1 小时 1 次），契合 Feed 生态且降低部署复杂度。
20. **无并发调度，PHP-FPM 隐式限并发**：单请求单 Bridge 串行执行，Detect/Findfeed 遍历 Bridge 也为串行且默认不发 HTTP；无连接池、无分布式锁、无惊群效应抑制，并发上限由 php-fpm `pm.max_children = 5` 隐式控制。
