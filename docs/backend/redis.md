# Redis

Redis 是一个开源的内存数据结构存储，可用作数据库、缓存和消息代理。Rust 生态通过 `redis` crate 提供异步/同步客户端，配合 `deadpool-redis` 实现生产级连接池管理。

**核心特点：**

| 特点 | 说明 |
|------|------|
| **内存存储** | 所有数据存储在内存中，读写速度极快（微秒级） |
| **丰富数据结构** | String、List、Hash、Set、ZSet、Stream、Bitmap 等 |
| **持久化** | RDB 快照 + AOF 日志，支持数据恢复 |
| **发布/订阅** | 内置 Pub/Sub 消息模式 |
| **原子操作** | 单命令原子性，支持 Lua 脚本和事务 |
| **集群支持** | 主从复制、哨兵、Cluster 模式 |

---

## 安装与项目初始化

```toml
# Cargo.toml
[dependencies]
redis           = { version = "0.27", features = ["tokio-comp", "connection-manager"] }
deadpool-redis  = "0.18"
tokio           = { version = "1", features = ["full"] }
serde           = { version = "1", features = ["derive"] }
serde_json      = "1"
anyhow          = "1"
```

### 快速连接

```rust
use redis::AsyncCommands;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = redis::Client::open("redis://127.0.0.1:6379")?;
    let mut conn = client.get_multiplexed_async_connection().await?;

    conn.set("hello", "world").await?;
    let val: String = conn.get("hello").await?;
    println!("{val}"); // world

    Ok(())
}
```

---

## 连接池

生产环境使用 `deadpool-redis` 管理连接池，避免频繁建立/销毁连接。

```rust
use deadpool_redis::{Config, Pool, Runtime};

pub fn create_pool(url: &str) -> Pool {
    let cfg = Config::from_url(url);
    cfg.create_pool(Some(Runtime::Tokio1)).expect("创建 Redis 连接池失败")
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = create_pool("redis://127.0.0.1:6379");
    let mut conn = pool.get().await?;

    redis::cmd("SET")
        .arg("key")
        .arg("value")
        .query_async::<()>(&mut *conn)
        .await?;

    Ok(())
}
```

### 带密码和数据库编号

```rust
// redis://:password@host:port/db
let pool = create_pool("redis://:mypassword@127.0.0.1:6379/1");
```

---

## 字符串（String）

最基础的数据类型，可存储文本、数字或二进制数据（上限 512 MB）。

```rust
use redis::AsyncCommands;

async fn string_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // SET / GET
    conn.set("name", "Alice").await?;
    let name: String = conn.get("name").await?;

    // 设置过期时间（秒）
    conn.set_ex("token", "abc123", 3600).await?;

    // 仅当键不存在时设置（分布式锁常用）
    let ok: bool = conn.set_nx("lock", "1").await?;

    // 自增 / 自减
    conn.set("counter", 10).await?;
    let n: i64 = conn.incr("counter", 1).await?;  // 11
    let n: i64 = conn.decr("counter", 3).await?;  // 8

    // 批量操作
    conn.mset(&[("k1", "v1"), ("k2", "v2")]).await?;
    let vals: Vec<String> = conn.mget(&["k1", "k2"]).await?;

    // 获取并更新 TTL
    let ttl: i64 = conn.ttl("token").await?;      // 剩余秒数，-1 永不过期，-2 不存在
    conn.expire("name", 60).await?;

    Ok(())
}
```

---

## 哈希（Hash）

适合存储对象字段，类似关系型数据库中的一行记录。

```rust
use redis::AsyncCommands;

async fn hash_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // HSET 单字段 / 多字段
    conn.hset("user:1", "name", "Alice").await?;
    conn.hset_multiple("user:1", &[
        ("name", "Alice"),
        ("age", "30"),
        ("email", "alice@example.com"),
    ]).await?;

    // HGET / HMGET
    let name: String = conn.hget("user:1", "name").await?;
    let fields: Vec<String> = conn.hmget("user:1", &["name", "age"]).await?;

    // HGETALL → HashMap
    use std::collections::HashMap;
    let user: HashMap<String, String> = conn.hgetall("user:1").await?;

    // HDEL / HEXISTS
    conn.hdel("user:1", "email").await?;
    let exists: bool = conn.hexists("user:1", "age").await?;

    // HINCRBY
    conn.hincr("user:1", "login_count", 1).await?;

    Ok(())
}
```

### 序列化结构体到 Hash

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

async fn save_user(conn: &mut impl AsyncCommands, user: &User) -> anyhow::Result<()> {
    let key = format!("user:{}", user.id);
    let json = serde_json::to_string(user)?;
    conn.set_ex(key, json, 86400).await?;
    Ok(())
}

async fn load_user(conn: &mut impl AsyncCommands, id: u64) -> anyhow::Result<Option<User>> {
    let key = format!("user:{id}");
    let json: Option<String> = conn.get(key).await?;
    Ok(json.as_deref().map(serde_json::from_str).transpose()?)
}
```

---

## 列表（List）

有序双向链表，支持从两端插入/弹出，适合消息队列、最新记录等场景。

```rust
use redis::AsyncCommands;

async fn list_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // LPUSH / RPUSH（返回列表长度）
    let len: i64 = conn.lpush("queue", "task1").await?;
    conn.rpush("queue", &["task2", "task3"]).await?;

    // LPOP / RPOP
    let task: Option<String> = conn.lpop("queue", None).await?;

    // 阻塞弹出（超时秒数，0 = 永久阻塞）
    let item: Option<(String, String)> = conn.blpop(&["queue"], 5.0).await?;

    // LRANGE（含两端，-1 表示末尾）
    let items: Vec<String> = conn.lrange("queue", 0, -1).await?;

    // LLEN / LINDEX
    let len: i64 = conn.llen("queue").await?;
    let first: String = conn.lindex("queue", 0).await?;

    // 保留最近 N 条（滑动窗口）
    conn.ltrim("recent_logs", 0, 99).await?;

    Ok(())
}
```

---

## 集合（Set）

无序不重复元素集合，支持交集、并集、差集运算。

```rust
use redis::AsyncCommands;

async fn set_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // SADD / SREM
    conn.sadd("tags:post:1", &["rust", "redis", "backend"]).await?;
    conn.srem("tags:post:1", "backend").await?;

    // SMEMBERS / SISMEMBER
    let tags: Vec<String> = conn.smembers("tags:post:1").await?;
    let exists: bool = conn.sismember("tags:post:1", "rust").await?;

    // SCARD（元素数量）
    let count: i64 = conn.scard("tags:post:1").await?;

    // 集合运算
    conn.sadd("tags:post:2", &["rust", "axum"]).await?;
    let inter: Vec<String> = conn.sinter(&["tags:post:1", "tags:post:2"]).await?; // 交集
    let union: Vec<String> = conn.sunion(&["tags:post:1", "tags:post:2"]).await?; // 并集
    let diff:  Vec<String> = conn.sdiff(&["tags:post:1", "tags:post:2"]).await?;  // 差集

    Ok(())
}
```

---

## 有序集合（ZSet / Sorted Set）

每个元素关联一个浮点分数，按分数自动排序，适合排行榜、优先级队列。

```rust
use redis::AsyncCommands;

async fn zset_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // ZADD（分数, 成员）
    conn.zadd("leaderboard", "Alice", 1500.0).await?;
    conn.zadd_multiple("leaderboard", &[
        (1200.0, "Bob"),
        (1800.0, "Charlie"),
        (900.0,  "Dave"),
    ]).await?;

    // ZRANGE（按分数升序，withscores 返回分数）
    let top: Vec<(String, f64)> = conn.zrange_withscores("leaderboard", 0, -1).await?;

    // ZREVRANGE（降序，取前 N 名）
    let top3: Vec<String> = conn.zrevrange("leaderboard", 0, 2).await?;

    // ZSCORE / ZRANK
    let score: f64 = conn.zscore("leaderboard", "Alice").await?;
    let rank: i64  = conn.zrevrank("leaderboard", "Alice").await?; // 从0开始的降序排名

    // ZINCRBY（增加分数）
    conn.zincr("leaderboard", "Bob", 100.0).await?;

    // 按分数范围查询
    let mid: Vec<String> = conn.zrangebyscore("leaderboard", 1000.0, 1600.0).await?;

    // ZCARD（成员总数）
    let total: i64 = conn.zcard("leaderboard").await?;

    Ok(())
}
```

---

## 过期与键管理

```rust
use redis::AsyncCommands;

async fn key_ops(conn: &mut impl AsyncCommands) -> anyhow::Result<()> {
    // 检查键是否存在
    let exists: bool = conn.exists("mykey").await?;

    // 设置/查看/删除过期时间
    conn.expire("mykey", 60).await?;
    let ttl: i64 = conn.ttl("mykey").await?;
    conn.persist("mykey").await?; // 移除过期时间

    // 删除键
    conn.del("mykey").await?;
    conn.del(&["k1", "k2", "k3"]).await?;

    // 重命名
    conn.rename("old_key", "new_key").await?;

    // 扫描匹配键（生产环境禁用 KEYS，用 SCAN）
    let mut cursor = 0u64;
    loop {
        let (next, keys): (u64, Vec<String>) = redis::cmd("SCAN")
            .arg(cursor)
            .arg("MATCH")
            .arg("user:*")
            .arg("COUNT")
            .arg(100)
            .query_async(conn).await?;
        for key in keys {
            println!("{key}");
        }
        cursor = next;
        if cursor == 0 { break; }
    }

    Ok(())
}
```

---

## 事务（Multi/Exec）

Redis 事务将多个命令打包为一个原子操作，要么全部执行，要么全部不执行（注意：不支持回滚）。

```rust
use redis::AsyncCommands;

async fn transaction_example(conn: &mut redis::aio::MultiplexedConnection) -> anyhow::Result<()> {
    // WATCH + MULTI/EXEC 实现乐观锁
    loop {
        redis::cmd("WATCH").arg("balance").query_async::<()>(conn).await?;

        let balance: i64 = conn.get("balance").await.unwrap_or(0);
        if balance < 100 {
            redis::cmd("UNWATCH").query_async::<()>(conn).await?;
            return Err(anyhow::anyhow!("余额不足"));
        }

        let result: Option<((),)> = redis::pipe()
            .atomic()
            .cmd("DECRBY").arg("balance").arg(100).ignore()
            .query_async(conn)
            .await?;

        if result.is_some() {
            break; // 事务成功
        }
        // result 为 None 说明 WATCH 的键被修改，重试
    }
    Ok(())
}
```

---

## Pipeline（管道）

批量发送多条命令，减少网络往返次数。

```rust
use redis::AsyncCommands;

async fn pipeline_example(conn: &mut redis::aio::MultiplexedConnection) -> anyhow::Result<()> {
    let (v1, v2, v3): (String, i64, bool) = redis::pipe()
        .cmd("SET").arg("p1").arg("hello").ignore()
        .cmd("INCR").arg("counter")
        .cmd("EXISTS").arg("p1")
        .query_async(conn)
        .await?;

    println!("counter={v2}, exists={v3}");
    Ok(())
}
```

---

## Lua 脚本

Lua 脚本在 Redis 服务端原子执行，适合需要读-改-写的复合操作。

```rust
use redis::Script;

async fn lua_example(conn: &mut redis::aio::MultiplexedConnection) -> anyhow::Result<()> {
    // 原子性地检查并消费限流令牌
    let script = Script::new(r#"
        local current = redis.call('GET', KEYS[1])
        if current and tonumber(current) >= tonumber(ARGV[1]) then
            return redis.call('DECRBY', KEYS[1], 1)
        end
        return -1
    "#);

    let remaining: i64 = script
        .key("rate:user:123")
        .arg(1)
        .invoke_async(conn)
        .await?;

    if remaining >= 0 {
        println!("剩余配额: {remaining}");
    } else {
        println!("配额不足");
    }

    Ok(())
}
```

---

## 发布/订阅（Pub/Sub）

```rust
use redis::AsyncCommands;

// 发布者
async fn publish(conn: &mut impl AsyncCommands, channel: &str, msg: &str) -> anyhow::Result<()> {
    let receivers: i64 = conn.publish(channel, msg).await?;
    println!("消息发送给 {receivers} 个订阅者");
    Ok(())
}

// 订阅者
async fn subscribe_loop(client: &redis::Client) -> anyhow::Result<()> {
    let conn = client.get_async_pubsub().await?;
    let mut pubsub = conn;
    pubsub.subscribe("notifications").await?;

    use futures_util::StreamExt;
    let mut stream = pubsub.on_message();
    while let Some(msg) = stream.next().await {
        let payload: String = msg.get_payload()?;
        println!("收到消息: {payload}");
    }
    Ok(())
}
```

---

## 常用场景

### 缓存（Cache-Aside 模式）

```rust
use deadpool_redis::Pool;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Article {
    id: u64,
    title: String,
    content: String,
}

async fn get_article(pool: &Pool, id: u64) -> anyhow::Result<Article> {
    let mut conn = pool.get().await?;
    let key = format!("article:{id}");

    // 先查缓存
    if let Ok(json) = redis::cmd("GET").arg(&key).query_async::<String>(&mut *conn).await {
        return Ok(serde_json::from_str(&json)?);
    }

    // 缓存未命中，查数据库（示意）
    let article = Article {
        id,
        title: format!("文章 {id}"),
        content: "内容...".to_string(),
    };

    // 写入缓存，TTL 10 分钟
    let json = serde_json::to_string(&article)?;
    redis::cmd("SET").arg(&key).arg(&json).arg("EX").arg(600)
        .query_async::<()>(&mut *conn).await?;

    Ok(article)
}
```

### 分布式限流（滑动窗口）

```rust
use redis::Script;

/// 返回 true 表示允许请求，false 表示超出限制
async fn rate_limit(
    conn: &mut redis::aio::MultiplexedConnection,
    key: &str,
    limit: u64,
    window_secs: u64,
) -> anyhow::Result<bool> {
    let script = Script::new(r#"
        local key    = KEYS[1]
        local limit  = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now    = tonumber(ARGV[3])

        redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)
        local count = redis.call('ZCARD', key)
        if count < limit then
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, window)
            return 1
        end
        return 0
    "#);

    use std::time::{SystemTime, UNIX_EPOCH};
    let now = SystemTime::now().duration_since(UNIX_EPOCH)?.as_millis() as u64;

    let allowed: i64 = script
        .key(key)
        .arg(limit)
        .arg(window_secs)
        .arg(now)
        .invoke_async(conn)
        .await?;

    Ok(allowed == 1)
}
```

### 分布式锁

```rust
use redis::AsyncCommands;
use std::time::Duration;
use uuid::Uuid;

pub struct RedisLock {
    key: String,
    token: String,
}

impl RedisLock {
    /// 尝试获取锁，ttl_secs 为自动释放时间
    pub async fn acquire(
        conn: &mut impl AsyncCommands,
        key: &str,
        ttl_secs: u64,
    ) -> anyhow::Result<Option<Self>> {
        let token = Uuid::new_v4().to_string();
        let ok: bool = conn.set_options(
            key,
            &token,
            redis::SetOptions::default()
                .conditional_set(redis::ExistenceCheck::NX)
                .with_expiration(redis::SetExpiry::EX(ttl_secs)),
        ).await?;

        if ok {
            Ok(Some(Self { key: key.to_string(), token }))
        } else {
            Ok(None)
        }
    }

    /// 释放锁（Lua 脚本保证原子性：只释放自己持有的锁）
    pub async fn release(&self, conn: &mut redis::aio::MultiplexedConnection) -> anyhow::Result<()> {
        let script = redis::Script::new(r#"
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            end
            return 0
        "#);
        script.key(&self.key).arg(&self.token).invoke_async::<i64>(conn).await?;
        Ok(())
    }
}
```

---

## 与 Axum 集成

```toml
# Cargo.toml
[dependencies]
axum           = "0.7"
tokio          = { version = "1", features = ["full"] }
deadpool-redis = "0.18"
redis          = { version = "0.27", features = ["tokio-comp"] }
serde          = { version = "1", features = ["derive"] }
serde_json     = "1"
```

```rust
use axum::{extract::State, routing::get, Json, Router};
use deadpool_redis::{Config, Pool, Runtime};
use redis::AsyncCommands;
use serde_json::{json, Value};

#[derive(Clone)]
struct AppState {
    redis: Pool,
}

async fn get_counter(State(state): State<AppState>) -> Json<Value> {
    let mut conn = state.redis.get().await.unwrap();
    let count: i64 = conn.incr("visit_count", 1).await.unwrap_or(0);
    Json(json!({ "count": count }))
}

#[tokio::main]
async fn main() {
    let cfg = Config::from_url("redis://127.0.0.1:6379");
    let pool = cfg.create_pool(Some(Runtime::Tokio1)).unwrap();

    let state = AppState { redis: pool };
    let app = Router::new()
        .route("/counter", get(get_counter))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 常用命令速查

| 场景 | 命令 | Rust 方法 |
|------|------|-----------|
| 字符串读写 | `SET / GET` | `conn.set() / conn.get()` |
| 带过期时间 | `SET key val EX 60` | `conn.set_ex()` |
| 不存在则设置 | `SET key val NX` | `conn.set_nx()` |
| 自增计数器 | `INCR / INCRBY` | `conn.incr()` |
| 哈希字段 | `HSET / HGET / HGETALL` | `conn.hset() / conn.hget() / conn.hgetall()` |
| 列表推入/弹出 | `LPUSH / RPOP` | `conn.lpush() / conn.rpop()` |
| 阻塞弹出 | `BLPOP` | `conn.blpop()` |
| 集合操作 | `SADD / SMEMBERS / SINTER` | `conn.sadd() / conn.smembers() / conn.sinter()` |
| 有序集合 | `ZADD / ZREVRANGE / ZSCORE` | `conn.zadd() / conn.zrevrange() / conn.zscore()` |
| 发布消息 | `PUBLISH` | `conn.publish()` |
| 删除键 | `DEL` | `conn.del()` |
| 检查存在 | `EXISTS` | `conn.exists()` |
| 剩余过期 | `TTL` | `conn.ttl()` |
| 安全扫描键 | `SCAN` | `redis::cmd("SCAN")` |
