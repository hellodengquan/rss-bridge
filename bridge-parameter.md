# RSS-Bridge 参数化 Bridge 与依赖项联动全链路解析

## 一、全局视角：参数从声明到缓存的完整链路

```
PARAMETERS 常量声明
       │
       ▼
FrontpageAction::render()  ──→  渲染 HTML 表单（上下文分组、global 合并）
       │
       ▼
用户填写表单 → GET 请求（query string）
       │
       ▼
DisplayAction::createResponse()
       │
       ├─→ 剔除无关参数（token/action/bridge/format/...）
       │
       ├─→ BridgeAbstract::setInput()
       │       │
       │       ├─→ ParameterValidator::validateInput()    ──→  类型校验 & 值清洗
       │       │
       │       ├─→ ParameterValidator::getQueriedContext() ──→  上下文推断
       │       │
       │       └─→ setInputWithContext()                   ──→  默认值回填 & global 注入
       │
       ├─→ BridgeAbstract::collectData()                   ──→  业务逻辑中使用 getInput()
       │
       └─→ CacheMiddleware / DisplayAction                  ──→  缓存键 = "http_" + json_encode($request->toArray())
```

---

## 二、参数声明机制（PARAMETERS 常量）

### 2.1 两层结构：Context → Parameter

参数声明在 `BridgeAbstract::PARAMETERS` 常量中，采用 **两层嵌套数组**：

```php
const PARAMETERS = [
    // Level 1: Context Name（命名上下文 / 数字索引匿名上下文 / "global"）
    'By keyword' => [
        // Level 2: Parameter Definitions（参数名 => 参数规格）
        'q' => [
            'name'          => 'Keyword',       // 必填，显示给用户的名称
            'type'          => 'text',          // 可选：text|number|list|checkbox，默认 text
            'required'      => true,            // 可选：是否必填
            'title'         => 'Insert keyword',// 可选：hover 提示
            'exampleValue'  => 'bird',          // 可选：placeholder 示例值
            'pattern'       => '[a-z]+',        // 可选：text 类型的正则校验
            'defaultValue'  => '',              // 可选：默认值（行为因 type 而异）
        ],
        'media' => [
            'name'          => 'Media',
            'type'          => 'list',
            'values' => [                       // list 类型必填：option 显示名 => option value
                'All (Photos & videos)' => 'all',
                'Photos' => 'photos',
                'Videos' => 'videos',
            ],
            'defaultValue'  => 'all',
        ],
    ],
];
```

**源码位置**: `lib/BridgeAbstract.php:21` (`const PARAMETERS = [];`)

### 2.2 三种 Context 形态

| 形态 | 声明方式 | 前端表现 | 示例 |
|------|----------|----------|------|
| **命名上下文** | `'Context Name' => [...]` | 显示分组标题 `<h5>`，每个上下文一个独立表单 | FlickrBridge 的 `'By keyword'` / `'By username'` |
| **匿名上下文** | `[0] => [...]` 或直接 `[...]` | 无标题，单个表单 | MinecraftBridge 的 `[[ 'category' => ... ]]` |
| **global 上下文** | `'global' => [...]` | 不单独展示，合并到其他上下文的表单中 | YoutubeBridge 的 `duration_min` / `duration_max` |

**源码位置**: `actions/FrontpageAction.php:104-129`（渲染时对三种形态的分支处理）

### 2.3 defaultValue 的类型相关行为

| type | defaultValue 语义 | 回填逻辑 |
|------|-------------------|----------|
| `text` | 任意文本字符串 | `BridgeAbstract.php:228-230`: 仅当用户未填时才回填 |
| `number` | 任意数字 | 同 text |
| `checkbox` | `"checked"` 表示默认勾选 | `BridgeAbstract.php:213-215`: 未提交时默认 `false` |
| `list` | 匹配 values 中的 name 或 value | `BridgeAbstract.php:217-226`: 无 defaultValue 时取 values 第一项 |

### 2.4 内置 LIMIT 参数

`BridgeAbstract` 提供了一个受保护的常量，供子类直接引用：

```php
protected const LIMIT = [
    'name' => 'Limit',
    'type' => 'number',
    'title' => 'Maximum number of items to return',
];
```

Bridge 子类可这样使用：`'limit' => self::LIMIT`，如 `GQMagazineBridge.php:36`。

---

## 三、依赖联动机制

### 3.1 上下文推断：从用户输入反推 Context

当用户提交表单时，请求中不含 `context` 参数（或为空），系统需从输入参数集合推断用户意图的上下文。

**核心方法**: `ParameterValidator::getQueriedContext()` (`lib/ParameterValidator.php:61-121`)

推断算法：

```
1. 遍历所有 context：
   a. 检查用户输入是否「完全属于」该 context（排除 global 参数后，不应有额外字段）
      → 不属于则跳过
   b. 检查该 context 的「必填参数」是否全部有值
      → 有值则标记 true（完全匹配）
      → 有必填缺值则标记 false（不满足）
      → 无任何输入则标记 null（空上下文）

2. 汇总判定：
   - sum = 0：无匹配
      → 有 context 参数则用其值
      → 否则找第一个 null 的上下文（参数均为空的上下文）
      → 否则返回 null（参数缺失）
   - sum = 1：唯一匹配 → 返回该 context 名
   - sum > 1：多上下文混合 → 返回 false（抛出 "Mixed context parameters" 异常）
```

### 3.2 global 参数合并

**渲染侧合并** (`FrontpageAction.php:117-119`):
```php
if (array_key_exists('global', $parameters)) {
    $contextParameters = array_merge($contextParameters, $parameters['global']);
}
```

**运行侧合并** (`BridgeAbstract::setInputWithContext()`, `BridgeAbstract.php:195-253`):
```php
$contextNames = [$queriedContext];
if (array_key_exists('global', $parameters)) {
    $contextNames[] = 'global';
}
// 遍历 $contextNames，对每个 context 中的参数执行默认值回填

// 最后将 global 参数值复制到 queriedContext
if (array_key_exists('global', $parameters)) {
    foreach ($parameters['global'] as $name => $parameter) {
        // 如果用户提交了就用提交值，否则用默认值
        $this->inputs[$queriedContext][$name]['value'] = $value;
    }
}
```

合并后只保留 `queriedContext` 的参数：
```php
$this->inputs = [$queriedContext => $this->inputs[$queriedContext]];
```

### 3.3 Bridge 间继承依赖

Bridge 之间可通过 PHP 继承实现代码复用，例如 `ReleasesSwitchBridge` 继承自 `Releases3DSBridge`：

```php
// bridges/ReleasesSwitchBridge.php
if (!class_exists('Releases3DSBridge')) {
    include('Releases3DSBridge.php');
}
class ReleasesSwitchBridge extends Releases3DSBridge { ... }
```

`FeedExpander` 也是典型的继承扩展，它继承 `BridgeAbstract` 并扩展了 `collectExpandableDatas()` 方法，子类只需实现 `parseItem()` 即可。

### 3.4 关于 visible / depends 等前端联动

**当前 rss-bridge 代码中不存在** `visible`、`depends`、`dependency` 等前端参数联动机制。前端 JS (`static/rss-bridge.js`) 仅实现了：

- 搜索过滤 (`rssbridge_list_search`)
- 桥梁折叠展开 (`rssbridge_toggle_bridge`)
- 示例值点击填入 (`rssbridge_use_placeholder_value`)
- URL 自动检测订阅源 (`rssbridge_feed_finder`)

参数间的联动完全由 **上下文机制** 实现：不同上下文拥有不同参数集合，用户选择上下文即确定了可用参数范围。这是一种「粗粒度」联动，而非参数级别的条件显隐。

---

## 四、用户输入校验流程

### 4.1 入口：BridgeAbstract::setInput()

```php
// lib/BridgeAbstract.php:138-179
public function setInput(array $input)
{
    // 1. 提取 context 提示（可选，来自隐藏字段）
    $contextName = $input['context'] ?? null;

    // 2. 无参数的 Bridge 不接受任何输入
    if (!$parameters) {
        if ($input) throwClientException('Unexpected parameters');
        return;
    }

    // 3. 校验 & 清洗（$input 按引用传递，会被原地修改）
    $validator = new ParameterValidator();
    $errors = $validator->validateInput($input, $parameters);

    // 4. 推断上下文（若未通过隐藏字段指定）
    if (empty($this->queriedContext)) {
        $this->queriedContext = $validator->getQueriedContext($input, $parameters);
    }

    // 5. 将清洗后的输入绑定到上下文
    $this->setInputWithContext($input, $this->queriedContext);
}
```

### 4.2 ParameterValidator::validateInput()

**源码**: `lib/ParameterValidator.php:8-58`

按类型分发校验：

| type | 校验方法 | PHP 过滤器 | 失败返回 |
|------|----------|-----------|----------|
| `text` (无 pattern) | `validateTextValue()` | `filter_var($value)` | `null` |
| `text` (有 pattern) | `validateTextValue($value, $pattern)` | `FILTER_VALIDATE_REGEXP` | `null` |
| `number` | `validateNumberValue()` | `FILTER_VALIDATE_INT` | `null` |
| `checkbox` | `validateCheckboxValue()` | `FILTER_VALIDATE_BOOLEAN` (带 `FILTER_NULL_ON_FAILURE`) | `null` |
| `list` | `validateListValue()` | `filter_var($value)` + `in_array` 检查 | `null` |

**关键细节**：
- `$input` 是 **按引用传递** 的，校验方法会直接修改 `$input[$name]` 的值为清洗后的结果
- 未在 PARAMETERS 中注册的参数会报 `"Parameter is not registered!"` 错误
- 校验结果为 `null` 且 `required === true` 的参数会报 `"Parameter is invalid!"` 错误

### 4.3 默认值回填：setInputWithContext()

**源码**: `lib/BridgeAbstract.php:181-263`

回填顺序：

```
1. 将用户输入绑定到对应 context
2. 遍历 queriedContext + global 中的所有参数
3. 对缺失值按类型回填：
   - checkbox → false
   - list → values 第一项 或 defaultValue
   - 其他 → defaultValue（有则填，无则留空）
4. 将 global 参数值复制到 queriedContext
5. 清理，只保留 queriedContext 的参数
```

### 4.4 getInput()：业务层取值

```php
// lib/BridgeAbstract.php:265-268
protected function getInput($input)
{
    return $this->inputs[$this->queriedContext][$input]['value'] ?? null;
}
```

Bridge 子类在 `collectData()` 中通过此方法读取参数值。

### 4.5 getKey()：获取 list 类型的键名

```php
// lib/BridgeAbstract.php:277-306
public function getKey($input)
```

对于 list 类型参数，`getInput()` 返回的是 option 的 value，而 `getKey()` 返回的是对应的 key（显示名）。支持两级嵌套的 values 结构。

---

## 五、缓存键拼装逻辑

### 5.1 HTTP 响应缓存键

**构建方式**: `CacheMiddleware.php:24` 和 `DisplayAction.php:50`

```php
$cacheKey = 'http_' . json_encode($request->toArray());
```

其中 `$request->toArray()` 返回的是完整的 `$_GET` 数组（`lib/http.php:248-251`）。

**缓存键的组成**：
```
http_ + json_encode({
    "action":    "display",
    "bridge":    "Flickr",
    "context":   "By keyword",
    "q":         "bird",
    "media":     "all",
    "sort":      "relevance",
    "format":    "Atom",
    "token":     "xxx",        // 如有认证
    "_noproxy":  "",           // 如有代理
    "_cache_timeout": "3600"   // 如有自定义超时
})
```

**重要特征**：
- **所有 GET 参数** 都参与缓存键计算，包括 `format`、`token`、`_noproxy`、`_cache_timeout` 等
- 不同格式的同一请求会产生不同的缓存条目
- 参数值的顺序差异会导致不同的缓存键（JSON 序列化的键顺序依赖 PHP 数组顺序）

### 5.2 Bridge 内部缓存键

Bridge 可通过 `saveCacheValue` / `loadCacheValue` 存取内部缓存：

```php
// lib/BridgeAbstract.php:325-333
protected function loadCacheValue(string $key, $default = null)
{
    return $this->cache->get($this->getShortName() . '_' . $key, $default);
}

protected function saveCacheValue(string $key, $value, int $ttl = 86400)
{
    $this->cache->set($this->getShortName() . '_' . $key, $value, $ttl);
}
```

内部缓存键格式：`{BridgeShortName}_{key}`，例如 `YoutubeBridge_youtube_rate_limit`。

### 5.3 缓存 TTL 来源

```php
// DisplayAction.php:57-63
$ttl = $request->get('_cache_timeout');
if (Configuration::getConfig('cache', 'custom_timeout') && isset($ttl)) {
    $ttl = (int) $ttl;
} else {
    $ttl = $bridge->getCacheTimeout();  // 来自 Bridge 的 CACHE_TIMEOUT 常量
}
```

优先级：用户自定义超时（需配置允许） > Bridge 定义的 CACHE_TIMEOUT > 默认 3600 秒。

### 5.4 错误报告缓存键

```php
// DisplayAction.php:175
$cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
```

---

## 六、全链路协作关系总结

```
┌─────────────────────────────────────────────────────────────────┐
│                     PARAMETERS 常量声明                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Context 1: [param_a, param_b, ...]                     │   │
│  │  Context 2: [param_c, param_d, ...]                     │   │
│  │  global:    [param_x, param_y, ...]                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────┬──────────────────────────────────┬─────────────────┘
            │                                  │
     渲染路径（读）                      运行路径（读+写）
            │                                  │
            ▼                                  ▼
┌──────────────────────┐          ┌──────────────────────────┐
│  FrontpageAction     │          │  DisplayAction           │
│  ::render()          │          │  ::createResponse()      │
│                      │          │                          │
│  • 遍历 context      │          │  • 剔除无关参数          │
│  • 合并 global       │          │  • setInput()            │
│  • 渲染 HTML 表单    │          │    ├ validateInput()     │
│  • 添加系统参数      │          │    ├ getQueriedContext() │
│    (_noproxy,        │          │    └ setInputWithContext()│
│     _cache_timeout)  │          │  • collectData()         │
└──────────┬───────────┘          │  • getItems()            │
           │                      └──────────┬───────────────┘
           │                                 │
           ▼                                 ▼
     用户浏览器                         ┌──────────────┐
     GET ?action=display                │ CacheMiddleware│
         &bridge=Flickr                 │               │
         &context=By+keyword            │ cacheKey =    │
         &q=bird                        │ "http_" +     │
         &media=all                     │  json_encode( │
         &format=Atom                   │   $_GET       │
           │                            │  )            │
           │                            └──────┬───────┘
           │                                   │
           └───────────► 请求 ─────────────────┘
                           │
                    ┌──────▼───────┐
                    │  Cache Store │
                    │  (SQLite等)  │
                    └──────────────┘
```

### 关键协作要点

1. **PARAMETERS 是唯一真相源**：声明、渲染、校验、默认值回填全部依赖这同一个常量
2. **上下文是参数联动的唯一机制**：没有参数级联显隐，只有上下文级分组
3. **global 参数是跨上下文共享的桥梁**：声明在 global 中的参数自动合并到所有上下文
4. **缓存键包含全部 GET 参数**：参数值的任何差异（包括 format、token）都产生不同缓存
5. **校验与清洗一体**：`ParameterValidator::validateInput()` 同时完成类型校验和值清洗，原地修改 `$input`
6. **默认值回填在 setInputWithContext 中完成**：校验只处理用户提交的值，未提交的参数由回填逻辑根据类型补全
