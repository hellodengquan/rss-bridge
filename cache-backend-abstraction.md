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
