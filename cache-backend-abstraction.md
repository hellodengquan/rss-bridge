# RSS-Bridge 缓存后端抽象机制

## 概述

RSS-Bridge 的缓存系统采用了**接口抽象 + 工厂模式**的设计，使不同存储后端（文件、SQLite、Memcached 等）通过统一的 `CacheInterface` 接入抓取流程。上层业务代码只依赖接口，不依赖具体实现，从而实现后端的可替换性。

---

## 一、核心接口：CacheInterface

所有缓存后端都必须实现 `CacheInterface`（`lib/CacheInterface.php`）：

```php
interface CacheInterface
{
    public function get(string $key, $default = null);
    public function set(string $key, $value, ?int $ttl = null): void;
    public function delete(string $key): void;
    public function clear(): void;
    public function prune(): void;
}
```

| 方法 | 职责 |
|------|------|
| `get($key, $default)` | 按 key 读取缓存，未命中或已过期返回 `$default` |
| `set($key, $value, $ttl)` | 写入缓存，`$ttl` 为秒数，`null` 表示永不过期，`0` 表示不缓存 |
| `delete($key)` | 删除指定 key |
| `clear()` | 清空所有缓存 |
| `prune()` | 清理已过期的缓存条目 |

**关键约定**：所有后端在 `set()` 中对 `$ttl` 的处理逻辑一致——`null` 永久存储，`0` 不存储，正整数表示存活秒数。

---

## 二、工厂模式：CacheFactory

`CacheFactory`（`lib/CacheFactory.php`）负责根据配置选择并实例化具体的缓存后端。

### 2.1 发现机制

工厂通过扫描 `caches/` 目录（常量 `PATH_LIB_CACHES`）自动发现可用的缓存类：

```php
foreach (scandir(PATH_LIB_CACHES) as $file) {
    if (preg_match('/^([^.]+)Cache\.php$/U', $file, $m)) {
        $cacheNames[] = $m[1];
    }
}
```

匹配规则：文件名以 `Cache.php` 结尾，前缀部分即为缓存名称。例如 `FileCache.php` → 名称 `File`。

### 2.2 名称规范化

用户传入的名称会经过两次裁剪：

1. 去掉 `.php` 后缀（如果有的话）
2. 去掉 `Cache` 后缀（如果有的话）

这意味着 `"File"`、`"FileCache"`、`"FileCache.php"` 都能匹配到 `FileCache` 类。

### 2.3 实例化分支

工厂的 `create()` 方法通过 `switch` 语句对已知后端做定制化构造，`default` 分支作为通用兜底：

```
┌─────────────────┐
│  create($name)  │
└────────┬────────┘
         │
    名称规范化 & 发现
         │
    ┌────▼────┐
    │ switch  │
    └────┬────┘
         │
   ┌─────┼──────────────┬───────────────┬──────────────┬────────────┐
   │     │              │               │              │            │
   ▼     ▼              ▼               ▼              ▼            ▼
NullCache  FileCache   SQLiteCache   MemcachedCache  ...      default
(无参)   (logger,      (logger,      (logger,              (无参构造)
          config)       config)       host, port)
```

- **NullCache**：空实现，所有操作均为空操作，适用于禁用缓存的场景
- **FileCache**：需要 `path`（缓存目录）和 `enable_purge` 配置
- **SQLiteCache**：需要 `file`（数据库文件路径）、`timeout`、`enable_purge` 配置，且依赖 `sqlite3` 扩展
- **MemcachedCache**：需要 `host` 和 `port` 配置，且依赖 `memcached` 扩展
- **default**：对未在 switch 中特殊处理的缓存类，使用无参构造（如 `ArrayCache`）

每个分支在构造前都会做**前置检查**（目录是否存在且可写、PHP 扩展是否加载、配置是否完整），不满足则抛出异常。

### 2.4 配置驱动

工厂的选择由配置文件中 `cache.type` 决定，在 DI 容器（`lib/dependencies.php`）中完成绑定：

```php
$container['cache'] = function ($c) {
    $cacheFactory = $c['cache_factory'];
    return $cacheFactory->create(Configuration::getConfig('cache', 'type'));
};
```

整个应用生命周期内只创建一个缓存实例，作为单例注入到各组件中。

---

## 三、五个缓存后端实现对比

| 后端 | 存储 | key 哈希 | 过期管理 | prune 实现 | 依赖 |
|------|------|----------|----------|------------|------|
| **NullCache** | 无 | — | — | 空操作 | 无 |
| **ArrayCache** | 内存数组 | 原始 key | TTL + 过期判断 | 遍历删除过期项 | 无 |
| **FileCache** | 磁盘文件 | MD5 | TTL + 过期判断 | 遍历目录删除过期文件 | 可写目录 |
| **SQLiteCache** | SQLite 数据库 | SHA1 (raw) | TTL + 过期判断 | SQL `DELETE WHERE updated <= now` | sqlite3 扩展 |
| **MemcachedCache** | Memcached 服务 | SHA1 (hex) | 原生 TTL | 空操作（服务端自管理） | memcached 扩展 |

### 3.1 key 哈希策略差异

- **ArrayCache**：直接使用原始 key 作为数组下标
- **FileCache**：`md5($key) . '.cache'` 作为文件名
- **SQLiteCache**：`sha1($key, raw_binary)` 作为 BLOB 主键
- **MemcachedCache**：`sha1($key)` 作为 hex 字符串 key

### 3.2 过期判断差异

- **FileCache / ArrayCache / SQLiteCache**：在 `get()` 时判断过期，过期则删除并返回默认值
- **MemcachedCache**：依赖 Memcached 服务端原生 TTL 机制，`get()` 返回 `false` 即视为未命中
- **NullCache**：永不存在命中

### 3.3 TTL 为 0 的语义

所有后端一致：`set()` 时若 `$ttl === 0`，直接 `return`，不写入任何数据。

### 3.4 过期时的清理行为差异

| 后端 | `get()` 发现过期后 | 是否立即删除 |
|------|-------------------|------------|
| **ArrayCache** | 返回 `$default` | 是（`unset`） |
| **FileCache** | 返回 `$default` | 是（`unlink` 文件） |
| **SQLiteCache** | 返回 `$default` | **否**（保留过期记录，靠 `prune()` 清理） |
| **MemcachedCache** | 由服务端处理，客户端感知不到 | 服务端自行管理 |

> **注意**：SQLiteCache 在 `get()` 中发现过期条目时仅注释了 `// delete?` 但并未实际执行删除操作，过期数据会一直保留直到 `prune()` 被调用。这是一个潜在的存储空间浪费点。

---

## 四、缓存键命名空间设计与碰撞防护

RSS-Bridge 通过**多层前缀 + 后端哈希**的组合策略管理缓存键命名空间，不同业务层使用不同的前缀，避免键名冲突。

### 4.1 命名空间分层

| 层级 | 前缀 | 用途 | 示例 key |
|------|------|------|---------|
| HTTP 响应层 | `http_` | 完整 HTTP 响应缓存（Middleware + DisplayAction 共享） | `http_{...json...}` |
| 错误计数层 | `error_reporting_` | Bridge 错误报告计数 | `error_reporting_YoutubeBridge_500` |
| 服务端响应层 | `server_` | 原始 HTTP 服务端响应缓存（getContents） | `server_https://example.com/feed_null` |
| 页面内容层 | `pages_` | 解析后 HTML 内容缓存（getSimpleHTMLDOMCached） | `pages_https://example.com/page` |
| Bridge 业务层 | `{BridgeShortName}_` | 各 Bridge 内部数据缓存（自动前缀） | `YoutubeBridge_token` |
| 全局业务层 | 无前缀 | 直接使用裸 key 的全局缓存（TwitterClient 等） | `twitter` |

### 4.2 Bridge 业务层的自动命名空间

`BridgeAbstract` 提供的便捷方法会自动加上 Bridge 短名前缀：

```php
// 实际 key 为 "YoutubeBridge_rate_limit"
$this->loadCacheValue('rate_limit');
$this->saveCacheValue('rate_limit', $data, 3600);
```

这层防护确保不同 Bridge 即使使用相同的业务 key（如 `token`、`rate_limit`）也不会互相冲突。

### 4.3 存储层哈希防护

在业务 key 之下，各缓存后端还会对 key 做一次哈希转换，实现第二层防护：

| 后端 | 哈希方式 | 作用 |
|------|---------|------|
| **FileCache** | `md5($key)` | 将任意长度的 key 转为固定长度文件名，避免文件系统路径/文件名限制 |
| **SQLiteCache** | `sha1($key, raw)` | 将任意长度的 key 转为 20 字节 BLOB 主键，节省索引空间 |
| **MemcachedCache** | `sha1($key)` | 将任意长度的 key 转为 40 字符 hex 字符串，符合 Memcached key 长度限制（250 字节） |
| **ArrayCache** | 原始 key | 内存数组直接使用，无需转换 |

**碰撞防护原理**：
- MD5：128 位输出，碰撞概率约 2^(-64)（生日悖论）
- SHA-1：160 位输出，碰撞概率约 2^(-80)
- 在 RSS-Bridge 的使用规模下（几千到几万条缓存），哈希碰撞可以忽略不计

### 4.4 命名空间的潜在风险

并非所有代码都遵循了命名空间约定，存在一些"裸 key"的使用：

- **TwitterClient**：直接使用 `'twitter'` 作为 key（全局单例数据）
- **部分 Bridge 直接调用 `$this->cache->get/set()`**：如 `InstagramBridge` 用 `'InstagramBridge_' . $username`、`ElloBridge` 用 `'ElloBridge_key'`、`SoundcloudBridge` 用 `'SoundCloudBridge_client_id'`。这些 Bridge 手动加了前缀，但格式不统一
- **BlueskyBridge**：直接用 URL 作为 key（如 `https://public.api.bsky.app/...`）

这些裸 key 虽然在当前业务中没有冲突风险，但如果未来新增的缓存 key 与之重名，可能导致难以排查的 bug。

---

## 五、缓存失效回源策略

RSS-Bridge 采用 **Cache-Aside（旁路缓存）** 模式：应用代码主动管理缓存的读写，缓存不主动回源。缓存未命中或过期时，**直接回源抓取**，不提供"回退到上一份缓存"的降级机制。

### 5.1 各层级的回源行为

#### 5.1.1 HTTP 响应层（CacheMiddleware + DisplayAction）

```
请求到达
   │
   ▼
CacheMiddleware::get($key)
   │
   ├─ 命中且未过期 ──→ 直接返回缓存响应
   │
   └─ 未命中 / 已过期
            │
            ▼
      执行 DisplayAction（真实抓取）
            │
            ▼
      成功 → DisplayAction 写入缓存（TTL = Bridge::CACHE_TIMEOUT）
      失败 → CacheMiddleware 写入缓存（TTL = 5~15 分钟随机）
```

**特点**：
- 失败响应也会被缓存（短 TTL），避免对出错的 Bridge 反复执行
- 成功响应的 TTL 由各个 Bridge 自行定义（`const CACHE_TIMEOUT`，默认 3600 秒）

#### 5.1.2 服务端响应层（getContents）

`getContents()` 的回源策略最特殊——它实现了**条件请求 + 304 协商**的智能回源：

```
getContents($url)
   │
   ▼
cache->get("server_$url")
   │
   ├─ 命中（可能已过期）
   │     │
   │     ├─ 提取 Last-Modified / ETag
   │     └─ 发起条件请求（If-Modified-Since / If-None-Match）
   │           │
   │           ├─ 200 OK → 用新响应更新缓存，返回新内容
   │           └─ 304 Not Modified → 复用缓存中的 body，不更新缓存
   │
   └─ 未命中
         │
         ▼
      发起普通请求
         │
         ├─ 200/201/202 → 写入缓存（TTL = 10 天），返回内容
         └─ 其他 → 抛出 HttpException
```

**关键洞察**：
- `getContents()` **不检查缓存是否过期**——即使缓存已过期，只要存在就会用于条件请求
- 这是一种"过期但仍有用"的策略：过期缓存作为条件请求的验证凭据，若服务端返回 304 则直接复用，节省带宽
- 只有当服务端返回 200 时才会更新缓存内容和 TTL
- 若服务端返回 `Cache-Control: no-cache` / `no-store`，则完全不缓存

#### 5.1.3 页面内容层（getSimpleHTMLDOMCached）

```
getSimpleHTMLDOMCached($url, $ttl)
   │
   ▼
cache->get("pages_$url")
   │
   ├─ 命中且未过期 → 直接返回解析后的 DOM
   │
   └─ 未命中 / 已过期 → 调用 getContents() 抓取 → 解析 DOM → 写入缓存 → 返回
```

**注意**：`getSimpleHTMLDOMCached` 有自己独立的缓存（`pages_` 前缀），而 `getContents` 内部又有一层缓存（`server_` 前缀），两者形成**两级缓存**。如果 `pages_` 缓存失效但 `server_` 缓存还有效，实际只需要重新做 DOM 解析，不需要真的发 HTTP 请求。

#### 5.1.4 Bridge 内部缓存

Bridge 内部缓存的回源策略完全由各个 Bridge 自行决定，常见模式：

| 模式 | 说明 | 示例 |
|------|------|------|
| **令牌缓存** | 缓存 API token，过期后重新申请 | SpotifyBridge、ElloBridge |
| **速率限制标记** | 触发限流时写入一个短 TTL 标记，后续请求检查到标记直接跳过 | YoutubeBridge、SpotifyBridge、RedditBridge |
| **标识映射** | 缓存用户名到 ID 的映射，长期有效 | InstagramBridge、BlueskyBridge |
| **数据缓存** | 缓存中间抓取结果，减少重复请求 | PepperBridgeAbstract（标题缓存） |

### 5.2 有没有"降级到上一份缓存"？

**没有。** RSS-Bridge 所有缓存层级都不提供"缓存失效但回源失败时，回退到过期缓存"的降级策略。

唯一的例外是 `getContents()` 的 304 协商机制——但那是**主动验证**而非**被动降级**：它会真的发请求去问服务端"内容变了吗"，只有服务端说"没变（304）"时才用旧缓存。如果服务端挂了或返回错误，`getContents()` 会直接抛出异常，不会回退到缓存。

**业务影响**：
- 若目标网站暂时不可用，用户会直接看到错误页面，而不是上一次成功抓取的旧 Feed
- 这种设计保证了数据的新鲜度，但牺牲了一定的可用性
- 如果需要" stale-while-revalidate "（过期时先返回旧数据，后台异步更新）之类的策略，需要自行在 Bridge 层实现

---

## 六、各缓存后端性能对比与选型建议

### 6.1 性能维度对比

| 维度 | NullCache | ArrayCache | FileCache | SQLiteCache | MemcachedCache | Redis（假想） |
|------|-----------|------------|-----------|-------------|----------------|--------------|
| **读取延迟** | ~0 | ~0.01ms | 0.1~1ms | 0.05~0.5ms | 0.1~1ms | 0.1~1ms |
| **写入延迟** | ~0 | ~0.01ms | 0.5~5ms | 0.1~1ms | 0.1~1ms | 0.1~1ms |
| **并发能力** | — | 单进程 | 差（文件锁） | 中（WAL 模式） | 好 | 好 |
| **数据容量** | — | 内存限制 | 磁盘限制 | 磁盘限制 | 内存限制 | 内存+持久化 |
| **prune 开销** | — | O(n) 内存遍历 | O(n) 磁盘遍历 | O(1) SQL 删除 | 0（自动） | 0（自动） |
| **部署复杂度** | 无 | 无 | 低 | 低（需扩展） | 中（需服务） | 中（需服务） |
| **多进程共享** | — | 否 | 是 | 是 | 是 | 是 |

> **说明**：以上数据为典型场景估算，实际性能取决于硬件、数据量、网络延迟等因素。

### 6.2 各后端适用场景详解

#### FileCache — 适合小流量单机部署

**优势**：
- 零依赖，只需一个可写目录
- 数据持久化，重启不丢失
- 每个 key 一个文件，便于排查问题

**劣势**：
- 大量小文件导致 inode 消耗高
- prune 时需要遍历整个目录，O(n) 开销
- 并发写入存在竞态（无文件锁保护）
- 文件系统 I/O 瓶颈明显

**选型建议**：
- 个人使用、日请求量 < 1 万的场景
- Bridge 数量少、缓存条目不多的场景
- 快速上手、不想安装额外服务的场景

#### SQLiteCache — 中等流量的均衡选择

**优势**：
- 单文件存储，不耗 inode
- 支持索引，`get()` 为 O(log n)
- `prune()` 为 SQL 条件删除，效率远高于 FileCache
- WAL 模式支持并发读
- 数据持久化

**劣势**：
- 写入是串行的（SQLite 写锁独占）
- 高写入量下可能成为瓶颈
- 数据库文件可能膨胀（需要定期 `VACUUM`）

**选型建议**：
- 日请求量 1 万 ~ 10 万的中等流量
- 缓存条目多（> 1 万）、prune 频繁的场景
- 希望比 FileCache 性能好但又不想部署额外服务的场景

#### MemcachedCache — 高并发分布式场景

**优势**：
- 纯内存操作，读写延迟低
- 服务端自动管理过期，无需 prune
- 支持分布式横向扩展
- 并发能力强

**劣势**：
- 数据不持久化，重启丢失
- 内存容量有限
- 需要部署和维护 Memcached 服务
- 缓存穿透（全量 miss 时可能打垮后端）

**选型建议**：
- 日请求量 > 10 万的高流量场景
- 多台 RSS-Bridge 实例共享缓存的部署
- 能容忍冷启动时缓存全空的场景

#### ArrayCache — 测试与单次请求内缓存

**优势**：
- 速度最快（纯内存数组）
- 零依赖
- 完全可控，适合单元测试

**劣势**：
- 请求结束即销毁，无法跨请求共享
- 仅在单次 PHP 执行生命周期内有效

**选型建议**：
- 单元测试和集成测试
- 需要在单次请求内避免重复计算的场景（作为一级缓存）

#### NullCache — 禁用缓存

**适用场景**：
- 调试开发，强制每次都重新抓取
- 数据实时性要求极高，完全不能接受缓存的场景
- 测试缓存失效路径

### 6.3 选型决策树

```
需要缓存吗？
   ├─ 否 → NullCache
   │
   └─ 是
        ├─ 仅单次请求内有效？ → ArrayCache
        │
        └─ 需要跨请求共享
              ├─ 有 Memcached/Redis 服务？
              │     ├─ 有 + 高并发 + 可容忍数据丢失 → Memcached
              │     └─ 有 + 需要持久化 + 丰富功能 → Redis（需自行实现）
              │
              └─ 无外部服务
                    ├─ 缓存条目少（< 1 万） + 低写入 → FileCache
                    └─ 缓存条目多 + 较高性能要求 → SQLiteCache
```

### 6.4 关于 Redis 后端

RSS-Bridge 目前未内置 Redis 缓存实现，但添加它的成本很低（遵循 `CacheInterface` 即可）。Redis 相比 Memcached 的优势：

- 支持数据持久化（RDB / AOF）
- 更丰富的数据结构（虽然 RSS-Bridge 场景只用简单 key-value）
- 支持 TTL 精度更高
- 内置 LRU 淘汰策略
- 可通过 Redis Cluster 横向扩展

如果你的部署已经在使用 Redis 技术栈，添加 `RedisCache` 是比 Memcached 更好的选择。

---

## 七、缓存与抓取流程的关联

缓存不是孤立存在的，它在四个层次上介入抓取流程，形成完整的缓存读写链路：

```
HTTP 请求
   │
   ▼
RssBridge::main()
   │
   ├─ 中间件层 ── CacheMiddleware（HTTP 响应级缓存）
   │
   └─ Action 层 ── DisplayAction
                      │
                      ├─ Bridge 业务层 ── BridgeAbstract::loadCacheValue / saveCacheValue
                      │
                      └─ HTTP 客户端层 ── getContents()（服务端响应级缓存）
```

### 7.1 第一层：CacheMiddleware — HTTP 响应缓存

**位置**：`middlewares/CacheMiddleware.php`

**作用**：对 `DisplayAction` 的完整 HTTP 响应做缓存。

**读缓存流程**：
1. 构造缓存 key：`'http_' + json_encode(请求参数)`
2. 调用 `$this->cache->get($cacheKey)` 查找缓存
3. 若命中，检查 `If-Modified-Since` / `Last-Modified`，可能直接返回 304
4. 若未命中，继续执行后续中间件和 Action

**写缓存流程**：
- 响应码 200：由 `DisplayAction` 内部自行写入（见下文）
- 响应码 400/403/404/429/500/503：写入缓存，TTL 为 5~15 分钟随机值
- 其他响应码：写入缓存，TTL 为 5 分钟

**清理**：1% 的请求会触发 `prune()`，清理过期条目。

### 7.2 第二层：DisplayAction — Feed 数据缓存

**位置**：`actions/DisplayAction.php`

**作用**：在 Bridge 执行完毕后，将成功的响应写入缓存。

**写缓存**：
```php
$cacheKey = 'http_' . json_encode($request->toArray());
// ...
if ($response->getCode() === 200) {
    $ttl = $bridge->getCacheTimeout();  // 默认 3600 秒
    $this->cache->set($cacheKey, $response, $ttl);
}
```

**注意**：`CacheMiddleware` 和 `DisplayAction` 使用**相同的 key 前缀 `http_`** 和**相同的请求参数序列化方式**，所以它们操作的是同一条缓存记录。Middleware 负责读取，DisplayAction 负责写入，形成了读写分离的协作关系。

**错误计数**：`DisplayAction` 还使用缓存来记录 Bridge 的错误次数（key 前缀 `error_reporting_`），用于判断是否超过报告阈值。

### 7.3 第三层：getContents() — 服务端响应缓存

**位置**：`lib/contents.php`

**作用**：缓存对外部 HTTP 请求的原始响应，避免重复请求目标网站。

**读缓存流程**：
1. 构造缓存 key：`'server_' + url + requestBodyHash`
2. 读取缓存，若命中则提取 `Last-Modified` 和 `ETag`
3. 将这些值附加到新请求的 `If-Modified-Since` / `If-None-Match` 头中
4. 若服务端返回 304，用缓存中的 body 补全响应

**写缓存**：
- 响应码 200/201/202：写入缓存，TTL 为 10 天（86400 * 10），除非服务端返回 `no-cache` 或 `no-store`
- 响应码 304：不写缓存，使用缓存中的旧 body
- 其他：抛出 `HttpException`

**额外缓存**：`getSimpleHTMLDOMCached()` 函数对解析后的 HTML 内容做额外缓存（key 前缀 `pages_`），避免重复解析 DOM。

### 7.4 第四层：Bridge 内部缓存

**位置**：`lib/BridgeAbstract.php`

每个 Bridge 实例内部都持有 `CacheInterface` 实例，通过便捷方法读写：

```php
protected function loadCacheValue(string $key, $default = null) {
    return $this->cache->get($this->getShortName() . '_' . $key, $default);
}

protected function saveCacheValue(string $key, $value, int $ttl = 86400) {
    $this->cache->set($this->getShortName() . '_' . $key, $value, $ttl);
}
```

Bridge 可在 `collectData()` 中自由使用这两个方法缓存中间数据（如 token、分页信息等），key 前缀为 Bridge 短名，避免不同 Bridge 之间的 key 冲突。

---

## 八、依赖注入与对象传递链路

整个缓存实例的创建与传递过程：

```
配置: cache.type = "File"
        │
        ▼
dependencies.php
  $container['cache_factory'] = new CacheFactory($logger)
  $container['cache'] = $cacheFactory->create("File")  ──→  FileCache 实例
        │
        ├─→ CacheMiddleware($container['cache'])
        │
        ├─→ DisplayAction($container['cache'], ...)
        │
        ├─→ BridgeFactory($container['cache'], ...)
        │       │
        │       └─→ new XxxBridge($this->cache, $this->logger)
        │               │
        │               └─→ Bridge 内部使用 $this->cache
        │
        └─→ getContents() 中通过 global $container['cache'] 使用
```

**核心要点**：

1. **单一实例**：整个应用共享同一个 `CacheInterface` 实例，由 DI 容器管理生命周期
2. **接口依赖**：`CacheMiddleware`、`DisplayAction`、`BridgeFactory`、`BridgeAbstract` 的构造函数中类型声明均为 `CacheInterface`，不绑定具体实现
3. **桥接传递**：`BridgeFactory::create()` 将同一个 cache 实例注入到每个新创建的 Bridge 中
4. **全局访问**：`getContents()` 和 `getSimpleHTMLDOMCached()` 通过 `global $container` 获取缓存实例（历史遗留设计，非 DI 注入）

---

## 九、添加新缓存后端

若需添加新的缓存后端（如 Redis），只需：

1. 在 `caches/` 目录下创建 `RedisCache.php`
2. 实现 `CacheInterface` 的 5 个方法
3. 在 `CacheFactory::create()` 的 `switch` 中添加 `RedisCache` 分支（如需特殊构造参数），或依赖 `default` 分支的无参构造
4. 在配置文件中设置 `cache.type = "Redis"`

无需修改任何上层代码（Middleware、Action、Bridge），因为它们只依赖 `CacheInterface`。

---

## 十、缓存预热与启动加载策略

### 10.1 当前行为：无预热，懒加载填充

RSS-Bridge **没有缓存预热机制**。应用启动时不会主动加载任何数据到缓存中，所有缓存条目都是由实际请求触发后逐步填充的。

启动时的缓存初始化流程：

```
请求到达
   │
   ▼
bootstrap.php
   │  加载 autoload、常量、工具函数
   │
   ▼
Configuration::loadConfiguration()
   │  解析 config.ini.php
   │  如果项目根目录存在 DEBUG 文件且内容为空 → 强制 cache.type = "array"
   │
   ▼
dependencies.php
   │  注册 DI 容器服务
   │  $container['cache'] = $cacheFactory->create(cache.type)
   │
   ▼
RssBridge::main()
   │  此时缓存实例已创建，但内部为空
   │
   ▼
CacheMiddleware::invoke()
   │  cache->get() → 未命中（空缓存）
   │
   ▼
DisplayAction::invoke()
   │  执行 Bridge 抓取
   │  cache->set() → 首次写入
   │
   ▼
后续请求
   │  逐步填充缓存
```

**关键特征**：

1. **冷启动全空**：无论使用哪个后端，首次启动时缓存都是空的
2. **按需填充**：每个缓存条目只在第一次被请求时创建
3. **无批量预热**：没有"启动时预加载所有 Bridge 的 Feed"之类的机制

### 10.2 开发模式的特殊行为

当项目根目录存在 `DEBUG` 文件（内容为空）时，`Configuration` 会自动将 `cache.type` 切换为 `array`（`lib/Configuration.php:38-43`）：

```php
if (file_exists(__DIR__ . '/../DEBUG')) {
    $debug = trim(file_get_contents(__DIR__ . '/../DEBUG'));
    if ($debug === '') {
        self::setConfig('system', 'env', 'dev');
        self::setConfig('cache', 'type', 'array');
    }
}
```

这意味着开发模式下每次请求结束后缓存自动销毁，等效于禁用缓存。开发者可以观察到每次请求都走完整抓取路径。

### 10.3 各后端启动特征

| 后端 | 启动时操作 | 冷启动表现 | 预热可能性 |
|------|-----------|-----------|-----------|
| **NullCache** | 无 | 所有请求直达后端 | 无意义 |
| **ArrayCache** | 无 | 进程首次请求全 miss | 不可跨请求预热 |
| **FileCache** | 无（不扫描目录） | 首次 `get()` 时按需读取文件 | 已有文件即预热 |
| **SQLiteCache** | 打开/创建数据库文件 | 首次 `get()` 时 SQL 查询 | 已有数据即预热 |
| **MemcachedCache** | `addServer()`（不立即连接） | 首次 `get()` 时才连接 | 已有数据即预热 |

**FileCache / SQLiteCache 的隐式预热**：由于它们使用磁盘持久化，服务重启后缓存数据不会丢失。重启后的第一次请求如果命中已有缓存文件/记录，就直接返回——这等效于"被动预热"。而 MemcachedCache 重启后数据丢失，必须从冷状态开始。

### 10.4 预热策略建议

如果业务上希望减少冷启动影响，可考虑以下方案：

| 策略 | 实现方式 | 适用场景 |
|------|---------|---------|
| **定时预热脚本** | 编写 cron 脚本，定期请求高频 Bridge 的 Feed URL | 专用部署、高可用要求 |
| **健康检查触发** | 在负载均衡器健康检查中访问核心 Bridge URL，同时完成缓存填充 | 容器化部署（K8s liveness/readiness） |
| **选择持久化后端** | 使用 FileCache / SQLiteCache 替代 MemcachedCache | 不容忍冷启动空缓存的场景 |
| **多级缓存** | 在 Bridge 层实现本地 ArrayCache + 远端持久化缓存的两级策略 | 高并发场景 |

> **注意**：预热需要遵守目标网站的速率限制。RSS-Bridge 各 Bridge 的 `CACHE_TIMEOUT` 就是为此设计的——默认 3600 秒（1 小时），预热频率不应超过此间隔。

---

## 十一、缓存后端迁移路径（File → Redis）

### 11.1 迁移的必要性

典型迁移场景：从小规模 FileCache 部署升级到 Redis，以解决：
- 文件系统 inode 耗尽
- 并发写入竞态
- prune 扫描目录的 O(n) 开销
- 多实例无法共享缓存

### 11.2 数据格式差异分析

迁移的核心难点在于各后端的**数据存储格式不同**：

| 维度 | FileCache | SQLiteCache | MemcachedCache | Redis（假想） |
|------|-----------|-------------|----------------|--------------|
| **key 编码** | `md5($key)` | `sha1($key, raw)` | `sha1($key)` | 建议 `sha1($key)` |
| **value 编码** | PHP serialize | PHP serialize | 原生（Memcached 自动序列化） | 需选择（见下文） |
| **过期存储** | 内嵌在 value 中 | 单独列 `updated` | 服务端管理 | 服务端管理（TTL） |
| **元数据** | `key`/`expiration`/`value` | `key`/`updated`/`value` | 仅 value | 仅 value |

**关键问题**：FileCache 的 key 经过了 `md5()` 哈希，而如果 Redis 使用 `sha1()` 哈希，两者无法直接映射。原始 key 在 FileCache 的 value 中有保存（`$item['key']`），可以作为迁移桥接。

### 11.3 迁移方案

#### 方案一：直接切换（推荐，零停机）

由于 RSS-Bridge 的缓存是 **Cache-Aside 模式**，缓存丢失不会导致功能异常，只会导致短暂的全量回源。因此最简单的迁移方式是：

```
1. 部署 Redis 服务
2. 实现 RedisCache 类（遵循 CacheInterface）
3. 在 CacheFactory 中注册 RedisCache
4. 修改配置 cache.type = "Redis"
5. 重启应用
```

**优点**：简单、无风险
**缺点**：冷启动期间全量回源，可能短暂增加目标网站请求量
**适用**：缓存可接受短暂空窗的场景

#### 方案二：双写迁移（平滑过渡）

如果需要平滑过渡，避免冷启动冲击：

```
阶段一：双写
┌──────────────┐     ┌──────────┐
│ Application  │────→│ FileCache │  （读）
│              │     └──────────┘
│              │────→│ RedisCache │  （写）
└──────────────┘     └──────────┘

阶段二：验证 Redis 数据完整性后，切换读源到 Redis

阶段三：移除 FileCache
```

**实现要点**：
1. 创建 `DualWriteCache` 装饰器，实现 `CacheInterface`
2. `get()` 优先从 FileCache 读（旧数据），miss 时从 Redis 读
3. `set()` 同时写入两个后端
4. 运行一段时间后，Redis 中的数据逐渐覆盖 FileCache
5. 确认 Redis 数据完整后，切换为纯 Redis

**适用**：对可用性要求极高、不能容忍冷启动空窗的场景

#### 方案三：脚本迁移（历史数据迁移）

如果需要将 FileCache 中的历史数据导入 Redis：

```php
// migrate_file_to_redis.php（示意代码）
$fileCache = new FileCache($logger, ['path' => PATH_CACHE]);
$redisCache = new RedisCache($logger, $redisConfig);

foreach (scandir(PATH_CACHE) as $filename) {
    if (!str_ends_with($filename, '.cache')) continue;

    $data = file_get_contents(PATH_CACHE . $filename);
    $item = unserialize($data);
    if ($item === false) continue;

    $originalKey = $item['key'];           // FileCache 保存了原始 key
    $expiration  = $item['expiration'];     // 0 = 永久, 否则为 unix timestamp
    $value       = $item['value'];

    $ttl = 0;
    if ($expiration > 0) {
        $ttl = $expiration - time();
        if ($ttl <= 0) continue;           // 已过期，跳过
    }

    $redisCache->set($originalKey, $value, $ttl > 0 ? $ttl : null);
}
```

**注意**：
- FileCache 的 value 中保存了原始 key（`$item['key']`），这是迁移的关键
- SQLiteCache 同样可以用类似方式迁移，从 `storage` 表读取原始 key
- MemcachedCache 不保存原始 key（无法反查），不适合作为迁移源
- 迁移脚本应在低流量时段执行

### 11.4 迁移检查清单

| 步骤 | 检查项 |
|------|--------|
| 部署前 | Redis 服务可用、PHP redis 扩展已安装、RedisCache 类已实现并通过测试 |
| 切换前 | 配置文件 `cache.type` 已更新、CacheFactory 已注册 RedisCache 分支 |
| 切换后 | HealthAction 返回 200、首个 Bridge 请求正常返回、日志无缓存相关异常 |
| 观察期 | 监控回源请求量是否逐步回落、Redis 内存使用是否稳定、TTL 是否正常过期 |
| 收尾 | 确认 FileCache 数据不再被引用后，清理旧缓存文件/目录 |

---

## 十二、缓存监控指标与命中率统计

### 12.1 当前状态：无内置监控

RSS-Bridge **没有内置的缓存命中率统计或监控指标**。`CacheInterface` 的 `get()` / `set()` 方法不记录任何计量数据，也不输出命中率、延迟等指标。

目前代码中唯一的可观测性来自日志，但仅覆盖异常场景：

| 组件 | 日志级别 | 触发条件 |
|------|---------|---------|
| `FileCache` | `warning` | 反序列化失败、写入失败 |
| `SQLiteCache` | `error` / `warning` | 反序列化失败、SQL 执行异常 |
| `MemcachedCache` | `warning` | 写入失败（附带 resultCode/resultMessage/errorCode） |
| `CacheMiddleware` | 无 | 不记录命中/未命中 |

**结论**：正常流量下，缓存操作完全"静默"，没有任何可观测的指标输出。

### 12.2 关键监控指标定义

若需构建缓存监控体系，以下指标值得关注：

#### 核心指标

| 指标 | 定义 | 计算方式 | 告警建议 |
|------|------|---------|---------|
| **命中率** | 缓存命中次数占总查询次数的比例 | `hits / (hits + misses) * 100%` | < 50% 持续 10 分钟 |
| **绝对命中量** | 单位时间内命中次数 | 计数器 | 突降至 0 可能是缓存清空 |
| **绝对未命中量** | 单位时间内未命中次数 | 计数器 | 突增可能是缓存失效 |
| **平均读取延迟** | `get()` 操作的平均耗时 | 计时器 | FileCache > 10ms, SQLiteCache > 5ms |
| **平均写入延迟** | `set()` 操作的平均耗时 | 计时器 | FileCache > 50ms |
| **缓存条目总数** | 当前存储的 key 数量 | 各后端特有方式 | 接近容量上限时告警 |

#### 分层指标

| 层级 | 指标 | 采集点 |
|------|------|--------|
| **HTTP 响应层** | Feed 命中率 / 304 返回率 | `CacheMiddleware::__invoke()` |
| **服务端响应层** | 源站请求次数 / 304 协商次数 | `getContents()` |
| **页面内容层** | DOM 解析次数 / 缓存命中次数 | `getSimpleHTMLDOMCached()` |
| **Bridge 业务层** | 各 Bridge 缓存命中率 | `BridgeAbstract::loadCacheValue()` |

#### 后端特有指标

| 后端 | 指标 | 采集方式 |
|------|------|---------|
| **FileCache** | 缓存目录文件数、磁盘使用量 | `scandir()` + `filesize()` |
| **SQLiteCache** | 数据库文件大小、记录数 | `SELECT COUNT(*) FROM storage` |
| **MemcachedCache** | 内存使用率、 eviction 数、当前连接数 | Memcached `stats` 命令 |
| **Redis** | 内存使用、key 数、hit/miss 统计 | Redis `INFO` 命令 |

### 12.3 监控实现方案

#### 方案一：装饰器模式（推荐）

在不修改 `CacheInterface` 的前提下，用装饰器包装任意缓存后端：

```php
class InstrumentedCache implements CacheInterface
{
    private CacheInterface $inner;
    private int $hits = 0;
    private int $misses = 0;
    private array $readLatencies = [];

    public function get(string $key, $default = null)
    {
        $start = microtime(true);
        $result = $this->inner->get($key, $default);
        $elapsed = microtime(true) - $start;

        $this->readLatencies[] = $elapsed;

        if ($result !== $default) {
            $this->hits++;
        } else {
            $this->misses++;
        }
        return $result;
    }

    public function getStats(): array
    {
        return [
            'hits'      => $this->hits,
            'misses'    => $this->misses,
            'hit_rate'  => $this->hits + $this->misses > 0
                ? $this->hits / ($this->hits + $this->misses)
                : 0,
            'avg_read_latency_ms' => $this->readLatencies
                ? array_sum($this->readLatencies) / count($this->readLatencies) * 1000
                : 0,
        ];
    }

    // set(), delete(), clear(), prune() 委托给 $this->inner
}
```

**接入方式**：在 DI 容器中包装原始缓存实例

```php
$container['cache'] = function ($c) {
    $rawCache = $cacheFactory->create(Configuration::getConfig('cache', 'type'));
    return new InstrumentedCache($rawCache);
};
```

**优点**：不侵入现有代码、不改变接口、可随时开启/关闭

#### 方案二：日志埋点

在关键路径添加 debug 级别日志：

```php
// CacheMiddleware 中
$cachedResponse = $this->cache->get($cacheKey);
$this->logger->debug('Cache ' . ($cachedResponse ? 'hit' : 'miss'), [
    'key' => $cacheKey,
    'layer' => 'http_response',
]);

// getContents() 中
$cachedResponse = $cache->get($cacheKey);
$this->logger->debug('Cache ' . ($cachedResponse ? 'hit' : 'miss'), [
    'key' => $cacheKey,
    'layer' => 'server_response',
]);
```

在 `dev` 环境下（`system.env = dev`），DEBUG 级别日志会输出到 `error_log`，可用于本地调试。生产环境可通过 `logging.file_path` + `logging.file_level = DEBUG` 将日志写入文件，再由外部工具（如 ElasticSearch / Grafana Loki）聚合分析。

**优点**：改动最小，利用现有日志基础设施
**缺点**：日志量大时影响性能，需要外部工具聚合

#### 方案三：Prometheus 指标暴露

添加一个 `/metrics` 端点（新 Action），暴露 Prometheus 格式的指标：

```
# TYPE rssbridge_cache_hits_total counter
rssbridge_cache_hits_total{backend="file",layer="http"} 1234
rssbridge_cache_hits_total{backend="file",layer="server"} 567

# TYPE rssbridge_cache_misses_total counter
rssbridge_cache_misses_total{backend="file",layer="http"} 89
rssbridge_cache_misses_total{backend="file",layer="server"} 12

# TYPE rssbridge_cache_latency_seconds summary
rssbridge_cache_latency_seconds{backend="file",operation="get",quantile="0.5"} 0.0003
rssbridge_cache_latency_seconds{backend="file",operation="get",quantile="0.99"} 0.0021
```

**适用**：已有 Prometheus + Grafana 监控体系的生产部署

### 12.4 外部监控（无需改代码）

在不修改 RSS-Bridge 代码的前提下，可通过以下方式监控缓存状态：

| 方法 | 监控对象 | 实现方式 |
|------|---------|---------|
| **文件系统监控** | FileCache 文件数/目录大小 | `find cache/ -name '*.cache' \| wc -l` + `du -sh cache/` |
| **SQLite 数据库监控** | SQLiteCache 记录数/文件大小 | `sqlite3 cache.db "SELECT COUNT(*) FROM storage"` + `ls -lh cache.db` |
| **Memcached 监控** | MemcachedCache 内存/命中率 | `echo "stats" \| nc memcached 11211` |
| **Redis 监控** | RedisCache 内存/命中率/eviction | `redis-cli INFO stats` |
| **HTTP 响应时间监控** | 整体缓存效果 | 对比有缓存/无缓存时的响应时间 |
| **日志分析** | 缓存异常 | 解析日志中的 `warning`/`error` 级别缓存消息 |

---

## 十三、缓存集群与分片策略

### 13.1 当前状态：不支持集群

RSS-Bridge 当前的所有缓存后端都是**单节点设计**，没有内置的集群或分片能力：

| 后端 | 集群支持 | 分片能力 | 一致性散列 |
|------|---------|---------|-----------|
| **NullCache** | — | — | — |
| **ArrayCache** | 否（进程内） | 否 | — |
| **FileCache** | 否（单机磁盘） | 否 | — |
| **SQLiteCache** | 否（单机文件） | 否 | — |
| **MemcachedCache** | **部分**（客户端支持多节点） | 否（无一致性散列） | 否（取模分片） |
| **Redis**（假想） | 依赖 Redis Cluster / 哨兵 | 依赖服务端分片 | 服务端管理 |

**关键发现**：`MemcachedCache` 当前的实现（`caches/MemcachedCache.php:16-20`）只调用了一次 `addServer($host, $port)`，仅支持单个 Memcached 节点：

```php
$this->conn = new \Memcached();
if (!$this->conn->addServer($host, $port)) {
    throw new \Exception('Unable to add memcached server');
}
```

虽然 PHP 的 `Memcached` 扩展本身支持多节点和一致性散列选项，但 RSS-Bridge 并未暴露这些配置。

### 13.2 多节点部署时的缓存一致性问题

RSS-Bridge 在多实例（多机器/多 PHP-FPM 进程池）部署时，不同后端的缓存共享能力差异显著：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  RSS-Bridge  │     │  RSS-Bridge  │     │  RSS-Bridge  │
│  Instance A  │     │  Instance B  │     │  Instance C  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│                    缓存后端                              │
│                                                          │
│  FileCache   → 各实例写本地磁盘，完全隔离，无共享        │
│  SQLiteCache → 各实例写本地文件，完全隔离，无共享        │
│  ArrayCache  → 各实例内存独立，完全隔离，无共享          │
│  Memcached   → 所有实例连同一个 Memcached，可共享        │
│  Redis       → 所有实例连同一个 Redis，可共享            │
└──────────────────────────────────────────────────────────┘
```

**FileCache / SQLiteCache 在多实例下的问题**：
- **缓存命中率低**：每个实例独立缓存，相同的请求被不同实例命中时仍然要回源
- **缓存不一致**：不同实例看到的缓存状态可能不同（用户看到的内容因负载均衡路由到不同实例而不同）
- **浪费存储**：N 个实例就有 N 份重复缓存数据
- **目标站点压力放大**：回源请求量随实例数线性增长

**结论**：多节点部署必须使用 MemcachedCache 或 RedisCache，否则缓存基本失效。

### 13.3 Memcached 多节点与取模分片

PHP `Memcached` 扩展原生支持多节点，使用方式是多次调用 `addServer()`：

```php
// MemcachedCache 当前只加一个节点，扩展后可加多个
$conn = new \Memcached();
$conn->addServer('cache-01', 11211);
$conn->addServer('cache-02', 11211);
$conn->addServer('cache-03', 11211);
```

**默认分片算法**：PHP Memcached 默认使用 **取模哈希**（`hash(key) % N`），其中 N 为节点数。

```
key → crc32(key) → % 3 → 选择节点 0/1/2

例如:
  "http_abc"  →  1542  →  % 3 = 0  →  cache-01
  "http_def"  →  9876  →  % 3 = 0  →  cache-01
  "http_ghi"  →  7341  →  % 3 = 1  →  cache-02
  "http_jkl"  →  5612  →  % 3 = 2  →  cache-03
```

**取模分片的致命缺陷**：当节点数变化时（如增减节点），几乎所有 key 的映射都会改变，导致**大规模缓存失效**。

从 3 节点扩展到 4 节点时，大约 `3/4`（75%）的 key 会映射到新的节点，缓存命中率骤降，目标站点压力激增。

### 13.4 一致性散列策略

一致性散列（Consistent Hashing）通过将 key 和节点都映射到一个 0~2^32 的环形空间来解决取模分片的问题：

```
         0
         │
   2^32 ─┼─ node-01 (虚拟点1, 虚拟点2, ...)
         │
         ├── key A → 找到顺时针最近节点 = node-03
         │
         ┼─ node-02
         │
         ├── key B → 找到顺时针最近节点 = node-02
         │
         ┼─ node-03
         │
     2^32-1
```

**PHP Memcached 原生支持**：通过 `setOption(\Memcached::OPT_DISTRIBUTION, \Memcached::DISTRIBUTION_CONSISTENT)` 启用。

一致性散列的效果：
- 增加 1 个节点：只有约 `1/N` 的 key 需要迁移，其余保持稳定
- 减少 1 个节点：只有该节点上的 key 需要重新分配

**RSS-Bridge 中启用的方式**（需修改 `MemcachedCache`）：

```php
class MemcachedCache implements CacheInterface
{
    public function __construct(Logger $logger, array $servers)
    {
        $this->conn = new \Memcached();
        $this->conn->setOption(\Memcached::OPT_DISTRIBUTION, \Memcached::DISTRIBUTION_CONSISTENT);
        $this->conn->setOption(\Memcached::OPT_LIBKETAMA_COMPATIBLE, true); // Ketama 算法
        foreach ($servers as $server) {
            $this->conn->addServer($server['host'], $server['port']);
        }
    }
}
```

同时需要在 `CacheFactory` 中读取 `MemcachedCache.servers` 配置数组而非单个 `host`/`port`。

### 13.5 集群架构设计

#### 方案一：Memcached + 客户端分片（一致性散列）

```
┌──────────────┐
│ RSS-Bridge   │
│ 所有实例     │──┐
└──────────────┘  │
                  │  客户端一致性散列
┌──────────────┐  │  (PHP Memcached 扩展内置)
│ RSS-Bridge   │──┤
│ 所有实例     │  │
└──────────────┘  │
                  ▼
        ┌───────────────────────────────┐
        │ cache-01  cache-02  cache-03  │
        │ (32GB)    (32GB)    (32GB)    │
        └───────────────────────────────┘
        总容量 ~96GB，无副本，单节点故障丢 1/N 数据
```

**优缺点**：
- ✅ 简单，不依赖额外代理层
- ✅ PHP Memcached 原生支持，配置即开
- ❌ 节点故障时该节点数据丢失（回源压力上升）
- ❌ 所有 RSS-Bridge 实例配置必须完全一致（节点列表顺序会影响散列）

#### 方案二：Twemproxy / Nutcracker 代理分片

```
                    ┌────────────────────┐
                    │   Twemproxy        │
                    │  (代理层，统一端口) │
                    └────────┬───────────┘
                             │  代理端一致性散列
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Memcached-01 │     │ Memcached-02 │     │ Memcached-03 │
└──────────────┘     └──────────────┘     └──────────────┘
```

**优缺点**：
- ✅ 客户端零改造，RSS-Bridge 认为自己只连一个 Memcached
- ✅ 代理层管理节点增删，对应用透明
- ❌ 增加新的基础设施组件和故障点
- ❌ 代理本身可能成为性能瓶颈

#### 方案三：Redis Cluster（服务端分片）

```
                    Redis Cluster（6 节点，3 主 3 从）
        ┌────────────────────────────────────────────────────┐
        │                                                    │
        │  Master-01 ── Slave-01   (slot 0-5460)            │
        │  Master-02 ── Slave-02   (slot 5461-10922)        │
        │  Master-03 ── Slave-03   (slot 10923-16383)       │
        │                                                    │
        └────────────────────┬───────────────────────────────┘
                             │  MOVED/ASK 重定向协议
                    ┌────────┴──────────┐
                    │  RSS-Bridge 集群   │
                    │  (phpredis 扩展)   │
                    └───────────────────┘
```

**优缺点**：
- ✅ 服务端管理分片，客户端只需连接任意节点
- ✅ 支持主从复制，节点故障自动故障转移
- ✅ 可水平扩展（增加 master 节点 + 重新分片）
- ❌ 需要 Redis 技术栈支持
- ❌ 客户端需支持 Cluster 协议（phpredis 原生支持）

### 13.6 分片策略选型建议

| 场景 | 推荐方案 | 节点数 | 容量 |
|------|---------|--------|------|
| 单实例 + 小流量 | 单机 Memcached / SQLite | 1 | ≤ 10GB |
| 多实例 + 中等流量 | Memcached + 一致性散列（客户端） | 2~5 | 10~100GB |
| 多实例 + 高可用 | Redis Cluster（3 主 3 从） | 6+ | 100GB+ |
| 超大流量 + 专业运维 | Twemproxy + Memcached 集群 | 5+ | 200GB+ |

---

## 十四、持久化缓存的快照与备份策略

### 14.1 各后端的持久化能力

| 后端 | 持久化 | 存储介质 | 数据可备份 | 恢复可行性 |
|------|--------|---------|-----------|-----------|
| **NullCache** | 无 | — | — | — |
| **ArrayCache** | 无 | 内存 | 否 | 否 |
| **FileCache** | ✅ 全量持久化 | 磁盘文件 | ✅ 可直接复制 | ✅ 拷贝回目录即可 |
| **SQLiteCache** | ✅ 全量持久化 | 单数据库文件 | ✅ 单文件备份 | ✅ 拷贝回即可 |
| **MemcachedCache** | ❌ 纯内存 | 内存 | 否（除非 `memcached-tool dump`） | ❌ 重启丢失 |
| **Redis**（假想） | ✅ RDB / AOF | 磁盘 + 内存 | ✅ 内置备份机制 | ✅ 多种恢复方式 |

### 14.2 FileCache 备份方案

FileCache 的数据是每个 key 对应一个独立文件（MD5 哈希命名），备份策略最简单。

#### 冷备份（停机后拷贝）

```bash
# 步骤1：停止 RSS-Bridge（如 PHP-FPM、Nginx）
systemctl stop php-fpm nginx

# 步骤2：完整拷贝缓存目录
cp -r /var/www/rss-bridge/cache /var/backup/rss-bridge-cache-$(date +%Y%m%d)

# 步骤3：启动 RSS-Bridge
systemctl start php-fpm nginx
```

**优点**：数据一致，无拷贝期间的写冲突
**缺点**：需要停机窗口，不适用于高可用场景

#### 热备份（在线复制）

```bash
# 使用 rsync 增量备份，不停止服务
rsync -av --delete \
    /var/www/rss-bridge/cache/ \
    /var/backup/rss-bridge-cache/

# 如果担心复制期间的文件写入不一致，可以使用 LVM 快照
lvcreate --size 10G --snapshot --name cache-snap /dev/vg0/lv-cache
mount /dev/vg0/cache-snap /mnt/cache-snap
cp -r /mnt/cache-snap /var/backup/
umount /mnt/cache-snap
lvremove /dev/vg0/cache-snap
```

**关键风险**：FileCache 没有文件锁，热备份期间如果有并发写入，备份数据可能包含部分写入的损坏文件。但由于 FileCache 是 Cache-Aside 模式，即使损坏也只是下次读取失败后重新回源，不会导致功能异常。

#### 恢复

```bash
# 清空旧缓存（可选）
rm -rf /var/www/rss-bridge/cache/*.cache

# 还原备份
cp -r /var/backup/rss-bridge-cache-20250614/* /var/www/rss-bridge/cache/

# 修正权限
chown -R www-data:www-data /var/www/rss-bridge/cache/
chmod 755 /var/www/rss-bridge/cache/
```

### 14.3 SQLiteCache 备份方案

SQLite 是单文件数据库，备份需要特别注意写入期间的一致性。

#### 方式一：`.backup` 命令（推荐）

SQLite 内置了在线备份机制，比直接拷贝文件更安全：

```bash
# 在线备份（不影响读写，SQLite 自己做一致性保证）
sqlite3 /var/www/rss-bridge/cache/cache.db ".backup '/var/backup/cache-$(date +%Y%m%d).db'"
```

内部原理：SQLite 会创建一个快照，逐页复制，即使期间有写入也能保证备份的一致性。

#### 方式二：VACUUM INTO（同时压缩）

```bash
# 导出一个"干净"的数据库（无碎片、无已删除页）
sqlite3 /var/www/rss-bridge/cache/cache.db \
    "VACUUM INTO '/var/backup/cache-clean-$(date +%Y%m%d).db'"
```

这不仅是备份，还相当于做了一次数据库压缩（VACUUM 回收已删除记录的空间）。

#### 方式三：直接拷贝（仅冷备份可用）

```bash
# 仅在 RSS-Bridge 停止时使用
systemctl stop php-fpm
cp /var/www/rss-bridge/cache/cache.db /var/backup/
systemctl start php-fpm
```

**严禁在运行时直接拷贝**：如果拷贝期间 SQLite 正在写入 WAL（Write-Ahead Log），拷贝出来的文件可能损坏。

#### 定期 VACUUM 维护

SQLiteCache 的 `clear()` 和 `prune()` 只删除记录但不回收磁盘空间。长期运行后数据库文件会膨胀。建议定期执行：

```bash
# 每周执行一次，回收空闲空间（可在业务低峰期）
sqlite3 /var/www/rss-bridge/cache/cache.db "VACUUM;"
```

#### 恢复

```bash
# 替换数据库文件
systemctl stop php-fpm
cp /var/backup/cache-20250614.db /var/www/rss-bridge/cache/cache.db
chown www-data:www-data /var/www/rss-bridge/cache/cache.db
systemctl start php-fpm
```

### 14.4 Memcached 快照（有限支持）

Memcached 本质是纯内存缓存，**不提供可靠的持久化机制**。但有有限的导出方式：

```bash
# 使用 libmemcached 自带的 memcached-tool 导出（非事务性，仅供参考）
memcached-tool cache-host:11211 dump > cache-dump-$(date +%Y%m%d).txt

# 导入
memcached-tool cache-host:11211 load < cache-dump-20250614.txt
```

**重要限制**：
- `dump` 命令只能导出**当前未过期**的条目，但**不会保留 TTL**（导入后所有条目变为永久或设置相同 TTL）
- 导出过程不是原子的，期间有写入会导致导出数据不一致
- 生产环境不依赖 Memcached 作为持久化数据源

### 14.5 Redis 快照与备份（假想后端）

如果添加 RedisCache 后端，Redis 提供两级持久化：

#### RDB 快照（默认）

```bash
# 触发一次 BGSAVE（后台异步快照）
redis-cli BGSAVE

# 配置自动快照（redis.conf）
save 900 1    # 900 秒内有 1 次写操作 → 快照
save 300 10   # 300 秒内有 10 次写操作 → 快照
save 60 10000 # 60 秒内有 10000 次写操作 → 快照
```

生成的 `dump.rdb` 文件存储在 Redis 工作目录，直接拷贝即可备份。

#### AOF 日志（Append-Only File）

```
# redis.conf
appendonly yes
appendfsync everysec   # 每秒 fsync，性能与可靠性的平衡
```

AOF 记录每一条写入命令，类似数据库的 redo log。恢复时重新执行所有命令即可还原数据。

#### 恢复优先级

Redis 启动时优先检查 AOF 文件（因为数据更完整），只有 AOF 不存在时才加载 RDB。

### 14.6 备份策略模板

| 场景 | 备份频率 | 保留策略 | 验证方式 |
|------|---------|---------|---------|
| **个人部署**（FileCache） | 每周 1 次 | 保留最近 2 份 | 抽检 key 是否存在 |
| **中型部署**（SQLiteCache） | 每日 1 次，BGSAVE 在线备份 | 保留最近 7 日 + 每周 1 份保留 1 月 | 每周恢复到测试库验证完整性 |
| **大型部署**（Redis Cluster） | RDB 每日备份 + AOF 实时追加 | RDB 保留 30 日 + AOF 保留 7 日 | 定期演练故障恢复流程 |
| **极高可用** | 跨地域复制（主从/集群） + 每日冷备份 | 冷备份保留 90 日 | 每月灾难恢复演练 |

---

## 十五、缓存过期事件订阅与业务回调

### 15.1 当前状态：无事件机制

RSS-Bridge 的 `CacheInterface` 是**纯 CRUD 接口**，没有任何事件通知机制：

```php
interface CacheInterface
{
    public function get(string $key, $default = null);
    public function set(string $key, $value, ?int $ttl = null): void;
    public function delete(string $key): void;
    public function clear(): void;
    public function prune(): void;
    // 没有 onExpire / onSet / onDelete 等事件回调
}
```

各后端在 `get()` 中检测到过期时，要么直接删除（FileCache、ArrayCache），要么静默忽略（SQLiteCache），**不会向上层代码发出任何通知**。

### 15.2 业务上需要过期回调的场景

虽然 RSS-Bridge 当前没有使用过期回调，但以下场景在业务上是合理的：

| 场景 | 回调时机 | 期望行为 |
|------|---------|---------|
| **令牌自动刷新** | API access_token 过期时 | 自动发起请求刷新 token，避免下次请求时临时刷新导致延迟 |
| **速率限制标记解除** | Bridge 的 `rate_limit` key 过期时 | 记录日志或重置内部限流状态 |
| **缓存预热触发** | 高频 Bridge 的缓存过期时 | 后台异步重新抓取，实现 stale-while-revalidate |
| **监控告警** | 大批量缓存过期时 | 触发告警，预防缓存雪崩 |
| **资源释放** | 大对象缓存过期时 | 触发关联的临时文件/目录清理 |

### 15.3 过期事件的实现层次

缓存过期事件的实现有两个层次：**惰性过期（懒检测）** 和 **主动过期（服务端推送）**。

#### 层次一：惰性过期检测（PHP 端可实现）

惰性过期依赖于 `get()` 调用时才检测到 key 已过期。这是 FileCache / ArrayCache / SQLiteCache 目前采用的方式。

**实现方案**：在缓存实现中注册过期回调，`get()` 检测到过期时触发。

```php
// 在 CacheInterface 中新增事件注册方法
interface CacheInterface
{
    // ...原有方法...
    public function onExpire(callable $callback): void;
}

// 以 FileCache 为例
class FileCache implements CacheInterface
{
    private array $expireCallbacks = [];

    public function onExpire(callable $callback): void
    {
        $this->expireCallbacks[] = $callback;
    }

    public function get(string $key, $default = null)
    {
        $cacheFile = $this->createCacheFile($key);
        // ...读取与反序列化...
        $expiration = $item['expiration'] ?? time();

        if ($expiration !== 0 && $expiration <= time()) {
            // 触发过期回调
            foreach ($this->expireCallbacks as $callback) {
                try {
                    $callback($key, $item['value'] ?? null);
                } catch (\Throwable $e) {
                    $this->logger->warning('Expire callback failed', ['key' => $key]);
                }
            }
            $this->delete($key);
            return $default;
        }
        return $item['value'];
    }
}
```

**缺点**：
- 只能检测被 `get()` 访问到的 key。如果某个 key 过期后从未被访问，回调永远不会触发
- 回调在请求线程内执行，会增加用户请求的响应延迟

#### 层次二：主动过期（服务端推送，仅 Redis/Memcached 支持）

主动过期由缓存服务端在 key 到期时主动通知客户端，不需要等待 `get()` 调用。

**Redis Keyspace Notifications**：

```bash
# redis.conf 中启用键空间通知（接收过期事件）
notify-keyspace-events Ex
```

订阅方式（PHP 端）：

```php
$redis = new \Redis();
$redis->connect('redis-host', 6379);

// 订阅 0 号数据库的过期事件
$redis->subscribe(['__keyevent@0__:expired'], function ($redis, $channel, $message) {
    $expiredKey = $message;

    // 根据 key 前缀分发业务回调
    if (str_starts_with($expiredKey, 'server_')) {
        // 服务端响应缓存过期，可异步预刷新
        // refresh_source_cache($expiredKey);
    } elseif (str_starts_with($expiredKey, 'http_')) {
        // Feed 响应缓存过期
        // preheat_feed_cache($expiredKey);
    }
});
```

**优点**：
- 真正的"到期即通知"，不依赖 `get()` 调用
- 回调可以在独立进程中执行，不阻塞用户请求

**缺点**：
- 仅 Redis 原生支持（Memcached 不提供）
- Redis 的过期事件是"尽力而为"——如果 Redis 崩溃，部分事件会丢失
- 需要额外的常驻进程（或 Swoole/ReactPHP 的事件循环）来维护订阅连接

### 15.4 完整事件模型设计

若需要扩展 `CacheInterface` 支持事件，建议采用以下模型：

```php
interface CacheInterface
{
    // 原有 CRUD 方法...

    public function on(string $event, callable $callback): void;
}

// 事件类型定义
final class CacheEvents
{
    public const BEFORE_GET  = 'before_get';   // 读取前
    public const AFTER_GET   = 'after_get';    // 读取后（含命中/未命中）
    public const BEFORE_SET  = 'before_set';   // 写入前
    public const AFTER_SET   = 'after_set';    // 写入后
    public const BEFORE_DEL  = 'before_delete';// 删除前
    public const AFTER_DEL   = 'after_delete'; // 删除后
    public const ON_HIT      = 'hit';          // 缓存命中
    public const ON_MISS     = 'miss';         // 缓存未命中
    public const ON_EXPIRE   = 'expire';       // 缓存过期（主动或惰性）
    public const ON_EVICT    = 'evict';        // 内存不足被淘汰（仅 Redis/Memcached）
}
```

**典型用法**：

```php
// 命中率统计
$cache->on(CacheEvents::ON_HIT, fn($k) => $stats->increment('hits'));
$cache->on(CacheEvents::ON_MISS, fn($k) => $stats->increment('misses'));

// 令牌自动刷新
$cache->on(CacheEvents::ON_EXPIRE, function ($key, $oldValue) {
    if ($key === 'SpotifyBridge_token') {
        // 后台队列任务：重新获取 token
        enqueue_job(new RefreshSpotifyTokenJob());
    }
});

// 慢查询日志
$cache->on(CacheEvents::AFTER_GET, function ($key, $value, $elapsedMs) {
    if ($elapsedMs > 100) {
        $logger->warning("Slow cache get: {$key} took {$elapsedMs}ms");
    }
});
```

### 15.5 事件模型的实现成本对比

| 方案 | 可实现性 | 过期事件精度 | 性能影响 | 推荐度 |
|------|---------|-------------|---------|--------|
| **惰性检测 + 装饰器** | 高（纯 PHP 实现，不依赖后端） | 低（仅被访问到的 key） | 中（请求线程内执行） | ⭐⭐⭐ |
| **惰性检测 + 后端扩展** | 高（修改各缓存类） | 低（仅被访问到的 key） | 中（请求线程内执行） | ⭐⭐ |
| **Redis 键空间通知** | 中（需常驻进程） | 高（服务端推送） | 低（独立进程） | ⭐⭐⭐⭐ |
| **修改 CacheInterface 接口** | 低（需改所有后端和测试） | 视后端而定 | 视实现而定 | ⭐ |

### 15.6 业务回调的最佳实践

无论采用哪种实现方式，都应遵守以下原则：

1. **回调必须非阻塞**：过期回调不应执行耗时操作（如 HTTP 请求），应将任务投递到消息队列由后台 worker 执行
2. **回调异常必须隔离**：单个回调出错不能影响缓存操作本身，也不能影响其他回调
3. **回调必须幂等**：同一条过期事件可能被触发多次（极端情况），回调逻辑必须可重入
4. **事件不保证可靠性**：不要在过期回调中处理核心业务逻辑（如计费、库存扣减），缓存事件是"尽力而为"的
5. **避免回调链过长**：不要在回调中又触发缓存操作，防止递归和死循环

---

## 十六、缓存与底层存储的一致性选择

### 16.1 RSS-Bridge 的一致性模型：Cache-Aside（旁路缓存）

RSS-Bridge 严格采用 **Cache-Aside（旁路缓存）模式**，这是所有缓存与"底层存储"（即被 Bridge 抓取的第三方网站 API/网页）之间交互的唯一模式。

Cache-Aside 的核心逻辑：

```
          读流程                          写流程（Bridge 抓取成功）
             │                                │
             ▼                                ▼
    查询缓存                          直接回源抓取（目标网站）
             │                                │
       ┌─────┴─────┐                    解析数据
       │           │                          │
     命中        未命中                      写入缓存
       │           │                          │
       ▼           ▼                          ▼
   返回缓存     回源抓取                  返回数据
               / 目标网站 │
                    │
                    ▼
               写入缓存
                    │
                    ▼
               返回数据
```

这与其他模式的关键区别：

| 模式 | 谁负责同步缓存和存储 | RSS-Bridge 是否适用 |
|------|-------------------|-------------------|
| **Cache-Aside** | 应用代码手动管理读写 | ✅ **当前使用** |
| **Write-Through** | 写时同时写缓存和存储 | ❌ 无"存储"可写（目标站不可写入） |
| **Write-Behind（Write-Back）** | 先写缓存，异步刷入存储 | ❌ 无"存储"可写 |
| **Read-Through** | 缓存层自动回源填充 | ❌ RSS-Bridge 的缓存是被动的 |
| **Refresh-Ahead** | 缓存过期前自动后台刷新 | ❌ 未实现 |

### 16.2 为什么 RSS-Bridge 只能用 Cache-Aside

RSS-Bridge 是一个**只读的聚合服务**——它从第三方网站抓取数据并转换为 Feed，**不向目标网站写入任何数据**。因此不存在"写回存储"的概念，Write-Through 和 Write-Behind 模式天然不适用。

```
RSS-Bridge 架构中的数据流方向：

┌──────────────┐    读（HTTP GET）    ┌──────────────────┐
│  RSS-Bridge  │ ───────────────────→ │  目标网站/API    │
│   (缓存层)    │ ←─────────────────── │  (唯一数据源)    │
└──────────────┘    返回 HTML/JSON    └──────────────────┘
         │
         ▼
    （单向）写回自身缓存
```

### 16.3 一致性语义：最终一致 + 弱一致

由于 Cache-Aside 模式加上 TTL 过期策略，RSS-Bridge 的缓存与目标网站之间天然是**弱一致性**（或最终一致性）：

```
T0: 目标站发布了新文章（RSS-Bridge 不知道）
T1: 用户请求 → RSS-Bridge 缓存命中 → 返回旧数据（不一致窗口）
T2: 缓存过期（CACHE_TIMEOUT 到了）
T3: 用户请求 → 回源抓取 → 获得新数据 → 更新缓存 → 返回新数据
T3+: 所有请求返回新数据（最终一致）
```

**不一致窗口长度** = `CACHE_TIMEOUT`（各 Bridge 自定义，默认 3600 秒 = 1 小时）

这意味着用户看到的 Feed 数据最多可能滞后 1 小时。对 RSS 阅读场景来说，这是完全可以接受的折衷。

### 16.4 主动失效机制的缺失

Cache-Aside 模式通常配合**主动失效**（当底层存储变化时主动删除缓存）来减少不一致窗口。但 RSS-Bridge 不具备这个能力：

| 主动失效触发方式 | RSS-Bridge 是否可用 | 原因 |
|----------------|-------------------|------|
| **写操作后删缓存** | ❌ | 不向目标站写数据 |
| **目标站 Webhook 通知** | ❌ | 绝大多数目标站不提供数据变更通知 |
| **数据库 binlog 订阅** | ❌ | 无权访问目标站数据库 |
| **定时轮询检测变更** | ⚠️ 理论可行 | 当前未实现，且等同于缩短 TTL |

**结论**：RSS-Bridge 只能依赖 **TTL 被动过期** 来实现数据更新，没有主动失效手段。

### 16.5 getContents() 的特殊一致性：条件请求 + 304 协商

虽然整体是 Cache-Aside + TTL，但 `getContents()` 引入了一层额外的一致性保障：**HTTP 条件请求（If-Modified-Since / If-None-Match）**。

```
普通 Cache-Aside（CacheMiddleware 层）：
  get() 命中未过期 → 直接返回，不联系目标站
  get() 未命中/过期 → 回源抓取 → 写缓存 → 返回

getContents 的增强 Cache-Aside：
  get() 命中（不检查过期！）→ 提取 Last-Modified/ETag
                     → 条件请求目标站
                          ├─ 304 → 用缓存内容（一致性比 TTL 更高）
                          └─ 200 → 更新缓存（获得最新数据）
  get() 未命中 → 普通请求 → 写缓存 → 返回
```

**getContents 的一致性优于普通 Cache-Aside**：
- 即使缓存已过 TTL，只要目标站内容没变（304），就复用缓存且不浪费带宽
- 即使缓存还在 TTL 内，只要目标站内容变了，也能在条件请求时发现（前提是 TTL 已过，否则根本不发请求）
- 但本质还是 TTL 驱动，TTL 内的数据是"盲信任"的

### 16.6 一致性选择的业务权衡

| 一致性级别 | 实现方式 | 数据延迟 | 目标站压力 | RSS-Bridge 现状 |
|-----------|---------|---------|-----------|----------------|
| **强一致** | 每次请求都回源 | 0 | 极高 | 仅 DEBUG 模式（ArrayCache）接近 |
| **读己之写** | 写后立即删缓存 | ~0 | 高 | ❌ 无法实现（无写操作） |
| **最终一致（短 TTL）** | TTL = 60~300 秒 | 1~5 分钟 | 较高 | 部分实时性强的 Bridge 设置 |
| **最终一致（长 TTL）** | TTL = 3600~86400 秒 | 1~24 小时 | 低 | ✅ **默认策略** |
| **条件请求增强** | TTL + ETag/Last-Modified | TTL + 协商延迟 | 中等 | ✅ getContents 层已实现 |

**设计建议**：
- 对更新频率低的 Bridge（如博客周刊），可设置长 TTL（86400 秒 = 1 天）
- 对更新频率高的 Bridge（如新闻、社交媒体），设置短 TTL（300~900 秒）并依赖 304 协商节省带宽
- 不要把 TTL 设为 0（禁用缓存），这会把 RSS-Bridge 变成纯代理，失去了最核心的价值

---

## 十七、缓存击穿与热点 Key 的本地二级缓存策略

### 17.1 三种缓存异常场景辨析

| 术语 | 定义 | 触发条件 | 风险等级 |
|------|------|---------|---------|
| **缓存穿透**（Penetration） | 查询不存在的数据，缓存和存储都没有，每次都打到存储 | 恶意请求不存在的 Bridge / URL | 中 |
| **缓存击穿**（Breakdown） | 单个热点 Key 过期瞬间，大量并发请求同时回源 | 热门 Bridge 的 Feed 缓存过期 | **高** |
| **缓存雪崩**（Avalanche） | 大量 Key 同时过期，或缓存服务宕机，全量请求打到存储 | 缓存重启 / TTL 全部设为同一值 | **极高** |

RSS-Bridge 在高流量部署下最容易遇到的是**缓存击穿**：某个热门 Bridge（如 YoutubeBridge、TwitterBridge）的 Feed 缓存一旦过期，瞬间可能有成百上千个并发请求同时回源抓取，极可能触发目标网站的速率限制（429 Too Many Requests）。

### 17.2 当前状态：无任何击穿防护

RSS-Bridge **没有内置任何防缓存击穿机制**。`CacheMiddleware` 的逻辑简单直接：

```php
// CacheMiddleware::__invoke()（简化示意）
$cacheKey = $this->makeCacheKey($request);
$cachedResponse = $this->cache->get($cacheKey);

if ($cachedResponse !== null) {
    return $cachedResponse;  // 命中，直接返回
}

// 未命中 —— 直接执行抓取（无锁、无单飞、无排队）
$response = $next($request);

// 写缓存
$this->cache->set($cacheKey, $response, $this->computeTtl($response));
return $response;
```

当热点 Key 过期时，N 个并发请求会同时执行到 `$next($request)`，产生 N 次回源抓取。

### 17.3 防击穿策略一：互斥锁（Mutex Lock）

最经典的防击穿手段：未命中时先尝试获取锁，只有拿到锁的请求去回源，其他请求等待或返回降级数据。

```php
class LockingCacheMiddleware
{
    public function __invoke(Request $request, callable $next): Response
    {
        $cacheKey = $this->makeCacheKey($request);
        $cached = $this->cache->get($cacheKey);

        if ($cached !== null) {
            return $cached;
        }

        // 尝试获取锁（用缓存自身实现分布式锁）
        $lockKey = 'lock:' . $cacheKey;
        $lockAcquired = $this->cache->set($lockKey, '1', 30, true); // NX + 30s TTL

        if ($lockAcquired) {
            // 拿到锁 → 回源抓取
            try {
                $response = $next($request);
                $this->cache->set($cacheKey, $response, $this->computeTtl($response));
                return $response;
            } finally {
                $this->cache->delete($lockKey);
            }
        }

        // 没拿到锁 → 等待后重试（自旋，或直接返回默认值）
        usleep(50000); // 50ms
        return $this->cache->get($cacheKey) ?? $this->fallbackResponse();
    }
}
```

**适用场景**：回源成本高（目标站速率限制严格）但热点 Key 数量不多的情况。

**注意事项**：
- 锁必须有 TTL，防止持锁进程崩溃导致死锁
- `set($key, $value, $ttl, true)` 中的 `true` 表示 NX（Only set if not exists），需要各缓存后端支持——目前 RSS-Bridge 的 `CacheInterface::set()` 没有 `$nx` 参数，需要扩展接口
- Redis/Memcached 原生支持 NX，FileCache/SQLiteCache 需要模拟（先 get 再 set 有竞态，需用文件锁/SQLite 事务保证原子性）

### 17.4 防击穿策略二：Singleflight（单飞模式）

Singleflight（Go 语言标准库模式）是比互斥锁更高效的方式：**同一个 Key 的并发回源请求只执行一次，其他请求共享结果**。

```php
class SingleflightCache
{
    private array $inflight = []; // key => Deferred

    public function getOrFetch(string $key, callable $fetcher, int $ttl)
    {
        $cached = $this->cache->get($key);
        if ($cached !== null) {
            return $cached;
        }

        // 已有同 key 的回源正在进行？
        if (isset($this->inflight[$key])) {
            // 等待另一个请求的结果（共享）
            return $this->inflight[$key]->await();
        }

        // 第一个请求 → 执行回源
        $deferred = new Deferred();
        $this->inflight[$key] = $deferred;

        try {
            $result = $fetcher();
            $this->cache->set($key, $result, $ttl);
            $deferred->resolve($result);
            return $result;
        } catch (\Throwable $e) {
            $deferred->reject($e);
            throw $e;
        } finally {
            unset($this->inflight[$key]);
        }
    }
}
```

**与互斥锁的区别**：
- 互斥锁：其他请求**阻塞等待**，锁释放后再去读缓存（可能需要再次查询）
- Singleflight：其他请求**挂起等待**同一份回源结果，完成后所有等待者同时拿到数据，少一次缓存查询

**RSS-Bridge 限制**：PHP-FPM 是**多进程模型**，每个请求是独立进程。`$inflight` 数组存在 PHP 内存中，**无法跨进程共享**。因此：
- 单 PHP-FPM 进程内 Singleflight 有效（同一进程处理的并发请求会合并）
- 多进程部署下 Singleflight 无效（需要配合 APCu 或分布式锁实现跨进程版本）

### 17.5 防击穿策略三：本地二级缓存（L1 + L2）

在远端缓存（Redis/Memcached）之上，增加一层进程内缓存（ArrayCache / APCu）作为 L1，形成两级缓存架构：

```
          请求
            │
            ▼
   ┌─────────────────┐
   │  L1: 本地缓存    │  (进程内 ArrayCache / APCu，TTL 较短: 10~30s)
   └───────┬─────────┘
        命中└────→ 直接返回（极快，无网络开销）
           未命中
            │
            ▼
   ┌─────────────────┐
   │  L2: 远端缓存    │  (Redis / Memcached，TTL 较长: 300~3600s)
   └───────┬─────────┘
        命中└────→ 写回 L1 → 返回
           未命中
            │
            ▼
      回源抓取（目标网站）
            │
            ▼
      同时写 L2 + L1 → 返回
```

**优势**：
- L1 命中率即使只有 20%，也能大幅减少 L2 的网络请求量
- 热点 Key 在 L1 TTL 内完全不会打到 L2，更不会回源
- L2 故障时，L1 可作为降级缓存（部分数据可用）
- L2 击穿瞬间，L1 仍然提供服务（只要 L1 TTL > 击穿窗口）

**L1 缓存的选择**：

| 方案 | 跨进程共享 | 性能 | PHP 要求 | 适用场景 |
|------|-----------|------|---------|---------|
| **ArrayCache** | ❌（仅当前进程） | ⭐⭐⭐⭐⭐ | 无需扩展 | 单进程 / CLI 模式 |
| **APCu** | ✅（同 PHP-FPM 进程池内） | ⭐⭐⭐⭐ | 需要 `apcu` 扩展 | 多进程生产部署 |

**RSS-Bridge 中的集成方式**：

```php
// 创建 L1 + L2 组合缓存
class TwoLevelCache implements CacheInterface
{
    public function __construct(
        private CacheInterface $l1,  // APCuCache / ArrayCache
        private CacheInterface $l2,  // RedisCache / MemcachedCache
        private int $l1Ttl = 30,     // L1 的 TTL，显著短于 L2
    ) {}

    public function get(string $key, $default = null)
    {
        // L1 查找
        $value = $this->l1->get($key, $this->sentinel);
        if ($value !== $this->sentinel) {
            return $value;
        }

        // L1 miss → L2 查找
        $value = $this->l2->get($key, $this->sentinel);
        if ($value !== $this->sentinel) {
            // L2 hit → 回填 L1
            $this->l1->set($key, $value, $this->l1Ttl);
            return $value;
        }

        return $default;
    }

    public function set(string $key, $value, ?int $ttl = null): void
    {
        // 双写：同时写 L2 和 L1
        $this->l2->set($key, $value, $ttl);
        $this->l1->set($key, $value, $this->l1Ttl);
    }

    public function delete(string $key): void
    {
        // 先删 L2，再删 L1（防止短暂不一致时 L1 返回旧值）
        $this->l2->delete($key);
        $this->l1->delete($key);
    }
}
```

**L1/L2 TTL 比例建议**：L1 TTL 设为 L2 TTL 的 5%~10%。例如 L2=3600s（1h），则 L1=180~360s。这个比例是命中率和一致性的折衷。

### 17.6 防击穿策略四：提前续期（Stale-While-Revalidate）

缓存即将过期但还未过期时，由第一个请求触发**后台异步刷新**，其他请求继续返回旧数据：

```
           TTL = 3600s
  ├────────────────────────────┤
                                过期点
  ├────────────────────┬───────┤
       正常期         续期窗口
       直接返回        第一个请求触发后台刷新
                        其他请求继续返回旧缓存
```

```php
// 在缓存写入时额外记录一个"续期触发时间"（TTL 的 80% 处）
$item = [
    'value'       => $data,
    'expireAt'    => time() + $ttl,
    'refreshAt'   => time() + (int)($ttl * 0.8), // 80% TTL 时开始续期
];

// 读取时检查是否进入续期窗口
public function getWithStaleWhileRevalidate(string $key, $default = null)
{
    $item = $this->rawGet($key);
    if ($item === null) {
        return $default;
    }

    if (time() >= $item['refreshAt'] && time() < $item['expireAt']) {
        // 进入续期窗口但还没过期 → 异步刷新，同时返回旧值
        $this->triggerAsyncRefresh($key); // 投递到消息队列
    }

    if (time() >= $item['expireAt']) {
        return $default; // 已过期，走正常回源
    }

    return $item['value'];
}
```

**RSS-Bridge 限制**：PHP-FPM 无原生异步任务机制，需要配合消息队列（RabbitMQ/Redis Queue）或 cron 定时任务实现。

### 17.7 防雪崩补充：TTL 随机抖动

除了防击穿，还需要防雪崩。最简单有效的手段是给 TTL 加**随机抖动**（jitter）：

```php
// 原始：所有请求同一 TTL = 3600，大量 Key 会在同一时间过期
$ttl = 3600;

// 改进：在 ±10% 范围内随机化，打散过期时间
$ttl = 3600 + random_int(-360, 360);  // 3240 ~ 3960 秒
```

RSS-Bridge 的 `CacheMiddleware` 对错误响应已经做了抖动（`random_int(300, 900)` 秒），但成功响应还没有。建议成功响应也加抖动。

### 17.8 策略组合建议

| 场景 | 推荐组合 |
|------|---------|
| **个人部署**（低流量） | 无需额外防护，当前 Cache-Aside 足够 |
| **中型部署**（中等流量） | L1（APCu）+ L2（Redis）二级缓存 + TTL 抖动 |
| **大型部署**（高流量） | L1/L2 二级缓存 + 分布式互斥锁防击穿 + 后台续期 + TTL 抖动 |
| **极高流量** | 以上全部 + CDN 边缘缓存（Nginx proxy_cache）在 RSS-Bridge 之前再加一层 |

---

## 十八、缓存多语言客户端 SDK 的兼容性

### 18.1 兼容性问题的来源

RSS-Bridge 当前是**纯 PHP 单体应用**，所有缓存读写都由 PHP 代码完成。但在以下场景中，会出现**多语言/多系统共享缓存**的需求：

1. **多服务微服务化**：RSS-Bridge 拆分为 PHP（Feed 生成）+ Go/Python（抓取 Worker）+ Node.js（Webhook 服务），多个服务共享同一个 Redis/Memcached
2. **运维工具链**：用 Python/Shell 写的缓存分析、预热、迁移脚本需要读写 RSS-Bridge 的缓存
3. **监控系统**：Prometheus/Grafana 的 Exporter（通常 Go 编写）需要读取缓存指标
4. **CDN/边缘节点**：Varnish/Nginx 的 Lua 脚本需要直接查询 RSS-Bridge 的缓存

在这些场景下，**缓存的编码协议必须跨语言一致**——否则 PHP 写入的数据 Go 读不懂，反之亦然。

### 18.2 当前编码协议：PHP 专属，不可跨语言

RSS-Bridge 的 FileCache 和 SQLiteCache 使用 **PHP `serialize()` / `unserialize()`** 作为值的编码格式：

| 后端 | Key 编码 | Value 编码 | TTL 存储方式 |
|------|---------|-----------|-------------|
| **FileCache** | `md5($key)`（文件名） | PHP `serialize(['key', 'expiration', 'value'])` | 内嵌在序列化数据中 |
| **SQLiteCache** | `sha1($key, raw)`（BLOB 主键） | PHP `serialize($value)` | 单独列 `updated`（整数） |
| **MemcachedCache** | `sha1($key)` | Memcached 扩展自动处理（默认 igbinary 或 PHP serialize） | 传给 Memcached 服务端 |
| **ArrayCache** | 原始 `$key` | 原始 PHP 变量 | 进程内存变量 |

**PHP `serialize()` 的跨语言问题**：

```php
// PHP 写入
$value = ['title' => 'Hello', 'items' => [1, 2, 3]];
file_put_contents('cache.bin', serialize($value));
// 存储内容: a:2:{s:5:"title";s:5:"Hello";s:5:"items";a:3:{i:0;i:1;i:1;i:2;i:2;i:3;}}
```

这种格式是 PHP 独有的，其他语言没有标准解析器：
- Python：需要第三方库 `phpserialize`（功能有限，不支持对象）
- Go：无成熟库，只能手动解析
- Node.js：`php-serialize` npm 包（同样不支持对象）
- Java/JVM：几乎不可用

更严重的是，如果缓存 value 包含 PHP 对象（如 `Response` 对象），`serialize()` 会写入类名和属性——其他语言完全无法还原为对应的对象结构。

### 18.3 存储格式标准化方案

#### 方案一：JSON 编码（推荐，跨语言最友好）

将 value 的编码格式统一为 JSON，所有语言都能原生解析：

```php
// 写入
$payload = [
    'v'   => 1,                    // 格式版本号，方便未来升级
    'ts'  => time(),               // 写入时间戳
    'ttl' => $ttl,                 // TTL（秒）
    'd'   => $this->normalize($value), // 标准化后的数据（纯数组/标量）
];
$serialized = json_encode($payload, JSON_UNESCAPED_UNICODE);

// 读取
$payload = json_decode($serialized, true);
if ($payload && $payload['v'] === 1) {
    return $payload['d'];
}
```

**JSON 编码的注意事项**：
1. **资源丢失**：PHP 对象的方法、私有属性、类信息都会丢失——必须在写入前将对象序列化为纯数组
2. **类型模糊**：JSON 不区分 `int`/`float` 大数字、不区分关联数组和对象（PHP `json_decode` 的 `assoc=true` 可全部转为数组）
3. **不支持二进制**：二进制数据需 base64 编码（RSS-Bridge 的缓存主要是文本，影响不大）
4. **循环引用**：含循环引用的 PHP 对象无法 JSON 编码

**Response 对象的标准化**：

`CacheMiddleware` 层缓存的是 `Response` 对象，需要定义跨语言的 JSON 结构：

```json
{
  "v": 1,
  "ts": 1718323200,
  "ttl": 3600,
  "d": {
    "body": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>...",
    "statusCode": 200,
    "headers": {
      "Content-Type": ["application/rss+xml; charset=utf-8"],
      "Last-Modified": ["Sat, 14 Jun 2025 00:00:00 GMT"]
    }
  }
}
```

#### 方案二：MessagePack / CBOR（紧凑二进制 JSON）

如果缓存体积大、JSON 的字符串开销不可接受，可用 MessagePack（紧凑二进制格式）替代：

```
JSON:  {"title":"Hello","items":[1,2,3]}  →  32 字节
MsgPack: 等价二进制                         →  19 字节（约省 40%）
```

各语言的 MsgPack 库支持情况：
- PHP：`msgpack` 扩展（PECL）
- Python：`msgpack`（pip）
- Go：`github.com/vmihailenco/msgpack/v5`
- Node.js：`@msgpack/msgpack`（npm）

**优势**：比 JSON 体积小 30%~50%，解析速度更快
**劣势**：二进制格式，不可人类可读，调试不如 JSON 方便

#### 方案三：Protocol Buffers（强类型，高性能）

如果对性能和类型安全有极高要求，可用 Protobuf 定义缓存 schema：

```protobuf
syntax = "proto3";

package rssbridge.cache;

message CacheEntry {
  int32 version     = 1;
  int64 timestamp   = 2;
  int32 ttl_seconds = 3;
  bytes data        = 4;  // 根据 key 前缀选择不同的子 message
}

message HttpResponse {
  int32 status_code     = 1;
  string body           = 2;
  map<string, HeaderValues> headers = 3;
}

message HeaderValues {
  repeated string values = 1;
}
```

**适用场景**：已有 Protobuf 技术栈、极高流量、对序列化开销敏感的场景

### 18.4 Key 格式标准化

除了 value，key 的格式也需要跨语言一致：

| 组件 | 当前实现 | 标准化建议 |
|------|---------|-----------|
| **Key 前缀** | PHP 字符串拼接（`http_`、`server_`、`pages_`、`{Bridge}_`） | ✅ 已天然跨语言，所有语言字符串拼接一致 |
| **Key 哈希** | FileCache: `md5($key)`, SQLiteCache: `sha1($key)`, Memcached: `sha1($key)` | **统一使用 SHA-1 hex**（所有语言都有标准库） |
| **TTL 语义** | FileCache: timestamp（过期绝对时间）, SQLiteCache: updated timestamp, Memcached: 秒数（相对 TTL） | **统一使用相对秒数 TTL**（Memcached/Redis 原生语义，写入时由客户端计算绝对时间） |

**标准化后的 Key 协议**：

```
原始 key (PHP): "http_" . json_encode([path, queryParams])
       ↓
跨语言通用:      prefix + ":" + raw_key (可选 URL 编码转义特殊字符)
       ↓
存储层:          sha1_hex(通用 key)  →  "a9993e364706816aba3e25717850c26c9cd0d89d"
```

各语言的 SHA-1 实现对照：

| 语言 | SHA-1 代码 |
|------|-----------|
| **PHP** | `sha1($key)` |
| **Python** | `hashlib.sha1(key.encode()).hexdigest()` |
| **Go** | `fmt.Sprintf("%x", sha1.Sum([]byte(key)))` |
| **Node.js** | `crypto.createHash('sha1').update(key).digest('hex')` |
| **Java** | `DigestUtils.sha1Hex(key)` (Apache Commons) |

### 18.5 跨语言缓存操作兼容性矩阵

假设 RSS-Bridge 改用 `JSON + SHA-1 key + 秒级 TTL` 的标准化协议，各语言操作缓存的兼容性：

| 操作 | PHP | Python | Go | Node.js | 说明 |
|------|-----|--------|----|---------|------|
| **计算存储 key** | ✅ | ✅ | ✅ | ✅ | SHA-1 hex 标准 |
| **读取 value** | ✅ | ✅ | ✅ | ✅ | JSON 标准解析 |
| **写入 value** | ✅ | ✅ | ✅ | ✅ | JSON 标准编码 |
| **解析 TTL** | ✅ | ✅ | ✅ | ✅ | 秒级整数 |
| **反序列化为 Response 对象** | ✅ | ⚠️ | ⚠️ | ⚠️ | 需各语言自行实现 Response 类 + 映射逻辑 |
| **读取 Bridge 业务数据**（数组） | ✅ | ✅ | ✅ | ✅ | JSON 直接可用 |
| **条件请求元数据**（ETag/Last-Modified） | ✅ | ✅ | ✅ | ✅ | 缓存 headers 字段中 |

### 18.6 渐进式迁移策略

由于修改编码协议涉及**数据不兼容**，不能一步到位，需要渐进式迁移：

```
阶段 1：双写
┌──────────┐
│  PHP     │──→ JSON 格式（新版）  ←── Python/Go 工具读写（使用新版）
│ RSS-Bridge│──→ serialize 格式（旧版）  ←── 兼容旧数据
└──────────┘
读取时先尝试 JSON 解析，失败则 fallback 到 unserialize

阶段 2：切换读
PHP 代码只读取 JSON 格式，不再读取 serialize 格式
旧数据通过 prune/TTL 过期自动清理

阶段 3：清理
移除 serialize 相关代码，文档中注明 JSON + SHA-1 为正式缓存协议
```

### 18.7 标准化后的缓存协议文档示例

如果完成了标准化，建议将以下内容固化为接口规范，方便其他语言 SDK 参考实现：

```
RSS-Bridge Cache Protocol v1.0

1. Key 格式:
   - 业务 key: "{prefix}:{payload}"
     prefix ∈ {http, error_reporting, server, pages, lock}
     或 "{BridgeShortName}:{payload}"
   - 存储 key: sha1_hex(业务 key)，40 字符小写十六进制

2. Value 格式（JSON）:
   {
     "v":   1,                // 协议版本，当前为 1
     "ts":  1718323200,       // unix 写入时间戳
     "ttl": 3600,             // TTL（秒），0 表示永久
     "d":   <任意 JSON 值>    // 实际业务数据
   }

3. 操作语义:
   - get(key): 返回 d 字段，若 ts+ttl < now 视为过期
   - set(key, value, ttl): 按上述 JSON 格式写入
   - delete(key): 物理删除
   - clear(): 删除所有 key
   - prune(): 删除 ts+ttl < now 的所有 key（TTL 由服务端管理的后端无需实现）
```

