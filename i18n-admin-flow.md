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

## 七、桥接页面运行时多语言切换的代码路径

### 7.1 核心结论：项目没有框架级的多语言切换

RSS-Bridge **整体框架层完全没有实现 UI 语言切换机制**。具体证据：

| 检查点 | 代码位置 | 实际情况 |
|-------|---------|---------|
| 语言 GET 参数 | `$_GET['lang']` / `$_GET['locale']` | 全局搜索无匹配 |
| Cookie 语言记忆 | `$_COOKIE['lang']` | 全局搜索无匹配 |
| 浏览器语言检测 | `$_SERVER['HTTP_ACCEPT_LANGUAGE']` | 全局搜索无匹配（仅在 curl 请求第三方站点时使用） |
| PHP 语言设置 | `setlocale()` / `mb_internal_encoding()` | 全局搜索无匹配 |
| 翻译函数 | `__()` / `t()` / `gettext()` | 全局搜索无匹配 |
| 模板 lang 属性 | `templates/base.html.php:2` / `templates/html-format.html.php:2` | 硬编码为 `<html lang="en">` |
| 前端 JS | `static/rss-bridge.js` | 纯搜索和参数填充逻辑，无语言切换代码 |

框架层所有界面文案（如 "Generate feed"、"Email:"、"Find feed by URL" 等）均在模板文件和 `FrontpageAction` 中**直接硬编码为英文**，无任何抽象层。

### 7.2 存在的"多语言"：桥接级别的目标网站语言参数

项目中确实存在大量 `language` 相关代码，但这些不是 UI 语言切换，而是**让终端用户选择"要爬取的目标网站使用哪种语言版本"**，完全是桥接的业务参数。

**典型桥接示例**：

#### 7.2.1 WikipediaBridge — 通过 language 参数选择不同语言维基

**文件**：`bridges/WikipediaBridge.php:13-43, 45-123`

```php
const PARAMETERS = [ [
    'language' => [
        'name' => 'Language',
        'type' => 'list',
        'title' => 'Select your language',
        'values' => [
            'English' => 'en',
            'Русский' => 'ru',
            'Dutch' => 'nl',
            // ...
        ]
    ],
    // ...
]];

// collectData() 中根据参数选择解析函数
$function = 'getContents' . ucfirst(strtolower($this->getInput('language')));
// en -> getContentsEn(), de -> getContentsDe(), fr -> getContentsFr() ...
if (!method_exists($this, $function)) {
    throwServerException('A function to get the contents for your language is missing...');
}
$this->$function($html, $subject, $fullArticle);
```

**用户选择流程图**：
```
用户在下拉框选择 "Русский"
    ↓
URL 参数: ?action=display&bridge=WikipediaBridge&language=ru
    ↓
collectData() 解析
    ↓
构造函数名 getContentsRu()
    ↓
调用该私有方法，解析 ru.wikipedia.org 的 DOM 结构
    ↓
输出俄语内容的 Feed（标题和正文都是俄语原文，非翻译）
```

> 注意：这里的"语言切换"只是**切换爬取的源站点**（从 en.wikipedia 切到 ru.wikipedia），Feed 内容是目标站点的原文，RSS-Bridge 自身不做任何翻译。

#### 7.2.2 NHKWorldJapanShowBridge — 唯一自带 UI 级多语言的桥接

**文件**：`bridges/NHKWorldJapanShowBridge.php:13-175, 303-315`

这个桥接是全项目中**唯一实现了自身 UI 文案多语言翻译**的桥接。它自实现了一套独立的 i18n 机制，与框架完全无关。

**定义多语言标签**（`bridges/NHKWorldJapanShowBridge.php:64-175`）：
```php
protected static $labels = [
    'length' => [
        'ar' => 'المدة:',
        'en' => 'Length:',
        'zh' => '时长:',
        'fr' => 'Durée:',
        // ... 共 17 种语言
    ],
    'broadcast' => [ /* ... */ ],
    'availableuntil' => [ /* ... */ ],
    'watchdirectly' => [ /* ... */ ],
    'watchonplayer' => [ /* ... */ ],
];
```

**翻译函数**（`bridges/NHKWorldJapanShowBridge.php:303-315`）：
```php
protected function getLocaleString($string)
{
    $language = $this->getInput('language');
    // Step 1: 命中请求的语言 → 返回
    if (isset(self::$labels[$string][$language])) {
        return self::$labels[$string][$language];
    }
    // Step 2: 回退到英语 → 返回
    if (isset(self::$labels[$string]['en'])) {
        return self::$labels[$string]['en'];
    }
    // Step 3: 连英语都没有 → 空字符串
    return '';
}
```

**渲染中使用**（`bridges/NHKWorldJapanShowBridge.php:263-271`）：
```php
$description .= $this->getLocaleString('length') . ' ' . $movielength . '<br>';
$description .= $this->getLocaleString('broadcast') . ' ' . $broadcastdate . '<br>';
$description .= $this->getLocaleString('availableuntil') . ' ' . $voddate . '<br>';
$description .= '<a href="...">' . $this->getLocaleString('watchdirectly') . '</a>';
```

**语言影响的其他表现**：
- 日期格式：`en` 语言用 `'F j, Y'`（如 "April 10, 2025"），其他语言用 `'Y-m-d'`（`bridges/NHKWorldJapanShowBridge.php:249-250`）
- 文本方向：阿拉伯语/波斯语/乌尔都语用 RTL（从右到左），其他用 LTR（`bridges/NHKWorldJapanShowBridge.php:177-179, 251`）

#### 7.2.3 其他桥接的 language 参数模式

其他桥接的 `language` 参数基本都是将语言代码拼接到目标 URL 中，让目标站点返回对应语言内容：

| 桥接 | 参数形式 | URL 拼接方式 |
|-----|---------|-------------|
| WhatsAppBlogBridge | `language` | `https://blog.whatsapp.com/?lang=` + code |
| NovayaGazetaEuropeBridge | `language` | 主页 URL + `?lang=` + code |
| WebfailBridge | `language` | `https://` + code + `.webfail.com` |
| NHKWorldJapanShowBridge | `language` | `/nhkworld/` + code + `/shows/...` |
| SamsungMobileChangelogBridge | 无参数，代码内置 | 在页面中查找 `<option value=EN>` 对应的 URL |

### 7.3 代码路径总览

**完整的运行时路径（以 WikipediaBridge 为例）**：

```
index.php
  ↓ lib/bootstrap.php / lib/dependencies.php
  ↓ RssBridge::main() (lib/RssBridge.php:13-40)
  ↓ action=Frontpage（首页渲染阶段）
FrontpageAction::__invoke()
  ↓ 读取 WikipediaBridge::PARAMETERS
  ↓ 发现 'language' 字段是 type=list，有 6 个 values
  ↓ FrontpageAction::renderForm() → 渲染为 <select> 下拉框
  ↓ templates/frontpage.html.php 输出
  ↓ 用户在浏览器看到下拉框 "Language"，选择 "Русский" 并提交
  ↓ 请求 URL: ?action=display&bridge=WikipediaBridge&language=ru&subject=tfa
  ↓ action=Display（Feed 生成阶段）
DisplayAction::__invoke()
  ↓ DisplayAction::createResponse()
  ↓ $bridge->loadConfiguration()  ← 无 CONFIGURATION，跳过
  ↓ $bridge->setInput(['language' => 'ru', 'subject' => 'tfa'])
  ↓ ParameterValidator 校验 → 通过
  ↓ $bridge->collectData()
  ↓   getURI() 返回 https://ru.wikipedia.org
  ↓   $function = 'getContentsRu'
  ↓   $this->getContentsRu($html, $subject, $fullArticle)
  ↓   解析俄语维基的 DOM 结构，生成 Feed items
  ↓ 按 format 参数输出（HTML/Atom/RSS/JSON...）
```

---

## 八、Admin 后台缓存清理对 i18n 资源的影响与触发条件

### 8.1 核心结论：缓存系统与"i18n 资源"完全无关

RSS-Bridge 没有 Web 管理后台，也没有"清理 i18n 资源缓存"的机制。原因是：

> **项目的"语言资源"（桥接 PARAMETERS、NAME、DESCRIPTION 等）不经过缓存系统，它们是 PHP 类常量，每次请求由 PHP 解释器直接从 PHP 文件加载。**

缓存系统只缓存**运行时抓取的数据**，与"语言资源"（文本元数据）完全隔离。

### 8.2 缓存系统的完整结构

#### 8.2.1 缓存接口与实现

**接口定义**：`lib/CacheInterface.php:3-14`
```php
interface CacheInterface
{
    public function get(string $key, $default = null);
    public function set(string $key, $value, ?int $ttl = null): void;
    public function delete(string $key): void;
    public function clear(): void;      // 清空全部
    public function prune(): void;      // 清理过期项
}
```

**五种实现对比**：

| 实现类 | 文件 | 存储介质 | `clear()` 实现 | `prune()` 实现 |
|-------|-----|---------|---------------|---------------|
| `FileCache` | `caches/FileCache.php` | 本地文件 `cache/*.cache` | `scandir()` 遍历删除所有 `.cache` 文件 | 遍历所有文件，反序列化检查 `expiration <= time()` 则删除 |
| `SQLiteCache` | `caches/SQLiteCache.php` | SQLite 数据库文件 | `DELETE FROM storage` | `DELETE FROM storage WHERE updated > 0 AND updated <= now` |
| `MemcachedCache` | `caches/MemcachedCache.php` | Memcached 服务 | `$conn->flush()` | 空实现（Memcached 自带 TTL 自动淘汰） |
| `ArrayCache` | `caches/ArrayCache.php` | 进程内存数组 | `$this->data = []` | 遍历数组删除过期项 |
| `NullCache` | `caches/NullCache.php` | 无（纯占位） | 空实现 | 空实现 |

**实例化位置**：`lib/dependencies.php:66-71` → `CacheFactory::create()`（`lib/CacheFactory.php:15-110`）

#### 8.2.2 缓存 Key 命名空间与缓存内容

通过全局搜索 `$cache->set(` 和 `$cacheKey =`，项目中有四类缓存数据：

| Key 前缀 | 设置位置 | 缓存内容 | TTL |
|---------|---------|---------|-----|
| `http_` | `middlewares/CacheMiddleware.php:24, 50, 53` | DisplayAction 的完整 `Response` 对象（含 body、headers、status code） | 成功 200：由桥接控制；错误 4xx/5xx：5-15 分钟随机 |
| `server_` | `lib/contents.php:71, 119` | `getContents()` 抓取的远程 HTTP Response | 固定 10 天（864000 秒），除非响应头含 `no-cache`/`no-store` |
| `pages_` | `lib/contents.php:236, 240` | `getSimpleHTMLDOMCached()` 抓取的 HTML 字符串 | 默认 1 天（86400 秒），调用方可覆盖 |
| `error_reporting_` | 桥接内部（未在主路径使用） | 错误上报计数 | - |

**没有任何缓存 Key 存储"翻译字符串"或"语言资源"。**

### 8.3 缓存清理的触发条件

项目没有提供 Admin Web 界面或 CLI 命令来主动清理缓存。缓存清理/失效只有以下四种被动触发方式：

#### 触发方式 1：随机被动清理（1% 概率）

**位置**：`middlewares/CacheMiddleware.php:56-60`
```php
// For 1% of requests, prune cache
if (rand(1, 100) === 1) {
    // This might be resource intensive!
    $this->cache->prune();
}
```
- **触发时机**：每次经过 CacheMiddleware 的请求（仅 DisplayAction 会被缓存）
- **行为**：调用 `prune()` 清理所有已过期的缓存项
- **对 i18n 的影响**：零。只清理 `http_` 前缀的 Feed Response 缓存

#### 触发方式 2：懒过期（读取时检查）

**位置**：
- `caches/FileCache.php:40-45`（get 方法）
- `caches/SQLiteCache.php:64-77`（get 方法）
- `caches/ArrayCache.php:18-24`（get 方法）

```php
// FileCache 示例
$expiration = $item['expiration'] ?? time();
if ($expiration === 0 || $expiration > time()) {
    return $item['value'];
}
$this->delete($key);   // 读时发现过期，立即删除
return $default;
```
- **触发时机**：读取缓存 Key 时
- **行为**：若已过期则删除该 Key 并返回默认值
- **对 i18n 的影响**：零。PARAMETERS 等语言资源不走缓存系统

#### 触发方式 3：Admin 修改 config.ini.php 切换缓存类型

**位置**：`config.default.ini.php:142-170`

```ini
[cache]
type = "File"   ; 可改为: File / SQLite / Memcached / Array / Null
```

- **触发方式**：管理员手动修改 `config.ini.php` 的 `[cache] type` 值
- **行为**：下次请求时 `CacheFactory::create()` 创建不同的缓存后端实例
- **实际效果**：相当于"逻辑上清空缓存"（切换到新后端就无法读取旧后端的数据了），但旧后端的物理文件/数据不会被删除
- **对 i18n 的影响**：零。与语言资源无关

#### 触发方式 4：DEBUG 文件触发切换到 ArrayCache

**位置**：`lib/Configuration.php:36-39`

```php
if (file_exists(__DIR__ . '/../DEBUG')) {
    $defaultConfig['system']['env'] = 'dev';
    $defaultConfig['cache']['type'] = 'array';
}
```
- **触发方式**：管理员在项目根目录创建名为 `DEBUG` 的空文件
- **行为**：强制使用 `ArrayCache`（进程内内存缓存），每次 PHP 进程结束缓存自动丢失
- **对 i18n 的影响**：零。但会导致 Feed 内容缓存完全失效，所有请求都重新抓取

### 8.4 缓存 TTL 与语言参数的关系

虽然缓存系统不存储"语言资源"，但**语言参数会参与缓存 Key 的生成**，导致不同语言版本的 Feed 会被分别缓存。

**CacheMiddleware 的 Key 生成**（`middlewares/CacheMiddleware.php:24`）：
```php
$cacheKey = 'http_' . json_encode($request->toArray());
```

由于 `$request->toArray()` 包含全部 GET 参数，如果桥接有 `language` 参数，那么：
- `http_{bridge=WikipediaBridge,language=en,...}` → 英语版缓存
- `http_{bridge=WikipediaBridge,language=ru,...}` → 俄语版缓存
- `http_{bridge=WikipediaBridge,language=de,...}` → 德语版缓存

这些是**互相独立的缓存条目**。清理缓存（比如 prune）时会一起被清理，但这不是"清理 i18n 资源"，只是清理已缓存的各语言 Feed 内容。

---

## 九、语言资源热加载与回退到默认语言的代码机制

### 9.1 热加载机制：依赖 PHP OPCache，无项目级实现

RSS-Bridge 的"语言资源"（桥接类中的 PARAMETERS、NAME、DESCRIPTION 常量）**完全没有项目级的热加载机制**。它们的加载/更新完全依赖 PHP 解释器本身的行为。

#### 9.1.1 正常请求流程中的语言资源加载

```
浏览器请求 index.php
    ↓
PHP-FPM/Apache PHP 模块接收请求
    ↓
Zend Engine 编译 PHP 文件为 opcode
    ├─ 若 OPCache 启用且有缓存且未过期 → 直接用 opcode 缓存
    └─ 否则 → 重新编译 bridges/*Bridge.php 等所有 PHP 文件
    ↓
读取类常量（static::PARAMETERS、static::NAME 等）
    ↓
FrontpageAction 渲染为 HTML
```

**关键点**：
- 桥接常量是**编译期确定**的，运行时无法修改
- 修改 `bridges/WikipediaBridge.php` 中的 `PARAMETERS` 后，是否立即生效取决于 OPCache 配置：
  - `opcache.validate_timestamps=1`（默认）→ 检查文件 mtime，变更后自动重新编译，相当于"热加载"
  - `opcache.validate_timestamps=0`（生产常用优化）→ 必须重启 PHP-FPM 或调用 `opcache_reset()` 才生效
- 项目代码中**没有任何地方**调用 `opcache_reset()` 或 `opcache_invalidate()`

#### 9.1.2 热加载的边界：哪些修改可以"热生效"

| 修改内容 | 热加载（OPCache validate_timestamps=1） | 需要重启 |
|---------|--------------------------------------|---------|
| 修改桥接类 `const PARAMETERS` 的 name/title/exampleValue | ✅ PHP 重新编译即生效 | ❌ |
| 修改桥接类 `const NAME` / `DESCRIPTION` | ✅ 同上 | ❌ |
| 修改 `templates/*.html.php` 模板文案 | ✅ 模板是 `require` 加载，每次重新编译 | ❌ |
| 修改 `config.default.ini.php` / `config.ini.php` | ✅ `Configuration::loadConfiguration()` 每次请求都重新 parse_ini_file | ❌ |
| 修改 `static/` 下的 JS/CSS | ✅ 静态文件由 Web 服务器直接服务，不受 PHP OPCache 影响 | ❌ |
| 修改 `composer.json` / 新增依赖 | ❌ 需重新 `composer dump-autoload` | - |
| 修改 `spl_autoload_register` 路径配置 | ❌ autoload 映射未更新 | ✅ |

> 管理员若在 `config.ini.php` 中修改了某个桥接的 `[BridgeName]` 配置段，无需重启，下次请求即生效（`Configuration::loadConfiguration()` 在 `lib/config.php:1-13` 中每次请求都重新 `parse_ini_file`）。

### 9.2 CONFIGURATION（管理员配置）的默认值回退

**位置**：`lib/BridgeAbstract.php:119-148` `loadConfiguration()`

```php
public function loadConfiguration()
{
    foreach (static::CONFIGURATION as $optionName => $optionValue) {
        $section = $this->getShortName();
        $configurationOption = Configuration::getConfig($section, $optionName);

        if ($configurationOption !== null) {
            // 优先级 1: config.ini.php 中有对应配置 → 直接使用
            $this->configuration[$optionName] = $configurationOption;
        } elseif (isset($optionValue['required']) && $optionValue['required'] === true) {
            // 优先级 2: required=true 且缺失 → 抛异常，中止请求
            throw new \Exception(sprintf('Missing configuration option: %s', $optionName));
        } elseif (isset($optionValue['defaultValue'])) {
            // 优先级 3: 存在 defaultValue → 使用默认值
            $this->configuration[$optionName] = $optionValue['defaultValue'];
        }
        // 优先级 4: 未设置 required，也无 defaultValue → 不赋值，getOption() 时返回 null
    }
}
```

**完整回退链**：
```
config.ini.php [BridgeName] 段中定义了该 key
    ↓ 存在
使用该值
    ↓ 不存在
CONFIGURATION['required'] === true
    ↓ 是
抛 HTTP 500 异常: "Missing configuration option: xxx"
    ↓ 否
isset(CONFIGURATION['defaultValue'])
    ↓ 是
使用 defaultValue
    ↓ 否
$this->configuration[$optionName] 未设置
    ↓ 调用 getOption($optionName)
返回 null（BridgeAbstract.php:86-89 中 return $this->configuration[$name] ?? null）
```

**示例**：以 `bridges/TelegramBridge.php:19-24` 为例
```php
const CONFIGURATION = [
    'max_pages' => [
        'required'      => false,
        'defaultValue'  => 1,
    ],
];
```
若管理员未在 config.ini.php 中设置 `[TelegramBridge] max_pages`，则自动回退为 `1`。

### 9.3 PARAMETERS（用户参数）的 defaultValue 回退

**位置**：`lib/BridgeAbstract.php:181-263` `setInputWithContext()`

用户提交表单时，未填写/未选中的字段按以下规则回退：

#### 9.3.1 四种回退规则

| 参数 type | 用户未提交时的行为 | 代码位置 |
|----------|-----------------|---------|
| `checkbox` | 固定回退为 `false` | `lib/BridgeAbstract.php:235-240` |
| `list` (下拉框) | 优先使用 `defaultValue`；若无则使用 `values` 数组的第一项 | `lib/BridgeAbstract.php:241-248` |
| `text` / `number` / 其他 | 优先使用 `defaultValue`；若无则不赋值（留空） | `lib/BridgeAbstract.php:217-230` |

**关键代码片段**（`lib/BridgeAbstract.php:215-248`）：
```php
if (isset($this->inputs[$context][$name])) {
    // 用户已提交值 → 跳过回退
    continue;
}

switch ($parameter['type'] ?? 'text') {
    case 'checkbox':
        // checkbox 未勾选 = false
        $this->inputs[$context][$name]['value'] = false;
        break;
    case 'list':
        if (!isset($parameter['defaultValue'])) {
            // 无 defaultValue → 取 values 的第一个
            $parameter['defaultValue'] = array_values($parameter['values'])[0] ?? '';
        }
        // fall-through
    default:
        if (isset($parameter['defaultValue'])) {
            $value = $parameter['defaultValue'];
            $this->inputs[$context][$name]['value'] = $value;
        }
}
```

#### 9.3.2 global 参数的特殊复制回退

**位置**：`lib/BridgeAbstract.php:250-263`

`global` 上下文的参数会被自动复制到实际查询的上下文。如果用户只填写了某个上下文的参数，`global` 的默认值也会被同步应用：

```php
// 复制 global 参数到实际 queriedContext
foreach (static::PARAMETERS['global'] ?? [] as $name => $parameter) {
    if (!isset($this->inputs[$context][$name])) {
        if (isset($parameter['defaultValue'])) {
            $this->inputs[$context][$name]['value'] = $parameter['defaultValue'];
        } elseif (($parameter['type'] ?? 'text') === 'checkbox') {
            $this->inputs[$context][$name]['value'] = false;
        }
    }
}
```

### 9.4 桥接内部级别的语言回退（如 NHKWorldJapanShowBridge）

全项目唯一实现了"语言翻译回退"的代码位于 `bridges/NHKWorldJapanShowBridge.php:303-315`（见 7.2.2 节详述），机制如下：

```
请求 language = 'zh' (中文)
    ↓
查找 $labels['length']['zh']
    ↓ 存在
返回 '时长:'
    ↓ 不存在（比如请求了未支持的小语种）
查找 $labels['length']['en']
    ↓ 存在
返回 'Length:'
    ↓ 不存在（理论上不会出现，en 是最完整的）
返回 ''
```

### 9.5 非语言类的通用 fallback 机制

除了上述语言/配置相关的回退，项目中还有几处通用的 fallback 与"回退到默认"相关：

#### 9.5.1 桥接元数据 fallback

**位置**：`lib/BridgeAbstract.php:62-72`
```php
public function getName()
{
    // NAME 常量未定义 → 回退到类名（去掉 "Bridge" 后缀）
    return static::NAME ?? $this->getShortName();
}

public function getURI()
{
    // URI 常量未定义 → 回退到 GitHub 项目地址
    return static::URI ?? 'https://github.com/RSS-Bridge/rss-bridge/';
}
```

#### 9.5.2 Feed 项字段 fallback

**位置**：`formats/HtmlFormat.php:39`、`formats/AtomFormat.php:101-106`、`formats/MrssFormat.php:121-127`
```php
// HtmlFormat: 标题为空 → 回退到 '(no title)'
'title' => $item->getTitle() ?? '(no title)',

// AtomFormat/MrssFormat:
// - item URI 为空 → 回退到 Feed 级 URI
// - item ID 为空 → 回退到 title + content 的哈希
```

#### 9.5.3 缩略图 fallback

**位置**：`lib/html.php:595`
```php
$fallbackUri = $thumbnailJpegBaseUri . '/maxresdefault.jpg';
// YouTube 视频缩略图：尝试多张不同分辨率的图，失败回退到 maxresdefault.jpg
```

### 9.6 回退机制总览图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       配置/参数回退总览                               │
├───────────────────────────┬─────────────────────────────────────────┤
│ CONFIGURATION (管理员端)   │ PARAMETERS (用户端)                      │
├───────────────────────────┼─────────────────────────────────────────┤
│ 1. config.ini.php 中定义    │ 1. 用户表单提交值                         │
│                           │                                         │
│ 2. required=true 抛异常    │ 2. checkbox → false                     │
│                           │                                         │
│ 3. defaultValue           │ 3. list → defaultValue / values[0]      │
│                           │                                         │
│ 4. 隐式 null (getOption)   │ 4. text/number → defaultValue / null    │
│                           │                                         │
│                           │ 5. global 参数自动复制同步到当前上下文       │
└───────────────────────────┴─────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    语言级回退（仅 NHKWorldJapanShowBridge）           │
├─────────────────────────────────────────────────────────────────────┤
│ 1. $labels[$key][$requestedLanguage] 命中                             │
│                                                                     │
│ 2. 回退 $labels[$key]['en']                                          │
│                                                                     │
│ 3. 回退空字符串 ''                                                    │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      通用元数据 fallback                              │
├─────────────────────────────────────────────────────────────────────┤
│ NAME 未定义     → 类名短名称                                           │
│ URI  未定义     → GitHub 项目地址                                       │
│ Feed 标题为空   → '(no title)'                                        │
│ Feed item URI  → Feed 级 URI                                          │
│ YouTube 缩略图  → maxresdefault.jpg                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十、扩展：若要实现真正的 i18n

当前项目无多语言能力。若需接入 i18n，需在以下位置做改造：

1. **桥接元数据层**：将 `NAME`、`DESCRIPTION`、`PARAMETERS[name/title/exampleValue]` 改为翻译键或支持多语言数组
2. **前端渲染层**：`FrontpageAction::render()` 中引入 `__()` 翻译函数包裹所有输出文本
3. **错误消息层**：`Configuration::throwConfigError()`、异常消息、`DisplayAction` 中的错误提示统一走翻译
4. **模板层**：`templates/*.html.php` 中的硬编码文案提取为翻译调用
5. **Admin 配置**：增加 `[system] default_locale` 配置项，按请求参数切换语言
