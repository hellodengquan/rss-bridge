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

## 十、Admin 后台多用户并发编辑同一 Bridge 配置的冲突处理

### 10.1 核心结论：不存在并发冲突处理机制，因为项目根本没有 Web 后台

经过全量代码扫描，RSS-Bridge **完全没有 Web 管理后台，也没有任何配置写入 API**。所谓"admin 后台"只是通过手动编辑 `config.ini.php` 文件来完成，因此不存在"多用户同时通过 Web 后台编辑同一 bridge 配置"的场景。

**关键证据**：
- 全局搜索无任何 `saveConfig`、`writeConfig`、`file_put_contents` 写配置文件的代码
- `Configuration::setConfig()`（`lib/Configuration.php:169-172`）仅为内存内赋值，不持久化到磁盘
- `actions/` 目录下 7 个 Action（Frontpage/Display/List/Findfeed/Detect/Connectivity/Health）均为只读操作，无任何"保存配置"的接口
- 无 `flock()`、`mutex`、`semaphore` 等任何并发锁原语的使用

### 10.2 配置的唯一写入方式：管理员手动编辑文件

管理员只能通过以下方式修改配置，完全绕过了 PHP 应用层：

| 修改方式 | 操作方式 | 并发风险 |
|---------|---------|---------|
| 修改 `config.ini.php` | SSH / SFTP 编辑文件 | 依赖操作系统文件锁和编辑器本身的冲突检测 |
| 设置环境变量 `RSSBRIDGE_*` | Docker Compose / `.env` / shell export | 由部署工具控制 |
| 创建/删除 `DEBUG` 文件 | `touch DEBUG` / `rm DEBUG` | 触发 `env=dev` 和 `cache=array`（见 8.3 触发方式 4）|
| 创建/编辑 `whitelist.txt` | 写入桥接白名单 | 每行一个桥接类名 |

### 10.3 Configuration 类的写入能力边界

**`setConfig()` 仅为请求级内存写入**（`lib/Configuration.php:169-172`）：
```php
public static function setConfig(string $section, string $key, $value): void
{
    self::$config[strtolower($section)][strtolower($key)] = $value;
}
```
- `self::$config` 是 `private static` 静态变量，生命周期 = PHP 请求生命周期
- 每次请求结束后内存自动释放，**不会持久化**
- 该方法仅在 `loadConfiguration()` 内部调用，用于从 ini 文件和环境变量加载配置

**`loadConfiguration()` 全流程中没有任何磁盘写入**（`lib/Configuration.php:18-156`），所有操作都是读取：
1. `parse_ini_file(config.default.ini.php)` → 只读
2. `parse_ini_file(config.ini.php)` → 只读（如果存在）
3. `file_get_contents(DEBUG)` → 只读（如果存在）
4. `file_get_contents(whitelist.txt)` → 只读（如果存在）
5. `getenv()` → 只读环境变量

### 10.4 若真的发生并发编辑：OS 级别的冲突

如果两名管理员同时通过 `vim` / `nano` 编辑同一台服务器上的 `config.ini.php`：

```
管理员 A: vim config.ini.php  → 修改 [TelegramBridge] max_pages = 5 → :wq
管理员 B: vim config.ini.php  → 修改 [TelegramBridge] max_pages = 10 → :wq
                                                              ↓
                                        后保存者覆盖先保存者的修改（Last Write Wins）
```

此时的冲突处理完全依赖：
1. **Vim/Nano 等编辑器自身的 swap file 检测**：编辑器会检测到文件已被修改并警告
2. **操作系统文件系统**：无内置冲突合并，纯文件级别覆盖
3. **PHP 端无感知**：下次请求时 `loadConfiguration()` 重新 `parse_ini_file`，读取到最终写入的版本

### 10.5 桥接级 CONFIGURATION 的"热更新"时序

虽然没有 Web 写入，但修改 `config.ini.php` 对桥接 CONFIGURATION 的生效路径是确定的：

```
T0: 管理员 A 编辑 config.ini.php，写入 [TelegramBridge] max_pages = 5
T1: 请求 1 到达 index.php
    → lib/config.php: parse_ini_file(config.ini.php) → 读到 max_pages = 5
    → Configuration::loadConfiguration() → setConfig('telegrambridge', 'max_pages', 5)
    → DisplayAction: $bridge->loadConfiguration()
        → Configuration::getConfig('TelegramBridge', 'max_pages') = 5
        → 生效
T2: 管理员 B 编辑 config.ini.php，覆盖写入 max_pages = 10
T3: 请求 2 到达
    → 重新 parse_ini_file → 读到 max_pages = 10
    → 对请求 2 生效。请求 1 的内存值仍为 5（但请求 1 已结束，无影响）
```

**关键点**：
- 无缓存，无竞态。每个请求独立从磁盘读取最新 ini 文件
- `parse_ini_file` 是原子文件读取操作，读取到的要么是旧完整内容要么是新完整内容（取决于 OS 文件系统的写入原子性）
- 如果管理员在 `parse_ini_file` 执行瞬间写入了一半文件 → PHP 会解析失败并在 `lib/Configuration.php:24-26` 抛异常 `Error parsing ini config`

### 10.6 项目中唯一涉及并发写入的地方：FileCache

全项目中唯一需要考虑并发写入安全的是 `FileCache`，但它**完全没有加锁**：

**`caches/FileCache.php:48-69`**
```php
public function set($key, $value, ?int $ttl = null): void
{
    $item = [
        'key'           => $key,
        'expiration'    => $ttl === null ? 0 : time() + $ttl,
        'value'         => $value,
    ];
    $cacheFile = $this->createCacheFile($key);
    $bytes = file_put_contents($cacheFile, serialize($item));
    // 无 flock(LOCK_EX) 保护！
}
```

- 未使用 `flock($fp, LOCK_EX)` 进行独占写锁
- 并发请求写入同一 cache key 时可能产生部分写入的损坏文件
- 但 FileCache 在 `get()` 时有损坏容错（`caches/FileCache.php:35-38`）：
  ```php
  if ($item === false) {
      $this->logger->warning(sprintf('Failed to unserialize: %s', $cacheFile));
      $this->delete($key);
      return $default;
  }
  ```
  → 损坏的缓存文件会被直接删除并当作缓存 miss 处理，不会导致错误

---

## 十一、i18n 资源贡献流程（社区翻译 PR 合入与文件结构）

### 11.1 核心结论：项目没有独立的"语言资源"或翻译系统

RSS-Bridge 不存在 `.po` / `.mo` / `.json` / `.xliff` 等任何语言包文件，也没有专门的 `lang/` 或 `locales/` 目录。所有面向用户的英文文案**直接硬编码在源代码中**，因此不存在传统意义上的"i18n 资源贡献流程"。

### 11.2 项目目录结构：与文案/翻译相关的文件位置

所有硬编码文案的分布（即需要"贡献翻译"时会修改的文件）：

```
291-rss-bridge/
├── bridges/                          # ⭐ 每个桥接是独立文件，含大量英文文案
│   ├── YoutubeBridge.php            #   const NAME/DESCRIPTION
│   ├── WikipediaBridge.php          #   PARAMETERS[name/title/exampleValue]
│   ├── NHKWorldJapanShowBridge.php  #   唯一的多语言样例：protected static $labels[...]
│   └── ... (400+ 桥接文件)
│
├── actions/
│   ├── FrontpageAction.php          # ⭐ 'Disable proxy (%s)' / 'Cache timeout in seconds'
│   ├── DisplayAction.php            # ⭐ 'Missing bridge name parameter' / 'Bridge not found'
│   ├── DetectAction.php             #   'You must specify a url'
│   ├── BasicAuthMiddleware.php      #   'Please authenticate...'
│   ├── TokenAuthenticationMiddleware.php  # 'Missing token' / 'Invalid token'
│   ├── MaintenanceMiddleware.php    #   '503 Service Unavailable'
│   └── SecurityMiddleware.php       #   'Query parameter "..." is not a string.'
│
├── templates/
│   ├── frontpage.html.php           # ⭐ 首页所有文案：'Email:' / 'Find feed by URL'
│   ├── base.html.php                #   <html lang="en"> 硬编码
│   ├── error.html.php               #   通用错误页结构
│   ├── exception.html.php           # ⭐ 大量硬编码错误解释文案
│   ├── html-format.html.php         #   '← back to rss-bridge' / 'Donate to maintainer'
│   ├── bridge-error.html.php        #   'Find similar bugs' / 'Create GitHub Issue'
│   └── token.html.php
│
├── lib/
│   ├── Configuration.php            # ⭐ 'Is not a valid email address' / 'Must be dev or prod'
│   ├── ParameterValidator.php       #   'Parameter is invalid!' / 'Parameter is not registered!'
│   └── utils.php                    #   throwClientException / throwServerException 辅助函数
│
├── docs/                            # ⭐ 项目文档（全英文 Markdown，独立于 PHP 代码）
│   └── ...
│
└── .github/
    └── CONTRIBUTING.md              # 贡献指南（仅引用文档链接）
```

### 11.3 桥接 PARAMETERS 文案的"贡献"流程 = 普通代码 PR

如果社区贡献者需要修改某桥接的 `name` / `title` / `exampleValue`（相当于修改"翻译"），流程与修复 bug 完全相同：

**文件：`.github/CONTRIBUTING.md` + `docs/04_For_Developers/02_Pull_Request_policy.md`**

#### 标准 PR 流程：

```
贡献者 fork 仓库
    ↓
git checkout -b fix/youtube-typo
    ↓
编辑 bridges/YoutubeBridge.php：
    const PARAMETERS = [
        'By username' => [
            'u' => [
-                'name' => 'username',
+                'name' => 'Username or handle',
-                'exampleValue' => 'LinusTechTips',
+                'exampleValue' => '@LinusTechTips',
```

#### Commit 命名规范（`docs/04_For_Developers/02_Pull_Request_policy.md:15-17`）：

| 修改对象 | commit message 格式 | 示例 |
|---------|-------------------|------|
| 桥接文件 | `[BridgeName] Feature` | `[YoutubeBridge] Fix typo in parameter name` |
| 其他文件 | `[FileName] Feature` | `[FrontpageAction.php] Add multilingual support` |
| 跨多文件 | `category: feature` | `bridges: Fix various typos in exampleValue` |

#### CI 校验（`.github/workflows/`）：

```
提交 PR
    ↓
┌─────────────────────────────────────┐
│ GitHub Actions CI                   │
│  ├─ tests.yml → phpunit             │  单元测试通过
│  ├─ lint.yml → phpcs + phpcompatibility │ 代码风格合规
│  ├─ dockerbuild.yml                 │  Docker 镜像可构建
│  └─ prhtmlgenerator.yml             │  自动生成 PR 测试页
└─────────────────────────────────────┘
    ↓
维护者 Code Review
    ↓
合并（Squash and merge）
```

### 11.4 真正的"多语言贡献"唯一案例：NHKWorldJapanShowBridge

全项目唯一实现多语言标签的桥接，贡献者添加新语言翻译的流程：

**文件位置**：`bridges/NHKWorldJapanShowBridge.php:64-175`

```php
protected static $labels = [
    'length' => [
        'en' => 'Length:',
        'zh' => '时长:',
        // ↓↓↓ 贡献者添加新语言 ↓↓↓
        'ja' => '長さ:',
    ],
    'broadcast' => [
        'en' => 'Broadcast:',
        'zh' => '播出:',
        'ja' => '放送:',
    ],
    // ... 同样为每个 label key 添加 ja 翻译
];
```

同时如需 RTL（从右到左）语言支持，添加语言代码到：
```php
protected static $rtlLanguages = [
    'ar','fa','ur'
    // 例如新增希伯来语: ,'he'
];
```

### 11.5 CI 中对桥接文案的自动化校验

**测试文件**：`tests/BridgeImplementationTest.php`

```php
// 校验 PARAMETERS 中 defaultValue 的规范性（非空字符串等）
if (isset($options['defaultValue'])) {
    if (is_string($options['defaultValue'])) {
        $this->assertNotEquals('', $options['defaultValue'], $field . ': empty defaultValue');
    }
}
```

没有对 `name` / `title` 做 i18n 相关校验（因为没有多语言机制）。

### 11.6 文档系统的贡献流程（docs/）

项目文档位于 `docs/` 目录，使用 [Daux.io](https://daux.io/) 静态站点生成器。所有文档为纯英文 Markdown，也无多语言版本。

**贡献方式**：直接编辑 `docs/**/*.md`，提交 PR，CI 中 `documentation.yml` workflow 会自动构建并发布到 GitHub Pages。

---

## 十二、桥接抛出的错误消息国际化路径与缺翻译回退机制

### 12.1 核心结论：错误消息完全无国际化，全部硬编码英文

全项目所有错误消息（exception message、error page text、validation error）均为**英文硬编码**，没有任何翻译抽象层。"缺翻译回退"的概念在本项目中不成立——因为根本就没有"翻译"这一层。

### 12.2 错误消息的抛出与捕获全链路

```
桥接 collectData() / 参数校验 / 中间件
    ↓ 抛出
┌─────────────────────────────────────────────────────┐
│ 四种异常类型                                          │
│  1. ClientException        → 400 Bad Request (用户错) │
│  2. RateLimitException     → 429 Too Many Requests   │
│  3. HttpException          → 继承自 \Exception         │
│     └─ CloudFlareException → 特殊子类型                │
│  4. \Exception (通用)      → 500 Internal Error       │
└─────────────────────────────────────────────────────┘
    ↓
ExceptionMiddleware::__invoke() 捕获 (middlewares/ExceptionMiddleware.php:14-23)
    ↓
渲染 templates/exception.html.php → 所有文案硬编码英文
```

### 12.3 四种异常类型的使用场景

| 异常类 | 抛出函数 | HTTP 状态码 | 日志级别 | 使用场景 |
|-------|---------|------------|---------|---------|
| `ClientException` | `throwClientException()` (lib/utils.php:254-257) | 400 | DEBUG | 用户输入错误（参数缺失、格式不对） |
| `RateLimitException` | `throwRateLimitException()` (lib/utils.php:264-267) | 429 | DEBUG | 目标站点限流 |
| `HttpException` / `CloudFlareException` | `HttpException::fromResponse()` (lib/http.php:23-35) | 源站返回码 | ERROR | 远程抓取失败（被 CloudFlare 拦截、404 等） |
| `\Exception` (通用) | `throwServerException()` / `throw new \Exception()` (lib/utils.php:259-262) | 500 | ERROR | 桥接内部逻辑错误（DOM 结构变更、解析失败） |

**典型示例**（来自 `bridges/YoutubeBridge.php:201`）：
```php
// 用户没填必填参数 → ClientException (英文硬编码)
throwClientException("You must either specify either:\n - YouTube username (?u=...)\n - Channel id (?c=...)\n - Playlist id (?p=...)\n - Search (?s=...)");
```

**典型示例**（来自 `bridges/WikipediaBridge.php:116`）：
```php
// 所选语言没有对应的解析函数 → ServerException (英文硬编码，含动态变量)
throwServerException('A function to get the contents for your language is missing (\'' . $function . '\')!');
```

### 12.4 DisplayAction 中错误的分类处理

**文件**：`actions/DisplayAction.php:91-123`

DisplayAction 在 `try/catch` 中执行桥接逻辑，对不同异常做差异化处理：

```php
try {
    $bridge->loadConfiguration();
    $bridge->setInput($input);
    $bridge->collectData();
} catch (\Throwable $e) {
    if ($e instanceof ClientException) {
        // 用户错 → DEBUG 级别日志（低噪声）
    } elseif ($e instanceof RateLimitException) {
        // 限流 → 立即返回 429 错误页
        return new Response(render(...exception.html.php...), 429);
    } elseif ($e instanceof HttpException) {
        if (in_array($e->getCode(), [429, 503])) {
            // 源站限流/不可用 → 立即返回对应状态码错误页
            return new Response(render(...), $e->getCode());
        }
        // 其他 Http 错误（404/403 等）→ 静默，走通用错误处理
    } else {
        // 其他所有异常 → ERROR 级别日志 + 记录堆栈
    }

    // 根据配置决定最终输出方式（[error] output）
    switch (Configuration::getConfig('error', 'output')) {
        case 'feed':  // 将错误包装为 Feed Item（默认）
        case 'http':  // 返回 HTTP 错误页
        case 'none':  // 静默输出空 Feed
    }
}
```

### 12.5 错误页面模板中的硬编码英文文案

**文件**：`templates/exception.html.php`

这个模板是错误消息最终呈现的地方，所有文案均为英文硬编码，按 HTTP 状态码分支显示：

| 状态码/类型 | 硬编码英文文案 | 代码行号 |
|-----------|-------------|---------|
| CloudFlare | `'The website is protected by CloudFlare'` / `'RSS-Bridge tried to fetch a website...'` | L9-16 |
| 400 | `'400 Bad Request'` / `'This is usually caused by...'` | L19-24 |
| 403 | `'403 Forbidden'` / `'...refuses to authorize it.'` | L26-32 |
| 404 | `'404 Page Not Found'` / `'...doesn\'t exists.'` | L34-40 |
| 429 | `'429 Too Many Requests'` / `'...told us to try again later.'` | L42-48 |
| 503 | `'503 Service Unavailable'` / `'Common causes are...'` | L50-56 |
| 其他 code | MDN 链接 `'https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/...'` | L68-71 |
| code=10 (空 Feed) | `'The rss feed is completely empty'` | L75-80 |
| code=11 (XML 解析失败) | `'There is something wrong with the rss feed'` | L83-88 |
| 所有异常通用 | `'Details'` / `'Type:'` / `'Code:'` / `'Message:'` / `'Trace'` / `'Context'` / `'Go back'` | L91-146 |

**关键代码**（`templates/exception.html.php:8-17`，CloudFlare 分支）：
```php
<?php if ($e instanceof CloudFlareException): ?>
    <h2>The website is protected by CloudFlare</h2>
    <p>
        RSS-Bridge tried to fetch a website.
        The fetching was blocked by CloudFlare.
        CloudFlare is anti-bot software.
        Its purpose is to block non-humans.
    </p>
<?php endif; ?>
```

### 12.6 参数校验错误消息

**文件**：`lib/ParameterValidator.php:49,54`

```php
if (is_null($input[$name]) && ...required...) {
    $errors[] = ['name' => $name, 'reason' => 'Parameter is invalid!'];
}
// ...
if (!$registered) {
    $errors[] = ['name' => $name, 'reason' => 'Parameter is not registered!'];
}
```

- 只有两种固定错误消息，均为英文硬编码
- 这些 errors 数组目前**未被实际渲染到前端**（仅在 `validateInput()` 返回，但调用方 `BridgeAbstract::setInput()` 没有使用）

### 12.7 错误输出的三种模式（[error] output 配置）

**文件**：`config.default.ini.php:132-140` + `actions/DisplayAction.php:107-123`

```ini
[error]
output = "feed"   ; feed | http | none
```

| output 模式 | 行为 | 用户看到的内容 |
|------------|------|--------------|
| `feed`（默认） | 将异常包装成一个 Feed Item，混入 Feed 输出 | 桥接 Feed 中出现一条特殊的错误条目，标题为 `'Bridge returned error %s! (%s)'`（英文硬编码，见 L152），内容为完整 exception.html.php 渲染结果 |
| `http` | 直接返回 HTTP 500 + exception.html.php | 完整错误页面（所有英文硬编码） |
| `none` | 吞掉错误，返回空 Feed | 用户看到空 Feed，无任何错误提示 |

### 12.8 "缺翻译回退"在本项目中的实际情况

由于没有翻译层，不存在"缺翻译"的场景。但存在以下几种等价的"回退"行为：

#### 回退 1：异常消息原样显示

**位置**：`templates/exception.html.php:102-104`
```php
<div class="error-message">
    <strong>Message:</strong> <?= e(sanitize_root($e->getMessage())) ?>
</div>
```
- 桥接抛出的英文异常消息会被 **直接原样输出**，没有任何翻译或转换
- 例如 `'The URL you provided is invalid!'` → 原封不动显示给用户

#### 回退 2：NHKWorldJapanShowBridge 的标签缺失

**位置**：`bridges/NHKWorldJapanShowBridge.php:303-315`
```php
protected function getLocaleString($string)
{
    $language = $this->getInput('language');
    if (isset(self::$labels[$string][$language])) {
        return self::$labels[$string][$language];   // ← 命中请求语言
    }
    if (isset(self::$labels[$string]['en'])) {
        return self::$labels[$string]['en'];        // ← 回退到英语
    }
    return '';                                        // ← 连英语都没有，回退空串
}
```
这是全项目唯一存在"语言回退链"的地方，三级回退：`请求语言 → 英语 → 空字符串`。

#### 回退 3：Feed 输出错误兜底

**位置**：`actions/DisplayAction.php:152`
```php
$title = sprintf('Bridge returned error %s! (%s)', $e->getCode(), $uniqueIdentifier);
```
当桥接抛异常且 `output=feed` 时，Feed 条目标题固定为此英文格式字符串，错误详情则复用 `exception.html.php` 模板。

### 12.9 错误消息全链路示意图

```
桥接代码 throwClientException('Invalid username!')
    ↓
DisplayAction catch → DEBUG 日志记录
    ↓
[error] output 配置判断
    ├─ feed → createFeedItemFromException()
    │          ↳ 标题: sprintf('Bridge returned error %s! (%s)', ...)  英文硬编码
    │          ↳ 内容: render(bridge-error.html.php)
    │                     ↳ render(exception.html.php)
    │                         ↳ 按状态码显示对应英文解释文案
    │                         ↳ 显示 $e->getMessage() 原样英文消息
    │
    ├─ http → render(exception.html.php)  → 完整英文错误页
    │
    └─ none → 静默空 Feed
```

---

## 十三、Admin 鉴权与 Session 管理代码路径

### 13.1 核心结论：无 Session，鉴权仅靠两种中间件一次性校验

RSS-Bridge **完全没有 Session 管理**，也没有任何登录/登出流程。鉴权仅通过两个中间件在每个请求开始时做一次性无状态校验：
- **HTTP Basic Auth**（用户名+密码，`BasicAuthMiddleware`）
- **Token 校验**（URL 参数携带 token，`TokenAuthenticationMiddleware`）

**关键证据**：
- 全局无 `session_start()` / `$_SESSION` / `session_name()` 等任何 Session 相关代码
- 无登录表单 / 登录接口 / 登出接口
- 无 Cookie 设置（`setcookie()`）代码
- 鉴权是**每个请求独立校验**的无状态模式

### 13.2 中间件栈与执行顺序

**文件**：`lib/RssBridge.php:25-38`

```php
$middlewares = [
    new BasicAuthMiddleware(),          // 第 1 层: HTTP Basic Auth
    new CacheMiddleware($this->container['cache']),
    new ExceptionMiddleware($this->container['logger']),
    new SecurityMiddleware(),
    new MaintenanceMiddleware(),
    new TokenAuthenticationMiddleware(), // 第 6 层: Token 校验
];
```

**执行顺序（洋葱模型）**：
```
请求进入
    ↓
BasicAuthMiddleware
    ↓
CacheMiddleware
    ↓
ExceptionMiddleware
    ↓
SecurityMiddleware
    ↓
MaintenanceMiddleware
    ↓
TokenAuthenticationMiddleware
    ↓
实际 Action 执行（Frontpage / Display 等）
```

> 注意：`array_reverse($middlewares)` 是装饰器模式的经典实现，实际执行顺序与数组定义顺序一致。

### 13.3 BasicAuthMiddleware 详细流程

**文件**：`middlewares/BasicAuthMiddleware.php`

```php
public function __invoke(Request $request, $next): Response
{
    // Step 1: 检查是否启用鉴权
    if (!Configuration::getConfig('authentication', 'enable')) {
        return $next($request);  // 未启用，直接放行
    }

    // Step 2: 密码为空配置错误（500 错误，无 i18n）
    if (Configuration::getConfig('authentication', 'password') === '') {
        return new Response('The authentication password cannot be the empty string', 500);
    }

    // Step 3: 从 PHP 全局变量读取 Basic Auth 头
    $user = $request->server('PHP_AUTH_USER');
    $password = $request->server('PHP_AUTH_PW');

    // Step 4: 未携带凭证 → 返回 401 触发浏览器弹窗
    if ($user === null || $password === null) {
        $html = render(__DIR__ . '/../templates/error.html.php', [
            'message' => 'Please authenticate in order to access this instance!',
        ]);
        return new Response($html, 401, ['WWW-Authenticate' => 'Basic realm="RSS-Bridge"']);
    }

    // Step 5: 使用 hash_equals() 安全比对密码（防时序攻击）
    if (
        (Configuration::getConfig('authentication', 'username') !== $user)
        || (!hash_equals(Configuration::getConfig('authentication', 'password'), $password))
    ) {
        // 用户名或密码错误 → 再次 401
        $html = render(__DIR__ . '/../templates/error.html.php', [
            'message' => 'Please authenticate in order to access this instance!',
        ]);
        return new Response($html, 401, ['WWW-Authenticate' => 'Basic realm="RSS-Bridge"']);
    }

    // Step 6: 鉴权通过，放行
    return $next($request);
}
```

**关键设计点**：
- 使用 `hash_equals()` 而非 `===`，防止时序攻击（timing attack）
- 直接读取 `$_SERVER['PHP_AUTH_USER']` / `$_SERVER['PHP_AUTH_PW']`，依赖 PHP/Apache 的 Basic Auth 解析
- 失败时返回 `WWW-Authenticate` 头，触发浏览器原生登录弹窗
- 用户名/密码**明文存储**在 `config.ini.php` 中，无哈希加密

**配置对应项**（`config.default.ini.php:120-129`）：
```ini
[authentication]
enable = false
username = "admin"
password = ""
```

### 13.4 TokenAuthenticationMiddleware 详细流程

**文件**：`middlewares/TokenAuthenticationMiddleware.php`

```php
public function __invoke(Request $request, $next): Response
{
    // Step 1: 检查是否启用 token 鉴权
    if (! Configuration::getConfig('authentication', 'token')) {
        return $next($request);  // 未配置 token，直接放行
    }

    // Step 2: 从 GET 参数读取 token
    $token = $request->get('token');

    // Step 3: 无 token → 显示 token 输入表单
    if (! $token) {
        return new Response(render(__DIR__ . '/../templates/token.html.php', [
            'message'   => 'Missing token',
            'token'     => '',
        ]), 401);
    }

    // Step 4: 安全比对 token
    if (! hash_equals(Configuration::getConfig('authentication', 'token'), $token)) {
        return new Response(render(__DIR__ . '/../templates/token.html.php', [
            'message'   => 'Invalid token',
            'token'     => $token,
        ]), 401);
    }

    // Step 5: 鉴权通过，将 token 写入请求属性（方便后续使用）
    $request = $request->withAttribute('token', $token);

    return $next($request);
}
```

**Token 表单模板**（`templates/token.html.php`）：
```html
<h1>Authentication with token required</h1>
<p><?= e($message) ?></p>
<form action="" method="get" autocomplete="off">
    <label for="token">Token:</label>
    <input type="text" name="token" id="token" placeholder="token" value="<?= e($token) ?>">
    <input type="submit" value="OK">
</form>
```

**关键设计点**：
- token 以 GET 参数形式传递（`?token=xxx`），方便在 Feed URL 中直接携带
- 提交表单使用 `method="get"`，token 会出现在 URL 中
- 同样使用 `hash_equals()` 防时序攻击
- 表单 `autocomplete="off"`，防止浏览器保存敏感 token

**配置对应项**（`config.default.ini.php:130`）：
```ini
[authentication]
token = ""
```

### 13.5 鉴权校验时序（两种方式同时启用）

```
用户请求: https://rss-bridge.example.com/?action=display&bridge=YoutubeBridge&u=test&token=mysecrettoken
    ↓
index.php → Request::fromGlobals()
    ↓
RssBridge::main() 构造中间件链
    ↓
BasicAuthMiddleware 检查:
  [authentication][enable] = true  ✓
  $_SERVER['PHP_AUTH_USER'] = 'admin'  ✓
  hash_equals 密码比对通过  ✓
    ↓
CacheMiddleware 读缓存 miss  ✓
    ↓
ExceptionMiddleware 包装 try/catch  ✓
    ↓
SecurityMiddleware 校验 GET 参数都是字符串  ✓
    ↓
MaintenanceMiddleware 检查维护模式  ✓
    ↓
TokenAuthenticationMiddleware 检查:
  [authentication][token] = 'mysecrettoken'  ✓
  $_GET['token'] = 'mysecrettoken'  ✓
  hash_equals 比对通过  ✓
  $request = $request->withAttribute('token', 'mysecrettoken')
    ↓
DisplayAction 执行，生成 Feed
```

### 13.6 其他安全相关中间件

#### SecurityMiddleware

**文件**：`middlewares/SecurityMiddleware.php`
```php
// 确保所有 GET 参数都是字符串（防数组注入攻击）
foreach ($request->toArray() as $key => $value) {
    if (!is_string($value)) {
        return new Response(render(__DIR__ . '/../templates/error.html.php', [
            'message' => "Query parameter \"$key\" is not a string.",
        ]), 400);
    }
}
```

#### MaintenanceMiddleware

**文件**：`middlewares/MaintenanceMiddleware.php`
```php
if (!Configuration::getConfig('system', 'enable_maintenance_mode')) {
    return $next($request);
}
return new Response(render(__DIR__ . '/../templates/error.html.php', [
    'title' => '503 Service Unavailable',
    'message' => 'RSS-Bridge is down for maintenance.',
]), 503);
```

### 13.7 鉴权方式对比

| 特性 | HTTP Basic Auth | Token 鉴权 |
|-----|----------------|-----------|
| 配置开关 | `[authentication] enable = true` | `[authentication] token = "xxx"` |
| 凭证传递 | `Authorization: Basic <base64>` 头 | GET 参数 `?token=xxx` |
| 浏览器原生支持 | ✅ 自动弹窗 | ❌ 需自定义表单 |
| 适合场景 | 人工访问浏览器 | 程序/Feed 阅读器订阅 |
| 凭证记忆 | 浏览器会缓存直到关闭 | 需每次在 URL 中携带 |
| 防时序攻击 | ✅ `hash_equals()` | ✅ `hash_equals()` |
| 安全级别 | 中（HTTPS 下安全） | 中（URL 可能被日志记录） |
| 可同时启用 | ✅ | ✅ |

> 注意：两种鉴权是**逻辑与**关系，同时启用时必须都通过才能访问。

---

## 十四、Bridge 配置文件的序列化方式与 Schema 校验

### 14.1 核心结论：仅使用 PHP 原生 INI 格式，无 Schema 校验框架

RSS-Bridge 的配置系统**完全基于 PHP 原生 `parse_ini_file()` 函数**，使用 INI 格式作为序列化方式。没有使用 JSON Schema、XML Schema、YAML 或任何配置校验库。Schema 校验通过 `Configuration::loadConfiguration()` 中的硬编码 `if` 语句逐字段完成。

### 14.2 配置文件格式与序列化

#### 14.2.1 INI 文件格式规范

**文件**：`config.default.ini.php`（头几行）
```ini
<?php
; This is a comment. Lines starting with semicolon are ignored.
; Exit if called directly
if(!defined('RSSBRIDGE')) {
    die('No direct access allowed!');
}
?>
; <?php exit; ?> DO NOT REMOVE THIS LINE

[system]
; Defines the environment: dev or prod.
; "dev" enables debugging and disables caching for most actions.
; "prod" is the standard mode.
; Default: "prod"
env = "prod"
```

**关键设计**：
- 文件是 `.ini.php` 扩展名，开头嵌入 PHP 代码防止直接访问
- 使用标准 INI 格式：`[section]` 段 + `key = value` 键值对
- 注释以 `;` 开头
- 字符串值用双引号包裹

#### 14.2.2 解析方式

**文件**：`lib/Configuration.php:23` + `lib/config.php:7`
```php
$config = parse_ini_file(
    __DIR__ . '/../config.default.ini.php',
    true,                  // process_sections = true → 多维数组
    INI_SCANNER_TYPED      // 自动类型转换（数字→int，"true"/"false"→bool）
);
```

**INI_SCANNER_TYPED 模式下的自动类型转换**：

| INI 中写法 | 解析后 PHP 类型 |
|-----------|----------------|
| `env = "prod"` | string `"prod"` |
| `enable = true` | bool `true` |
| `timeout = 3600` | int `3600` |
| `limit = 10.5` | float `10.5` |
| `enabled_bridges[] = "YoutubeBridge"` | array `["YoutubeBridge", ...]` |

**反序列化/加载完整流程**：
```
Configuration::loadConfiguration($customConfig, $env)
    ↓
1. parse_ini_file(config.default.ini.php, true, INI_SCANNER_TYPED)
    → 得到 $config 多维数组
    ↓
2. 遍历所有 section/key → setConfig() 存入 self::$config
    ↓
3. parse_ini_file(config.ini.php)（如果存在）
    → 相同键覆盖默认值
    ↓
4. file_get_contents(DEBUG) + file_get_contents(whitelist.txt)
    → 特殊配置项
    ↓
5. 遍历 $_ENV / getenv()，匹配 RSSBRIDGE_* 前缀
    → 再次覆盖（最高优先级）
    ↓
6. 硬编码 schema 校验（见 14.3）
```

#### 14.2.3 序列化反序列化边界

配置系统**只有反序列化（读），没有序列化（写）**。`Configuration::setConfig()` 只写内存不写磁盘。

其他序列化技术在项目中的使用：

| 技术 | 使用场景 | 文件位置 |
|-----|---------|---------|
| `parse_ini_file()` | 配置文件加载 | `lib/Configuration.php:23,32` |
| `serialize()` / `unserialize()` | FileCache / SQLiteCache 存储完整 PHP 对象（Response、Feed 数据） | `caches/FileCache.php:34,60`、`caches/SQLiteCache.php:67,86` |
| `json_encode()` / `json_decode()` | 缓存 Key 生成、error_reporting 计数、Twitter API 调用 | `lib/utils.php:6-21`、`middlewares/CacheMiddleware.php:24` |
| `Json::encode/decode()` | 缓存数据 JSON 序列化 | `lib/utils.php:6-21` |

### 14.3 Schema 校验：硬编码逐字段校验

**文件**：`lib/Configuration.php:84-156`

所有校验都是硬编码的 `if` 语句，没有使用任何校验框架。每个可配置项都有对应的校验逻辑。

**完整校验清单**：

| 配置项 | 校验规则 | 代码行号 | 失败处理 |
|-------|---------|---------|---------|
| `system.env` | 必须是 `'dev'` 或 `'prod'` | L84-86 | `throwConfigError()` → HTTP 500 + `exit(1)` |
| `system.enabled_bridges` | 必须是 array 类型 | L88-90 | 同上 |
| `system.timezone` | 必须是字符串 + 必须在 `timezone_identifiers_list()` 返回值中 | L92-97 | 同上 |
| `proxy.url` | 必须是字符串 | L99-101 | 同上 |
| `proxy.by_bridge` | 必须是 bool 类型 | L103-105 | 同上 |
| `proxy.name` | 必须是字符串 | L107-110 | 同上 |
| `cache.type` | 必须是字符串 | L112-114 | 同上 |
| `cache.custom_timeout` | 必须是 bool 类型 | L116-118 | 同上 |
| `authentication.enable` | 必须是 bool 类型 | L120-122 | 同上 |
| `authentication.username` | 必须是字符串 | L124-126 | 同上 |
| `authentication.password` | 必须是字符串 | L128-130 | 同上 |
| `admin.email` | 非空时必须通过 `FILTER_VALIDATE_EMAIL` | L132-137 | 同上 |
| `admin.donations` | 必须是 bool 类型 | L139-141 | 同上 |
| `error.output` | 必须是字符串 + 必须是 `'feed'`/`'http'`/`'none'` | L143-148 | 同上 |
| `error.report_limit` | 必须是数字 + 必须 >= 1 | L150-155 | 同上 |

**校验失败处理**（`lib/Configuration.php:192-197`）：
```php
private static function throwConfigError($section, $key, $message = '')
{
    http_response_code(500);
    print ("Config [$section] => [$key] is invalid. $message");
    exit(1);  // 直接终止整个程序
}
```
- 英文硬编码错误消息，无翻译
- 直接 `exit(1)` 终止，不走异常处理系统

### 14.4 环境变量到配置的映射与校验

**文件**：`lib/Configuration.php:55-82`

环境变量名格式：`RSSBRIDGE_<SECTION>_<KEY>`

```php
foreach ($env as $envName => $envValue) {
    $nameParts = explode('_', $envName);
    if ($nameParts[0] === 'RSSBRIDGE') {
        $header = $nameParts[1];                    // section
        $key = implode('_', array_slice($nameParts, 2));  // key（下划线重组）
        $key = strtolower($key);

        // 特殊处理：enabled_bridges 按逗号拆分为数组
        if ($key === 'enabled_bridges') {
            $envValue = explode(',', $envValue);
            $envValue = array_map('trim', $envValue);
        }

        // 字符串 "true"/"false" → bool 类型转换
        if ($envValue === 'true' || $envValue === 'false') {
            $envValue = filter_var($envValue, FILTER_VALIDATE_BOOLEAN);
        }

        self::setConfig($header, $key, $envValue);
    }
}
```

**示例映射**：
```
RSSBRIDGE_SYSTEM_ENV=dev                 → section=system, key=env, value='dev'
RSSBRIDGE_SYSTEM_ENABLED_BRIDGES=YoutubeBridge,TelegramBridge
                                        → section=system, key=enabled_bridges, value=['YoutubeBridge','TelegramBridge']
RSSBRIDGE_ADMIN_DONATIONS=true          → section=admin, key=donations, value=true (bool)
RSSBRIDGE_CACHE_CUSTOM_TIMEOUT=false    → section=cache, key=custom_timeout, value=false (bool)
```

> 注意：环境变量先被 `setConfig()` 写入内存，然后**一起**在 L84-156 做 schema 校验。校验发生在环境变量加载之后。

### 14.5 桥接级 CONFIGURATION 的校验机制

桥接自定义配置（`const CONFIGURATION`）的校验**不在 Configuration 类中**，而是在每个桥接首次加载时由 `BridgeAbstract::loadConfiguration()` 完成。

**文件**：`lib/BridgeAbstract.php:119-148`
```php
public function loadConfiguration()
{
    foreach (static::CONFIGURATION as $optionName => $optionValue) {
        $section = $this->getShortName();
        $configurationOption = Configuration::getConfig($section, $optionName);

        if ($configurationOption !== null) {
            $this->configuration[$optionName] = $configurationOption;
        } elseif (isset($optionValue['required']) && $optionValue['required'] === true) {
            // 校验: required=true 但配置缺失 → 抛异常
            throw new \Exception(sprintf('Missing configuration option: %s', $optionName));
        } elseif (isset($optionValue['defaultValue'])) {
            $this->configuration[$optionName] = $optionValue['defaultValue'];
        }
    }
}
```

**校验逻辑非常简单**：
- 只校验 `required` 字段是否存在
- **不校验数据类型**（字符串/数字/布尔等）
- 不校验值范围（最大/最小值）
- 不校验格式（URL、邮箱等）
- 缺失且非 required → 用 `defaultValue`，或留空

### 14.6 用户参数 PARAMETERS 的校验

**文件**：`lib/ParameterValidator.php:8-59`

用户表单提交的参数（PARAMETERS）在 `BridgeAbstract::setInput()` 中由 `ParameterValidator` 校验：

```php
public function validateInput(array &$input, array $parameters): array
{
    foreach ($input as $name => $value) {
        // Step 1: 检查参数是否已注册（防止未注册参数注入）
        if (!$registered) {
            $errors[] = ['name' => $name, 'reason' => 'Parameter is not registered!'];
            continue;
        }

        // Step 2: 按 type 做类型校验
        switch ($contextParameters[$name]['type']) {
            case 'number':
                $input[$name] = filter_var($value, FILTER_VALIDATE_INT);
                break;
            case 'checkbox':
                $input[$name] = filter_var($value, FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE);
                break;
            case 'list':
                // 检查值是否在 values 枚举列表中
                if (!in_array($filteredValue, $expectedValues)) {
                    return null;
                }
                break;
            case 'text':
                if (isset($pattern)) {
                    $input[$name] = filter_var($value, FILTER_VALIDATE_REGEXP, [
                        'options' => ['regexp' => '/^' . $pattern . '$/']
                    ]);
                }
                break;
        }

        // Step 3: required 字段校验
        if (is_null($input[$name]) && ...required...) {
            $errors[] = ['name' => $name, 'reason' => 'Parameter is invalid!'];
        }
    }
    return $errors;
}
```

**校验能力对比**：

| 校验能力 | Configuration（管理员配置） | PARAMETERS（用户参数） |
|---------|----------------------------|----------------------|
| required 必填 | ✅ 桥接级检查 | ✅ |
| 类型校验（int/bool/string） | ❌ 仅在系统级配置硬编码 | ✅ 基于 type 字段 |
| 正则 pattern 校验 | ❌ | ✅ text 类型 |
| 枚举值校验 | ❌ | ✅ list 类型 |
| 邮箱/URL 格式校验 | ✅ 仅 admin.email 特殊处理 | ❌ |
| 值范围校验（min/max） | ❌ | ❌ |

### 14.7 序列化与校验全链路图

```
管理员编辑 config.ini.php
    ↓ 写入磁盘
[system]
env = "prod"
enabled_bridges[] = YoutubeBridge
enabled_bridges[] = TelegramBridge

[TelegramBridge]
max_pages = 5
    ↓
用户请求到达
    ↓
lib/config.php:
    $config = parse_ini_file(config.default.ini.php, true, INI_SCANNER_TYPED)
    $customConfig = parse_ini_file(config.ini.php, true, INI_SCANNER_TYPED)
    Configuration::loadConfiguration($customConfig, getenv())
    ↓
Configuration::loadConfiguration():
    1. 加载 default.ini → setConfig() 存入内存
    2. 加载 custom.ini → 覆盖
    3. 加载 DEBUG / whitelist.txt → 覆盖
    4. 加载环境变量 RSSBRIDGE_* → 覆盖
    5. Schema 校验（15 个硬编码 if）
        → 全部通过 → 继续
        → 任一失败 → throwConfigError() → HTTP 500 exit
    ↓
DisplayAction:
    $bridge->loadConfiguration()
        → 读取 [TelegramBridge] max_pages = 5
        → 桥接级 required 校验
        → 成功存入 $this->configuration
    ↓
    $bridge->setInput($input)
        → ParameterValidator::validateInput()
        → 按 PARAMETERS 的 type/pattern/required 校验
        → 成功存入 $this->inputs
    ↓
    $bridge->collectData() → 生成 Feed
```

---

## 十五、CLI 命令的国际化处理及 Web 翻译资源复用

### 15.1 核心结论：CLI 模式复用 Web 全部代码路径，同样无国际化

RSS-Bridge 的 CLI 模式**完全复用 Web 模式的所有代码**，包括错误消息、模板渲染、异常处理等。因此 CLI 与 Web 一样，**没有任何国际化处理**，所有输出均为英文硬编码，不存在"复用 Web 翻译资源"的问题——因为 Web 端本身就没有翻译资源。

### 15.2 CLI 入口与代码路径

**文件**：`index.php:63-69`

```php
$argv = $argv ?? null;
if ($argv) {
    // CLI 模式: 解析命令行参数为 GET 参数
    parse_str(implode('&', array_slice($argv, 1)), $cliArgs);
    $request = Request::fromCli($cliArgs);
} else {
    // Web 模式: 从 $_GET/$_SERVER 读取
    $request = Request::fromGlobals();
}

// 后续流程完全相同
$rssBridge = new RssBridge($container);
$response = $rssBridge->main($request);
$response->send();
```

**命令行用法**（`docs/02_CLI/index.md`）：
```bash
php index.php action=display bridge=DansTonChat format=Json
php index.php action=list
php index.php action=display bridge=YoutubeBridge u=LinusTechTips format=Atom
```

### 15.3 Request 对象的两种构造方式

**文件**：`lib/http.php:210-224`

```php
// Web 模式
public static function fromGlobals(): self
{
    $self = new self();
    $self->get = $_GET;
    $self->server = $_SERVER;
    $self->attributes = [];
    return $self;
}

// CLI 模式
public static function fromCli(array $cliArgs): self
{
    $self = new self();
    $self->get = $cliArgs;
    // $self->server 未设置 → 始终为 null
    return $self;
}
```

**CLI 模式的限制**：
- `$self->server` 属性为空数组，所有 `$request->server()` 调用返回 `null`
- 依赖 `$_SERVER` 的功能在 CLI 下会失效

### 15.4 两种模式下鉴权中间件的行为差异

由于 CLI 模式下 `$request->server()` 返回 null，鉴权中间件行为不同：

#### BasicAuthMiddleware 在 CLI 下的行为

```php
$user = $request->server('PHP_AUTH_USER');     // CLI 下 → null
$password = $request->server('PHP_AUTH_PW');   // CLI 下 → null

if ($user === null || $password === null) {
    // 即使在 config.ini.php 中开启了 authentication.enable
    // CLI 下也会触发 401 错误！
    $html = render(__DIR__ . '/../templates/error.html.php', [
        'message' => 'Please authenticate in order to access this instance!',
    ]);
    return new Response($html, 401, ['WWW-Authenticate' => 'Basic realm="RSS-Bridge"']);
}
```

> **CLI 陷阱**：如果启用了 HTTP Basic Auth，CLI 调用会直接失败，因为无法传递 `PHP_AUTH_USER` 环境变量。需要先设置 `$_SERVER['PHP_AUTH_USER']` 和 `$_SERVER['PHP_AUTH_PW']` 全局变量。

#### TokenAuthenticationMiddleware 在 CLI 下的行为

Token 鉴权从 GET 参数读取，CLI 下**正常工作**：
```bash
php index.php action=display bridge=YoutubeBridge u=test format=Json token=mysecrettoken
```

### 15.5 Response::send() 在 CLI 下的行为

**文件**：`lib/http.php:379-388`

```php
public function send(): void
{
    http_response_code($this->code);       // CLI 下无效，但不报错
    foreach ($this->headers as $name => $values) {
        foreach ($values as $value) {
            header(sprintf('%s: %s', $name, $value));  // CLI 下会输出到 stderr 警告
        }
    }
    print $this->body;                       // CLI 下正常输出到 stdout
}
```

**CLI 下的输出问题**：
- `http_response_code()` 在 CLI 模式下无效（无 HTTP 协议）
- `header()` 会产生 PHP Warning：`Cannot modify header information - headers already sent`
- `print $this->body` 正常输出 Feed 内容到 stdout
- 如果输出的是 HTML 格式错误页，CLI 下会看到完整 HTML 标签

### 15.6 CLI 与 Web 代码复用对比

| 组件 | Web 模式 | CLI 模式 | 是否复用 |
|-----|---------|---------|---------|
| `RssBridge::main()` | ✅ | ✅ | ✅ 完全复用 |
| 中间件栈（6 个中间件） | ✅ | ✅ | ✅ 完全复用（行为有差异） |
| 所有 Action（Frontpage/Display 等） | ✅ | ✅ | ✅ 完全复用 |
| 错误消息文案 | 英文硬编码 | 英文硬编码 | ✅ 完全复用 |
| 异常模板 exception.html.php | ✅ | ✅ | ✅ 完全复用 |
| `Configuration::loadConfiguration()` | ✅ | ✅ | ✅ 完全复用 |
| `BridgeAbstract::loadConfiguration()` | ✅ | ✅ | ✅ 完全复用 |
| `ParameterValidator` | ✅ | ✅ | ✅ 完全复用 |
| 桥接 `collectData()` 逻辑 | ✅ | ✅ | ✅ 完全复用 |
| Feed 格式渲染（Atom/JSON/RSS 等） | ✅ | ✅ | ✅ 完全复用 |
| `$request->server('PHP_AUTH_*')` | ✅ 可用 | ❌ 返回 null | ❌ 不复用 |
| HTTP 响应头（`header()`） | ✅ 发送到浏览器 | ❌ 产生 Warning | ❌ 不复用 |

### 15.7 CLI 模式下的错误消息示例

**示例 1：参数缺失**
```bash
$ php index.php action=display bridge=YoutubeBridge format=Json
```
输出（完整 HTML）：
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <title>RSS-Bridge</title>
</head>
<body>
    <h1>Missing parameter</h1>
    <p>You must specify either:
 - YouTube username (?u=...)
 - Channel id (?c=...)
 - Playlist id (?p=...)
 - Search (?s=...)</p>
</body>
</html>
```
> 错误消息是桥接抛出的英文硬编码 `throwClientException()` 内容，经 `templates/exception.html.php` 渲染。

**示例 2：鉴权失败（Basic Auth 已启用）**
```bash
$ php index.php action=list
```
输出：
```html
<!DOCTYPE html>
<html lang="en">
<body>
    <p>Please authenticate in order to access this instance!</p>
</body>
</html>
```
> 同样是英文硬编码，与 Web 端完全相同。

### 15.8 CLI 模式下的"国际化"现状

**结论**：与 Web 端完全一致，无任何 i18n 能力。

| 检查项 | Web 模式 | CLI 模式 |
|-------|---------|---------|
| 翻译函数 `__()` / `t()` | ❌ | ❌ |
| 多语言包文件 | ❌ | ❌ |
| 语言 GET 参数 | ❌ | ❌ |
| Accept-Language 解析 | ❌ | ❌（无 HTTP 头） |
| 日期/时间本地化 | 仅 UTC 或配置时区 | 仅 UTC 或配置时区 |
| 错误消息语言 | 英文硬编码 | 英文硬编码 |
| 模板 lang 属性 | `<html lang="en">` | `<html lang="en">` |

### 15.9 CLI 下 Response send 的技术细节

由于 CLI 模式下没有 HTTP 协议层，`Response::send()` 会产生一些副作用：

1. **`http_response_code()`**：在 CLI 下调用不会影响实际退出码（始终为 0），也不会产生警告
2. **`header()`**：会产生 `PHP Warning: Cannot modify header information - headers already sent`，输出到 stderr
3. **退出码**：无论 Response code 是 200/400/404/500，PHP 进程退出码始终为 0
4. **内容类型**：不会设置 Content-Type，输出直接是原始 body（可能是 HTML/XML/JSON 等）

**如果要在脚本中正确处理 CLI 调用，应避免使用 `$response->send()`，改为**：
```bash
php index.php action=display bridge=YoutubeBridge u=test format=Json 2>/dev/null
```
重定向 stderr 以忽略 header() 警告。

---

## 十六、Admin 导入导出 Bridge 配置的批量校验

### 16.1 核心结论：无 Web 导入导出界面，仅支持 ini 文件批量配置

RSS-Bridge **没有 Web 管理后台，也没有"导入/导出"功能**。桥接的批量启用/禁用通过编辑 `config.ini.php` 中的 `[system] enabled_bridges` 数组来完成。批量校验逻辑在 `BridgeFactory` 构造函数中完成，无独立校验框架。

### 16.2 批量配置方式

#### 方式一：全部启用（通配符）

**文件**：`config.default.ini.php:30`
```ini
[system]
enabled_bridges[] = *
```

`*` 是特殊通配符，表示启用 `bridges/` 目录下所有的桥接类（通过 `scandir()` 扫描）。

#### 方式二：逐个列出

**文件**：`config.default.ini.php:14-29`
```ini
[system]
enabled_bridges[] = CssSelectorBridge
enabled_bridges[] = FeedMerge
enabled_bridges[] = Filter
enabled_bridges[] = Youtube
; ...
```

桥接名称支持大小写不敏感，且可以省略 `Bridge` 后缀（见 16.4 名称规范化）。

#### 方式三：whitelist.txt 白名单文件

**文件**：`lib/Configuration.php:46-51`
```php
if (file_exists(__DIR__ . '/../whitelist.txt')) {
    $enabledBridges = trim(file_get_contents(__DIR__ . '/../whitelist.txt'));
    if ($enabledBridges === '*') {
        self::setConfig('system', 'enabled_bridges', ['*']);
    } else {
        self::setConfig('system', 'enabled_bridges', array_filter(array_map('trim', explode("\n", $enabledBridges))));
    }
}
```

每行一个桥接名，文件级白名单。存在时会覆盖 ini 配置中的 `enabled_bridges`。

#### 方式四：环境变量

**文件**：`lib/Configuration.php:71-74`
```bash
export RSSBRIDGE_SYSTEM_ENABLED_BRIDGES="YoutubeBridge,TelegramBridge,Reddit"
```
按逗号分隔，自动拆分为数组。环境变量优先级最高。

### 16.3 批量校验逻辑（BridgeFactory 构造函数）

**文件**：`lib/BridgeFactory.php:11-42`

```php
public function __construct(CacheInterface $cache, Logger $logger)
{
    // Step 1: 扫描 bridges/ 目录，获取所有可用桥接类名
    foreach (scandir(__DIR__ . '/../bridges/') as $file) {
        if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
            $this->bridgeClassNames[] = $m[1];
        }
    }

    // Step 2: 读取配置中的 enabled_bridges
    $enabledBridges = Configuration::getConfig('system', 'enabled_bridges');
    if ($enabledBridges === null) {
        throw new \Exception('No bridges are enabled...');
    }

    // Step 3: 逐个校验 & 规范化
    foreach ($enabledBridges as $enabledBridge) {
        if ($enabledBridge === '*') {
            // 通配符：全部启用，直接复制数组并 break
            $this->enabledBridges = $this->bridgeClassNames;
            break;
        }
        $bridgeClassName = $this->createBridgeClassName($enabledBridge);
        if ($bridgeClassName) {
            // 校验通过：加入已启用列表
            $this->enabledBridges[] = $bridgeClassName;
        } else {
            // 校验失败：加入缺失列表 + INFO 日志，不抛出异常
            $this->missingEnabledBridges[] = $enabledBridge;
            $this->logger->info(sprintf('Bridge not found: %s', $enabledBridge));
        }
    }
}
```

**关键设计决策**：
- **容错而非阻断**：不存在的桥接名只是记录到 `missingEnabledBridges` 并打日志，不会导致整个系统 500
- **大小写不敏感**：名称比较时统一转小写
- **静默降级**：`FrontpageAction` 首页会显示警告（`Warning : Bridge "xxx" not found`），但功能正常

### 16.4 桥接名称规范化算法

**文件**：`lib/BridgeFactory.php:54-75`

```php
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    if (! in_array($nameLoweredCase, $namesLoweredCase)) {
        return null;  // 未找到
    }
    $index = array_search($nameLoweredCase, $namesLoweredCase);
    return $this->bridgeClassNames[$index];  // 返回原始大小写形式
}

public static function normalizeBridgeName(string $name)
{
    // 去掉 .php 后缀
    if (preg_match('/(.+)(?:\.php)/', $name, $matches)) {
        $name = $matches[1];
    }
    // 自动补全 Bridge 后缀
    if (!preg_match('/(Bridge)$/i', $name)) {
        $name = sprintf('%sBridge', $name);
    }
    return $name;
}
```

**规范化示例**：

| 输入 | normalize 后 | 查找结果 |
|-----|-------------|---------|
| `Youtube` | `YoutubeBridge` | ✅ YoutubeBridge |
| `youtube` | `youtubeBridge` | ✅ YoutubeBridge（大小写不敏感匹配） |
| `YoutubeBridge` | `YoutubeBridge` | ✅ YoutubeBridge |
| `YouTube` | `YouTubeBridge` | ✅ （模糊匹配） |
| `youtube.php` | `youtubeBridge` | ✅ |
| `NonExistent` | `NonExistentBridge` | ❌ null |

### 16.5 系统级 schema 校验

**文件**：`lib/Configuration.php:88-90`
```php
if (!is_array(self::getConfig('system', 'enabled_bridges'))) {
    self::throwConfigError('system', 'enabled_bridges', 'Is not an array');
}
```
- 仅校验**类型是否为数组**，不校验每个元素是否存在
- 元素存在性校验在 `BridgeFactory` 中做（见 16.3）

### 16.6 桥接级 CONFIGURATION 的批量加载与校验

每个桥接的专属配置（`const CONFIGURATION`）不在 `Configuration` 类中批量校验，而是**按需延迟加载**：

```
用户请求 action=display&bridge=TelegramBridge
    ↓
DisplayAction::__invoke()
    ↓
$bridge->loadConfiguration()
    ↓
遍历 static::CONFIGURATION（TelegramBridge::CONFIGURATION）
    ├─ 从 Configuration::getConfig('TelegramBridge', 'max_pages') 读取
    ├─ required=true 且缺失 → 抛异常
    └─ 有 defaultValue → 使用默认值
```

**无批量预校验**：只有当某个桥接被实际请求时，才会校验它的 CONFIGURATION。未被请求的桥接即使配置错误也不会被发现。

### 16.7 导入导出的等价操作

由于没有真正的"导入导出"功能，管理员通过以下方式实现批量配置管理：

| 操作 | 等价命令 |
|-----|---------|
| 导出当前桥接列表 | `ls bridges/ | grep Bridge.php` 或访问 `?action=list`（JSON） |
| 导入启用列表 | 编辑 `config.ini.php` 的 `enabled_bridges[]` 数组 |
| 批量禁用 | 注释掉对应的 `enabled_bridges[] = xxx` 行 |
| 全部启用 | `enabled_bridges[] = *` |

`ListAction` 提供 JSON 格式的桥接元数据导出（`actions/ListAction.php:13-35`）：
```json
{
    "bridges": {
        "YoutubeBridge": {
            "status": "active",
            "uri": "https://www.youtube.com",
            "name": "YouTube",
            "parameters": { ... },
            "description": "..."
        }
    },
    "total": 400
}
```
但没有对应的"导入"API。

---

## 十七、API 端点鉴权在公开 Bridge 与私有 Bridge 间的差异化处理

### 17.1 核心结论：无"公开/私有"桥接分级，鉴权是全站一刀切

RSS-Bridge **没有"公开桥接"和"私有桥接"的分级概念**。鉴权（HTTP Basic Auth / Token）是全站级别的开关——要么所有桥接都需要鉴权，要么所有桥接都不需要。

唯一的"分级"机制是 **`enabled_bridges` 白名单**：未启用的桥接不能访问，但这是功能开关而非鉴权分级。

### 17.2 鉴权与桥接可用性的两层控制

```
用户请求 action=display&bridge=XxxBridge
    ↓
第一层：鉴权中间件（全站通用，与具体桥接无关）
    ├─ BasicAuthMiddleware → 未通过 → 401 错误页
    └─ TokenAuthenticationMiddleware → 未通过 → 401 token 表单
    ↓
第二层：桥接白名单校验（DisplayAction 内）
    └─ BridgeFactory::isEnabled() 检查 → 未启用 → 400 "This bridge is not whitelisted"
    ↓
正常生成 Feed
```

### 17.3 各 Action 对桥接白名单的处理

| Action | 是否检查 isEnabled | 未启用时的行为 |
|-------|-------------------|--------------|
| `FrontpageAction`（首页） | ✅ 是（L31） | 不渲染该桥接的卡片，完全隐藏 |
| `DisplayAction`（Feed 生成） | ✅ 是（L36） | 返回 400 错误：`'This bridge is not whitelisted'` |
| `ListAction`（JSON 列表） | ✅ 是（L23） | 仍列出，但 `status: "inactive"` |
| `FindfeedAction`（发现 Feed） | ✅ 是（L32） | 跳过该桥接，不参与检测 |
| `DetectAction`（自动检测） | ❌ 否（自动检测） | - |
| `ConnectivityAction`（连通性） | ❌ 否（运维用） | - |
| `HealthAction`（健康检查） | ❌ 否（运维用） | - |

#### FrontpageAction 的白名单表现

**文件**：`actions/FrontpageAction.php:29-36`
```php
foreach ($bridgeClassNames as $bridgeClassName) {
    if ($this->bridgeFactory->isEnabled($bridgeClassName)) {
        $bridge = $this->bridgeFactory->create($bridgeClassName);
        $body .= self::render($bridge, $bridgeClassName, $token);
        $activeBridges++;
    }
}
```
- 未启用的桥接**不显示在首页**，用户看不到任何痕迹
- 首页计数只统计 active 桥接数

#### DisplayAction 的白名单表现

**文件**：`actions/DisplayAction.php:36-38`
```php
if (!$this->bridgeFactory->isEnabled($bridgeClassName)) {
    return new Response(render(__DIR__ . '/../templates/error.html.php', [
        'message' => 'This bridge is not whitelisted'
    ]), 400);
}
```
- 直接访问未启用桥接的 display 端点 → 返回 400 HTTP 错误
- 错误消息为英文硬编码

### 17.4 鉴权中间件与桥接的关系

**两个鉴权中间件对所有请求一视同仁**，不区分具体桥接：

```
所有请求（无论 bridge= 是什么）
    ↓
BasicAuthMiddleware
    ↓ 检查 [authentication] enable = true/false
    ↓ 如启用，校验 PHP_AUTH_USER / PHP_AUTH_PW
    ↓ 不通过 → 401，不会进入后续逻辑
    ↓
TokenAuthenticationMiddleware
    ↓ 检查 [authentication] token 是否配置
    ↓ 如配置，校验 $_GET['token']
    ↓ 不通过 → 401，不会进入后续逻辑
    ↓
DisplayAction 才开始检查具体桥接是否在白名单中
```

> 鉴权和白名单是**两个独立的控制层**：
> - 鉴权 = "你能不能访问这个网站"
> - 白名单 = "这个网站上有没有这个桥接"
>
> 未鉴权用户看不到任何桥接；鉴权但白名单外的桥接返回 400。

### 17.5 不存在的"私有桥接"特性

常见的"私有桥接"预期功能在本项目中均不存在：

| 预期的私有桥接特性 | 本项目是否支持 |
|------------------|--------------|
| 某些桥接需要登录才能访问 | ❌ 鉴权是全站的，不能按桥接单独设置 |
| 每个桥接有独立的访问密码 | ❌ 只有一个全局用户名/密码 / token |
| 不同用户看到不同的桥接列表 | ❌ 无用户系统，只有一个 admin 账户 |
| 桥接访问审计日志 | ❌ 无访问日志（仅有错误日志） |
| 按 IP 限制桥接访问 | ❌ 无 IP 黑白名单 |

### 17.6 白名单与鉴权的组合效果

| 场景 | 鉴权状态 | 桥接白名单 | 结果 |
|-----|---------|-----------|------|
| 未启用鉴权 + 白名单=`*` | - | 全部启用 | 所有桥接公开访问 |
| 未启用鉴权 + 白名单=部分 | - | 仅启用部分 | 公开访问部分桥接，其他报 400 |
| 启用鉴权 + 白名单=`*` | 未通过 | - | 401 登录弹窗 |
| 启用鉴权 + 白名单=`*` | 已通过 | 全部启用 | 可访问所有桥接 |
| 启用鉴权 + 白名单=部分 | 已通过 | 仅启用部分 | 可访问白名单内的桥接，其他 400 |

### 17.7 间接实现"私有桥接"的变通方式

虽然没有原生支持，但管理员可以通过以下方式近似实现：

#### 方式一：通过 enabled_bridges 隐藏敏感桥接

```ini
[system]
; 只启用"安全"的桥接，隐藏需要凭证的桥接
enabled_bridges[] = Youtube
enabled_bridges[] = Reddit
; TelegramBridge 不启用，通过内部文档告知用户手动添加
```

缺点：完全不启用就完全不能用，等于禁用而非"私有"。

#### 方式二：反向代理按路径鉴权

在 Nginx/Caddy 等反向代理层针对特定 bridge 的 URL 做额外鉴权：
```nginx
location ~* bridge=TelegramBridge {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd-telegram;
    proxy_pass http://rss-bridge;
}
```

这是项目外部的实现，与 RSS-Bridge 代码无关。

---

## 十八、定时任务跑批失败的恢复路径与重试上限

### 18.1 核心结论：无内置定时任务系统，仅在 HTTP 请求级有重试

RSS-Bridge **没有内置的定时任务（cron/job/queue）系统**，也没有"跑批"概念。所有桥接都是**按需触发**——用户请求时才执行抓取，失败就失败，没有后台重试任务。

项目中存在的重试机制只有两层：
1. **curl 请求级重试**：单次 HTTP 请求失败后的立即重试（`[http] retries` 配置）
2. **错误报告阈值**：错误累计到 `report_limit` 次才显示给用户（避免偶发错误打扰用户）

### 18.2 curl 请求级重试机制

**文件**：`lib/http.php:170-192`

```php
// This retry logic is a bit hard to understand, but it works
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        // 请求成功，跳出循环
        break;
    }
    if ($tries <= $config['retries']) {
        // 未达上限，继续循环（立即重试）
        continue;
    }
    // 达到重试上限，抛出异常
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

**关键特性**：
- **重试触发条件**：仅当 `curl_exec() === false` 时重试（即网络层错误，如连接超时、DNS 解析失败、连接拒绝等）
- **不重试的情况**：HTTP 状态码错误（404/500 等）**不会触发重试**，因为 `curl_exec` 仍返回 body（非 false）
- **重试次数配置**：`[http] retries`，默认 `1`（`config.default.ini.php:48-49`）
- **重试间隔**：**无间隔，立即重试**（无 backoff、无 sleep）
- **最大尝试次数** = `retries + 1`（首次 + N 次重试）
  - `retries=1` → 最多尝试 2 次
  - `retries=3` → 最多尝试 4 次

**重试流程图**：
```
调用 getContents() / getSimpleHTMLDOM()
    ↓
CurlHttpClient::request()
    ↓
curl_exec()
    ├─ 成功 (body !== false) → 返回 Response
    └─ 失败 (body === false)
        ├─ tries <= retries → 立即重试（continue）
        └─ tries > retries → 抛 HttpException
```

### 18.3 重试配置的传递链

```
config.default.ini.php [http] retries = 1
    ↓
Configuration::getConfig('http', 'retries')
    ↓
getContents() 构造 $config 数组（lib/contents.php:55）
    ↓
CurlHttpClient::request($url, $config)
    ↓
$defaultConfig = ['retries' => 2, ...]  ← 默认值
$config = array_merge($defaultConfig, $config);  ← 配置覆盖默认
    ↓
实际 retries 值 = min(配置文件值, 代码默认值)？不，是配置覆盖代码默认
```

> 注意：`CurlHttpClient` 代码中 `retries` 默认值为 `2`（L76），但 `getContents()` 会从配置读取覆盖它（L55），所以最终以 `config.default.ini.php` 中的 `retries = 1` 为准。

### 18.4 错误报告阈值（report_limit）

**文件**：`actions/DisplayAction.php:107-123, 172-191`

这不是"重试机制"，而是**错误抑制机制**——偶发错误不立即显示给用户，累计到一定次数后才报告。

```php
$reportLimit = Configuration::getConfig('error', 'report_limit');
$errorCount = 1;
if ($reportLimit > 1) {
    $errorCount = $this->logBridgeError($bridge->getName(), $e->getCode());
}
// 达到报告阈值才显示错误
if ($errorCount >= $reportLimit) {
    if ($errorOutput === 'feed') {
        $items = [$this->createFeedItemFromException($e, $bridge)];
    } elseif ($errorOutput === 'http') {
        return new Response(render('exception.html.php', ['e' => $e]), 500);
    } elseif ($errorOutput === 'none') {
        // 静默
    }
}
```

#### logBridgeError 计数逻辑

**文件**：`actions/DisplayAction.php:172-191`
```php
private function logBridgeError($bridgeName, $code)
{
    $cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
    $report = $this->cache->get($cacheKey);
    if ($report) {
        $report = Json::decode($report);
        $report['time'] = time();
        $report['count']++;      // 计数 +1
    } else {
        $report = [
            'error' => $code,
            'time' => time(),
            'count' => 1,        // 首次
        ];
    }
    $ttl = 86400 * 5;   // 5 天 TTL
    $this->cache->set($cacheKey, Json::encode($report), $ttl);
    return $report['count'];
}
```

**计数器特性**：
- 存储在**缓存系统**中（FileCache/SQLite 等），key = `error_reporting_{bridgeName}_{errorCode}`
- TTL = 5 天（86400 * 5 秒）
- 每次错误刷新 `time` 字段（重置 TTL 倒计时）
- 按**桥接名 + 错误码**分别计数
- 5 天内无错误则计数器自动过期消失

**与用户感知的关系**：
- `report_limit = 1`（默认）：第一次出错就显示给用户（无抑制）
- `report_limit = 3`：前两次出错用户看到空 Feed，第三次才显示错误条目
- 目的：避免偶发的、自动恢复的错误（如短暂网络波动）打扰用户

### 18.5 错误后的"恢复"路径

项目中没有显式的"失败恢复"机制，但存在以下几种自然恢复方式：

#### 恢复方式 1：下次请求自动重试

由于桥接是**按需执行**的，每次用户请求都是一次全新的抓取：
```
请求 1: YoutubeBridge → 网络超时 → 错误缓存计数 +1 → 空 Feed / 错误条目
请求 2: 5 分钟后用户刷新 → 重新抓取 → 成功 → 正常 Feed
```
每次请求都是独立的，失败不会导致后续请求被阻塞。

#### 恢复方式 2：缓存回退（304 Not Modified）

**文件**：`lib/contents.php:73-89`

如果之前有成功的缓存，且源站返回 304，会复用缓存内容：
```php
$cachedResponse = $cache->get($cacheKey);
if ($cachedResponse) {
    $lastModified = $cachedResponse->getHeader('last-modified');
    if ($lastModified) {
        $config['if_not_modified_since'] = $lastModified->getTimestamp();
    }
    $etag = $cachedResponse->getHeader('etag');
    if ($etag) {
        $httpHeadersNormalized['if-none-match'] = $etag;
    }
}
// ...
$response = $httpClient->request($url, $config);
// 304 Not Modified → 使用缓存
case 304:
    $response = $response->withBody($cachedResponse->getBody());
    break;
```

但这需要源站支持 304，且缓存未过期。

#### 恢复方式 3：错误计数过期重置

如果错误是偶发的，5 天后计数自动过期消失，相当于"自动恢复"：
```
Day 1: 错误 3 次 → 达到 report_limit=3 → 向用户显示错误
Day 2-6: 没有新请求 → 计数器 TTL 5 天后过期
Day 7: 新请求 → 重新计数，首次错误不显示（如果 report_limit>1）
```

### 18.6 无定时任务系统的佐证

全局搜索无以下相关代码：
- `cron` / `schedule` / `job` / `task`（定时任务相关）
- `max_retries` / `retry_count` / `backoff`（批处理重试）
- `queue` / `worker` / `daemon`（队列/守护进程）
- `pcntl_fork` / `exec` / `shell_exec`（进程管理）

Docker 入口脚本 `docker-entrypoint.sh` 也只是设置文件权限和启动 Apache，无 cron 配置。

### 18.7 社区常见的定时任务实现方式（项目外）

虽然项目本身没有，但 RSS-Bridge 的用户社区通常通过外部工具实现定时刷新：

| 方式 | 说明 |
|-----|------|
| `cron + curl` | 定时 curl 抓取 Feed URL，触发缓存更新 |
| `systemd timer` | 类似 cron 的 systemd 单元 |
| Feed 阅读器 | 读者的 RSS 阅读器定期拉取，自然触发 |
| `watchtower` / 类似工具 | 监控更新通知 |

这些都在 RSS-Bridge 代码库之外。

---

## 十九、扩展：若要实现真正的 i18n

当前项目无多语言能力。若需接入 i18n，需在以下位置做改造：

1. **桥接元数据层**：将 `NAME`、`DESCRIPTION`、`PARAMETERS[name/title/exampleValue]` 改为翻译键或支持多语言数组
2. **前端渲染层**：`FrontpageAction::render()` 中引入 `__()` 翻译函数包裹所有输出文本
3. **错误消息层**：`Configuration::throwConfigError()`、异常消息、`DisplayAction` 中的错误提示统一走翻译
4. **模板层**：`templates/*.html.php` 中的硬编码文案提取为翻译调用
5. **Admin 配置**：增加 `[system] default_locale` 配置项，按请求参数切换语言
