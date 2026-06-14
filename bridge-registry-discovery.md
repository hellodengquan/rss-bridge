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
