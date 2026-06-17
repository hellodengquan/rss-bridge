# RSS-Bridge 访问控制机制：Whitelist 与 Token 鉴权协作

## 一、整体架构概览

### 1.1 请求处理流水线

请求进入 RSS-Bridge 后，按以下顺序经过中间件（`lib/RssBridge.php:25-39`），最终到达具体 Action：

```
请求 → BasicAuthMiddleware → CacheMiddleware → ExceptionMiddleware
     → SecurityMiddleware → MaintenanceMiddleware → TokenAuthenticationMiddleware
     → Action (Frontpage / Display / List / ...)
```

**关键点**：Whitelist 检查不在中间件层，而是在各 Action 内部通过 `BridgeFactory::isEnabled()` 执行。Token 鉴权在中间件层完成，通过后向 Request 注入 `token` attribute。

### 1.2 配置源与加载顺序

配置加载顺序（后者覆盖前者），见 `lib/Configuration.php:18-82`：

| 优先级 | 配置源 | 说明 |
|--------|--------|------|
| 1（最低） | `config.default.ini.php` | 默认配置，随版本更新，不可修改 |
| 2 | `config.ini.php` | 用户自定义配置 |
| 3 | `whitelist.txt` | 遗留快捷方式，仅设置 `system.enabled_bridges` |
| 4（最高） | 环境变量 `RSSBRIDGE_*` | Docker 部署常用 |

> **注意**：`whitelist.txt` 的内容仅映射到 `system.enabled_bridges`，与 config.ini.php 中同名字段等价。

---

## 二、Whitelist：管理员声明可用 Bridge 的放行规则

### 2.1 Whitelist 的三种声明方式

**方式一：config.ini.php（推荐）**

```ini
[system]
enabled_bridges[] = *              ; 启用全部 bridge
; 或按需启用
;enabled_bridges[] = TwitchBridge
;enabled_bridges[] = TwitterBridge
```

**方式二：whitelist.txt（遗留）**

```bash
echo '*' > whitelist.txt                    # 全部启用
echo -e "TwitchBridge\nTwitterBridge" > whitelist.txt   # 指定启用
```

**方式三：环境变量**

```bash
RSSBRIDGE_system_enabled_bridges="TwitchBridge,TwitterBridge"
```

### 2.2 BridgeFactory 的启用判定逻辑

`lib/BridgeFactory.php:25-42` 在构造时完成桥接启用列表构建：

```
Configuration::getConfig('system', 'enabled_bridges')
    │
    ├─ 包含 '*'  → 将 bridges/ 目录下扫描到的所有 *Bridge.php 类名
    │              全部加入 $this->enabledBridges
    │
    └─ 列出具体名称 → 逐个 normalize（补全 Bridge 后缀、大小写匹配）
                     存在则加入 $this->enabledBridges
                     不存在则记录到 $this->missingEnabledBridges 并日志告警
```

判定接口：`BridgeFactory::isEnabled(string $bridgeName): bool`
→ 简单检查 `in_array($bridgeName, $this->enabledBridges)`

### 2.3 各 Action 中的放行检查点

Whitelist 检查**非全局中间件**，而是在各 Action 内部按需执行：

| Action | 检查位置 | 未放行行为 |
|--------|----------|-----------|
| **DisplayAction** | `actions/DisplayAction.php:36-38` | 返回 HTTP 400，消息 "This bridge is not whitelisted" |
| **FrontpageAction** | `actions/FrontpageAction.php:30-36` | 不在首页渲染 bridge 卡片，统计不计入 `active_bridges` |
| **ListAction** | `actions/ListAction.php:22-24` | **不拦截**，但将 status 标记为 `"inactive"`，前端可自行过滤 |

> **设计意图**：DisplayAction 是核心数据出口，必须严格校验；Frontpage 仅展示过滤；ListAction 提供全景视图供 API 客户端自行判断。

---

## 三、Token 模式下访问者身份识别

### 3.1 Token 鉴权触发条件

在 `middlewares/TokenAuthenticationMiddleware.php:9-11`：

```php
if (! Configuration::getConfig('authentication', 'token')) {
    return $next($request);  // token 为空字符串 → 直接放行，不鉴权
}
```

即：**当且仅当 config 中 `authentication.token` 被设置为非空值时，Token 模式才激活**。

### 3.2 Token 校验流程

```
请求到达 TokenAuthenticationMiddleware
    │
    ├─ config 中 authentication.token 为空？
    │     └─ 是 → 放行，不做任何处理
    │
    ├─ 请求参数中缺失 token？
    │     └─ 是 → 返回 HTTP 401，渲染 token.html.php 输入表单（"Missing token"）
    │
    ├─ hash_equals(config.token, 请求token) 不匹配？
    │     └─ 是 → 返回 HTTP 401，渲染 token.html.php 输入表单（"Invalid token"）
    │
    └─ 全部通过
          ├─ 将 token 写入 Request attribute: $request->withAttribute('token', $token)
          └─ 调用 $next($request) 放行
```

**安全细节**：使用 `hash_equals()` 做时序安全比较（`middlewares/TokenAuthenticationMiddleware.php:22`），防止侧信道攻击。

### 3.3 Token 通过后的身份透传

Token 验证成功后，系统**不做细粒度角色区分**（没有 admin/user 分级），仅有"已认证/未认证"二元状态。

Token 在后续流程中的两处使用：

1. **FrontpageAction 渲染表单**（`actions/FrontpageAction.php:15-16, 163-169`）：
   ```php
   $token = $request->getAttribute('token');
   // ... 若 token 模式开启且存在，则在所有 bridge 表单中注入隐藏字段
   if (Configuration::getConfig('authentication', 'token') && $token) {
       $form .= '<input type="hidden" name="token" value="{$token}" />';
   }
   ```
   → 保证用户在首页点击"Generate feed"生成的 URL 自动携带 token，无需手动拼接。

2. **DisplayAction 剥离参数**（`actions/DisplayAction.php:77`）：
   ```php
   $remove = ['token', 'action', 'bridge', 'format', ...];
   $input = array_diff_key($requestArray, array_fill_keys($remove, ''));
   $bridge->setInput($input);
   ```
   → token 不会传递给具体 bridge 的业务逻辑，仅用于访问控制。

### 3.4 与 Basic Auth 的协作关系

`BasicAuthMiddleware` 排在流水线**最前端**（`lib/RssBridge.php:26`），Token 中间件排在**最后**。两者为**独立开关**的并行关系：

| 场景 | authentication.enable | authentication.token | 效果 |
|------|----------------------|----------------------|------|
| 无鉴权 | false | 空 | 全部放行 |
| 仅 Basic Auth | true + 用户名/密码 | 空 | 需通过 HTTP Basic，通过后无需 token |
| 仅 Token | false | 非空 | 需通过 URL ?token=xxx |
| 两者同开 | true + 用户名/密码 | 非空 | **先过 Basic，再过 Token**，两道关卡 |

---

## 四、Whitelist 开关切换时的行为差异

### 4.1 维度一：从「全部放行」到「部分放行」

**状态迁移**：`enabled_bridges = ['*']` → `enabled_bridges = ['TwitterBridge', 'TwitchBridge']`

| 受影响模块 | 行为变化 |
|-----------|---------|
| **首页（FrontpageAction）** | 所有未列入白名单的 bridge 卡片消失，`active_bridges` 计数减少 |
| **DisplayAction** | 已保存的其他 bridge feed URL 立即返回 HTTP 400 "not whitelisted" |
| **ListAction** | status 字段从 `"active"` 变为 `"inactive"`，JSON 结构不变 |
| **BridgeFactory 启动** | 若列表中包含不存在的 bridge 名，记录 warning 级日志，不中断启动 |

### 4.2 维度二：从「部分放行」到「全部放行」

**状态迁移**：`enabled_bridges = ['TwitterBridge']` → `enabled_bridges = ['*']`

| 受影响模块 | 行为变化 |
|-----------|---------|
| **首页** | bridges/ 目录下全部 bridge 被渲染，`active_bridges` = 总数 |
| **DisplayAction** | 所有历史保存的 feed URL 恢复可访问 |
| **ListAction** | 所有 bridge status 变为 `"active"` |

### 4.3 维度三：新增白名单条目

**场景**：`enabled_bridges = ['TwitterBridge']` → 添加 `TwitchBridge`

| 检查点 | 行为 |
|--------|------|
| 名称规范化 | 支持 `Twitch` / `TwitchBridge` / `twitchbridge`，大小写不敏感（`lib/BridgeFactory.php:54-64`） |
| 缺失容忍 | 若填写拼写错误的 bridge 名，仅记录 info 日志到 `missingEnabledBridges`，首页顶部展示 Warning 横幅 |
| 生效时机 | 下次 HTTP 请求时自动生效（BridgeFactory 每次请求重新构造），无需重启服务 |

### 4.4 维度四：移除白名单条目 vs Token 失效的对比

| 现象 | 移除 Whitelist 条目 | Token 失效/缺失 |
|------|-------------------|-----------------|
| HTTP 状态码 | 400 Bad Request | 401 Unauthorized |
| 返回页面 | error.html.php 模板 | token.html.php 表单模板 |
| 错误消息 | "This bridge is not whitelisted" | "Missing token" / "Invalid token" |
| 首页表现 | bridge 卡片消失 | 展示 token 输入表单，无 bridge 列表 |
| 缓存行为 | CacheMiddleware 缓存 5~15 分钟（400 错误码） | CacheMiddleware 缓存 5~15 分钟（401 错误码） |

---

## 五、协作场景完整示例

### 场景 A：标准私用部署（Token + 全量 Bridge）

**配置**：
- `authentication.token = "my-secret-token"`
- `enabled_bridges[] = *`

**访问链路**：
```
用户访问 /?token=my-secret-token
  → BasicAuthMiddleware（跳过，authentication.enable=false）
  → TokenAuthenticationMiddleware（校验通过，注入 attribute）
  → FrontpageAction
       → 遍历所有 bridge，isEnabled() 全部为 true
       → 渲染所有卡片，每个表单内嵌 <input type=hidden name=token>
用户点击某 bridge Generate feed
  → 跳转到 /?action=display&bridge=Xxx&format=Html&token=my-secret-token
  → TokenAuthenticationMiddleware（通过）
  → DisplayAction
       → isEnabled("XxxBridge") = true
       → 正常返回 feed 数据
```

### 场景 B：公开只读部署（无 Token + 白名单限制）

**配置**：
- `authentication.token = ""` （空）
- `enabled_bridges[] = TwitterBridge`

**访问链路**：
```
匿名用户访问 /
  → TokenAuthenticationMiddleware（token 为空，直接放行）
  → FrontpageAction
       → 仅 isEnabled("TwitterBridge") = true，其他 bridge 不渲染
匿名用户构造 ?action=display&bridge=TwitchBridge
  → TokenAuthenticationMiddleware（放行）
  → DisplayAction
       → isEnabled("TwitchBridge") = false
       → 返回 400 "This bridge is not whitelisted"
```

### 场景 C：严格企业部署（Basic + Token + 部分白名单）

**配置**：
- `authentication.enable = true`，`username/password` 已设置
- `authentication.token = "corp-token-xyz"`
- `enabled_bridges[] = GithubIssueBridge`

**访问链路**：
```
请求进入
  → BasicAuthMiddleware 检查 Authorization 头
       → 缺失 → 401 + WWW-Authenticate: Basic 弹窗
       → 错误 → 401 重试
       → 正确 → 继续
  → TokenAuthenticationMiddleware 检查 ?token=
       → 缺失/错误 → 401 + token 输入表单
       → 正确 → 注入 attribute
  → DisplayAction 请求 GithubIssueBridge
       → isEnabled = true → 正常返回数据
  → DisplayAction 请求 TwitterBridge
       → isEnabled = false → 400 not whitelisted
```

---

## 六、关键代码索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|---------|---------|
| 中间件流水线编排 | `lib/RssBridge.php` | 25-39 |
| Whitelist 文件加载 | `lib/Configuration.php` | 46-53 |
| enabled_bridges 校验 | `lib/Configuration.php` | 88-90 |
| 环境变量 enabled_bridges 解析 | `lib/Configuration.php` | 71-74 |
| BridgeFactory 白名单构建 | `lib/BridgeFactory.php` | 25-42 |
| BridgeFactory isEnabled | `lib/BridgeFactory.php` | 49-52 |
| Bridge 名称规范化 | `lib/BridgeFactory.php` | 66-75 |
| DisplayAction 白名单检查 | `actions/DisplayAction.php` | 36-38 |
| DisplayAction 剥离 token | `actions/DisplayAction.php` | 77 |
| FrontpageAction 过滤未启用 bridge | `actions/FrontpageAction.php` | 30-36 |
| FrontpageAction 表单注入 token | `actions/FrontpageAction.php` | 163-169 |
| ListAction status 标记 | `actions/ListAction.php` | 22-24 |
| Token 中间件主逻辑 | `middlewares/TokenAuthenticationMiddleware.php` | 7-32 |
| Basic Auth 中间件 | `middlewares/BasicAuthMiddleware.php` | 10-37 |
| CacheMiddleware 错误缓存策略 | `middlewares/CacheMiddleware.php` | 46-54 |
