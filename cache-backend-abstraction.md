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

---

## 四、缓存与抓取流程的关联

缓存不是孤立存在的，它在三个层次上介入抓取流程，形成完整的缓存读写链路：

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

### 4.1 第一层：CacheMiddleware — HTTP 响应缓存

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

### 4.2 第二层：DisplayAction — Feed 数据缓存

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

### 4.3 第三层：getContents() — 服务端响应缓存

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

### 4.4 第四层：Bridge 内部缓存

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

## 五、依赖注入与对象传递链路

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

## 六、添加新缓存后端

若需添加新的缓存后端（如 Redis），只需：

1. 在 `caches/` 目录下创建 `RedisCache.php`
2. 实现 `CacheInterface` 的 5 个方法
3. 在 `CacheFactory::create()` 的 `switch` 中添加 `RedisCache` 分支（如需特殊构造参数），或依赖 `default` 分支的无参构造
4. 在配置文件中设置 `cache.type = "Redis"`

无需修改任何上层代码（Middleware、Action、Bridge），因为它们只依赖 `CacheInterface`。
