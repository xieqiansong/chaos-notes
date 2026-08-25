# Redis 学习记录

> 工程依托：`chaos-java/jdk8-platform/jdk8-redis-demo`（Spring Data Redis + Redisson，多模块覆盖）。
> 记录 Redis 的数据结构用法、分布式锁、限流等高频模式的实现与坑。

## 1. 常用数据结构与场景

| 结构 | demo 类 | 典型场景 |
|---|---|---|
| String | `StringCacheService` | 缓存、计数器、分布式锁 value |
| Hash | `CollectionService` | 对象字段缓存、购物车 |
| List/Set/ZSet | `CollectionService` / `RankService` | 队列、去重、排行榜（ZSet 按 score 排序） |
| Bitmap/HyperLogLog | （计划） | 在线统计、UV 估算 |

`RankService` 用 `ZSet` 做排行榜：`ZADD` 写入分数、`ZREVRANGE` 取 TopN、`ZINCRBY` 实时更新。

## 2. 分布式锁（SET NX EX + Lua 释放）

`DistributedLock` 基于 `SET key value NX EX`：

```java
// 加锁：key 不存在才能成功，原子 + 过期防止死锁
stringRedisTemplate.opsForValue().setIfAbsent(LOCK_KEY + lockKey, requestId, expireSeconds, TimeUnit.SECONDS);

// 释放：必须校验 value 后再删，防止误删他人锁
String RELEASE_SCRIPT =
    "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
stringRedisTemplate.execute(new DefaultRedisScript<>(RELEASE_SCRIPT, Long.class),
    Collections.singletonList(LOCK_KEY + lockKey), requestId);
```

**要点**：
- 加锁用 `setIfAbsent`（= `SET NX`），并带过期时间防死锁。
- 释放锁必须用 **Lua 脚本**保证「校验 value == 自己 requestId → 删除」原子完成，否则可能删掉别人刚获得的锁。
- `withLock(lockKey, expire, action)` 模板方法：加锁 → 业务 → `finally` 释放，避免忘记解锁。

## 3. 限流（固定窗口 + Lua 原子）

`RateLimitService` 用 Lua 做固定窗口限流：

```lua
local current = redis.call('incr', KEYS[1])
if current == 1 then redis.call('expire', KEYS[1], tonumber(ARGV[1])) end
if current > tonumber(ARGV[2]) then return 0 else return 1 end
```

**为什么用 Lua**：限流要判断「当前计数 + 是否超阈值」，两步操作需原子，否则并发下「读旧值→判断→写」竞态会让限流失效；用 Lua 把 `INCR + EXPIRE + 比较` 缩成一次原子执行。

**坑点**：固定窗口在窗口临界点有「双倍突发」（如 0s 与 1s 各放满窗口）。生产常用**滑动窗口 / 令牌桶**（Redis-Cell、Gateway 内置限流或 Sentinel 兜底）。

## 4. 其他高频模式

- **计数器** `CounterService`：`INCR` / `INCRBY`，注意 `INCR` 后 key 不带 TTL 会常驻，需主动 `EXPIRE`。
- **批量管道** `PipelineService`：`executePipelined` 把多条命令合并一次网络往返，高并发写入首选（呼应 performance 笔记的批量优化）。
- **发布订阅** `PubSubService`：`RedisTemplate.convertAndSend` + `@RedisListener` 监听器，轻量解耦，但无持久化、无 ACK，不适合可靠消息。
- **Redisson**：`RedissonTest` 验证 `RLock`（看门狗自动续期）、`RScoredSortedSet` 等，分布式锁生产更推荐 Redisson 而非手写 Lua。

## 5. 学习踩坑点

- **key 命名与 TTL**：所有业务 key 应有统一前缀（demo 用 `RedisKeyConstants`），避免冲突；缓存类 key 务必设过期，防止内存只增不减。
- **大 key / 热 key**：ZSet、Hash 元素过多会阻塞单线程；大列表用 `SCAN` 而非 `KEYS`。
- **序列化一致性**：`StringRedisTemplate` 与 `RedisTemplate` 默认序列化不同，混用会读不到对方写入的值。

## 6. 参考来源

- 工程：`chaos-java/jdk8-platform/jdk8-redis-demo`
- 性能专题：见 `performance/Redis批量入库-自适应批量大小.md`（批量命令量优化）
