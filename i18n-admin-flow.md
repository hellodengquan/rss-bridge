# RSS-Bridge i18n、Admin 配置与桥接页面协作流程

> 说明：经过代码分析，**本项目并未实现真正的 i18n（国际化多语言）系统**。所有面向用户的文案均直接硬编码为英文。下文所称的"语言资源"实际指的是桥接类中以常量形式定义的、在前端页面中展示的各种文本元数据。

---

## 一、整体架构概览

三者的协作关系如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                        配置加载层                                 │
│  config.default.ini.php → config.ini.php → 环境变量 RSSBRIDGE_*  │
│                   ↓                                              │
│          Configuration::loadConfiguration()                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        桥接类层 (Bridge)                          │
│  const NAME / URI / DESCRIPTION / MAINTAINER                     │
│  const PARAMETERS    ← 用户端参数定义 (表单渲染)                   │
│  const CONFIGURATION ← 管理员端配置 (从配置文件读取)               │
│  const CACHE_TIMEOUT                                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
               ┌────────────┴────────────┐
               ▼                         ▼
┌──────────────────────────┐  ┌───────────────────────────────────┐
│  FrontpageAction (首页)   │  │  DisplayAction (生成Feed)         │
│  - 读取桥接 PARAMETERS    │  │  - 桥接.loadConfiguration()       │
│  - 结合 Admin 配置渲染    │  │  - 桥接.setInput() + collectData()│
│    前端表单 HTML          │  │  - 按格式输出内容                  │
└──────────────────────────┘  └───────────────────────────────────┘
```

---

## 二、"语言资源"（文本元数据）定义与加载

### 2.1 语言资源的定义位置

所有可展示文本均定义在各桥接类（`bridges/*Bridge.php`）的**类常量**中，不存在独立的语言包文件。

核心常量定义于 `lib/BridgeAbstract.php:5-22`：

| 常量名           | 用途                          | 示例                                    |
|-----------------|-------------------------------|-----------------------------------------|
| `NAME`          | 桥接显示名称                   | `'YouTube'`                             |
| `URI`           | 源网站 URL                    | `'https://www.youtube.com'`             |
| `DESCRIPTION`   | 桥接功能描述（首页卡片显示）   | `'Returns the 10 newest videos...'`     |
| `MAINTAINER`    | 维护者 GitHub 用户名           | `'Niehztog'`                            |
| `DONATION_URI`  | 捐赠链接（Admin 可控制是否显示）| `''` (空则不显示)                       |
| `PARAMETERS`    | **用户端参数定义**（见 2.2）   | 多维数组，定义表单字段                  |
| `CONFIGURATION` | **管理员端配置定义**（见 3.4） | 多维数组，从配置文件读取                |
| `CACHE_TIMEOUT` | 缓存 TTL（秒）                 | `3600` (1小时)                          |

### 2.2 PARAMETERS — 用户端参数（表单渲染）

这是桥接页面最核心的"语言资源"，定义了前端表单的所有字段标签、提示、示例值。

**文件位置**：`lib/BridgeAbstract.php:21` + 各桥接类

**结构说明**：
```php
const PARAMETERS = [
    '上下文名称1' => [       // 命名上下文，会显示为 <h5> 标题
        '参数key' => [
            'name'          => '字段标签文本',    // ← 语言资源
            'type'          => 'text|number|list|checkbox',
            'title'         => '悬停提示文字',    // ← 语言资源
            'exampleValue'  => '输入框占位示例',  // ← 语言资源
            'defaultValue'  => '默认值',
            'required'      => true|false,
            'pattern'       => '正则验证',
            'values'        => [...],             // list 类型的选项
        ]
    ],
    'global' => [...]        // 特殊：合并到所有上下文
];
```

**示例**（来自 `bridges/YoutubeBridge.php:10-66`）：
```php
const PARAMETERS = [
    'By username' => [
        'u' => [
            'name' => 'username',
            'exampleValue' => 'LinusTechTips',
            'required' => true
        ]
    ],
    'Search result' => [
        's' => [
            'name' => 'search keyword',
            'exampleValue' => 'LinusTechTips',
            'required' => true
        ],
        'pa' => [
            'name' => 'page',
            'type' => 'number',
            'title' => 'This option is not work anymore...',
            'exampleValue' => 1
        ]
    ],
    'global' => [
        'duration_min' => [
            'name' => 'min. duration (minutes)',
            'type' => 'number',
            'title' => 'Minimum duration for the video in minutes',
            'exampleValue' => 5
        ]
    ]
];
```

**三种参数结构模式**：
1. **无参数**：`PARAMETERS = []` → 直接渲染"Generate feed"按钮
2. **单上下文 global**：`PARAMETERS = ['global' => [...]]` → 无上下文标题
3. **多命名上下文**：数组 key 为上下文名 → 每个上下文独立渲染 `<h5>` 标题 + 表单

### 2.3 语言资源的加载与使用流程

语言资源（即 PARAMETERS 和元数据常量）的读取链路：

```
index.php:13-14
    ↓ 引入
lib/bootstrap.php, lib/config.php
    ↓ 加载容器
lib/dependencies.php → BridgeFactory 实例化
    ↓ FrontpageAction::__invoke()
actions/FrontpageAction.php:30-36
    遍历 bridgeClassNames，对每个启用的桥接：
    - $bridge->getName()         ← 读取 NAME 常量 (lib/BridgeAbstract.php:62-65)
    - $bridge->getURI()          ← 读取 URI 常量
    - $bridge->getDescription()  ← 读取 DESCRIPTION 常量
    - $bridge->getParameters()   ← 读取 PARAMETERS 常量
    - $bridge->getMaintainer()   ← 读取 MAINTAINER 常量
    - $bridge->getDonationURI()  ← 读取 DONATION_URI 常量
    ↓ 调用
FrontpageAction::render()
    ↓ 将 PARAMETERS 转换为 HTML 表单 (见第四章)
templates/frontpage.html.php 输出
```

关键方法（`lib/BridgeAbstract.php:62-107`）：
```php
public function getName()        { return static::NAME ?? $this->getShortName(); }
public function getURI()         { return static::URI ?? 'https://github.com/...'; }
public function getDescription() { return static::DESCRIPTION; }
public function getParameters()  { return static::PARAMETERS; }
```

**特点**：采用 PHP 后期静态绑定（`static::`），子类常量覆盖父类。

---

## 三、Admin 后台配置系统

> 注意：RSS-Bridge **没有 Web 后台管理界面**。所有配置通过修改 `config.ini.php` 文件或设置环境变量完成。

### 3.1 配置的三级来源（优先级从低到高）

| 优先级 | 来源                        | 文件位置                                  | 说明                              |
|-------|----------------------------|------------------------------------------|----------------------------------|
| 1 (低)| `config.default.ini.php`   | 项目根目录（不可修改，更新时会被覆盖）     | 默认配置                          |
| 2    | `config.ini.php`           | 项目根目录（需手动创建，复制自 default） | 用户自定义配置，覆盖默认值        |
| 3 (高)| 环境变量 `RSSBRIDGE_*`      | 系统环境 / Docker env / `.env`          | 最高优先级，再次覆盖              |

**加载入口**：`lib/config.php:1-13`
```php
$config = [];
if (file_exists(__DIR__ . '/../config.ini.php')) {
    $config = parse_ini_file(__DIR__ . '/../config.ini.php', true, INI_SCANNER_TYPED);
}
Configuration::loadConfiguration($config, getenv());
```

### 3.2 Configuration 类核心逻辑

**文件**：`lib/Configuration.php`

**`loadConfiguration()` 执行流程（`lib/Configuration.php:18-156`）**：

```
Step 1: 解析 config.default.ini.php → 存入 self::$config
Step 2: 解析 config.ini.php → 覆盖相同配置项
Step 3: 检查 DEBUG 文件存在 → 若存在则设 env=dev, cache=array
Step 4: 检查 whitelist.txt 存在 → 载入 enabled_bridges 列表
Step 5: 遍历环境变量，匹配 RSSBRIDGE_前缀 → 再次覆盖
Step 6: 对所有关键配置项做类型/范围校验，失败则 HTTP 500 退出
```

**配置读取**：`lib/Configuration.php:158-164`
```php
public static function getConfig(string $section, string $key, $default = null)
{
    return self::$config[strtolower($section)][strtolower($key)] ?? $default;
}
```

### 3.3 标准配置段

定义于 `config.default.ini.php:1-190`，常用配置段：

| 配置段 (section)  | 关键字段                                 | 影响范围                       |
|------------------|------------------------------------------|-------------------------------|
| `[system]`       | env, enabled_bridges, timezone, message  | 系统环境、桥接白名单、全站消息 |
| `[admin]`        | email, telegram, donations               | **首页页脚联系方式、捐赠按钮** |
| `[proxy]`        | url, by_bridge, name                     | 代理设置、桥接表单的"禁用代理"选项 |
| `[cache]`        | type, custom_timeout                     | **桥接表单的"缓存超时"选项**   |
| `[authentication]` | enable, username, password, token       | HTTP Basic Auth、Token 鉴权    |
| `[error]`        | output, report_limit                     | 错误输出方式                   |
| `[youtube]`      | iframe, nocookie                         | YouTube 视频嵌入方式           |

### 3.4 桥接级专属配置（CONFIGURATION 常量）

桥接可在 `const CONFIGURATION` 中定义需要管理员在 config.ini.php 中设置的专属参数。

**定义位置**：`bridges/*Bridge.php`，如 `bridges/TelegramBridge.php:19-24`
```php
const CONFIGURATION = [
    'max_pages' => [
        'required'      => false,
        'defaultValue'  => 1,
    ],
];
```

**读取流程**（`lib/BridgeAbstract.php:119-136` `loadConfiguration()`）：
```php
public function loadConfiguration()
{
    foreach (static::CONFIGURATION as $optionName => $optionValue) {
        $section = $this->getShortName();  // 如 "TelegramBridge"
        // 从 [TelegramBridge] 段读取 max_pages 配置
        $configurationOption = Configuration::getConfig($section, $optionName);

        if ($configurationOption !== null) {
            $this->configuration[$optionName] = $configurationOption;
        } elseif (isset($optionValue['required']) && $optionValue['required'] === true) {
            throw new \Exception(sprintf('Missing configuration option: %s', $optionName));
        } elseif (isset($optionValue['defaultValue'])) {
            $this->configuration[$optionName] = $optionValue['defaultValue'];
        }
    }
}
```

**对应 config.ini.php**：
```ini
[TelegramBridge]
max_pages = 5          ; 管理员设置，用户端不可见
```

**桥接内使用**：`$this->getOption('max_pages')` （`lib/BridgeAbstract.php:86-89`）

**设计意图**：将"配置"与"参数"解耦：
- `PARAMETERS` = 终端用户每次请求时填写（如用户名、搜索关键词）
- `CONFIGURATION` = 部署者一次性设置（如 API Key、最大页数），对终端用户透明

### 3.5 环境变量映射规则

环境变量名格式：`RSSBRIDGE_<SECTION>_<KEY>`

**映射逻辑**（`lib/Configuration.php:55-82`）：
```
RSSBRIDGE_SYSTEM_TIMEZONE → section=system, key=timezone
RSSBRIDGE_ADMIN_EMAIL     → section=admin,  key=email
RSSBRIDGE_YOUTUBE_IFRAME  → section=youtube,key=iframe
RSSBRIDGE_SYSTEM_ENABLED_BRIDGES=Telegram,Youtube → 特殊处理: 按逗号拆分为数组
```

---

## 四、桥接页面渲染（PARAMETERS → HTML 表单）

### 4.1 入口：FrontpageAction

**文件**：`actions/FrontpageAction.php`

**`__invoke()` 主流程**（`actions/FrontpageAction.php:13-49`）：
```php
public function __invoke(Request $request): Response
{
    // 1. 获取所有桥接类名
    $bridgeClassNames = $this->bridgeFactory->getBridgeClassNames();

    // 2. 遍历启用的桥接，逐个调用 render() 生成卡片 HTML
    $body = '';
    foreach ($bridgeClassNames as $bridgeClassName) {
        if ($this->bridgeFactory->isEnabled($bridgeClassName)) {
            $bridge = $this->bridgeFactory->create($bridgeClassName);
            $body .= self::render($bridge, $bridgeClassName, $token);
        }
    }

    // 3. 注入 Admin 配置到模板变量
    return new Response(render(__DIR__ . '/../templates/frontpage.html.php', [
        'messages'          => $messages,
        'admin_email'       => Configuration::getConfig('admin', 'email'),   // ← Admin 配置
        'admin_telegram'    => Configuration::getConfig('admin', 'telegram'),// ← Admin 配置
        'bridges'           => $body,
        'active_bridges'    => $activeBridges,
        'total_bridges'     => count($bridgeClassNames),
    ]));
}
```

### 4.2 Admin 配置对桥接卡片的动态注入

**在 `FrontpageAction::render()` 中，Admin 配置会修改 PARAMETERS**：

**注入 1：代理禁用选项**（`actions/FrontpageAction.php:63-72`）
```php
if (Configuration::getConfig('proxy', 'url') && Configuration::getConfig('proxy', 'by_bridge')) {
    $proxyName = Configuration::getConfig('proxy', 'name') ?: Configuration::getConfig('proxy', 'url');
    $parameters['global']['_noproxy'] = [
        'name' => sprintf('Disable proxy (%s)', $proxyName),  // ← 运行时生成的文本
        'type' => 'checkbox',
    ];
}
```
> 效果：当管理员启用了按桥接控制代理时，所有桥接表单自动增加一个"禁用代理"复选框。

**注入 2：自定义缓存超时**（`actions/FrontpageAction.php:74-80`）
```php
if (Configuration::getConfig('cache', 'custom_timeout')) {
    $parameters['global']['_cache_timeout'] = [
        'name' => 'Cache timeout in seconds',
        'type' => 'number',
        'defaultValue' => $bridge->getCacheTimeout()
    ];
}
```
> 效果：当管理员启用自定义缓存时，所有桥接表单自动增加"缓存超时"输入框。

**条件显示 3：捐赠按钮**（`actions/FrontpageAction.php:136-144`）
```php
if (Configuration::getConfig('admin', 'donations') && $bridge->getDonationURI()) {
    // 显示 "maintainer ~ Donate" 链接
} else {
    // 仅显示 maintainer
}
```

### 4.3 PARAMETERS → HTML 渲染逻辑

**`renderForm()` 方法**（`actions/FrontpageAction.php:150-243`）：

```
输入: bridgeClassName, contextName, parameters, token
输出: 完整的 <form> HTML 字符串

流程:
  1. 写入隐藏字段: action=display, bridge=桥接类名
  2. 若启用 token 认证 → 追加 token 隐藏字段
  3. 若有上下文名 → 追加 context 隐藏字段
  4. 遍历 parameters 中的每个字段:
     - 生成 <label>（使用 parameter['name']）
     - 按 type 调用 getTextInput/getNumberInput/getListInput/getCheckboxInput
     - 若有 parameter['title'] → 生成带 title 的 <i class="info">
     - 若有 parameter['exampleValue'] → 生成右键可用的占位提示
  5. 生成 "Generate feed" 提交按钮
```

**字段类型对应关系**：

| PARAMETERS type | 生成的 HTML                           |
|----------------|---------------------------------------|
| `text` (默认)  | `<input type="text">` + placeholder   |
| `number`       | `<input type="number">`               |
| `checkbox`     | `<input type="checkbox">`             |
| `list`         | `<select>` + `<option>` (支持 optgroup)|

### 4.4 首页页脚的 Admin 信息

**文件**：`templates/frontpage.html.php:47-57`

```php
<?php if ($admin_email): ?>
    <div>Email: <a href="mailto:..."><?= e($admin_email) ?></a></div>
<?php endif; ?>

<?php if ($admin_telegram): ?>
    <div>Url: <a href="<?= e($admin_telegram) ?>"><?= e($admin_telegram) ?></a></div>
<?php endif; ?>
```

> 管理员在 `[admin]` 段设置的 `email` 和 `telegram` 会出现在首页底部。

---

## 五、用户提交请求：DisplayAction 中的协作

当用户在桥接页面填写表单并点击"Generate feed"后，数据流向：

### 5.1 请求接收与校验

**文件**：`actions/DisplayAction.php`

**`__invoke()` 流程**（`actions/DisplayAction.php:19-67`）：
```
1. 从 Request 读取 bridge, format, _noproxy, _cache_timeout
2. BridgeFactory 校验桥接名是否存在、是否在白名单
3. 若 _noproxy 开启且 admin.proxy.by_bridge=true → 定义 NOPROXY 常量
4. 创建桥接实例
5. createResponse() 处理业务逻辑
6. 按 _cache_timeout 或桥接默认值写入缓存
```

### 5.2 桥接配置与参数的加载顺序

**`createResponse()` 方法**（`actions/DisplayAction.php:73-90`）：
```php
$bridge->loadConfiguration();              // Step 1: 从 Admin 配置加载 CONFIGURATION

// 过滤掉系统参数，只保留桥接业务参数
$input = array_diff_key($requestArray, [
    'token','action','bridge','format',
    '_noproxy','_cache_timeout','_error_time','_'
]);

$bridge->setInput($input);                 // Step 2: 校验并注入用户端 PARAMETERS
$bridge->collectData();                    // Step 3: 桥接逻辑采集数据
```

### 5.3 用户参数校验与上下文推断

**`setInput()` 流程**（`lib/BridgeAbstract.php:138-179`）：
```
1. 提取 context 字段（如有）→ 设置 queriedContext
2. ParameterValidator::validateInput()
   - 遍历用户输入，检查是否在 PARAMETERS 中注册
   - 按 type 做类型转换（number/checkbox/list/text）
   - 检查 required 字段是否为空
3. 若未显式指定 context → ParameterValidator::getQueriedContext()
   - 通过用户输入与各上下文参数做匹配，推断当前请求属于哪个上下文
   - global 上下文会被自动合并
4. setInputWithContext()
   - 将用户输入写入 $this->inputs[queriedContext]
   - 对未提交的参数应用 defaultValue
   - 将 global 参数复制到 queriedContext
```

**`getQueriedContext()` 算法**（`lib/ParameterValidator.php:61-121`）：
```
对每个上下文 context:
  若用户输入中有任何一个不属于该上下文且不属于 global → 跳过
  检查该上下文的所有 required 字段是否都已满足 → 标记 true/false/null
最后:
  若有且仅有一个上下文为 true → 返回该上下文
  若全部为 null（无必填参数）→ 返回第一个无参数上下文
  否则返回 false (混合上下文) 或 null (缺少必填)
```

---

## 六、关键协作点汇总

### 6.1 Admin 配置 → 桥接页面渲染 协作矩阵

| Admin 配置项                   | 作用位置                                  | 效果                                   |
|-------------------------------|------------------------------------------|---------------------------------------|
| `system.enabled_bridges`      | `BridgeFactory::__construct()`           | 决定哪些桥接出现在首页                  |
| `system.message`              | `render()` (lib/html.php:12-17)          | 全站顶部显示系统消息横幅                |
| `admin.email` / `telegram`    | `templates/frontpage.html.php`           | 页脚显示管理员联系方式                  |
| `admin.donations`             | `FrontpageAction::render():136`          | 控制是否显示 Donate 链接               |
| `proxy.url` + `by_bridge`     | `FrontpageAction::render():63`           | 表单自动注入"Disable proxy"复选框       |
| `cache.custom_timeout`        | `FrontpageAction::render():74`           | 表单自动注入"Cache timeout"输入框       |
| `authentication.token`        | `FrontpageAction::renderForm():163`      | 表单自动注入 token 隐藏字段             |
| 桥接专属 `[BridgeName].*`     | `BridgeAbstract::loadConfiguration()`    | 桥接内部逻辑使用，用户端不可见          |

### 6.2 PARAMETERS 与 CONFIGURATION 对比

| 维度            | PARAMETERS                     | CONFIGURATION                         |
|----------------|-------------------------------|--------------------------------------|
| 受众            | 终端用户（每次请求填写）         | 管理员部署时一次性配置                 |
| 来源            | 表单 GET 参数                  | config.ini.php / 环境变量              |
| 可见性          | 前端页面可见可改                | 用户端不可见                          |
| 定义格式        | `name/type/title/required...`  | `required/defaultValue`               |
| 校验类          | `ParameterValidator`           | `Configuration::loadConfiguration()`  |
| 读取方式        | `$this->getInput('key')`       | `$this->getOption('key')`             |

### 6.3 关键文件索引

| 职责                  | 文件路径                                      | 关键行号                       |
|----------------------|-----------------------------------------------|-------------------------------|
| 配置核心类            | `lib/Configuration.php`                       | 18-156, 158-164               |
| 配置加载入口          | `lib/config.php`                              | 1-13                          |
| 默认配置文件          | `config.default.ini.php`                      | 全部                          |
| 桥接抽象基类          | `lib/BridgeAbstract.php`                      | 1-22, 62-136, 138-263         |
| 参数校验器            | `lib/ParameterValidator.php`                  | 8-121                         |
| 桥接工厂              | `lib/BridgeFactory.php`                       | 11-42, 44-52                  |
| 首页动作              | `actions/FrontpageAction.php`                 | 13-49, 51-148, 150-324        |
| Feed生成动作          | `actions/DisplayAction.php`                   | 19-67, 69-144                 |
| 模板渲染函数          | `lib/html.php`                                | 6-30, 85-146                  |
| 首页模板              | `templates/frontpage.html.php`                | 47-57                         |
| 页面基模板            | `templates/base.html.php`                     | 22-28 (消息渲染)              |

---

## 七、扩展：若要实现真正的 i18n

当前项目无多语言能力。若需接入 i18n，需在以下位置做改造：

1. **桥接元数据层**：将 `NAME`、`DESCRIPTION`、`PARAMETERS[name/title/exampleValue]` 改为翻译键或支持多语言数组
2. **前端渲染层**：`FrontpageAction::render()` 中引入 `__()` 翻译函数包裹所有输出文本
3. **错误消息层**：`Configuration::throwConfigError()`、异常消息、`DisplayAction` 中的错误提示统一走翻译
4. **模板层**：`templates/*.html.php` 中的硬编码文案提取为翻译调用
5. **Admin 配置**：增加 `[system] default_locale` 配置项，按请求参数切换语言
