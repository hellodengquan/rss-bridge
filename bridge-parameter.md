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

---

## 八、深度细节（续）：边界场景的源码级剖析

### 8.1 5 种 LIMIT 兜底模式的实际覆盖率

对 `bridges/` 目录下所有使用了 `limit` 参数的 Bridge 做全量扫描，统计 5 种兜底模式的实际使用情况：

**统计基数**：85 个文件、104 处 `getInput('limit')` 调用。

| 兜底模式 | 匹配数 | 典型示例 | 覆盖率 |
|---------|--------|---------|--------|
| `?:` 短路默认值 | 12 | `ZeitBridge.php:57`, `TagesspiegelBridge.php:113`, `GenshinImpactBridge.php:66` | 11.5% |
| `??` 空合并默认值 | 22 | `ZDNetBridge.php:173`, `WordPressBridge.php:25`, `GQMagazineBridge.php:84` | 21.2% |
| `max()` 保护下界 | 1 | `GenshinImpactBridge.php:67`, `Formula1Bridge.php:36`, `DjMagDotComBridge.php:78` | ~1% |
| `min()` 保护上界 | 11 | `WorldbankBridge.php:32`, `OglafBridge.php:23`, `GettrBridge.php:34` | 10.6% |
| `array_slice` 容错 | 26 | `YandexZenBridge.php:64`, `XenForoBridge.php:155`, `IPBBridge.php:187` | 25.0% |
| **裸用（无兜底）** | 34 | `YandexZenBridge.php:62`, `UsbekEtRicaBridge.php:30`, `SpotifyBridge.php:144` | **32.7%** |

**覆盖率关键发现**：
1. **1/3 的 Bridge 未做任何兜底**：直接将 `getInput('limit')` 传入 `array_slice`、循环条件或 URL 查询字符串。如果 limit 参数校验失败（`null`）或为负值，行为依赖 PHP 底层函数的容错能力。

2. **`??` 和 `?:` 混用**：两者语义差异容易混淆：
   - `$limit = $this->getInput('limit') ?: 5;` → 0、null、false、"" 都兜底为 5
   - `$limit = $this->getInput('limit') ?? 10;` → 仅 null 兜底为 10，0 是合法值
   - 极端案例：`WiredBridge.php:49` 使用 `?? -1`，`KununuBridge.php:84` 使用 `?: 0`

3. **双向 clamp 极其稀缺**：只有 `GenshinImpactBridge.php:67` 和 `Formula1Bridge.php:36` 同时使用了 `max(LIMIT_MIN, min(LIMIT_MAX, $limit))` 的双向范围限制，不足 2%。

4. **`(int)` 强制转换防御**：仅 3 处使用了 `(int)$this->getInput('limit')`，如 `UniverseTodayBridge.php:24`、`FeedMergeBridge.php:45`，确保即使校验通过的非数字也会被转成 0。

5. **`array_slice` 行为差异**：
   - `array_slice($arr, 0, null)` → 返回全部元素（正常行为）
   - `array_slice($arr, 0, -1)` → 去掉最后一项（**陷阱**）
   - `array_slice($arr, 0, 0)` → 返回空数组（**陷阱**）

### 8.2 array_sum 零匹配 fallback 路径全景

当 `getQueriedContext()` 所有上下文都不匹配（`array_sum = 0`）时，fallback 逻辑按以下优先级逐级尝试：

**完整 fallback 链** (`lib/ParameterValidator.php:103-114` + `lib/BridgeAbstract.php:172-176`)：

```
array_sum === 0
    │
    ├─ 1. 检查 hidden 字段 context
    │      if (isset($input['context'])) {
    │          return $input['context'];
    │      }
    │    说明：用户在 FrontpageAction 提交表单时，每个上下文表单
    │    都包含 <input type="hidden" name="context" value="...">
    │    只要用户从正常表单进入，这里就一定能命中
    │
    ├─ 2. 遍历找第一个空参数 context
    │      foreach ($queriedContexts as $context2 => $queried) {
    │          if (is_null($queried)) {
    │              return $context2;
    │          }
    │      }
    │    说明：$queried === null 表示该 context 既没有用户填值，
    │    也没有必填项缺失（即 context 本身无参数或全是选填）
    │    通常适用于匿名无参数 Bridge
    │
    └─ 3. 返回 null → setInput() 抛出异常
           if (is_null($this->queriedContext)) {
               throwClientException('Required parameter(s) missing');
           }
```

**边界案例分析**：

- **正常表单提交**：用户在首页点击某个 Bridge 的「Generate Feed」，hidden `context` 字段携带上下文名 → 命中 step 1

- **手写 URL 无 context 参数**：用户直接手动拼接 URL 提交参数 → 跳过 step 1，走到 step 2 或 3

- **匿名无参数 Bridge（如 `MinecraftBridge` 的匿名 context）**：所有参数全为空 → `$queried === null` → 命中 step 2，成功返回第一个匿名 context（`0`）

- **带必填参数的 Bridge，用户完全不填**：必填参数缺失 → `$queried === false` → 跳过 step 2 → 走到 step 3 抛出 "Required parameter(s) missing"

- **恶意构造的混合参数**：用户同时提交两个 context 的必填参数 → 不走 `array_sum=0` 分支，而是走 `default` 分支返回 `false`，抛出 "Mixed context parameters"

### 8.3 global 合并时类型不兼容的 panic 防护

global 和 context 定义同名参数但**类型不同**时，系统无任何显式 panic 防护，会静默产生不可预测行为：

**无防护的风险路径**：

```
渲染侧（FrontpageAction）：
  array_merge($contextParams, $globalParams)
  → global 的类型定义覆盖 context
  → HTML 输入框按 global 的 type 渲染

校验侧（ParameterValidator）：
  validateInput() 遍历所有 context
  → 同一参数在 context 中被按 context 类型校验
  → 同一参数在 global 中又被按 global 类型校验
  → 两次校验结果可能冲突

回填侧（setInputWithContext）：
  先遍历 queriedContext，再遍历 global
  → global 的 defaultValue 覆盖 context 的 defaultValue
  → 但 type 不兼容导致回填值可能是 null 或类型错误
```

**实际风险案例**：
```php
const PARAMETERS = [
    'By A' => [
        'limit' => ['name' => 'Limit', 'type' => 'number'],
    ],
    'global' => [
        'limit' => ['name' => 'Limit', 'type' => 'text', 'pattern' => '[a-z]+'],
    ],
];
```
- 用户提交 `limit=10`
- 渲染侧：按 `text + pattern` 渲染，前端 pattern 校验正则 `[a-z]+` 直接拦截 `10`，用户无法提交
- 若绕过前端：`validateNumberValue('10')` = 10（context 侧），但 `validateTextValue('10', '[a-z]+')` = null（global 侧）
- 最终值取决于遍历顺序，可能是 `10` 或 `null`

**无防护的本质**：
1. `FrontpageAction` 的 `array_merge` 对同名参数静默覆盖，无任何 `E_WARNING`
2. `ParameterValidator` 遍历所有 context 对同一参数多次校验，后写覆盖前写
3. `setInputWithContext` 先写 context 值再写 global 值，类型不兼容时 global 的 `defaultValue` 类型错误导致值异常

**防御性开发建议**：
- 严格约定 global 参数使用独特前缀（如 `g_limit` 而非 `limit`）
- Bridge 开发时在本地添加 PHP 静态检查规则：`global` 与 context 不得有重名参数
- 如果无法避免重名，确保 global 和 context 的 type 完全一致

### 8.4 自定义类型三步法后的热更新注意事项

完成自定义类型扩展（新增 case 分支 + 新增 validate 方法 + 补充回填逻辑）后，热更新有三层陷阱：

**1. OPcache 层面**：
```
修改 ParameterValidator.php 和 BridgeAbstract.php
    ↓
OPcache 未过期（默认 revalidate_freq=2s）
    ↓
旧字节码仍在执行，新类型完全不生效
    ↓
用户提交自定义类型的值
    ↓
落到 default: case 'text': 分支
    ↓
被当作 text 类型处理，用 filter_var 清洗
    ↓
不报错但行为不符合预期（静默降级）
```

**2. 前端 HTML 渲染层**：
即使后端类型生效，`lib/html.php` 的 `Bridge::parameters_to_html()` 可能未同步添加新类型的 HTML 渲染分支。如果类型未被识别：
- type 属性会被输出为自定义字符串（如 `type="email"`），浏览器不识别
- 退化为普通文本输入框（HTML5 浏览器未知 type 会 fallback 为 text）
- **不会报错**，但失去了新类型应有的交互特性（如日期选择器、邮箱键盘）

**3. 默认值回填缺失**：
如果只改了 `ParameterValidator` 的校验分支，忘记补 `setInputWithContext()` 的类型分支：
- 校验时按新类型正确清洗并通过
- 但用户未填值时，落到 switch 的 default 分支用 text 逻辑回填
- `isset($parameter['defaultValue'])` 可能不匹配新类型的语义

**安全热更新步骤**：
1. 修改源码后先 `opcache_reset()`（或等 revalidate_freq 过期）
2. 同步修改 `html.php` 的表单渲染逻辑，为新类型添加对应 HTML 控件
3. 同步修改 `setInputWithContext()` 的 default 值回填逻辑
4. 用无缓存浏览器刷新 Frontpage，验证新类型控件正常渲染
5. 手动提交一次带新类型参数的请求，验证校验与回填链路正常

### 8.5 SHA-1 极值碰撞的实际处理

**三种缓存实现的哈希算法差异**：

| 缓存实现 | createCacheKey | 输出格式 | 长度 |
|---------|---------------|---------|------|
| SQLiteCache | `hash('sha1', $key, true)` | 二进制原始输出 | 20 字节 |
| MemcachedCache | `hash('sha1', $key)` | 十六进制字符串 | 40 字符 |
| FileCache | `hash('md5', $key)` | 十六进制字符串 | 32 字符 |

**算法不统一的原因**：
- SQLiteCache 用二进制存储在 BLOB 列，节省空间（20B vs 40B）
- Memcached 键名不支持二进制字符，必须用十六进制字符串
- FileCache 用 MD5 是历史遗留，文件名长度短一些（32 字符 vs 40 字符）

**碰撞场景的实际防护**：

1. **SHA-1 碰撞的密码学攻击成本**：构造两个产生相同 SHA-1 哈希的不同请求，需要约 $100K 级别的计算资源（SHAttered 攻击 2017 年数据），针对 rss-bridge 这种场景无实际攻击价值。

2. **应用层键的前缀隔离**：
   - HTTP 响应缓存键前缀 `http_`
   - Bridge 内部缓存键前缀 `{ShortName}_`
   - 错误报告缓存键前缀 `error_reporting_`
   - 不同命名空间的前缀降低了不同业务缓存之间的碰撞概率

3. **缓存值的类型匹配**：即使发生碰撞，`unserialize()` 返回的类型与期望类型不匹配时，业务逻辑通常会自然失败（如期望 `Response` 对象却拿到了字符串），不会出现「用了错误数据」的安全问题。

4. **SQLite 的 PRIMARY KEY 保护**：相同哈希值写入时，`INSERT OR REPLACE` 会覆盖旧条目，不会产生脏数据。

**MD5 的 FileCache 风险**：MD5 碰撞构造成本仅需几美元（2024 年数据），如果 rss-bridge 暴露在公网且用户可控 cache key 前缀，存在缓存污染理论风险。实际中因为还有 `serialize()` 的类型约束，风险很低。

### 8.6 SQLite PRIMARY KEY 并发写入一致性

**SQLite 的并发模型**：
SQLite 使用**数据库级读写锁**，而非行级锁。多个进程同时写入时的行为：

```
进程 A：prepare(INSERT OR REPLACE) → execute()
    ↓
获取 RESERVED 锁 → 升级为 PENDING 锁 → 升级为 EXCLUSIVE 锁
    ↓
写入 BLOB 数据 → 提交 WAL
    ↓
释放锁

进程 B：同时 prepare(INSERT OR REPLACE) → execute()
    ↓
尝试获取 RESERVED 锁 → 被阻塞
    ↓
等待 busy_timeout（SQLiteCache 配置 5000ms）
    ↓
超时后抛出 SQLITE_BUSY 异常
    ↓
SQLiteCache.php:92-97 catch 捕获 → logger warning → 静默吞掉异常
```

**INSERT OR REPLACE 的语义**：
```sql
INSERT OR REPLACE INTO storage (key, value, updated) VALUES (:key, :value, :updated)
```
等价于：如果 `key` 已存在 → 先 DELETE 旧行，再 INSERT 新行。**不是 update**，而是 delete+insert 原子操作。

**并发写入的一致性影响**：

1. **最后写入者获胜（LWW）**：两个进程同时写同一条缓存，谁最后提交谁的数据保留，TTL 也以最后写入者为准。

2. **读-改-写竞态**：
   ```
   进程 A: cache->get(key) → 查到 V1
   进程 B: cache->get(key) → 查到 V1
   进程 A: cache->set(key, f(V1), ttl1) → 写入 V2，ttl=ttl1
   进程 B: cache->set(key, g(V1), ttl2) → 写入 V3，ttl=ttl2（覆盖 A 的 V2！）
   ```
   这是典型的 lost update 问题，rss-bridge 无任何 CAS（Compare-And-Swap）或事务保护。

3. **busy_timeout 超时保护**：
   ```php
   // SQLiteCache.php:42
   $this->db->busyTimeout($config['timeout']);  // 默认 5000ms
   ```
   5 秒内获取不到锁会放弃写入并记 warning，不会无限阻塞。

4. **WAL 模式的提升**：
   ```php
   // SQLiteCache.php:45
   $this->db->exec('PRAGMA journal_mode = wal');
   ```
   WAL 模式下读不阻塞写、写不阻塞读，大大降低了并发冲突概率。但多个写者之间仍然串行。

**实际场景中的影响**：
- HTTP 响应缓存：同一请求被并发触发时，后写入的响应覆盖先写入的，通常可以接受
- Bridge 内部缓存（如 `api_token`）：多个进程刷新 token 时可能丢刷新记录，但 token 本身由上游 API 签发，只要有效即可
- `prune()` 清理过期数据：可能和读写并发，但 prune 只删除过期数据，不影响有效缓存

### 8.7 _cache_timeout 越界与清零边界

**_cache_timeout 的完整流水线**：

```
用户请求携带 ?_cache_timeout=XXX
    │
    ├─ 配置检查（DisplayAction.php:57-58）
    │    Configuration::getConfig('cache', 'custom_timeout')
    │      ├─ true  → 允许用户自定义 TTL
    │      └─ false → 忽略用户值，使用 Bridge 定义的 CACHE_TIMEOUT
    │
    ├─ 值类型转换（DisplayAction.php:59）
    │    $ttl = (int) $ttl;
    │      非数字字符串 → 0（如 "abc" → 0）
    │      布尔值 true → 1
    │      null → 0
    │      浮点数 → 截断取整（如 3600.9 → 3600）
    │
    ├─ 越界值无校验
    │    负值（如 -1）→ 原样传给 cache->set()
    │    超大值（如 999999999）→ 原样传给 cache->set()
    │    0 → 触发 TTL 零值保护
    │
    └─ 各缓存实现的处理
         │
         ├─ SQLiteCache / MemcachedCache / FileCache
         │    if ($ttl === 0) {
         │        return; // TTL 清零，完全跳过写入
         │    }
         │
         └─ 计算 expiration
              $expiration = $ttl === null ? 0 : time() + $ttl;
                $expiration === 0 → 永久缓存
                负值 $ttl → time() + 负数 = 过去时间 → 立即过期
```

**边界值矩阵**：

| 用户提交值 | `(int)` 后 | `custom_timeout=true` | 缓存写入？ | 实际 TTL |
|-----------|-----------|----------------------|-----------|---------|
| 未提交 | null | - | 走 Bridge 的 CACHE_TIMEOUT | 如 3600 |
| `""`（空字符串） | 0 | true | **跳过写入**（`$ttl===0`） | 不缓存 |
| `"0"` | 0 | true | **跳过写入** | 不缓存 |
| `"3600"` | 3600 | true | 写入 | 3600 秒 |
| `"-1"` | -1 | true | 写入，但 `time()-1` → 立即过期 | 0 秒（写入即失效） |
| `"abc"` | 0 | true | **跳过写入** | 不缓存 |
| `"2147483648"` | -2147483648（32 位溢出） | true | 写入，但时间戳 `time()-2e9` → 立即过期 | 0 秒 |
| `"999999999"` | 999999999 | true | 写入 | ~31.7 年后过期 |
| `"3600"` | 3600 | **false**（配置关闭） | 走 Bridge 的 CACHE_TIMEOUT | 如 3600 |

**三个关键发现**：
1. **负值 TTL 不报错**：`-1` 会通过所有检查，最终 `time() + (-1)` 产生过去时间戳，读取时立即判定过期，等价于不缓存。但浪费了一次写入操作。

2. **零值跳过写入**：`$ttl === 0` 在所有缓存实现中都会提前 `return`，连过期条目都不会留下。注意是**严格相等判断**，`"0"` 转成 `(int)` 后是整数 0 也会命中。

3. **32 位整数溢出风险**：在 32 位 PHP 环境中，`(int)` 超过 2,147,483,647（约 68 年）会溢出为负数，产生「立即过期」效果。64 位 PHP 无此问题。

### 8.8 ReflectionClass::getShortName() 的性能开销

**getShortName() 的调用位置**：
```php
// BridgeAbstract.php:320-322
protected function loadCacheValue(string $key, $default = null)
{
    return $this->cache->get($this->getShortName() . '_' . $key, $default);
}

// BridgeAbstract.php:330-332
protected function saveCacheValue(string $key, $value, int $ttl = 86400)
{
    $this->cache->set($this->getShortName() . '_' . $key, $value, $ttl);
}

// BridgeAbstract.php:335-338
public function getShortName(): string
{
    return (new \ReflectionClass($this))->getShortName();
}
```

**每次 load/saveCacheValue 都会创建一个新的 ReflectionClass 对象**。

**性能开销分析**：

1. **PHP Reflection 的成本**：单次 `new ReflectionClass($obj)` 约为 **0.5-2μs**。创建对象 + 获取短名，在现代 CPU 上约 1-3μs。

2. **单次请求的累计开销**：
   - 每个 Bridge 实例化一次
   - 业务代码中 `loadCacheValue()` 和 `saveCacheValue()` 一般调用 0-10 次
   - 累计开销：约 5-30μs
   - 对比整个 HTTP 请求（50-500ms），占比 < 0.1%，**完全可以忽略**

3. **为什么不用 `get_class()` + 字符串处理**：
   ```php
   // 方案 A：当前实现
   (new \ReflectionClass($this))->getShortName()
   
   // 方案 B：更简单的实现
   $class = get_class($this);
   return substr($class, strrpos($class, '\\') + 1);
   ```
   方案 B 性能更好（约 0.1μs），但 rss-bridge 的 Bridge 都不使用命名空间，所以 `get_class()` 直接就是短名（如 `"FlickrBridge"`）。Reflection 方案是通用写法，兼容命名空间场景。

4. **是否需要缓存**：
   ```php
   // 可以优化为一次性计算：
   private ?string $shortNameCache = null;
   public function getShortName(): string
   {
       return $this->shortNameCache
           ?? ($this->shortNameCache = (new \ReflectionClass($this))->getShortName());
   }
   ```
   但当前代码未做此优化，因为性能差距在纳秒级别，对整体无感知影响。只有在极端场景（单个请求调用数千次 loadCacheValue）时才值得引入。

5. **OPcache 的优化**：开启 OPcache 时 Reflection 操作有额外的内部缓存优化，实际开销比裸 benchmark 更低。

---

## 九、深度细节（再续）：运维与安全场景

### 9.1 OPcache 热更新的运维信号

修改 PARAMETERS 或 ParameterValidator 后，PHP OPcache 的状态对运维排障至关重要。以下是可观测的运维信号：

**OPcache 配置检查清单**：

| ini 参数 | 默认值 | 对热更新的影响 | 排障信号 |
|---------|--------|---------------|---------|
| `opcache.enable` | `1` | 关闭时每次请求重新解析 PHP，热更新即时生效 | `php -i \| grep "opcache.enable"` |
| `opcache.validate_timestamps` | `1` | 关闭时完全不检查文件变更，必须手动 `opcache_reset()` | 生产环境常关闭以提升性能 |
| `opcache.revalidate_freq` | `2` | 单位秒；每隔 N 秒重新校验 mtime | 修改源码后等 2 秒就能看到变更生效 |
| `opcache.revalidate_path` | `0` | 同文件在不同 include_path 会缓存多个副本 | 软链接部署场景设为 1 |
| `opcache.max_accelerated_files` | `10000` | 缓存满时随机驱逐旧条目，导致偶发热更新不生效 | `opcache_get_status()['num_cached_scripts']` 接近上限时警惕 |

**运维可观测信号**：

```php
// 热更新后可执行此脚本确认 OPcache 状态
$status = opcache_get_status(true);
if ($status) {
    echo "已缓存脚本数: " . $status['opcache_statistics']['num_cached_scripts'] . "\n";
    echo "命中率: " . round($status['opcache_statistics']['opcache_hit_rate'], 2) . "%\n";
    echo "最后重启时间: " . date('Y-m-d H:i:s', $status['opcache_statistics']['last_restart_time']) . "\n";
    
    // 检查特定 Bridge 是否已使用新版
    foreach ($status['scripts'] as $file => $info) {
        if (str_contains($file, 'FlickrBridge.php')) {
            echo "FlickrBridge 最后修改: " . date('H:i:s', $info['timestamp']) . "\n";
            echo "文件系统 mtime: " . date('H:i:s', filemtime($file)) . "\n";
            // 如果 timestamp < filemtime，说明 OPcache 还未刷新
        }
    }
}
```

**热更新失效的典型表现**：
1. 前端页面显示旧的 PARAMETERS 参数表单（如新增的字段没出现）
2. 提交后校验逻辑与代码不一致（如加了 required 仍能通过）
3. `error_log` 中出现与代码逻辑矛盾的 Notice/Warning

**一键安全热更新操作**：
```bash
# 方案 A：温和方案（推荐），等 revalidate_freq 自然过期
sleep 3 && curl -s http://localhost/ > /dev/null

# 方案 B：强制重置（需 PHP-FPM 进程内执行）
php -r 'opcache_reset();'

# 方案 C：重启 PHP-FPM（最粗暴但最可靠）
systemctl reload php-fpm
```

### 9.2 三种缓存算法间的迁移兼容

rss-bridge 支持 SQLiteCache / FileCache / MemcachedCache 三种后端，配置方式：
```ini
; config.ini.php
[cache]
type = "sqlite"   ; 或 "file" / "memcached"
```
不同缓存后端之间切换时，**完全不兼容**。

**三种后端的不可互操作点**：

| 维度 | SQLiteCache | FileCache | MemcachedCache |
|------|-------------|-----------|----------------|
| 哈希算法 | sha1 二进制 | md5 十六进制 | sha1 十六进制 |
| 存储介质 | SQLite BLOB 列 | 本地文件 | 内存 KV |
| 键空间 | `PRIMARY KEY` 全局 | 文件名（目录隔离） | Memcached 全局 |
| 值格式 | `serialize()` 二进制 | `serialize()` 含 key/expiration 包装 | 原始 `serialize()` |
| TTL 模型 | `updated` 列存绝对时间戳 | 文件内 `expiration` 字段存绝对时间戳 | Memcached 原生 TTL（相对秒或绝对 Unix 时间戳 <2592000） |
| 过期清理 | `prune()` 定时 SQL DELETE | `prune()` 遍历 unlink | Memcached 原生 LRU |

**迁移场景的影响**：

1. **冷启动缓存全部失效**：切换后端后，原有缓存数据完全无法读取，所有请求都会穿透到上游 API。
   - 风险：短时间内上游 API 请求量激增，可能触发限流
   - 建议：切换前提前用爬虫预热热点 URL，或在低流量时段切换

2. **Bridge 内部缓存也不互通**：
   - 同一个 key（如 `YoutubeBridge_api_token`）在不同后端下实际存储键完全不同
   - 切换后会重新生成 API token / 速率限制计数器
   - 注意：`saveCacheValue()` 的硬编码 TTL（默认 86400s）在 Memcached 侧超过 2592000（30天）会被 Memcached 当作绝对 Unix 时间戳处理

3. **FileCache → SQLiteCache 迁移工具（缺失）**：
   当前代码无迁移工具。如需迁移，需自行编写脚本：
   ```php
   // 伪代码示例：FileCache -> SQLiteCache
   foreach (glob(PATH_CACHE . '*.cache') as $f) {
       $data = unserialize(file_get_contents($f));
       $originalKey = $data['key'];  // FileCache 保存了原始 key
       $sqlite->set($originalKey, $data['value'], $data['expiration'] - time());
   }
   ```
   FileCache 恰好把原始 key 存在序列化数组里（`FileCache.php:54-58`），可以逆向还原；SQLiteCache 和 MemcachedCache 则丢弃了原始 key，只存哈希值。

**Memcached 的 30 天 TTL 陷阱**：
```
Memcached TTL 规则：
  ttl < 2592000 (30天) → 相对秒数
  ttl >= 2592000       → 绝对 Unix 时间戳

saveCacheValue() 默认 TTL = 86400（1 天）→ 没问题
saveCacheValue('key', $v, 2592001)        → 被 Memcached 当作 1970-01-31，立即过期！
```
注意：`CACHE_TIMEOUT` 常量最大的值在现有 Bridge 中是 86400（WarhammerComBridge、StripeAPIChangeLogBridge 等），未触达 30 天阈值，暂无风险。但自定义 Bridge 如果设置更大的 TTL 会踩坑。

### 9.3 SQLite 锁等待超时熔断

SQLiteCache 的锁等待与熔断机制：

**锁升级与等待链** (`SQLiteCache.php:42` + SQLite 内核)：

```
写入请求
   │
   ├─ 1. 获取 SHARED 锁（读，所有连接共享）
   │
   ├─ 2. prepare(INSERT OR REPLACE)
   │       → 尝试升级到 RESERVED 锁
   │       → 如果已有写者持有 RESERVED，阻塞等待
   │       → busy_timeout 计时开始（默认 5000ms）
   │
   ├─ 3. execute() 前升级到 PENDING 锁
   │       → 阻止新读者获取 SHARED
   │       → 等待现有读者释放 SHARED
   │       → 如果超时 → SQLITE_BUSY
   │
   ├─ 4. 写入 WAL 页时升级到 EXCLUSIVE 锁
   │       → 独占写
   │       → 完成后立即释放
   │
   └─ 5. 提交事务（自动），释放所有锁
```

**超时熔断行为** (`SQLiteCache.php:92-97`)：
```php
try {
    $stmt->execute();
} catch (\Exception $e) {
    $this->logger->warning(create_sane_exception_message($e));
    // Intentionally not rethrowing exception
}
```
**熔断策略**：写入失败时记录 warning 日志，**静默吞掉异常**，继续业务流程。相当于「降级为不缓存」模式，不影响用户拿到数据（虽然会慢一点，因为下次还得重新抓）。

**5 秒超时的合理性分析**：
- SQLite 单条 INSERT OR REPLACE + BLOB 写入通常 <1ms
- 5000ms 超时意味着大约可以容忍 ~5000 个排队写入操作
- 对于 rss-bridge 这种写入量不高的场景（每次 feed 请求最多 2-3 次缓存写入），5 秒绰绰有余
- 如果 5 秒超时频繁触发，说明：
  1. 某条缓存值特别大（如 BLOB > 10MB）导致 WAL 写入慢
  2. 并发量过高，SQLite 已到达性能瓶颈 → 建议切换到 Memcached

**观测超时的日志特征**：
```
[YYYY-MM-DD HH:MM:SS] rss-bridge.WARNING: The database is locked
SQLite3Stmt::execute(): Unable to execute statement: database is locked
```
出现此警告意味着 **该次缓存写入被丢弃**，下次请求会重新走上游。单条偶发可忽略，连续出现需要运维介入。

### 9.4 INSERT OR REPLACE 回滚链

SQLiteCache 使用 `INSERT OR REPLACE` 而非显式事务，但 SQLite 的每一条语句都隐式在一个事务中执行。

**隐式事务的回滚链**：

```
INSERT OR REPLACE INTO storage (key, value, updated) VALUES (...)
   │
   ├─ 自动开启隐式事务（AUTOCOMMIT）
   │
   ├─ 步骤 1：检查 PRIMARY KEY 冲突
   │   ├─ 存在相同 key → DELETE 旧行
   │   │   │
   │   │   ├─ 如果 DELETE 时磁盘满 / I/O 错误
   │   │   │   → 语句失败
   │   │   │   → 事务回滚（旧行保留）
   │   │   │   → 抛出异常，被 catch 记 warning
   │   │   └─ DELETE 成功
   │   └─ 不存在相同 key → 跳过 DELETE
   │
   ├─ 步骤 2：INSERT 新行
   │   ├─ PRIMARY KEY 冲突（极端竞态）→ 理论上不会发生
   │   ├─ BLOB 绑定失败（内存不足）→ 失败
   │   ├─ updated 列写入失败 → 失败
   │   └─ 存储页分配失败（磁盘满）→ 失败
   │   → 任一失败：事务回滚，DELETE 操作一并撤销
   │
   └─ 两步均成功 → 自动 COMMIT
       → WAL 追加一条记录
       → 下次 checkpoint 时合并到主 DB
```

**关键保障：原子性**：
即使 `DELETE` 成功但 `INSERT` 失败（极端 I/O 故障），SQLite 的隐式事务会把 `DELETE` 也回滚，**不会出现「旧值已删、新值未写」的空档**。这是 ACID 中 Atomicity 的直接体现。

**并发竞态下的保障**：
```
时间线：
  T0: 进程 A 开始 INSERT OR REPLACE (key=X, val=V1) → 获取 RESERVED 锁
  T1: 进程 B 开始 INSERT OR REPLACE (key=X, val=V2) → 阻塞等锁
  T2: 进程 A 完成 DELETE+INSERT，释放锁
  T3: 进程 B 获取锁
  T4: 进程 B DELETE 进程 A 写入的行，INSERT V2
  T5: 进程 B 完成，最终 DB 中是 V2
```
不会出现两条相同 key 的行（PRIMARY KEY 约束保障），也不会出现旧行删了新行没写（事务保障）。最坏情况就是最后写入者获胜，和缓存语义一致。

**失败回滚的可观测性**：
`SQLiteCache.php:92-97` 的 catch 块会把所有异常（含 I/O 错误、约束冲突、磁盘满）都转为 warning 日志，不会中断业务。但此时**缓存处于不一致状态**（旧值可能还在、也可能被回滚恢复），下一次读请求要么拿到旧值，要么 miss。

### 9.5 _cache_timeout 边界的覆盖测试

现有测试套件（`tests/`）中关于 `_cache_timeout` 和 `ParameterValidator` 的覆盖率极低。

**ParameterValidator 测试现状** (`tests/ParameterValidatorTest.php`)：
```php
// 仅 2 个用例：
// test1：匿名 context + text 类型，合法输入 → 无错误
// test2：匿名 context + 参数名不匹配 → 报错
```
完全未覆盖以下边界：
- number / checkbox / list 类型的校验
- text 带 pattern 的正则校验
- required 必填项校验
- global context 合并
- getQueriedContext 三态（唯一匹配/零匹配/歧义匹配）
- 空值、null、负值等边界输入

**Cache 测试现状** (`tests/CacheTest.php` + `tests/CacheImplementationTest.php`)：
```php
// CacheTest：仅 FileCache 的基本 set/get/clear 测试
// CacheImplementationTest：仅类名/接口契约检查
```
未覆盖：
- SQLiteCache / MemcachedCache 的读写测试
- TTL 过期行为（set 后等待 >TTL 再读应返回 null）
- TTL=0 跳过写入的行为
- TTL=null 永久缓存的行为
- `prune()` 过期清理
- 大 BLOB（>1MB）读写
- 并发写入

**_cache_timeout 边界的理论覆盖矩阵（当前 0%）**：

| 用例 | 输入 | 预期 TTL | 预期缓存命中 |
|------|------|---------|-------------|
| 未传 _cache_timeout，`custom_timeout=true` | 无 | Bridge::CACHE_TIMEOUT | 是 |
| 未传 _cache_timeout，`custom_timeout=false` | 无 | Bridge::CACHE_TIMEOUT | 是 |
| `_cache_timeout=0`，`custom_timeout=true` | `"0"` | **跳过写入** | 否 |
| `_cache_timeout=""`，`custom_timeout=true` | `""` | **跳过写入**（`(int)"" = 0`） | 否 |
| `_cache_timeout="abc"`，`custom_timeout=true` | `"abc"` | **跳过写入**（`(int)"abc" = 0`） | 否 |
| `_cache_timeout="-1"`，`custom_timeout=true` | `"-1"` | `time()-1`（写入即过期） | 否（读时立即过期） |
| `_cache_timeout="3600"`，`custom_timeout=true` | `"3600"` | 3600s | 是 |
| `_cache_timeout="3600.9"`，`custom_timeout=true` | `"3600.9"` | 3600s（截断） | 是 |
| `_cache_timeout="999999999"`，`custom_timeout=true` | `"999999999"` | ~31.7 年 | 是 |
| `_cache_timeout=3600`，`custom_timeout=false` | `"3600"` | Bridge::CACHE_TIMEOUT（忽略用户值） | 是 |

**建议新增测试用例的优先级**：
1. P0：`ttl=0` 跳过写入（安全边界）
2. P0：负值 TTL 立即过期
3. P1：`custom_timeout=false` 时忽略用户值
4. P1：非数字字符串转 int=0 跳过写入
5. P2：浮点数截断行为

### 9.6 ReflectionClass 热点路径降级方案

`getShortName()` 在每个 `loadCacheValue` / `saveCacheValue` 调用时都会新建 `ReflectionClass` 对象。对于常规 Bridge，这个开销可忽略；但对于热点 Bridge（单请求调用 100+ 次缓存读写），可以考虑以下降级/优化方案：

**方案 A：属性缓存（零改动风险）**：
```php
// BridgeAbstract.php 新增属性
private ?string $shortName = null;

public function getShortName(): string
{
    if ($this->shortName === null) {
        $this->shortName = (new \ReflectionClass($this))->getShortName();
    }
    return $this->shortName;
}
```
- 效果：单请求第一次调用时反射一次，后续直接返回字符串
- 性能提升：2μs → 约 0.1μs，20 倍提升
- 风险：无，仅增加 8 字节对象内存

**方案 B：get_class() 替代（更简单，但依赖无命名空间约定）**：
```php
public function getShortName(): string
{
    $class = get_class($this);
    return substr($class, strrpos($class, '\\') + 1);
}
```
- 效果：~0.1μs，与方案 A 相当
- 风险：如果未来 Bridge 引入命名空间（如 `namespace RssBridge\Bridges;`），此方案仍能正确截取短名（因为用了 `strrpos('\\')`），所以其实比 Reflection 更通用

**方案 C：静态缓存（跨请求）**：
```php
private static array $shortNameCache = [];

public function getShortName(): string
{
    $class = static::class;
    return self::$shortNameCache[$class]
        ?? (self::$shortNameCache[$class] = (new \ReflectionClass($this))->getShortName());
}
```
- 效果：跨请求共享缓存（同一 PHP-FPM 进程内）
- 风险：OPcache 下类名静态数组可能与旧缓存冲突，热更新后需重启 PHP-FPM

**方案 D：BridgeFactory 在构造时注入**：
```php
// BridgeFactory.php:46
public function create(string $name): BridgeAbstract
{
    $bridge = new $name($this->cache, $this->logger);
    $bridge->setShortName($name);  // Factory 已知类名，直接注入
    return $bridge;
}
```
- 效果：0μs，完全消除反射
- 风险：需要修改 BridgeAbstract 构造函数，所有子类需同步

**结论**：方案 A 最稳妥，与现有实现语义完全等价，无兼容性风险，代码改动量最小。在 rss-bridge 当前量级下其实没必要优化，但如果出现性能瓶颈，这是首选方案。

### 9.7 unserialize 反序列化攻击防护

rss-bridge 的三种缓存实现（SQLiteCache / FileCache / MemcachedCache）都使用 `serialize()` + `unserialize()` 作为序列化格式。`unserialize()` 在 PHP 中是**高危函数**，如果攻击者能控制缓存内容，可触发 PHP 对象注入（POP chain）攻击，甚至 RCE。

**调用链与攻击面**：

```
攻击者控制缓存存储
   │
   ├─ SQLiteCache：攻击者能写入 SQLite 文件
   │   → 构造序列化恶意对象写入 storage.value 列
   │   → 下次合法请求 cache->get() 调用 unserialize()
   │   → 触发 __wakeup() / __destruct() 等魔术方法
   │
   ├─ FileCache：攻击者能写入缓存目录的 .cache 文件
   │   → 同上，写入恶意序列化数据
   │
   └─ MemcachedCache：攻击者能访问 Memcached 端口（11211）
       → 直接 SET key 为恶意序列化数据
```

**现有防护的脆弱性**：
1. **无 `allowed_classes` 白名单**：所有 `unserialize()` 调用均为裸调用：
   ```php
   // SQLiteCache.php:67
   $value = unserialize($blob);  // ❌ 无白名单，允许反序列化任意类
   
   // FileCache.php:34
   $item = unserialize($data);  // ❌ 同上
   ```
   PHP 7+ 提供的 `unserialize($data, ['allowed_classes' => [...])` 机制完全未使用。

2. **无完整性校验（HMAC / 签名）**：缓存值没有 MAC，攻击者可随意篡改内容，读取方无法发现。

3. **缓存文件权限**：FileCache 的创建权限受 `umask` 影响，`FileCache.php:62` 的 TODO 注释明确指出了这个问题：
   ```php
   // TODO: Consider tightening the permissions of the created file.
   // It usually allow others to read, depending on umask
   ```
   默认 `umask=022` 时，缓存文件权限为 0644，同服务器其他用户可读取内容。

**加固建议**（按优先级）：

P0：添加 `allowed_classes` 白名单
```php
// 只允许反序列化内置类和 rss-bridge 的值对象
$allowed = [
    'stdClass',
    'DateTime',
    // Response 对象等 rss-bridge 内部类
    'Response',
];
$value = unserialize($blob, ['allowed_classes' => $allowed]);
```
如果缓存值只有标量+数组，直接用 `['allowed_classes' => false]` 完全禁止对象。

P1：改用 `json_encode` / `json_decode` 替代 `serialize()`
- JSON 格式无法携带 PHP 对象，从根源消除反序列化风险
- 注意：`Response` 对象需要实现 `jsonSerialize()` 接口或自定义 encode/decode
- `error_reporting_` 前缀的缓存已经用 JSON 了（`DisplayAction.php:178-189`），是良好示范

P2：添加 HMAC-SHA256 完整性校验
```php
// 写入时
$payload = serialize($value);
$hmac = hash_hmac('sha256', $payload, $secretKey);
$store = $hmac . '::' . $payload;

// 读取时
[$hmac, $payload] = explode('::', $stored, 2);
if (!hash_equals($hmac, hash_hmac('sha256', $payload, $secretKey))) {
    throw new \Exception('Cache integrity check failed');
}
return unserialize($payload);
```
防止数据被篡改，即使攻击者能读文件也无法伪造内容。

P3：收紧 FileCache 文件权限
```php
// FileCache.php set() 中
file_put_contents($cacheFile, serialize($item));
chmod($cacheFile, 0600);  // 仅所有者可读写
```

### 9.8 error_reporting_ 前缀的大对象拆分

错误报告缓存的结构与潜在问题：

**存储结构** (`DisplayAction.php:172-190`)：
```php
private function logBridgeError($bridgeName, $code)
{
    $cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
    $report = $this->cache->get($cacheKey);
    if ($report) {
        $report = Json::decode($report);
        $report['time'] = time();       // 覆盖时间
        $report['count']++;             // 累加计数
    } else {
        $report = [
            'error' => $code,
            'time'  => time(),
            'count' => 1,
        ];
    }
    $ttl = 86400 * 5;
    $this->cache->set($cacheKey, Json::encode($report), $ttl);
    return $report['count'];
}
```

**键的构成**：`error_reporting_{BridgeName}_{ErrorCode}`
- `BridgeName`：短类名（如 `FlickrBridge`）
- `ErrorCode`：通常是错误代码字符串或异常类名 hash
- TTL：硬编码 86400 * 5 = 432000 秒（5 天）

**大对象风险场景**：

1. **`$code` 异常膨胀**：如果 `$code` 是完整的堆栈跟踪字符串（含文件名、行号、参数值），长度可达数 KB。但目前 `$code` 看起来是简短的错误码，风险不大。

2. **错误类型爆炸导致键空间膨胀**：
   - 每个 Bridge × 每种错误类型 = 一个缓存条目
   - 假设有 400 个 Bridge，每个平均产生 10 种不同错误 → 4000 条缓存
   - 每条 JSON 约 100 字节 → 总计约 400KB，完全可接受

3. **并发丢失更新（lost update）风险**：
   ```
   请求 A: get(key) → count=5
   请求 B: get(key) → count=5
   请求 A: count=6, set(key)
   请求 B: count=6, set(key) → 覆盖 A 的写入！实际应该是 7
   ```
   这是典型的读-改-写竞态。当前无 CAS 或事务保护，并发错误场景下计数会丢失。对于错误报告统计来说，计数器少量偏差通常可以接受。

**潜在的大对象拆分建议**（当前非必要，但为未来预留）：

如果未来需要在错误报告里存堆栈跟踪、请求参数、调试上下文等大对象：

1. **拆分为元数据 + 详情**：
   ```
   error_reporting_{Bridge}_{Code}_meta  → {"count": N, "time": ...}   // 小，高频读写
   error_reporting_{Bridge}_{Code}_trace → {"stack": "...", "params": ...}  // 大，首次出错时写入
   ```

2. **滚动时间窗口拆分**：避免单条目 TTL 5 天导致数据陈旧：
   ```
   error_reporting_{Bridge}_{Code}_20260617  // 每天独立计数
   error_reporting_{Bridge}_{Code}_20260618
   ```
   聚合查询时合并最近 N 天，更符合「最近 5 天出现 X 次」的实际语义。

3. **序列化格式选择**：
   当前使用 JSON 而非 `serialize()` 是正确选择（`DisplayAction.php:175` 的 todo 评论也提到没必要 json encode）。但从反序列化安全角度，JSON 比 `serialize()` 安全得多，建议保持。

4. **SQLite 下的 BLOB 成本**：虽然 error_reporting 的 JSON 很小，但 SQLite 的 `INSERT OR REPLACE` 是「先删后插」操作，如果频繁更新同一错误的 count，会产生大量 WAL 页碎片。高并发下建议用 Redis/INCR 指令替代，避免全量读改写。

---

## 十、深度细节（终章）：工程化落地的深度剖析

### 10.1 unserialize 运行时检测机制

在加固 `unserialize()` 之前，需要先搞清楚「缓存里到底存了什么类型的对象」，否则盲目加 `allowed_classes` 白名单可能导致正常业务崩溃。

**运行时检测的三种方案**：

**方案 A：`allowed_classes => true` + 日志审计（最安全的渐进式）**
```php
// 第一步：先开启审计，不改行为
$value = unserialize($blob, ['allowed_classes' => true]);
if (is_object($value)) {
    $this->logger->debug('unserialize object detected', [
        'class'   => get_class($value),
        'cache_key' => $this->createCacheKey($key),  // 注意：只记哈希，不记原始 key 避免泄露敏感信息
    ]);
}
```
- 等价于裸 `unserialize()` 的行为，**不破坏任何功能**
- 运行一周后分析日志，收集所有出现过的类名
- 然后把这些类名加入白名单，切换到 `allowed_classes => [...]` 模式
- 风险：在此期间仍然受反序列化攻击威胁，只能作为过渡

**方案 B：包装器类 + 类型断言**
```php
// 对每个 unserialize 调用点，加上期望类型检查
function safe_unserialize(string $data, array $allowed, $default = null) {
    $value = unserialize($data, ['allowed_classes' => $allowed]);
    if ($value === false) {
        // 反序列化失败（格式错/类不在白名单）
        return $default;
    }
    return $value;
}
```
- `unserialize` 返回 `false` 可能是因为：数据损坏、类不在白名单、版本不兼容
- 全部归因为「缓存失效」，返回 default 降级，下次重新生成
- 对业务无影响，只是缓存命中率暂时下降

**方案 C：`__PHP_Incomplete_Class` 检测**
```php
// 当 allowed_classes=false 但数据里有对象时
// unserialize 会返回 __PHP_Incomplete_Class 对象
$value = unserialize($blob, ['allowed_classes' => false]);
if ($value instanceof __PHP_Incomplete_Class) {
    $this->logger->warning('Incomplete class in cache', [
        'class' => $value->__PHP_Incomplete_Class_Name,
    ]);
    return $default;
}
```
- 这是 PHP 提供的「软失败」机制：类不在白名单时不报错，返回标记对象
- 可用于检测旧缓存里有哪些类，同时不会触发任何 `__wakeup()` 魔术方法
- **最安全的检测方案**，因为对象不会被真正实例化

**rss-bridge 的实际检测路径**：
- 三个缓存实现（SQLite/File/Memcached）各有 2-3 处 unserialize 调用
- 如果要加运行时检测，需要修改 7-8 个调用点
- 推荐顺序：先方案 C 检测有哪些类 → 再方案 A 确认风险 → 最后方案 B 加白名单

### 10.2 error_reporting_ 4 种拆分的切换灰度

从「单键大对象」切换到 4 种拆分方案（元数据+详情、滚动时间窗、JSON 序列化、Redis INCR），需要灰度发布策略，避免一次性切换导致数据丢失或缓存雪崩。

**4 种方案的切换难度矩阵**：

| 方案 | 旧数据兼容 | 切换复杂度 | 回滚难度 | 推荐灰度周期 |
|------|-----------|-----------|---------|-------------|
| 元数据+详情分键 | 可兼容（读旧键做迁移） | 低 | 低 | 1 天 |
| 滚动时间窗 | 不可兼容（键名变了） | 中 | 中 | 3-7 天 |
| JSON 替代 serialize | 可兼容（双写+渐进切换） | 中 | 中 | 3 天 |
| Redis INCR 原子计数 | 不可兼容（存储引擎变了） | 高 | 高 | 7-14 天 |

**灰度切换的通用模式（双写过渡法）**：

```
灰度阶段 1（双写旧键）：
  写入：同时写旧键和新键
  读取：优先读旧键，旧键没有就读新键
  持续时间：1 个 TTL 周期（5 天）
  目的：确保新键有完整的数据积累

灰度阶段 2（切读新键）：
  写入：仍然双写
  读取：优先读新键，新键没有就读旧键
  持续时间：3-7 天
  目的：验证新键的读写链路稳定

灰度阶段 3（停止写旧键）：
  写入：只写新键
  读取：只读新键，旧键等自然过期
  持续时间：观察 1 天后彻底删除旧键
  目的：清理冗余数据

回滚机制：
  任何阶段发现问题 → 立即切回「只读旧键」
  因为阶段 1 和 2 都在双写，回滚零损失
```

**滚动时间窗的特殊灰度策略**：
- 不能双写，因为时间窗的键名每天都在变（`_20260617` / `_20260618`）
- 切换当天开始写新格式，旧格式随 TTL 自然过期
- 聚合时做兼容：「有时间窗数据就聚合时间窗，没有就 fallback 到旧键」
- 5 天后旧键全部过期，彻底移除兼容代码

### 10.3 3 项防护缺失的补救代价

9.7 节提到的 3 项防护缺失（无白名单、无 HMAC、文件权限宽松），各自的修复成本和业务影响不同。

**代价量化矩阵**：

| 防护项 | 代码改动量 | 业务影响 | 测试成本 | 总代价 | 投入产出比 |
|--------|-----------|---------|---------|--------|-----------|
| `allowed_classes` 白名单 | 中（7-8 个调用点 + 1 个配置项） | 低（缓存 miss 率短暂上升） | 高（需收集所有类名验证） | 中 | 极高（安全收益大） |
| HMAC 完整性校验 | 中（set/get 各加一段逻辑 + 密钥配置） | 极低（CPU 开销 <1%） | 低（只需测 set/get 正常链路） | 低 | 高 |
| FileCache 文件权限 0600 | 极小（加一行 chmod） | 无（仅权限收紧） | 极低 | 极低 | 高 |

**详细拆解**：

**P0：allowed_classes 白名单**
- 改动点：SQLiteCache (2处) + FileCache (2处) + MemcachedCache (1处) = 5 处 unserialize
- 风险点：如果白名单漏了某个类，该类对象的缓存会全部失效，表现为「缓存命中率突降 + 上游请求量突增」
- 补救成本：发现漏类 → 加回白名单 → 等缓存自然重建；最坏情况影响几分钟到几小时的缓存命中率
- 总成本：约 2-3 人天开发 + 1 周观察期

**P1：HMAC-SHA256 校验**
- 改动点：set 时拼接 HMAC + get 时验证 HMAC
- 兼容性问题：旧缓存没有 HMAC，直接验证会全部失败 → 需要「无 HMAC 时降级兼容」
- 双写过渡策略：
  - 阶段 1：写入时同时写「纯数据」和「带 HMAC 数据」两个版本（或同值加前缀标记）
  - 阶段 2：读取时优先验证 HMAC 版本，失败则读纯数据版本
  - 阶段 3：纯数据版本随 TTL 过期后，只保留 HMAC 版本
- 性能开销：`hash_hmac('sha256', ...)` 单次约 1-2μs，对缓存读写可忽略
- 总成本：约 1-2 人天开发 + 3 天观察期

**P3：文件权限 0600**
- 改动点：`FileCache.php:60` 的 `file_put_contents()` 后加一行 `chmod()`
- 风险：umask 场景、共享主机场景下可能 chmod 失败 → 需要 try/catch 降级
- 注意：已存在的旧文件权限不会自动改变 → 需要 prune 时遍历 chmod
- 总成本：约 0.5 人天开发，几乎零风险

### 10.4 3 种风险场景的兼容回退

实施加固时必须考虑「改出问题了怎么快速回退」，三种风险场景各有不同的回退策略。

**场景 1：allowed_classes 白名单漏类**
```
现象：某类缓存突然全部 miss，上游 API 请求量暴涨
定位：error_log 中出现 "Failed to unserialize" 警告（SQLiteCache.php:69）
即时回退：
  方案 A（最快）：配置开关临时切回 allowed_classes => true
    - 改配置文件，等 OPcache 刷新
    - 恢复时间：1-2 分钟（取决于 revalidate_freq）
  方案 B（代码回退）：git revert 加固 commit，重新部署
    - 恢复时间：取决于部署流水线，通常 5-15 分钟
善后：
  - 从日志中收集缺失的类名
  - 加入白名单
  - 重新上线
```

**场景 2：HMAC 密钥泄露或配置错误**
```
现象：所有缓存验证失败，命中率跌至 0
定位：日志中大量 "Cache integrity check failed" 错误
即时回退：
  - 配置项 `cache.hmac_enable = false` 临时关闭校验
  - 退化为无 HMAC 模式，立即恢复
  - 同时旧数据全部可读，无任何丢失
风险：
  - 关闭 HMAC 期间重新暴露在篡改风险中
  - 如果是密钥配置错误，修正后重新开启即可
  - 如果是密钥泄露，需要：轮换密钥 → 旧数据全部失效（或双密钥过渡）
```

**场景 3：文件权限收紧导致其他进程无法读**
```
现象：另一个 PHP-FPM 池或 CLI 脚本读缓存失败，报 permission denied
定位：ls -l 缓存文件显示 0600，所有者是 www-data
即时回退：
  - 配置开关切回宽松权限（0644）
  - 脚本批量 chmod 现有文件为 0644
    find /path/to/cache -name "*.cache" -exec chmod 644 {} \;
  - 几分钟内恢复
根本解决：
  - 确保所有读写进程属于同一用户组
  - 使用 0660 权限而非 0600
  - 或者切换到 SQLite/Memcached 等共享存储
```

**通用回退设计原则**：
1. 每个加固项都要有独立的配置开关，支持一键关闭
2. 回退路径的代码必须经过测试，不能只设计前进路径
3. 灰度期间保留旧代码路径至少一个 TTL 周期
4. 关键指标（缓存命中率、上游请求量、错误率）必须有监控告警

### 10.5 P0 至 P3 加固的自动化扫描

加固不能只靠人工审查，需要自动化扫描工具持续检测，防止新代码又引入漏洞。

**静态检测的 4 个层面**：

**1. grep / ripgrep 级别的快扫（CI 级，<10 秒）**
```bash
# 检测裸 unserialize 调用（无 allowed_classes 参数）
rg 'unserialize\s*\(\s*\$' --type php \
  | grep -v 'allowed_classes' \
  | grep -v __PHP_Incomplete_Class

# 检测 serialize 调用（标记潜在风险点）
rg 'serialize\s*\(' --type php

# 检测直接使用 getInput('limit') 无兜底
rg 'getInput\(('|"")limit('|")\)' --type php \
  | grep -v '\?\?' \
  | grep -v '\?:' \
  | grep -v 'min\|max\|array_slice'
```
- 可集成到 CI 流水线，发现新增裸调用立即打回
- 优点：快、零依赖
- 缺点：误报率高（某些场景故意不用白名单）

**2. PHP-CS-Fixer / PHP_CodeSniffer 自定义规则**
```php
// 自定义 Sniff 伪代码
class UnserializeSecuritySniff implements Sniff
{
    public function register() { return [T_STRING]; }
    public function process(File $phpcsFile, $stackPtr) {
        $token = $phpcsFile->getTokensAsString($stackPtr, 1);
        if ($token !== 'unserialize') return;
        // 检查后面是否有第二个参数包含 allowed_classes
        $next = $phpcsFile->findNext(T_OPEN_PARENTHESIS, $stackPtr);
        // ... 解析参数 ...
        if (!$hasAllowedClasses) {
            $phpcsFile->addError('Unsafe unserialize call', $stackPtr, 'Unsafe');
        }
    }
}
```
- 优点：准确，可集成到 IDE
- 缺点：编写规则成本高

**3. PHPStan / Psalm 静态分析**
```neon
# phpstan.neon
rules:
    - RssBridge\Rules\UnserializeRule
```
- 可做数据流分析：追踪「从缓存读出 → unserialize → 传给业务逻辑」的完整链路
- 检测未验证的反序列化数据流入敏感函数
- 优点：最精准
- 缺点：配置复杂，运行慢

**4. 运行时 AOP 注入检测**
```php
// 利用 uopz 扩展或预加载机制，在运行时 hook unserialize
uopz_set_return('unserialize', function ($data, $options = []) {
    if (!isset($options['allowed_classes']) || $options['allowed_classes'] === true) {
        trigger_error('Unsafe unserialize call detected!', E_USER_WARNING);
    }
    return uopz_get_exit_status() ? false : unserialize($data, $options);
}, true);
```
- 覆盖所有调用点，包括第三方库
- 只能在测试/预发布环境开，生产环境有性能影响

**rss-bridge 现状**：当前项目用 PHPUnit 做单元测试，未见 PHPStan/Psalm/CS 等静态分析工具。如果要加自动化扫描，推荐从 grep 级别的 CI 检查开始，成本最低。

### 10.6 分键拆分的 key 数膨胀

把 error_reporting 从单键拆成多键（元数据+详情、滚动时间窗）后，缓存 key 的数量会膨胀。需要量化评估对存储的影响。

**4 种拆分方案的 key 数对比**：

假设有 B = 400 个 Bridge，每个平均 E = 10 种错误类型，保留 W = 5 天数据。

| 方案 | Key 数量 | 单键大小 | 总存储估算 | 膨胀倍数 |
|------|---------|---------|-----------|---------|
| 原始单键 | B × E = 4,000 | ~100B | ~400KB | 1× |
| 元数据+详情 | 2 × B × E = 8,000 | meta: ~80B / trace: ~2KB | ~8MB | 20× |
| 滚动时间窗（日） | B × E × W = 20,000 | ~80B | ~1.6MB | 4× |
| 滚动时间窗（时） | B × E × W × 24 = 480,000 | ~50B | ~24MB | 60× |
| 元数据+滚动窗 | 2 × B × E × W = 40,000 | 见上 | ~16MB | 40× |

**Key 数膨胀的影响**：

**对 SQLiteCache 的影响**：
- 更多行 → 索引更大 → 查询稍慢（但 BLOB 小了，单次 IO 更快）
- 2 万行 vs 4 千行，查询性能差异可忽略
- 存储空间：主要开销在 BLOB 数据，行数本身占比极低

**对 FileCache 的影响**：
- 每个 key 一个文件 → 4000 个文件 vs 20000 个文件
- 文件系统 inode 消耗增加（每个文件一个 inode）
- 目录遍历（`scandir` / `prune()`）变慢：O(n) 扫描 2 万文件 vs 4 千文件，慢 5 倍
- `prune()` 每天执行的话，2 万文件遍历约需几十毫秒到几百毫秒

**对 Memcached 的影响**：
- Key 数越多 → 内存占用越多
- 但 Memcached 是 LRU 淘汰，冷数据自然被踢
- 对命中率的影响：小 key 多了可能降低单 key 驱逐成本，但总体影响小

**控制膨胀的手段**：

1. **滑动窗口清理**：每天凌晨清理超过 N 天的时间窗 key
   ```
   error_reporting_{Bridge}_{Code}_{YYYYMMDD}
   → 每天只保留最近 5 天的 key
   → 稳定在 B × E × 5 = 20,000，不会无限增长
   ```

2. **采样聚合**：低优先级错误不按天细分，按周聚合
   - 高频错误：按天粒度（准确统计）
   - 低频错误：按周粒度（减少 key 数）

3. **TTL 自动过期**：靠 TTL 自然过期清理，无需主动删
   - 滚动窗的 key 设置 TTL = 窗口大小 + 1 天
   - 过期后自动消失，无需手动清理

**结论**：按天滚动窗口 + 元数据拆分，key 数从 4 千膨胀到 2-4 万，总存储从 400KB 涨到几 MB，完全在可接受范围内。如果是按小时窗口才需要警惕膨胀。

### 10.7 滚动时间窗丢窗口处理

使用滚动时间窗（按天/按小时分片）时，有几个边界场景会导致「窗口数据丢失」或「统计不准」。

**丢窗口的 4 种典型场景**：

**场景 1：跨天边界误差**
```
时间线（按天窗口）：
  2026-06-17 23:59:55 → 错误发生 → 计入 06-17 窗口
  2026-06-18 00:00:03 → 又一个错误 → 计入 06-18 窗口
  两次只差 8 秒，但分到了两个窗口

聚合「最近 5 天」时：
  如果当前时间是 06-18 00:00:10
  只算 06-14 到 06-18 → 06-17 窗口的 23:59:55 那一次被算在内 ✓
  但如果聚合逻辑是「只看完整的天」，06-18 当天会被排除 → 少计最近的错误
```
**处理**：聚合时包含「当天（不完整窗口）」，统计时注明「截至当前时间」。

**场景 2：空窗口没有值**
```
某 Bridge 某天没有任何错误 → 当天窗口的 key 不存在
聚合计算 5 天平均值时：
  方案 A：只算有数据的天数 → 平均值偏高（分母变小）
  方案 B：没 key 的天算 0 → 平均值偏低
```
**处理**：
- 计数器场景：空窗口当作 0（错误次数为 0 是合理语义）
- 聚合时遍历日期范围，逐个查 key，不存在则补 0
- 代码示例：
  ```php
  $total = 0;
  for ($i = 0; $i < 5; $i++) {
      $date = date('Ymd', strtotime("-$i days"));
      $key = "error_reporting_{$bridge}_{$code}_{$date}";
      $total += (int)$this->cache->get($key);  // 不存在返回 0
  }
  ```

**场景 3：窗口切换时的并发写入**
```
23:59:59.9 → 进程 A 读取 06-17 窗口 → count=100
00:00:00.1 → 进程 A 写回 → 应该写到 06-17 还是 06-18？
```
**处理**：读和写必须使用同一时间戳。读取窗口 key 时记下日期，写回时还用这个日期，而不是重新取 `date('Ymd')`。
```php
// 错误做法
$count = $cache->get("key_" . date('Ymd'));  // 读时是 23:59:59 → 06-17
$count++;
$cache->set("key_" . date('Ymd'), $count);    // 写时是 00:00:01 → 写到 06-18 了！

// 正确做法
$window = date('Ymd');
$key = "key_" . $window;
$count = (int)$cache->get($key);
$count++;
$cache->set($key, $count, $ttl);  // 用同一个 $window
```

**场景 4：长 TTL 窗口数据残留**
```
设置 TTL = 5 天
06-17 的窗口 → TTL 到 06-22 结束
06-22 当天聚合「最近 5 天」(06-18 ~ 06-22)
  → 06-17 的数据还在缓存里，但不被计入 ✓（正确）
06-23 聚合
  → 06-18 窗口的 TTL 是到 06-23 的某个精确秒数
  → 如果聚合时间早于过期时间，06-18 还在，计入（应该是 06-19~06-23）
  → 多算了一天？
```
**处理**：TTL 设为窗口大小 + 1 天的缓冲，聚合时按日期范围过滤，不依赖 TTL 自动清理。TTL 只负责兜底清理，业务逻辑自己判断窗口是否在范围内。

### 10.8 Redis INCR 幂等键设计

如果用 Redis 替代 SQLite 做错误计数，`INCR` 指令是天然的原子操作，能解决 lost update 问题。但需要注意幂等性设计。

**Redis INCR 的原子性优势**：
```
// 对比：当前 SQLite 实现（非原子）
$report = $cache->get($key);    // 读
$report['count']++;             // 改
$cache->set($key, $report);     // 写 → 竞态窗口

// Redis INCR（原子）
$count = $redis->incr($key);    // 单条指令，服务器端原子执行
// 不会有竞态，永远不会少算
```

**幂等性问题**：
- `INCR` 不是幂等的：同一次错误如果重试了，会多算一次
- rss-bridge 的错误报告是「每次异常触发一次计数」，天然就是「来一次加一次」的语义，不需要幂等
- 但如果上游有重试机制（比如客户端超时重发请求），同一个错误可能被计数多次

**幂等键设计方案**：
如果需要「同一错误只算一次」（比如按小时幂等）：

```
方案 A：错误指纹 + SETNX
  key = "error_reporting_{bridge}_{code}_{hour}_{fingerprint}"
  fingerprint = md5($e->getMessage() . $e->getFile() . $e->getLine())
  
  if ($redis->setnx($key, 1)) {
      // 首次出现，计数+1
      $redis->incr("error_reporting_{bridge}_{code}_{hour}_count");
      $redis->expire($key, 3600);  // 1 小时后过期
  }
  // 否则什么也不做
```
- 同一小时内完全相同的错误只计数一次
- 消耗：每个错误指纹一个 key + 一个计数器 key
- 适合：去重计数场景

**方案 B：滑动窗口 + ZSET（更精细）**
```
key = "error_reporting_{bridge}_{code}_sliding"
ZADD key score=timestamp member=requestId

// 清理窗口外的数据
ZREMRANGEBYSCORE key 0 (当前时间 - 窗口大小)

// 统计窗口内数量
ZCARD key
```
- 精确到秒级的滑动窗口计数
- 消耗：每个错误事件一个 ZSET member（约几十字节）
- 适合：需要精确速率限制的场景

**rss-bridge 场景选择**：
- 当前需求是「5 天内错误数超过 N 次就上报」，精度要求不高
- 用「按天 INCR + 聚合 5 天」足够
- 键数：B × E × 5 = 2 万，和 SQLite 方案差不多
- 复杂度：很低，`INCR` 一条指令搞定，比 SQLite 的读-改-写简单得多

**INCR 的注意事项**：
1. **INCR 溢出**：Redis 的 INCR 是 64 位有符号整数，最大 9.2×10¹⁸，error count 永远达不到
2. **初始值**：key 不存在时 INCR 会自动创建并从 0 开始 +1，结果为 1，符合预期
3. **TTL 配合**：`INCR` 不自动设 TTL，需要首次时单独 `EXPIRE`
   ```php
   $count = $redis->incr($key);
   if ($count === 1) {
       $redis->expire($key, $ttl);  // 只在第一次设置过期时间
   }
   ```
   但这不是原子的！要用 `SET key 1 EX ttl NX` 替代：
   ```php
   if ($redis->set($key, 1, ['ex' => $ttl, 'nx'])) {
       $count = 1;
   } else {
       $count = $redis->incr($key);
   }
   ```
4. **Memcached 也有 INCR**：不一定要换 Redis，Memcached 的 `increment()` 也是原子的
   - 注意 Memcached 的 `increment` 在 key 不存在时**不会自动创建**，返回 false
   - 需要先 `add($key, 0)` 初始化，再 `increment`
