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

---

## 七、深度细节：边界场景与扩展机制

### 7.1 PARAMETERS 热更新场景

**PHP 常量的不可变性**：`PARAMETERS` 是类级 `const` 常量，在 PHP 编译阶段绑定到类，一旦定义就**无法在运行时修改**。尝试 `$bridge::PARAMETERS = [...]` 会直接报错。

**热更新路径**：只有修改 `.php` 源码文件，让 PHP 重新解析才能更新 PARAMETERS。但这里有两层缓存需要注意：

1. **OPcache 字节码缓存**：生产环境通常启用 `opcache.enable=1`，修改源码后 PHP 不会立即重新解析。需调用 `opcache_reset()` 或等待 `opcache.revalidate_freq`（默认 2 秒）过期。

2. **BridgeFactory 类名缓存**：`BridgeFactory::__construct()` (`lib/BridgeFactory.php:18-23`) 在实例化时扫描 `bridges/` 目录生成可用类名列表。如果新增 Bridge 文件，需要重新实例化 `BridgeFactory`（通常每次请求都会重新创建，所以无影响）。

**热更新的风险**：
- PARAMETERS 变更不会自动作废已有缓存，旧参数生成的缓存键可能与新参数不兼容
- 前端用户可能看到旧表单（浏览器缓存），提交后与新参数结构不匹配
- 建议：修改 PARAMETERS 后手动清空缓存目录

### 7.2 Context 命名冲突解决

PHP 数组的键名具有**唯一性**，后声明的键会**静默覆盖**先声明的同名键。

**PARAMETERS 内部冲突**：
```php
const PARAMETERS = [
    'Context A' => [ 'q' => ... ],
    'Context A' => [ 'u' => ... ],  // 覆盖前者！
];
```
第二个 `'Context A'` 会完全覆盖第一个，前者的参数定义永久丢失，无任何警告。

**渲染侧冲突（global 合并）**：
```php
// FrontpageAction.php:117-119
$contextParameters = array_merge($contextParameters, $parameters['global']);
```
`array_merge()` 的行为是：**后面的数组覆盖前面的数组中相同字符串键**。如果某个 context 中的参数名与 global 中的参数名相同，**global 的定义会覆盖 context 的定义**。

**防范措施**：
- Bridge 开发者需自行确保 context 名称和参数名的唯一性
- global 参数命名应使用不易冲突的前缀（如 `global_` 或 `sys_`）
- 代码审查时重点检查 PARAMETERS 中的键名重复

### 7.3 LIMIT 边界值兜底

**内置 LIMIT 常量** (`lib/BridgeAbstract.php:25-31`) 只定义了参数元数据，**不包含任何边界校验逻辑**：
```php
protected const LIMIT = [
    'name' => 'Limit',
    'type' => 'number',
    'title' => 'Maximum number of items to return',
];
```

**校验层**：`ParameterValidator::validateNumberValue()` (`lib/ParameterValidator.php:137-144`) 仅用 `FILTER_VALIDATE_INT` 检查是否为合法整数，**不做 min/max 范围校验**。负值、零、极大值都能通过校验。

**业务层兜底**：边界值完全由 Bridge 业务代码自行处理，常见模式：

| 兜底模式 | 示例代码 | 说明 |
|---------|----------|------|
| `?:` 短路默认值 | `$limit = $this->getInput('limit') ?: 5;` | 0、null、false 都会触发兜底 |
| `??` 空合并 | `$limit = $this->getInput('limit') ?? 10;` | 仅 null 触发兜底，0 是有效值 |
| `max()` 保护下界 | `$limit = max(1, $this->getInput('limit'));` | 确保至少为 1 |
| `min()` 保护上界 | `$limit = min(50, $this->getInput('limit'));` | 限制最大返回数量 |
| `array_slice` 容错 | `array_slice($items, 0, $limit)` | `$limit` 为 null 时取全部，为负时从末尾截取 |

**风险**：如果业务代码未做边界兜底，用户传入 `-1` 或 `999999` 可能导致：
- 上游 API 被请求大量数据，触发限流
- `array_slice($items, 0, -1)` 意外截断最后一项
- 数据库查询无 `LIMIT` 导致全表扫描

### 7.4 getQueriedContext 歧义解析

当用户输入参数集合可能匹配多个上下文时，`getQueriedContext()` 有明确的歧义处理策略。

**歧义产生场景**：
```php
const PARAMETERS = [
    'By keyword'  => ['q' => ['name' => 'Query', 'required' => true]],
    'By username' => ['u' => ['name' => 'User', 'required' => true]],
];
```
如果用户同时提交 `q=bird&u=alice`，两个上下文的必填参数都有值，`array_sum($queriedContexts) = 2`。

**歧义处理流程** (`lib/ParameterValidator.php:103-120`)：
```php
switch (array_sum($queriedContexts)) {
    case 0:      // 无匹配 → 尝试找空参数 context
    case 1:      // 唯一匹配 → 返回 context 名
    default:     // 歧义匹配 → return false
}
```

**歧义产生后果**：`BridgeAbstract::setInput()` 检测到返回 `false` 后，会在 `lib/BridgeAbstract.php:170-176` 抛出：
```php
if ($this->queriedContext === false) {
    throwClientException('Mixed context parameters');
}
```

**消歧手段**：表单中可通过隐藏字段 `context` 显式指定目标上下文：
```html
<input type="hidden" name="context" value="By keyword">
```
当 `input['context']` 存在时，`getQueriedContext()` (`lib/ParameterValidator.php:106-108`) 会直接返回该值，**跳过自动推断**，从根源避免歧义。

### 7.5 global 合并覆盖语义

`global` 上下文与普通 context 的合并在**渲染侧**和**运行侧**都使用了「后发覆盖」语义。

**渲染侧合并** (`FrontpageAction.php:117-119`)：
```php
$contextParameters = array_merge($contextParameters, $parameters['global']);
```
`array_merge()` 中 `$parameters['global']` 作为第二个参数，其同名参数会**覆盖** `$contextParameters` 中的定义。

**运行侧合并** (`BridgeAbstract.php:235-253`)：
```php
foreach ($contextNames as $contextName) {
    foreach ($parameters[$contextName] as $name => $parameter) {
        // 回填默认值...
        if (isset($this->inputs[$queriedContext][$name])) {
            // 已存在，跳过（保留用户提交值）
        } else {
            $this->inputs[$queriedContext][$name]['value'] = $defaultValue;
        }
    }
}
```
`$contextNames` 的遍历顺序是 `[$queriedContext, 'global']`，所以 global 的默认值**不会覆盖**用户已提交的值，但会**覆盖** context 中已存在的同名参数的默认值。

**覆盖顺序优先级**（从高到低）：
1. 用户通过 GET/POST 提交的值
2. global 上下文中的 `defaultValue`
3. 普通 context 中的 `defaultValue`
4. 系统兜底（`false` / `values[0]` / `null`）

**潜在陷阱**：如果 global 和 context 定义了同名参数但类型不同，渲染侧使用 global 的类型定义，运行侧校验时也按 global 的类型校验，context 中的类型定义被完全忽略。

### 7.6 ParameterValidator 自定义类型扩展

`ParameterValidator` 目前支持 `text`、`number`、`checkbox`、`list` 四种类型。扩展新类型需要修改源码，**无插件化扩展机制**。

**当前类型分发逻辑** (`lib/ParameterValidator.php:24-42`)：
```php
switch ($contextParameters[$name]['type']) {
    case 'number':
        $input[$name] = $this->validateNumberValue($value);
        break;
    case 'checkbox':
        $input[$name] = $this->validateCheckboxValue($value);
        break;
    case 'list':
        $input[$name] = $this->validateListValue($value, $contextParameters[$name]['values']);
        break;
    default:
    case 'text':
        $input[$name] = $this->validateTextValue($value, $pattern ?? null);
        break;
}
```

**扩展新类型的三步法**：

1. **新增 case 分支**：在 switch 中添加新类型，例如 `case 'email':`

2. **新增校验方法**：添加 `private function validateEmailValue($value)`，返回 `null` 表示校验失败

3. **补充默认值回填逻辑**：在 `BridgeAbstract::setInputWithContext()` 中添加新类型的默认值处理

**扩展示例（email 类型）**：
```php
// ParameterValidator.php switch 中新增
case 'email':
    $input[$name] = $this->validateEmailValue($value);
    break;

// 新增方法
private function validateEmailValue($value)
{
    $filtered = filter_var($value, FILTER_VALIDATE_EMAIL);
    return $filtered === false ? null : $filtered;
}

// BridgeAbstract.php setInputWithContext() 中补充
case 'email':
    if (isset($parameter['defaultValue'])) {
        $value = $parameter['defaultValue'];
    }
    break;
```

**扩展限制**：
- `default: case 'text':` 是双重入口，未知类型会被当作 text 处理
- 新类型的参数元数据（如 `values` 用于 list）需自行在 PARAMETERS 中定义并在校验方法中读取
- 前端 HTML 渲染（`lib/html.php`）也需同步扩展，否则新类型会被渲染为普通 text input

### 7.7 缓存键碰撞防护

rss-bridge 通过**两级哈希**防止缓存键碰撞：

**第一级：应用层键构造**
```php
// HTTP 响应缓存
$cacheKey = 'http_' . json_encode($request->toArray());

// Bridge 内部缓存
$cacheKey = $this->getShortName() . '_' . $key;
```
应用层键是人类可读的，但可能很长（包含完整的 JSON 序列化）。

**第二级：存储层键哈希**
所有缓存实现（`SQLiteCache`、`MemcachedCache`、`FileCache`）都包含 `createCacheKey()` 方法：
```php
// SQLiteCache.php:131-134
private function createCacheKey($key)
{
    return hash('sha1', $key, true);  // 返回 20 字节二进制哈希
}
```

**SHA-1 哈希的碰撞防护能力**：
- 输出空间：160 位 → 约 1.46×10⁴⁸ 种可能
- 生日悖论下，产生碰撞需要约 10²⁴ 个缓存条目，实际系统中不可能发生
- 注意：SHA-1 已被密码学攻破，但此处用于缓存键去重而非安全签名，仍是足够的

**额外的唯一性约束**：
- `SQLiteCache.php:39`: `'key' BLOB PRIMARY KEY` — 数据库层强制执行唯一性
- `Memcached` / `Redis` 等 KV 存储天然按键名覆盖，不保证多客户端写入时的一致性

**碰撞场景**：理论上，如果两个不同的 `$_GET` 数组序列化后产生相同的 SHA-1 哈希，会导致「错误的缓存命中」。但这种概率可以忽略不计。

### 7.8 ShortName_key 的 TTL 串扰

**缓存键格式** (`lib/BridgeAbstract.php:320-332`)：
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

**getShortName() 的行为** (`lib/BridgeAbstract.php:335-338`)：
```php
public function getShortName(): string
{
    return (new \ReflectionClass($this))->getShortName();
}
```
返回的是**实际实例化类**的短名，而非定义该方法的类名。这意味着子类不会与父类共享缓存键。

**TTL 串扰的产生场景**：

1. **同一 Bridge 内的覆盖**：`SQLiteCache::set()` 使用 `INSERT OR REPLACE` (`SQLiteCache.php:88`)，如果同一 Bridge 对相同 `$key` 多次调用 `saveCacheValue()` 但传入不同 `$ttl`，**后一次的 TTL 会覆盖前一次**。

   ```php
   // 请求 A：缓存 1 小时
   $this->saveCacheValue('api_token', $token, 3600);

   // 请求 B（1 分钟后）：缓存 24 小时（覆盖！）
   $this->saveCacheValue('api_token', $token, 86400);
   ```
   结果：缓存实际存活 24 小时，而非预期的 1 小时。

2. **继承体系中的隔离**：假设有继承链 `BaseBridge → ChildBridge`：
   ```php
   class BaseBridge extends BridgeAbstract {
       protected function cacheSomething() {
           $this->saveCacheValue('shared_data', $data, 3600);
       }
   }
   class ChildBridge extends BaseBridge {}
   ```
   - `BaseBridge` 产生键：`BaseBridge_shared_data`
   - `ChildBridge` 产生键：`ChildBridge_shared_data`
   - **两者完全隔离**，不会串扰。`getShortName()` 返回实际类名，这是关键的隔离机制。

3. **HTTP 缓存的 TTL 优先级**：
   ```php
   // DisplayAction.php:57-63
   $ttl = $request->get('_cache_timeout');
   if (Configuration::getConfig('cache', 'custom_timeout') && isset($ttl)) {
       $ttl = (int) $ttl;
   } else {
       $ttl = $bridge->getCacheTimeout();  // 来自 CACHE_TIMEOUT 常量
   }
   ```
   用户可通过 `_cache_timeout` 参数自定义 TTL，这会覆盖 Bridge 定义的 `CACHE_TIMEOUT`。如果同一请求被不同用户用不同 `_cache_timeout` 访问，**后写入的 TTL 会覆盖先写入的**。

**防范措施**：
- Bridge 内部对同一 key 的 `saveCacheValue()` 调用应使用一致的 TTL
- 如需不同 TTL，使用不同的 key 后缀（如 `api_token_short`、`api_token_long`）
- 生产环境可关闭 `custom_timeout` 配置，避免用户随意设置 TTL
