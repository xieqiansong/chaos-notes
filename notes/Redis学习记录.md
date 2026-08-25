# Redis 学习记录

工程依托：[jdk8-redis-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-redis-demo)（`chaos-java` 仓库，Spring Data Redis + Redisson，多模块覆盖）。

## 1. 安装

单节点，挂载数据/配置/日志，开了 AOF（`appendonly yes`），映射到宿主机 `30102`。内存上限 1gb，淘汰策略 `allkeys-lru`（内存满时按 LRU 删所有 key）。

`docker-compose.yaml`：

```yaml
services:
  redis:
    image: redis:latest
    container_name: redis
    restart: unless-stopped
    volumes:
      - /data/redis/data:/data
      - /data/redis/redis.conf:/etc/redis/redis.conf
      - /data/redis/logs:/logs
    command: redis-server /etc/redis/redis.conf --appendonly yes
    ports:
      - 30102:6379
```

`redis.conf`：

```conf
maxmemory 1gb
maxmemory-policy allkeys-lru
```

`docker-compose up -d` 起来，客户端连 `localhost:30102`。

## 2. 常用数据结构

Spring 里用 `StringRedisTemplate` 操作，最常用的几个：

```java
// String：缓存 / 计数器
template.opsForValue().set("user:1:name", "张三", 30, TimeUnit.MINUTES);
template.opsForValue().increment("counter");

// Hash：对象字段缓存 / 购物车
template.opsForHash().put("cart:1", "sku-100", "2");

// ZSet：排行榜（按 score 排序）
template.opsForZSet().add("rank", "userA", 100);
template.opsForZSet().reverseRange("rank", 0, 9);   // Top 10
```

```redis
# 对应 Redis 指令
SET user:1:name 张三 EX 1800
INCR counter
HSET cart:1 sku-100 2
ZADD rank 100 userA
ZREVRANGE rank 0 9 WITHSCORES
```

## 3. 分布式锁

加锁用 `SET NX EX`（= `setIfAbsent` 带过期），释放必须用 Lua 校验 value 后再删，否则可能删掉别人的锁。

```java
// 加锁
template.opsForValue().setIfAbsent("lock:order", requestId, 30, TimeUnit.SECONDS);

// 释放（Lua 原子校验）
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
template.execute(new DefaultRedisScript<>(script, Long.class),
        Collections.singletonList("lock:order"), requestId);
```

```redis
# 对应 Redis 指令
SET lock:order <requestId> NX EX 30
# 释放：GET lock:order 比对 value，一致才 DEL lock:order
GET lock:order
DEL lock:order
```

> 真要上生产更推荐直接用 Redisson 的 `RLock`（带看门狗自动续期），不用自己手写 Lua。

## 4. 限流（固定窗口）

固定窗口限流，用 Lua 把 `INCR + EXPIRE + 判断` 合成一次原子执行，避免并发竞态。

```lua
local current = redis.call('incr', KEYS[1])
if current == 1 then redis.call('expire', KEYS[1], tonumber(ARGV[1])) end
if current > tonumber(ARGV[2]) then return 0 else return 1 end
```

```redis
# 对应 Redis 指令
INCR <key>
EXPIRE <key> <windowSeconds>
# 超过阈值返回 0（拒绝），否则返回 1（放行）
```

> 固定窗口在临界点有「双倍突发」，生产常用滑动窗口 / 令牌桶（Redis-Cell、Gateway、Sentinel）。

## 5. 批量管道

多条命令合并成一次网络往返，高并发写入首选。

```java
template.executePipelined((RedisCallback<Object>) connection -> {
    for (int i = 0; i < 1000; i++) {
        connection.stringCommands().set(("k:" + i).getBytes(), "v".getBytes());
    }
    return null;
});
```

```redis
# 管道内每条命令对应 Redis 指令（1000 条合并为一次网络往返发送）
SET k:0 v
SET k:1 v
...
SET k:999 v
```

## 6. 发布订阅

轻量解耦用，但无持久化、无 ACK，别当可靠消息队列。

```java
// 发
template.convertAndSend("topic:news", "hello");

// 收
@RedisListener(topics = "topic:news")
public void onMessage(String msg) {
    System.out.println("收到: " + msg);
}
```

```redis
# 对应 Redis 指令
PUBLISH topic:news hello
SUBSCRIBE topic:news
```

## 参考来源

- 工程：[jdk8-redis-demo](https://github.com/xieqiansong/chaos-java/tree/main/jdk8-platform/jdk8-redis-demo)
- 性能专题：见 `performance/Redis批量入库-自适应批量大小.md`（批量命令量优化）
