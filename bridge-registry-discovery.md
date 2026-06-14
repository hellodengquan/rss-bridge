# RSS-Bridge 桥接器注册与发现链路

本文逐段梳理一个 Bridge 实现从「写一个 PHP 文件」到「出现在前端列表」的完整链路。

---

## 全局流程总览

```
bridges/XxxBridge.php          ← 桥接器实现（约定式注册，无需手动声明）
       │
       ▼
lib/bootstrap.php              ← spl_autoload_register 扫描 bridges/ 目录实现自动加载
       │
       ▼
lib/BridgeFactory.php          ← 构造时 scandir() 发现所有桥接器类名，结合配置筛选 enabled 列表
       │
       ▼
lib/dependencies.php           ← DI 容器注册 BridgeFactory 及各 Action
       │
       ▼
lib/RssBridge.php              ← 请求路由：action 参数 → 容器取出对应 Action
       │
       ▼
actions/FrontpageAction.php    ← 调用 BridgeFactory 获取桥接器列表 → 渲染 HTML 卡片
actions/ListAction.php         ← 调用 BridgeFactory 获取桥接器列表 → 返回 JSON API
       │
       ▼
templates/frontpage.html.php   ← 前端页面展示桥接器卡片列表
```

---

## 第 1 环：桥接器实现（约定式注册）

**文件**: `bridges/XxxBridge.php`

RSS-Bridge 采用**约定优于配置**的方式注册桥接器——不需要任何显式注册声明，只需在 `bridges/` 目录下放置一个符合命名约定的 PHP 文件即可。

**命名约定**：文件名必须以 `Bridge.php` 结尾，如 `DemoBridge.php`、`GitHubBridge.php`。

**类定义**：类必须继承 `BridgeAbstract`，并覆写关键常量和方法：

```php
// bridges/DemoBridge.php
class DemoBridge extends BridgeAbstract
{
    const MAINTAINER = 'teromene';       // 维护者
    const NAME = 'DemoBridge';           // 显示名称
    const URI = 'https://github.com/...'; // 桥接目标站点
    const DESCRIPTION = 'Bridge used for demos'; // 描述（展示在卡片上）
    const CACHE_TIMEOUT = 15;            // 缓存超时秒数
    const PARAMETERS = [ ... ];          // 参数定义（决定前端表单）

    public function collectData() { ... } // 核心数据采集逻辑
}
```

**关键点**：桥接器本身不需要任何"注册"动作，文件放在 `bridges/` 目录即视为注册。

---

## 第 2 环：自动加载机制

**文件**: `lib/bootstrap.php:29-44`

```php
spl_autoload_register(function ($className) {
    $folders = [
        __DIR__ . '/../actions/',
        __DIR__ . '/../bridges/',      // ← 关键：bridges/ 在自动加载路径中
        __DIR__ . '/../caches/',
        __DIR__ . '/../formats/',
        __DIR__ . '/../lib/',
        __DIR__ . '/../middlewares/',
    ];
    foreach ($folders as $folder) {
        $file = $folder . $className . '.php';
        if (is_file($file)) {
            require $file;
        }
    }
});
```

当 PHP 首次引用一个未加载的类（如 `DemoBridge`）时，autoloader 会依次在上述目录中查找 `DemoBridge.php`。由于 `bridges/` 在扫描列表中，只要文件存在就会被自动 require。

**注意**：这只是按需加载，实际的"发现"逻辑在 `BridgeFactory` 中。

---

## 第 3 环：BridgeFactory — 发现与筛选

**文件**: `lib/BridgeFactory.php`

这是整个链路的**核心枢纽**，负责两件事：
1. 从文件系统**发现**所有桥接器类名
2. 结合配置**筛选**出启用的桥接器

### 3.1 发现：扫描 bridges/ 目录

```php
// BridgeFactory 构造函数（第 19-23 行）
foreach (scandir(__DIR__ . '/../bridges/') as $file) {
    if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
        $this->bridgeClassNames[] = $m[1];
    }
}
```

**机制**：`scandir()` 遍历 `bridges/` 目录，用正则 `/^([^.]+Bridge)\.php$/U` 提取类名。

- 匹配 `DemoBridge.php` → 提取 `DemoBridge`
- 匹配 `GitHubReleaseBridge.php` → 提取 `GitHubReleaseBridge`
- 不匹配 `.` 开头的隐藏文件、非 `Bridge.php` 结尾的文件

结果存入 `$this->bridgeClassNames` 数组，这就是**全量桥接器清单**。

### 3.2 筛选：结合配置确定启用列表

```php
// BridgeFactory 构造函数（第 25-41 行）
$enabledBridges = Configuration::getConfig('system', 'enabled_bridges');
foreach ($enabledBridges as $enabledBridge) {
    if ($enabledBridge === '*') {
        $this->enabledBridges = $this->bridgeClassNames;  // 通配符：全部启用
        break;
    }
    $bridgeClassName = $this->createBridgeClassName($enabledBridge);
    if ($bridgeClassName) {
        $this->enabledBridges[] = $bridgeClassName;
    } else {
        $this->missingEnabledBridges[] = $enabledBridge;  // 配置了但文件不存在
    }
}
```

**`enabled_bridges` 配置来源**（优先级从低到高）：

| 来源 | 文件/方式 | 说明 |
|------|-----------|------|
| 默认配置 | `config.default.ini.php` | `enabled_bridges[] = *`（默认全部启用） |
| 自定义配置 | `config.ini.php` | 覆盖默认值，可指定具体桥接器 |
| 白名单文件 | `whitelist.txt` | 旧式配置，内容为 `*` 或逐行列出 |
| 环境变量 | `RSSBRIDGE_SYSTEM_ENABLED_BRIDGES` | 最高优先级，逗号分隔 |

### 3.3 名称规范化与校验

```php
// BridgeFactory::createBridgeClassName（第 54-64 行）
public function createBridgeClassName(string $bridgeName): ?string
{
    $name = self::normalizeBridgeName($bridgeName);
    $namesLoweredCase = array_map('strtolower', $this->bridgeClassNames);
    $nameLoweredCase = strtolower($name);
    if (!in_array($nameLoweredCase, $namesLoweredCase)) {
        return null;  // 配置中写了但文件系统没有 → 返回 null
    }
    $index = array_search($nameLoweredCase, $namesLoweredCase);
    return $this->bridgeClassNames[$index];  // 返回精确大小写的类名
}
```

```php
// BridgeFactory::normalizeBridgeName（第 66-75 行）
public static function normalizeBridgeName(string $name)
{
    if (preg_match('/(.+)(?:\.php)/', $name, $matches)) {
        $name = $matches[1];    // 去掉 .php 后缀
    }
    if (!preg_match('/(Bridge)$/i', $name)) {
        $name = sprintf('%sBridge', $name);  // 自动补 Bridge 后缀
    }
    return $name;
}
```

这意味着配置中写 `GitHub`、`GitHubBridge`、`GitHubBridge.php` 都能匹配到 `GitHubBridge` 类。

### 3.4 暴露的查询接口

| 方法 | 作用 |
|------|------|
| `getBridgeClassNames()` | 返回全量桥接器类名数组（文件系统上发现的） |
| `isEnabled($name)` | 判断某个桥接器是否在启用列表中 |
| `create($name)` | 实例化一个桥接器（`new $name($cache, $logger)`） |
| `createBridgeClassName($name)` | 规范化名称 + 校验存在性 |
| `getMissingEnabledBridges()` | 返回配置中声明但文件不存在的桥接器 |

---

## 第 4 环：配置加载

**文件**: `lib/config.php` → `lib/Configuration.php`

### 加载顺序

```
1. config.default.ini.php   → 解析为默认配置
2. config.ini.php           → 解析为自定义配置，覆盖默认值
3. whitelist.txt            → 如果存在，覆盖 enabled_bridges
4. 环境变量 RSSBRIDGE_*     → 最高优先级覆盖
```

**关键配置项**：`[system] enabled_bridges[]`

```ini
; config.default.ini.php
[system]
enabled_bridges[] = *          ; 默认全部启用
```

```ini
; config.ini.php（用户自定义）
[system]
enabled_bridges[] = CssSelectorBridge
enabled_bridges[] = FilterBridge
enabled_bridges[] = Youtube
```

---

## 第 5 环：DI 容器与依赖注入

**文件**: `lib/dependencies.php`

```php
$container['bridge_factory'] = function ($c) {
    return new BridgeFactory($c['cache'], $c['logger']);  // ← BridgeFactory 在此实例化
};

$container[FrontpageAction::class] = function ($c) {
    return new FrontpageAction($c['bridge_factory']);      // ← 注入到 FrontpageAction
};

$container[ListAction::class] = function ($c) {
    return new ListAction($c['bridge_factory']);            // ← 注入到 ListAction
};

$container[DisplayAction::class] = function ($c) {
    return new DisplayAction($c['cache'], $c['logger'], $c['bridge_factory']);
};
```

**容器特性**（`lib/Container.php`）：懒加载 + 单例，首次 `offsetGet` 时执行工厂函数并缓存结果。

---

## 第 6 环：请求路由

**文件**: `index.php` → `lib/RssBridge.php`

### 入口

```php
// index.php
$rssBridge = new RssBridge($container);
$response = $rssBridge->main($request);
```

### 路由逻辑

```php
// RssBridge::main（第 13-40 行）
public function main(Request $request): Response
{
    $action = $request->get('action', 'Frontpage');     // 默认 action=Frontpage
    $actionName = strtolower($action) . 'Action';       // → "frontpageAction"
    $actionName = implode(array_map('ucfirst', explode('-', $actionName)));  // → "FrontpageAction"
    $filePath = __DIR__ . '/../actions/' . $actionName . '.php';
    if (!file_exists($filePath)) {
        return new Response(..., 400);  // action 不存在
    }

    $handler = $this->container[$actionName];  // ← 从容器取出 Action 实例
    // ... 中间件包装 ...
    return $action($request);
}
```

**关键 URL 映射**：

| URL 参数 | Action 类 | 作用 |
|----------|-----------|------|
| （无 action 参数） | `FrontpageAction` | 前端主页，展示桥接器卡片列表 |
| `action=list` | `ListAction` | JSON API，返回所有桥接器元数据 |
| `action=display&bridge=Xxx` | `DisplayAction` | 生成具体桥接器的 RSS Feed |
| `action=detect` | `DetectAction` | URL 检测匹配桥接器 |
| `action=findfeed` | `FindfeedAction` | 从 URL 查找可用 Feed |

---

## 第 7 环：前端暴露 — FrontpageAction（HTML 卡片）

**文件**: `actions/FrontpageAction.php`

这是用户访问首页时看到的桥接器列表。

### 执行流程

```php
// FrontpageAction::__invoke（第 13-49 行）
public function __invoke(Request $request): Response
{
    // 1. 获取全量桥接器类名
    $bridgeClassNames = $this->bridgeFactory->getBridgeClassNames();

    // 2. 对每个【已启用】的桥接器渲染卡片
    $body = '';
    foreach ($bridgeClassNames as $bridgeClassName) {
        if ($this->bridgeFactory->isEnabled($bridgeClassName)) {
            $bridge = $this->bridgeFactory->create($bridgeClassName);   // 实例化
            $body .= self::render($bridge, $bridgeClassName, $token);   // 渲染卡片
            $activeBridges++;
        }
    }

    // 3. 将卡片 HTML 嵌入模板
    return new Response(render(__DIR__ . '/../templates/frontpage.html.php', [
        'bridges'        => $body,          // 所有卡片 HTML 拼接
        'active_bridges' => $activeBridges,
        'total_bridges'  => count($bridgeClassNames),
    ]));
}
```

### 卡片渲染（render 方法）

```php
// FrontpageAction::render（第 51-148 行）
public static function render(BridgeAbstract $bridge, string $bridgeClassName, ?string $token): string
{
    // 从桥接器实例提取元数据
    $uri         = $bridge->getURI();
    $name        = $bridge->getName();
    $icon        = $bridge->getIcon();
    $description = $bridge->getDescription();
    $parameters  = $bridge->getParameters();

    // 生成 <section class="bridge-card"> HTML
    // 包含：名称链接、描述、参数表单、提交按钮、维护者信息
}
```

**每个卡片的结构**：

```html
<section class="bridge-card" id="bridge-DemoBridge" data-ref="DemoBridge" data-short-name="DemoBridge">
    <h2><a href="https://...">DemoBridge</a></h2>
    <p class="description">Bridge used for demos</p>
    <input type="checkbox" class="showmore-box" id="showmore-DemoBridge" />
    <label class="showmore" for="showmore-DemoBridge">Show more</label>

    <!-- 参数表单 -->
    <form method="GET" action="?" class="bridge-form">
        <input type="hidden" name="action" value="display" />
        <input type="hidden" name="bridge" value="DemoBridge" />
        <!-- 各参数输入控件... -->
        <button type="submit" name="format" value="Html">Generate feed</button>
    </form>

    <p class="maintainer">teromene</p>
</section>
```

---

## 第 8 环：前端暴露 — ListAction（JSON API）

**文件**: `actions/ListAction.php`

```php
public function __invoke(Request $request): Response
{
    $list = new \stdClass();
    $list->bridges = [];

    // 遍历【全部】桥接器（不仅是启用的）
    foreach ($this->bridgeFactory->getBridgeClassNames() as $bridgeClassName) {
        $bridge = $this->bridgeFactory->create($bridgeClassName);

        $list->bridges[$bridgeClassName] = [
            'status'      => $this->bridgeFactory->isEnabled($bridgeClassName) ? 'active' : 'inactive',
            'uri'         => $bridge->getURI(),
            'donationUri' => $bridge->getDonationURI(),
            'name'        => $bridge->getName(),
            'icon'        => $bridge->getIcon(),
            'parameters'  => $bridge->getParameters(),
            'maintainer'  => $bridge->getMaintainer(),
            'description' => $bridge->getDescription(),
        ];
    }
    $list->total = count($list->bridges);
    return new Response(Json::encode($list), 200, ['content-type' => 'application/json']);
}
```

**与 FrontpageAction 的区别**：ListAction 遍历的是**全量**桥接器并标记 status（active/inactive），而 FrontpageAction 只渲染**已启用**的桥接器。

---

## 第 9 环：前端模板

**文件**: `templates/frontpage.html.php`

```php
<!-- 搜索栏 -->
<section class="searchbar">
    <input type="text" name="searchfield" id="searchfield"
           placeholder="Insert URL or bridge name"
           onchange="rssbridge_list_search()"
           onkeyup="rssbridge_list_search()">
    <button type="button" id="findfeed" name="findfeed">Find Feed from URL</button>
</section>

<!-- 桥接器卡片列表（由 FrontpageAction 预渲染的 HTML 直接嵌入） -->
<?= raw($bridges) ?>

<!-- 页脚统计 -->
<p><?= $active_bridges ?>/<?= $total_bridges ?> active bridges.</p>
```

**前端交互**：`static/rss-bridge.js` 提供 `rssbridge_list_search()` 搜索过滤功能，通过 CSS 隐藏/显示卡片实现实时过滤。

---

## 完整调用时序

```
用户访问首页
    │
    ▼
index.php
    │ require bootstrap.php → 注册 autoloader
    │ require config.php    → 加载配置（含 enabled_bridges）
    │ require dependencies.php → 创建 DI 容器
    │
    ▼
RssBridge::main($request)
    │ action 默认 "Frontpage" → 拼出 "FrontpageAction"
    │ $container['FrontpageAction'] → 触发容器工厂函数
    │     │
    │     ▼
    │ new FrontpageAction($container['bridge_factory'])
    │     │
    │     ▼
    │ $container['bridge_factory'] → 触发容器工厂函数
    │     │
    │     ▼
    │ new BridgeFactory($cache, $logger)
    │     │ 构造时 scandir('bridges/') → 发现全部桥接器类名
    │     │ 读取 enabled_bridges 配置 → 筛选启用列表
    │
    ▼
FrontpageAction::__invoke($request)
    │ bridgeFactory->getBridgeClassNames() → 获取全量类名
    │ 对每个已启用类名:
    │   bridgeFactory->create($className) → 实例化桥接器
    │   self::render($bridge, ...)        → 渲染 HTML 卡片
    │
    ▼
render('templates/frontpage.html.php', ['bridges' => $cardsHtml])
    │
    ▼
Response → 发送给浏览器
```

---

## 关键设计特点总结

1. **零注册**：桥接器无需显式注册，文件放入 `bridges/` 目录即自动被发现
2. **双重发现**：autoloader 负责按需加载类文件，BridgeFactory 负责启动时扫描出全量类名清单
3. **配置驱动启禁**：`enabled_bridges` 配置决定哪些桥接器对外可见，支持通配符 `*`
4. **实例化延迟**：FrontpageAction/ListAction 遍历时才通过 `BridgeFactory::create()` 实例化，避免不必要的对象创建
5. **两种暴露方式**：FrontpageAction 渲染 HTML 卡片（仅启用），ListAction 返回 JSON（含未启用的 status 标记）

---

## 补充 A：桥接器加载失败的错误处理和剔除策略

RSS-Bridge 的错误处理策略是**"软失败、可观测、不中断"**——单个桥接器故障不会导致整体服务崩溃，但会通过日志和前端警告暴露出来。

### A.1 配置声明但文件不存在（missingEnabledBridges）

**文件**: `lib/BridgeFactory.php:25-41`

```php
foreach ($enabledBridges as $enabledBridge) {
    if ($enabledBridge === '*') {
        $this->enabledBridges = $this->bridgeClassNames;
        break;
    }
    $bridgeClassName = $this->createBridgeClassName($enabledBridge);
    if ($bridgeClassName) {
        $this->enabledBridges[] = $bridgeClassName;
    } else {
        // 关键：记录到 missingEnabledBridges，同时打 info 日志
        $this->missingEnabledBridges[] = $enabledBridge;
        $this->logger->info(sprintf('Bridge not found: %s', $enabledBridge));
    }
}
```

**处理策略**：
- 不抛异常、不中断启动流程
- 将缺失的桥接器名存入 `$this->missingEnabledBridges` 数组
- 通过 `Logger::info()` 记录到日志（级别为 INFO，生产环境默认不输出）

### A.2 前端页面警告

**文件**: `actions/FrontpageAction.php:22-27`

```php
foreach ($this->bridgeFactory->getMissingEnabledBridges() as $missingEnabledBridge) {
    $messages[] = [
        'body'  => sprintf('Warning : Bridge "%s" not found', $missingEnabledBridge),
        'level' => 'warning'
    ];
}
```

`$messages` 最终通过模板变量传递到首页顶部，以黄色警告条形式展示给管理员。

### A.3 DisplayAction 请求时的失败场景

**文件**: `actions/DisplayAction.php:21-38`

当用户通过 URL 直接请求某个桥接器时（`?action=display&bridge=Xxx`），会触发三道校验：

| 校验项 | 代码位置 | 失败行为 |
|--------|----------|----------|
| bridge 参数是否存在 | 第 25-27 行 | 返回 400 + 错误页 "Missing bridge name parameter" |
| 桥接器类是否存在（文件系统扫描） | 第 28-31 行 | 返回 404 + 错误页 "Bridge not found" |
| 桥接器是否在启用列表 | 第 36-38 行 | 返回 400 + 错误页 "This bridge is not whitelisted" |

### A.4 类加载本身的失败（PHP Fatal Error）

**autoloader 层** (`lib/bootstrap.php:29-44`)：只做 `require $file`，不 catch。如果桥接器文件有语法错误或依赖缺失，将触发 PHP Fatal Error，被 `index.php` 中的 `register_shutdown_function` 捕获并记录到错误日志：

```php
// index.php:47-59
register_shutdown_function(function () use ($logger) {
    $error = error_get_last();
    if ($error) {
        $logger->error(sprintf('(shutdown) %s: %s in %s line %s', ...));
    }
});
```

**注意**：这是进程级的 shutdown handler，单个桥接器文件语法错误不会导致整个进程崩溃（PHP-FPM 模式下每个请求独立进程），但会让该请求的响应中断。

### A.5 实例化失败

`BridgeFactory::create()` (`lib/BridgeFactory.php:44-47`) 直接 `new $name($cache, $logger)`，没有 try-catch。如果桥接器构造函数抛异常，会冒泡到外层 `ExceptionMiddleware` 处理，渲染异常页面并打 error 日志。

### A.6 ConnectivityAction（调试模式下的健康检查）

**文件**: `actions/ConnectivityAction.php`

仅在 `env=dev` 时可用，可手动检查单个桥接器目标站点的 HTTP 可达性：

```
?action=connectivity          → 返回 JS 自动探测页面，逐个发起 AJAX
?action=connectivity&bridge=X → 返回 JSON: {bridge, successful, http_code}
```

探测逻辑：对 `$bridge::URI` 发起 cURL 请求（超时 5 秒），HTTP 200 视为成功。

---

## 补充 B：桥接器版本兼容性与升级路径

RSS-Bridge 本身没有严格的桥接器版本号体系，兼容性主要通过**基类约束 + 单元测试**来保证。

### B.1 系统版本标识

**文件**: `lib/Configuration.php:10`

```php
private const VERSION = '2025-08-05';
```

RSS-Bridge 使用日期格式（`YYYY-MM-DD`）作为版本号，通过 `Configuration::getVersion()` 对外暴露（若检测到 `.git/HEAD` 会附加分支名和 commit 短哈希）。

### B.2 PHP 版本下限

**文件**: `index.php:3-6`

```php
if (version_compare(\PHP_VERSION, '7.4.0') === -1) {
    http_response_code(500);
    exit("RSS-Bridge requires minimum PHP version 7.4\n");
}
```

系统启动即硬校验 PHP ≥ 7.4，不满足直接 500 退出。

### B.3 桥接器实现规范测试（BridgeImplementationTest）

**文件**: `tests/BridgeImplementationTest.php`

这是桥接器兼容性的核心防线，覆盖所有 `bridges/*Bridge.php` 文件（通过 `dataBridgesProvider` 用 `glob` 自动枚举）。

#### 测试项一览

| 测试方法 | 校验内容 | 对应代码行 |
|----------|----------|-----------|
| `testClassName` | 类名首字母大写、不含空格、以 `Bridge` 结尾 | 17-23 |
| `testClassType` | 必须是 `BridgeAbstract` 的实例（含 `FeedExpander` 子类） | 28-32 |
| `testConstants` | `NAME/URI/DESCRIPTION/MAINTAINER` 必须为非空字符串；`PARAMETERS` 必须为数组；`CACHE_TIMEOUT` 必须为 ≥0 的整数 | 37-53 |
| `testParameters` | 多 context 时 context 名必须为非空字符串；参数 `name` 非空；`type` 只能是 `text/number/list/checkbox`；`list` 类型必须有 `values` 数组；`required` 仅适用于 `text/number`；`pattern/title/exampleValue/defaultValue` 校验 | 58-158 |
| `testMethodValues` | `getDescription/getMaintainer/getName/getURI/getIcon` 返回值类型及非空 | 163-185 |
| `testUri` | `URI` 常量和 `getURI()` 返回值必须通过 `FILTER_VALIDATE_URL` | 190-196 |

#### 测试执行方式

```bash
vendor/bin/phpunit tests/BridgeImplementationTest.php
```

该测试被 CI workflow (`.github/workflows/tests.yml`) 纳入 PR 检查，任何桥接器新增或修改若不符合规范会直接阻断合并。

### B.4 FeedExpander：桥接器升级兼容基类

**文件**: `lib/FeedExpander.php`

`FeedExpander` 是 `BridgeAbstract` 的子类，专为**已有 RSS/Atom Feed 的扩展增强**场景设计。它是桥接器升级的主要兼容路径——当目标站点本身有 Feed 但需要扩展内容（补全摘要、解析全文、添加图片等），新写法应继承 `FeedExpander` 而非直接继承 `BridgeAbstract`。

```php
abstract class FeedExpander extends BridgeAbstract
{
    // 封装了 Feed 抓取、解析、裁剪逻辑
    public function collectExpandableDatas(string $url, $maxItems = -1, $headers = [])

    // 子类覆写此方法对每条 item 做转换（默认原样返回）
    protected function parseItem(array $item) { return $item; }

    // 子类可覆写此方法对原始 XML 做预处理
    protected function prepareXml(string $xmlString): string

    // Feed 元数据优先取真实解析结果，降级到常量
    public function getURI()  { return $this->feed['uri']   ?? parent::getURI(); }
    public function getName() { return $this->feed['title'] ?? parent::getName(); }
    public function getIcon() { return $this->feed['icon']  ?? parent::getIcon(); }
}
```

目前已有 **大量桥接器通过 `FeedExpander` 实现**（搜索 `extends FeedExpander` 可得数十个），包括 `WordPressBridge`、`TagesschauBridge`、`TheGuardianBridge`、`SubstackBridge` 等主流桥接器。

### B.5 升级兼容策略总结

| 变更场景 | 兼容策略 |
|----------|----------|
| 目标站点从无 Feed → 有 Feed | 桥接器可从 `extends BridgeAbstract` 改为 `extends FeedExpander`，接口保持兼容 |
| 目标站点 Feed 格式变化 | 在 `prepareXml()` 中做 XML 修复，或在 `parseItem()` 中转换字段 |
| BridgeAbstract 新增抽象方法 | 现有桥接器自动触发测试失败，PR 流水线阻断 |
| 参数定义规范变更 | `BridgeImplementationTest::testParameters` 自动覆盖，不合规桥接器被标记 |
| PHP 版本升级 | `index.php` 硬校验 + CI phpunit.xml 中指定版本矩阵 |

**没有**针对单个桥接器的"废弃/Deprecated"标记机制，不兼容的桥接器直接通过测试失败暴露，由维护者在 PR 中修复。

---

## 补充 C：前端列表渲染时的过滤与排序逻辑

### C.1 排序策略

**后端不排序**，完全依赖文件系统顺序。

**排序来源**：`BridgeFactory` 构造时使用 `scandir(__DIR__ . '/../bridges/')`（`lib/BridgeFactory.php:19`）。PHP 的 `scandir()` 按**字母升序**返回目录条目，因此桥接器卡片的默认展示顺序就是文件名的 A-Z 字典序（例如 `ABCNewsBridge` 排在最前，`ZeitBridge` 排在最后）。

```
bridges/
  ABCNewsBridge.php      ← 第 1 个
  ABolaBridge.php        ← 第 2 个
  ...
  ZeitBridge.php         ← 最后一个
```

**注意**：PHP `scandir()` 对大小写敏感，大写字母排在小写字母前面（ASCII 码顺序）。项目中所有桥接器文件名均以大写开头，因此实际表现为标准的字典序。

### C.2 后端无过滤

`FrontpageAction` 只做**启禁过滤**（`isEnabled()` 判断），不做关键字过滤、分类过滤等任何业务过滤。所有启用的桥接器都会渲染到页面，过滤完全由前端 JavaScript 完成。

### C.3 前端实时搜索过滤

**文件**: `static/rss-bridge.js:1-30`

```javascript
function rssbridge_list_search() {
    var search = document.getElementById('searchfield').value;

    var bridgeCards = document.querySelectorAll('section.bridge-card');
    for (var i = 0; i < bridgeCards.length; i++) {
        var bridgeName        = bridgeCards[i].getAttribute('data-ref');
        var bridgeShortName   = bridgeCards[i].getAttribute('data-short-name');
        var bridgeDescription = bridgeCards[i].querySelector('.description');
        var bridgeUrlElement  = bridgeCards[i].getElementsByTagName('a')[0];
        var bridgeUrl         = bridgeUrlElement.toString();

        bridgeCards[i].style.display = 'none';          // 默认隐藏
        if (!bridgeName || !bridgeUrl) { continue; }

        var searchRegex = new RegExp(search, 'i');      // 不区分大小写
        if (bridgeName.match(searchRegex))        { bridgeCards[i].style.display = 'block'; }
        if (bridgeShortName.match(searchRegex))   { bridgeCards[i].style.display = 'block'; }
        if (bridgeDescription.textContent.match(searchRegex)) { bridgeCards[i].style.display = 'block'; }
        if (bridgeUrl.match(searchRegex))         { bridgeCards[i].style.display = 'block'; }
    }
}
```

#### 过滤字段

| 字段来源 | DOM 位置 | 说明 |
|----------|----------|------|
| `data-ref` | `<section class="bridge-card" data-ref="DemoBridge">` | 桥接器显示名称（`$bridge->getName()`） |
| `data-short-name` | `<section data-short-name="DemoBridge">` | 类短名（ReflectionClass::getShortName()） |
| `.description` 文本 | `<p class="description">...</p>` | 桥接器描述 |
| 首个 `<a>` href | `<h2><a href="https://...">...</a></h2>` | 桥接器目标站点 URL |

#### 匹配规则
- **不区分大小写**（正则 `'i'` 标志）
- **或关系**：四个字段中任一匹配即显示卡片
- **支持正则**：直接用用户输入构造 `RegExp`（用户可输入 `github|gitlab` 这类模式）
- **实时触发**：`onchange` + `onkeyup` 双事件绑定（`templates/frontpage.html.php:15-17`）

#### 搜索框初始化

```php
// templates/frontpage.html.php
<script>
    document.addEventListener('DOMContentLoaded', rssbridge_toggle_bridge);  // URL hash 展开
    document.addEventListener('DOMContentLoaded', rssbridge_list_search);   // 初始执行一次过滤
    document.addEventListener('DOMContentLoaded', rssbridge_feed_finder);   // Feed Finder 按钮绑定
</script>
```

页面加载完成即执行一次 `rssbridge_list_search()`——此时搜索框为空，`new RegExp('')` 匹配所有内容，因此全部卡片默认显示。

### C.4 锚点跳转与自动展开

**文件**: `static/rss-bridge.js:32-39`

```javascript
function rssbridge_toggle_bridge() {
    var fragment = window.location.hash.substr(1);   // 例如 "#bridge-DemoBridge"
    var bridge = document.getElementById(fragment);
    if (bridge !== null) {
        bridge.getElementsByClassName('showmore-box')[0].checked = true;  // 自动勾选展开
    }
}
```

当 URL 带 `#bridge-DemoBridge` 这样的 hash 时（如从搜索引擎点击或管理员分享链接），对应桥接器卡片的 "Show more" 复选框会被自动勾选，展开参数表单区。

### C.5 Feed Finder（URL → 桥接器反向匹配）

**文件**: `static/rss-bridge.js:47-124`

这是与搜索过滤互补的另一种发现方式：用户粘贴一个目标页面 URL，前端调用 `?action=findfeed` 后端接口，后端遍历所有桥接器的 `detectParameters($url)` 方法进行匹配，匹配成功的桥接器以 `.search-result` 卡片渲染在搜索栏下方。

```javascript
async function rssbridge_feed_search(event) {
    const input = document.getElementById('searchfield');
    let baseurl = window.location.protocol + window.location.pathname;
    let url = baseurl + '?action=findfeed&format=Html&url=' + content;
    const response = await fetch(url);
    const data = await response.json();
    rss_bridge_feed_display_found_feed(data);  // 渲染匹配结果
}
```

### C.6 过滤与排序策略总结

| 维度 | 行为 | 实现位置 |
|------|------|----------|
| 默认排序 | 按文件名字母升序（A→Z） | `scandir()` 天然顺序 |
| 自定义排序 | 无任何排序接口 | — |
| 启禁过滤 | 后端 `isEnabled()` 判断，未启用的桥接器不渲染 | `actions/FrontpageAction.php:31` |
| 关键字过滤 | 前端 JS，4 字段 OR 正则匹配，不区分大小写 | `static/rss-bridge.js:1-30` |
| 分类/标签过滤 | 不支持（桥接器无分类体系） | — |
| URL 反向匹配 | 前端 AJAX + 后端 `detectParameters()` | `?action=findfeed` |
| 锚点自动展开 | URL `#bridge-Xxx` 自动展开对应卡片 | `static/rss-bridge.js:32-39` |

---

## 补充 D：桥接器请求外部网站的限流与缓存机制

RSS-Bridge 的限流和缓存分为三层：**HTTP 请求层缓存**、**桥接器级缓存**、**Feed 响应层缓存**，每层独立运作、协同生效。

### D.1 限流控制

RSS-Bridge 本身**没有全局请求限流中间件**，但通过以下机制间接实现速率控制：

#### 桥接器缓存超时（CACHE_TIMEOUT）

**文件**: `lib/BridgeAbstract.php:18, 114-117`

```php
const CACHE_TIMEOUT = 3600;  // 默认 1 小时

public function getCacheTimeout()
{
    return static::CACHE_TIMEOUT;
}
```

每个桥接器通过 `CACHE_TIMEOUT` 常量声明自己的缓存 TTL（秒）。这是最核心的"限流"手段——同一参数的 Feed 请求在 TTL 内不会重复执行 `collectData()`，从而避免频繁请求目标网站。

各桥接器根据目标站点更新频率自行设定：
- 高频更新：`Vk2Bridge` 300 秒、`VixenBridge` 60 秒
- 中频更新：多数桥接器 3600 秒（1 小时）
- 低频更新：`WarhammerComBridge` 86400 秒（1 天）、`UnsplashBridge` 43200 秒（12 小时）

#### 用户自定义缓存超时

**文件**: `actions/FrontpageAction.php:74-80` + `config.default.ini.php:68`

```ini
[cache]
custom_timeout = false   ; 默认关闭
```

如果 `cache.custom_timeout = true`，前端每个桥接器卡片会多出一个"Cache timeout in seconds"输入框，用户可在请求时通过 `_cache_timeout` 参数自定义单次请求的缓存 TTL。

```php
// actions/DisplayAction.php:57-62
$ttl = $request->get('_cache_timeout');
if (Configuration::getConfig('cache', 'custom_timeout') && isset($ttl)) {
    $ttl = (int) $ttl;
} else {
    $ttl = $bridge->getCacheTimeout();
}
```

#### 连接超时与最大文件大小

**文件**: `lib/http.php:71, 94-98, 119-131`

```php
// 默认配置
'timeout' => 5,            // HTTP 请求超时 5 秒
'max_filesize' => null,    // 响应体大小限制
'max_redirections' => 5,   // 最多 5 次重定向
```

`max_filesize` 通过 `CURLOPT_MAXFILESIZE`（检查 Content-Length 头）和 `CURLOPT_PROGRESSFUNCTION`（流式传输中实时监控）双重保障，防止目标站点返回超大响应拖垮服务。配置项 `http.max_filesize` 单位为 MB（默认 20）。

### D.2 HTTP 请求层缓存（getContents）

**文件**: `lib/contents.php:36-138`

`getContents()` 是所有桥接器发起 HTTP 请求的标准入口，内置了完整的 HTTP 缓存机制。

#### 缓存 Key 生成

```php
$cacheKey = implode('_', ['server',  $url, $requestBodyHash]);
```

Key 由三部分组成：前缀 `server_` + URL + POST body 的 MD5（仅 POST 请求有）。

#### 缓存命中流程

```
请求进入
    │
    ▼
查 cache_key 是否命中
    ├─ 命中 → 提取 Last-Modified 和 ETag
    │          → 加入请求头（If-Modified-Since / If-None-Match）
    │          → 发请求
    │              ├─ 304 Not Modified → 直接使用缓存 body
    │              └─ 200 OK → 更新缓存（TTL 10 天）
    └─ 未命中 → 直接发请求 → 200 时写入缓存（TTL 10 天）
```

#### 缓存 TTL 与策略

- **成功响应（200/201/202）**：缓存 10 天（864000 秒），除非响应头 `Cache-Control` 含 `no-cache` / `no-store`
- **重定向（301/302/303）**：TODO 注释，未实现缓存
- **304 Not Modified**：复用缓存 body，不更新 TTL
- **其他状态码**：抛出 `HttpException`，不缓存

注意：这层缓存是**原始 HTTP 响应缓存**，与桥接器的 `CACHE_TIMEOUT` 无关，TTL 固定 10 天（但 HTTP 缓存协商机制会在内容变化时自动刷新）。

### D.3 页面 DOM 缓存（getSimpleHTMLDOMCached）

**文件**: `lib/contents.php:219-251`

桥接器可以调用 `getSimpleHTMLDOMCached($url, $ttl)` 来缓存解析后的 simple_html_dom 对象，默认 TTL 86400 秒（1 天）。

```php
function getSimpleHTMLDOMCached($url, $ttl = 86400, ...): \simple_html_dom {
    $cacheKey = 'pages_' . $url;
    $content = $cache->get($cacheKey);
    if (!$content) {
        $content = getContents($url, ...);
        $cache->set($cacheKey, $content, $ttl);
    }
    return str_get_html($content, ...);
}
```

这层缓存的 key 前缀是 `pages_`，与 `getContents()` 的 `server_` 前缀不冲突，相当于二级缓存。

### D.4 桥接器内部缓存（loadCacheValue / saveCacheValue）

**文件**: `lib/BridgeAbstract.php:325-333`

桥接器可以通过这两个方法存取自定义缓存数据，Key 会自动加上桥接器短名前缀隔离。

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

常用于存储桥接器内部状态，比如 API 访问令牌、分页游标等。

### D.5 Feed 响应层缓存（CacheMiddleware）

**文件**: `middlewares/CacheMiddleware.php`

这是最外层的缓存，对整个 `DisplayAction` 的 HTTP 响应做缓存。

```php
// 仅对 DisplayAction 生效
if ($action !== 'DisplayAction') {
    return $next($request);
}

$cacheKey = 'http_' . json_encode($request->toArray());
$cachedResponse = $this->cache->get($cacheKey);
```

#### 缓存策略

| 响应状态码 | 缓存 TTL | 说明 |
|-----------|----------|------|
| 200 | 由桥接器 `CACHE_TIMEOUT` 决定 | DisplayAction 内部已自行缓存，这里做占位 |
| 400 / 403 / 404 / 429 / 500 / 503 | 5 分钟 + 随机 1~10 分钟 | 错误响应缓存，防止雪崩 |
| 其他 | 5 分钟 | 兜底 |

额外支持 `If-Modified-Since` 协商缓存，客户端带该头且缓存未过期时返回 304。

#### 缓存清理

每 100 个请求随机触发一次 `$this->cache->prune()`，清理过期的缓存条目。

### D.6 缓存后端实现

**文件**: `caches/` 目录 + `lib/CacheFactory.php`

| 缓存实现 | 文件 | 特点 |
|---------|------|------|
| FileCache | `caches/FileCache.php` | 默认，文件系统存储，每个 key 一个文件 |
| SQLiteCache | `caches/SQLiteCache.php` | SQLite 单文件存储，适合大量小 key |
| MemcachedCache | `caches/MemcachedCache.php` | 分布式内存缓存 |
| ArrayCache | `caches/ArrayCache.php` | 进程内数组，调试模式默认使用 |
| NullCache | `caches/NullCache.php` | 空实现，永不命中 |

通过 `cache.type` 配置切换，默认 `file`。

### D.7 代理与绕过

**文件**: `lib/contents.php:100-102` + `actions/DisplayAction.php:40-48`

可配置全局 HTTP 代理（`proxy.url`），所有 `getContents()` 请求自动走代理。如果 `proxy.by_bridge = true`，每个桥接器卡片会多出一个"Disable proxy"复选框，用户可通过 `_noproxy` 参数单次绕过代理。

---

## 补充 E：用户自定义桥接器的热加载与隔离

需要先明确：**RSS-Bridge 没有正式的"用户自定义桥接器"功能，也没有热加载机制**。所有桥接器都是通过放置在 `bridges/` 目录下的 PHP 文件实现的，属于同一代码池。但可以从以下几个角度理解"自定义"和"隔离"的实践方式。

### E.1 没有热加载，但 PHP 本身是"热"的

由于 RSS-Bridge 是 PHP 程序（无状态、每次请求独立执行），"热加载"的含义与常驻进程框架不同：

- **文件系统级热加载**：往 `bridges/` 目录放入新的 `XxxBridge.php` 文件后，下一个请求就能自动发现并加载它——因为 `BridgeFactory` 每次实例化都会重新 `scandir()` 扫描目录
- **代码级热加载**：修改已有桥接器文件后，下一个请求会自动加载新代码（PHP-FPM 模式下除非开启了 opcache 且未过期）
- **无热卸载**：文件删除后桥接器自动消失，但不会影响正在处理中的请求

**关键位置**：`lib/BridgeFactory.php:19-23`

```php
foreach (scandir(__DIR__ . '/../bridges/') as $file) {
    if (preg_match('/^([^.]+Bridge)\.php$/U', $file, $m)) {
        $this->bridgeClassNames[] = $m[1];
    }
}
```

每次请求都会重新扫描目录，这是一种朴素但有效的"热加载"实现。

### E.2 白名单机制：启禁隔离

**文件**: `lib/Configuration.php:46-53`

虽然没有用户级隔离，但可以通过 `whitelist.txt`（旧式）或 `enabled_bridges` 配置实现**桥接器级别的启禁隔离**。

```
whitelist.txt 内容：
*                      ← 全部启用
# 或逐行列出：
CssSelectorBridge
FilterBridge
Youtube
```

白名单文件与 `config.ini.php` 的关系：白名单文件存在时**覆盖**配置中的 `enabled_bridges`。

### E.3 自定义桥接器的实践方式

如果用户需要添加自定义桥接器，标准做法是：

1. 将 `XxxBridge.php` 文件放入 `bridges/` 目录
2. 确保类继承 `BridgeAbstract` 并符合命名约定
3. 在 `config.ini.php` 的 `enabled_bridges` 中添加（若不是 `*` 模式）
4. 刷新页面即可看到

不需要重启服务，不需要注册命令，也不需要清理缓存（除非 Feed 结果已被缓存）。

### E.4 contrib/ 目录

项目根目录有一个 `contrib/` 目录（目前只有 `.gitkeep` 占位文件），从命名和开源项目惯例来看，这是预留的"用户贡献/自定义"目录，但**当前代码并未扫描该目录**。桥接器只能放在 `bridges/` 目录下才能被发现。

### E.5 隔离现状与局限

| 隔离维度 | 是否支持 | 说明 |
|---------|---------|------|
| 代码隔离 | 部分 | 每个桥接器一个类文件，但共享全局命名空间和 autoloader |
| 权限隔离 | 不支持 | 所有桥接器运行在同一进程、同一权限下 |
| 资源隔离 | 不支持 | 共享缓存、共享 HTTP 客户端、共享内存限制 |
| 启禁隔离 | 支持 | 通过 `enabled_bridges` / `whitelist.txt` 控制哪些桥接器可见 |
| 配置隔离 | 支持 | 每个桥接器有独立的配置段（如 `[TelegramBridge] max_pages = 1`） |
| 缓存隔离 | 自动 | 缓存 key 自动带桥接器短名前缀（`loadCacheValue`） |

**安全注意**：由于桥接器是原生 PHP 代码，自定义桥接器拥有与主程序完全相同的权限（可读写文件、发起网络请求等），添加不受信任的桥接器存在安全风险。

### E.6 与 WordPress 插件模式的对比

RSS-Bridge 的桥接器模式与 WordPress 插件有本质区别：

| 特性 | RSS-Bridge 桥接器 | WordPress 插件 |
|------|------------------|---------------|
| 注册机制 | 文件系统约定（自动扫描） | 插件头注释 + 主动激活 |
| 生命周期 | 请求级（每次重新扫描） | 持久化（激活状态存数据库） |
| 隔离程度 | 弱（同进程同权限） | 弱（同进程同权限，但有钩子机制） |
| 热加载 | 天然支持（PHP 无状态） | 需要手动激活/停用 |
| 管理界面 | 无（改配置文件） | 有后台插件管理页 |

---

## 补充 F：桥接器抓取失败的重试与降级链路

抓取失败是 RSS-Bridge 的常见场景（目标站点改版、限流、网络波动等）。系统设计了一套分层重试与降级机制，从底层 HTTP 到上层 Feed 输出逐层兜底。

### F.1 第一层：HTTP 请求重试

**文件**: `lib/http.php:170-192`

最底层的 `CurlHttpClient::request()` 内置了网络级重试：

```php
$tries = 0;
while (true) {
    $tries++;
    $body = curl_exec($ch);
    if ($body !== false) {
        break;  // 请求成功
    }
    if ($tries <= $config['retries']) {
        continue;  // 继续重试
    }
    // 达到最大重试次数，抛异常
    throw new HttpException(sprintf('cURL error %s: %s ...', ...));
}
```

**重试规则**：
- 触发条件：`curl_exec()` 返回 `false`（网络连接失败、超时、DNS 解析失败等）
- 重试次数：由 `http.retries` 配置控制，默认 1 次（即总共尝试 2 次：初始 + 1 次重试）
- 重试间隔：无退避，立即重试
- 不重试的情况：HTTP 错误状态码（如 404、500）不会触发重试，因为 `curl_exec` 不会返回 false
- 最大重定向：`max_redirections` 默认 5 次

### F.2 第二层：HTTP 缓存降级（304 / 过期缓存）

**文件**: `lib/contents.php:73-89, 126-129`

当本地有缓存但资源可能过期时，`getContents()` 会发送条件请求实现"软降级"：

```
有本地缓存
    │
    ▼
发条件请求（带 If-Modified-Since / If-None-Match）
    ├─ 200 OK → 用新内容，更新缓存
    └─ 304 Not Modified → 用缓存内容（静默降级，用户无感知）
```

如果目标站点完全不可达，**这层不会直接降级返回过期缓存**，而是直接抛异常（因为 `curl_exec` 失败 → 重试 → 抛 HttpException）。

### F.3 第三层：Feed 响应缓存降级

**文件**: `middlewares/CacheMiddleware.php:48-50`

`CacheMiddleware` 对错误响应也做缓存（5~15 分钟随机 TTL），防止错误雪崩：

```php
} elseif (in_array($response->getCode(), [400, 403, 404, 429, 500, 503])) {
    // Cache these responses for about ~10 mins on average
    $this->cache->set($cacheKey, $response, 60 * 5 + rand(1, 60 * 10));
}
```

但这是**缓存错误**而不是降级返回旧数据。如果桥接器本次抓取失败，用户会看到错误，而不是上一次成功的 Feed。

### F.4 第四层：错误报告阈值与输出模式

**文件**: `actions/DisplayAction.php:107-123`

真正的"降级"策略体现在错误输出模式上。系统不会一出错就把错误暴露给用户，而是通过 `error.report_limit` 控制错误报告频率。

#### 错误计数器

```php
// 记录错误次数（缓存中存储，TTL 5 天）
$cacheKey = 'error_reporting_' . $bridgeName . '_' . $code;
$report = $this->cache->get($cacheKey);
if ($report) {
    $report = Json::decode($report);
    $report['count']++;
} else {
    $report = ['error' => $code, 'time' => time(), 'count' => 1];
}
$this->cache->set($cacheKey, Json::encode($report), 86400 * 5);
```

#### 三种错误输出模式

由 `error.output` 配置控制：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `feed`（默认） | 错误次数达到 `report_limit` 后，在 Feed 中插入一条错误项作为降级 | 生产环境，保持 Feed 可用 |
| `http` | 错误次数达到 `report_limit` 后，直接返回 HTTP 500 错误页 | 调试环境，快速发现问题 |
| `none` | 静默忽略错误，返回空 Feed | 对可用性要求极高的场景 |

`report_limit` 默认值为 1（即每次出错都报告）。调高此值可以过滤偶发错误，只在持续失败时才通知用户。

### F.5 限流异常的特殊处理

**文件**: `lib/utils.php:264-267` + `actions/DisplayAction.php:94-96`

桥接器可以通过 `throwRateLimitException()` 主动抛出限流异常，这是一种**受控失败**：

```php
function throwRateLimitException(string $message = '')
{
    throw new RateLimitException($message);
}
```

`DisplayAction` 捕获到 `RateLimitException` 时，直接返回 HTTP 429 状态码：

```php
} elseif ($e instanceof RateLimitException) {
    $this->logger->debug(...);
    return new Response(render(...), 429);
}
```

这比通用异常多了一层语义——告诉调用方"这是限流，不是程序 bug"，方便上层做退避重试。

### F.6 CloudFlare 检测

**文件**: `lib/http.php:38-56`

系统内置了 CloudFlare 拦截检测，通过响应体中的 `<title>` 标签判断：

```php
final class CloudFlareException extends HttpException
{
    public static function isCloudFlareResponse(Response $response): bool
    {
        $cloudflareTitles = [
            '<title>Just a moment...',
            '<title>Please Wait...',
            '<title>Attention Required!',
            '<title>Security | Glassdoor',
            '<title>Access denied</title>',
        ];
        // ...
    }
}
```

`HttpException::fromResponse()` 会自动识别 CloudFlare 拦截并返回 `CloudFlareException` 子类。目前代码中没有针对 CloudFlare 的特殊降级逻辑（如自动切换代理），但子类化为后续扩展预留了空间。

### F.7 完整失败链路时序

```
桥接器 collectData() 发起请求
    │
    ▼
getContents()
    │
    ├─ 查缓存 → 命中 → 发条件请求 → 304 → 用缓存（成功路径）
    │
    └─ 网络请求
         │
         ├─ curl_exec 失败 → 重试 N 次 → 仍失败 → 抛 HttpException
         │
         └─ HTTP 错误码（4xx/5xx）→ 抛 HttpException
              │
              ▼
DisplayAction catch
    │
    ├─ RateLimitException → 返回 429 + 异常页
    ├─ ClientException    → 打 debug 日志
    └─ 其他 Exception      → 打 error 日志
         │
         ├─ 检查错误次数 < report_limit → 静默（返回空 Feed？不，实际会抛出）
         └─ 错误次数 >= report_limit
              │
              ├─ error.output = feed → 错误项插入 Feed
              ├─ error.output = http → 返回 500 异常页
              └─ error.output = none → 空 Feed
```

### F.8 关键局限

1. **无 stale-while-revalidate**：缓存过期后若目标站点不可达，不会返回过期内容兜底
2. **无熔断机制**：某个桥接器持续失败不会自动"熔断"停用，每次请求都会重试
3. **无退避策略**：HTTP 重试是立即重试，没有指数退避
4. **无降级数据**：失败时不会返回上一次成功的旧 Feed 数据（除非缓存层恰好命中）
5. **重试仅限网络层**：HTTP 429/503 等应用层限流不会触发重试
