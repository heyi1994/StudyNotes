# sqlx

sqlx 是 Rust 生态中最主流的异步数据库驱动库，支持 PostgreSQL、MySQL/MariaDB、SQLite 和 MSSQL。它最核心的优势是**编译期 SQL 检查**——通过 `query!` 宏在编译时连接真实数据库，验证 SQL 语法、列名、参数类型，把运行时错误提前暴露在编译阶段。

**核心特性：**

| 特性 | 说明 |
|------|------|
| **编译期 SQL 检查** | `query!` 宏在编译时检查 SQL 正确性和类型匹配 |
| **纯异步** | 基于 `tokio`/`async-std`，无阻塞 |
| **连接池** | 内置 `Pool`，自动管理连接生命周期 |
| **类型安全映射** | Rust 类型与数据库类型的双向静态映射 |
| **迁移工具** | 内置 `sqlx-cli`，版本化管理数据库结构变更 |
| **多数据库** | 同一套 API 支持 PostgreSQL、MySQL、SQLite |

---

## 安装

### Cargo.toml

```toml
[dependencies]
# PostgreSQL（生产环境推荐）
sqlx = { version = "0.7", features = [
    "runtime-tokio-rustls",   # tokio 运行时 + TLS
    "postgres",               # PostgreSQL 驱动
    "uuid",                   # Uuid 类型映射
    "chrono",                 # DateTime 类型映射
    "json",                   # serde_json::Value 映射
    "migrate",                # 运行迁移
] }

tokio  = { version = "1", features = ["full"] }
serde  = { version = "1", features = ["derive"] }
uuid   = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

# MySQL
# sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "mysql"] }

# SQLite（轻量、本地开发）
# sqlx = { version = "0.7", features = ["runtime-tokio-native-tls", "sqlite"] }
```

### 安装 sqlx-cli

```shell
# 安装命令行工具（用于迁移管理）
cargo install sqlx-cli --no-default-features --features rustls,postgres

# 支持多数据库
cargo install sqlx-cli --no-default-features --features rustls,postgres,mysql,sqlite

# 验证
sqlx --version
```

---

## 连接与连接池

### 创建连接池

```rust
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 从环境变量读取（推荐，避免硬编码密码）
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL 未设置");

    let pool = PgPoolOptions::new()
        .max_connections(20)       // 最大连接数
        .min_connections(2)        // 最小保持连接数
        .acquire_timeout(std::time::Duration::from_secs(5))   // 获取连接超时
        .idle_timeout(std::time::Duration::from_secs(600))    // 空闲连接超时
        .max_lifetime(std::time::Duration::from_secs(1800))   // 连接最大生存时间
        .connect(&database_url)
        .await?;

    println!("数据库连接成功，池大小：{}", pool.size());
    Ok(())
}
```

### DATABASE_URL 格式

```shell
# .env 文件
# PostgreSQL
DATABASE_URL=postgres://username:password@localhost:5432/dbname
DATABASE_URL=postgres://username:password@localhost:5432/dbname?sslmode=require

# MySQL
DATABASE_URL=mysql://username:password@localhost:3306/dbname

# SQLite
DATABASE_URL=sqlite://./data.db         # 相对路径
DATABASE_URL=sqlite:///absolute/path.db # 绝对路径
DATABASE_URL=sqlite::memory:            # 内存数据库（测试用）
```

### 懒连接（应用启动时不立即验证）

```rust
// connect_lazy：延迟到第一次查询时才建立连接
let pool = PgPoolOptions::new()
    .max_connections(10)
    .connect_lazy(&database_url)?;  // 注意：这里是 ? 不是 .await?
```

---

## 数据库迁移

### 目录结构

```
project/
├── .env
├── migrations/
│   ├── 20240101000000_create_users.sql
│   ├── 20240102000000_create_posts.sql
│   └── 20240103000000_add_user_index.sql
└── src/
    └── main.rs
```

### sqlx-cli 常用命令

```shell
# 创建数据库（根据 DATABASE_URL）
sqlx database create

# 删除数据库
sqlx database drop

# 创建新迁移文件（自动加时间戳前缀）
sqlx migrate add create_users
sqlx migrate add add_email_index

# 执行所有未运行的迁移
sqlx migrate run

# 回滚最近一次迁移
sqlx migrate revert

# 查看迁移状态
sqlx migrate info

# 生成离线查询缓存（CI 环境不需要数据库即可编译）
cargo sqlx prepare
```

### 迁移文件示例

```sql
-- migrations/20240101000000_create_users.sql
CREATE TABLE IF NOT EXISTS users (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(50) NOT NULL,
    email      VARCHAR(255) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,
    avatar_url TEXT,
    is_active  BOOLEAN     NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- 自动更新 updated_at 的触发器
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

```sql
-- migrations/20240102000000_create_posts.sql
CREATE TABLE IF NOT EXISTS posts (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id    UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      VARCHAR(255) NOT NULL,
    content    TEXT        NOT NULL,
    tags       TEXT[]      DEFAULT '{}',
    view_count INTEGER     NOT NULL DEFAULT 0,
    published  BOOLEAN     NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_posts_user_id   ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published) WHERE published = true;

CREATE TRIGGER posts_updated_at
    BEFORE UPDATE ON posts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### 在代码中运行迁移

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = connect_db().await?;

    // 应用启动时自动运行未执行的迁移
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await?;

    println!("迁移执行完毕");
    Ok(())
}
```

---

## 基础查询

### query! 宏（编译期检查）

`query!` 宏会在**编译时**连接数据库验证 SQL，返回匿名结构体。

```rust
use sqlx::PgPool;
use uuid::Uuid;

// 查询返回匿名结构体，通过字段名访问
async fn get_user_name(pool: &PgPool, id: Uuid) -> anyhow::Result<Option<String>> {
    let row = sqlx::query!(
        "SELECT name FROM users WHERE id = $1",
        id
    )
    .fetch_optional(pool)
    .await?;

    Ok(row.map(|r| r.name))
}

// 多列查询
async fn get_user_info(pool: &PgPool, id: Uuid) -> anyhow::Result<()> {
    let row = sqlx::query!(
        "SELECT id, name, email, created_at FROM users WHERE id = $1",
        id
    )
    .fetch_one(pool)   // 找不到则返回错误
    .await?;

    println!("ID: {}, 名字: {}, 邮箱: {}", row.id, row.name, row.email);
    Ok(())
}
```

### 执行方式

```rust
// fetch_one    - 返回恰好一行，0 行或多行都报错
// fetch_optional - 返回 0 或 1 行，多行报错
// fetch_all    - 返回所有行（Vec）
// fetch        - 返回流（Stream），适合大数据集
// execute      - 不返回行，返回影响行数（INSERT/UPDATE/DELETE）

let user = sqlx::query!("SELECT * FROM users WHERE id = $1", id)
    .fetch_one(&pool)
    .await?;

let user = sqlx::query!("SELECT * FROM users WHERE id = $1", id)
    .fetch_optional(&pool)
    .await?;  // Option<_>

let users = sqlx::query!("SELECT * FROM users ORDER BY created_at DESC")
    .fetch_all(&pool)
    .await?;  // Vec<_>

let result = sqlx::query!("DELETE FROM users WHERE id = $1", id)
    .execute(&pool)
    .await?;
println!("删除了 {} 行", result.rows_affected());
```

### query_as! 宏（映射到结构体）

```rust
use sqlx::FromRow;
use chrono::{DateTime, Utc};

#[derive(Debug, FromRow, serde::Serialize, serde::Deserialize, Clone)]
pub struct User {
    pub id:         Uuid,
    pub name:       String,
    pub email:      String,
    pub avatar_url: Option<String>,
    pub is_active:  bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

// 编译期检查 SQL，并映射到 User 结构体
async fn find_user(pool: &PgPool, id: Uuid) -> sqlx::Result<Option<User>> {
    sqlx::query_as!(
        User,
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at
         FROM users WHERE id = $1",
        id
    )
    .fetch_optional(pool)
    .await
}

async fn list_active_users(pool: &PgPool) -> sqlx::Result<Vec<User>> {
    sqlx::query_as!(
        User,
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at
         FROM users WHERE is_active = true
         ORDER BY created_at DESC"
    )
    .fetch_all(pool)
    .await
}
```

### query_scalar! 宏（单个标量值）

```rust
// 查询单个值（COUNT、MAX、SUM 等聚合函数）
async fn count_users(pool: &PgPool) -> sqlx::Result<i64> {
    sqlx::query_scalar!("SELECT COUNT(*) FROM users")
        .fetch_one(pool)
        .await
        .map(|c| c.unwrap_or(0))
}

async fn user_exists(pool: &PgPool, email: &str) -> sqlx::Result<bool> {
    sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM users WHERE email = $1)",
        email
    )
    .fetch_one(pool)
    .await
    .map(|e| e.unwrap_or(false))
}

async fn max_view_count(pool: &PgPool) -> sqlx::Result<Option<i32>> {
    sqlx::query_scalar!("SELECT MAX(view_count) FROM posts")
        .fetch_one(pool)
        .await
}
```

---

## CRUD 完整示例

```rust
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, sqlx::FromRow, serde::Serialize, serde::Deserialize, Clone)]
pub struct User {
    pub id:         Uuid,
    pub name:       String,
    pub email:      String,
    pub avatar_url: Option<String>,
    pub is_active:  bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

pub struct CreateUserDto {
    pub name:      String,
    pub email:     String,
    pub password:  String,
}

pub struct UpdateUserDto {
    pub name:       Option<String>,
    pub email:      Option<String>,
    pub avatar_url: Option<String>,
}

// ── 查询列表 ──────────────────────────────────────────────────────────
pub async fn list_users(
    pool: &PgPool,
    page: u32,
    per_page: u32,
) -> sqlx::Result<Vec<User>> {
    let offset = ((page.saturating_sub(1)) * per_page) as i64;
    let limit  = per_page as i64;

    sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users
        WHERE is_active = true
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
        "#,
        limit,
        offset,
    )
    .fetch_all(pool)
    .await
}

// ── 查询单个 ──────────────────────────────────────────────────────────
pub async fn find_user_by_id(pool: &PgPool, id: Uuid) -> sqlx::Result<Option<User>> {
    sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users WHERE id = $1
        "#,
        id,
    )
    .fetch_optional(pool)
    .await
}

pub async fn find_user_by_email(pool: &PgPool, email: &str) -> sqlx::Result<Option<User>> {
    sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users WHERE email = $1
        "#,
        email,
    )
    .fetch_optional(pool)
    .await
}

// ── 创建 ──────────────────────────────────────────────────────────────
pub async fn create_user(pool: &PgPool, dto: CreateUserDto) -> sqlx::Result<User> {
    // 实际项目中密码需要用 bcrypt/argon2 哈希
    let password_hash = format!("hashed_{}", dto.password);

    sqlx::query_as!(
        User,
        r#"
        INSERT INTO users (id, name, email, password, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())
        RETURNING id, name, email, avatar_url, is_active, created_at, updated_at
        "#,
        Uuid::new_v4(),
        dto.name,
        dto.email,
        password_hash,
    )
    .fetch_one(pool)
    .await
}

// ── 更新（部分字段）──────────────────────────────────────────────────
pub async fn update_user(
    pool: &PgPool,
    id: Uuid,
    dto: UpdateUserDto,
) -> sqlx::Result<Option<User>> {
    sqlx::query_as!(
        User,
        r#"
        UPDATE users
        SET
            name       = COALESCE($2, name),
            email      = COALESCE($3, email),
            avatar_url = COALESCE($4, avatar_url),
            updated_at = NOW()
        WHERE id = $1 AND is_active = true
        RETURNING id, name, email, avatar_url, is_active, created_at, updated_at
        "#,
        id,
        dto.name,
        dto.email,
        dto.avatar_url,
    )
    .fetch_optional(pool)
    .await
}

// ── 软删除 ──────────────────────────────────────────────────────────
pub async fn soft_delete_user(pool: &PgPool, id: Uuid) -> sqlx::Result<bool> {
    let result = sqlx::query!(
        "UPDATE users SET is_active = false, updated_at = NOW() WHERE id = $1 AND is_active = true",
        id,
    )
    .execute(pool)
    .await?;

    Ok(result.rows_affected() > 0)
}

// ── 硬删除 ──────────────────────────────────────────────────────────
pub async fn delete_user(pool: &PgPool, id: Uuid) -> sqlx::Result<bool> {
    let result = sqlx::query!("DELETE FROM users WHERE id = $1", id)
        .execute(pool)
        .await?;

    Ok(result.rows_affected() > 0)
}
```

---

## 事务

### 基础事务

```rust
use sqlx::PgPool;

pub async fn transfer_balance(
    pool:    &PgPool,
    from_id: Uuid,
    to_id:   Uuid,
    amount:  i64,
) -> sqlx::Result<()> {
    // begin() 开启事务，返回 Transaction
    let mut tx = pool.begin().await?;

    // 检查余额（使用 FOR UPDATE 加行锁，防止并发超扣）
    let balance = sqlx::query_scalar!(
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
        from_id
    )
    .fetch_one(&mut *tx)   // 注意：在事务上执行用 &mut *tx
    .await?
    .unwrap_or(0);

    if balance < amount {
        // 不需要显式 rollback，tx drop 时自动回滚
        return Err(sqlx::Error::Protocol("余额不足".to_string()));
    }

    sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount,
        from_id
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount,
        to_id
    )
    .execute(&mut *tx)
    .await?;

    // 手动提交
    tx.commit().await?;
    Ok(())
}
```

### 保存点（Savepoint）

```rust
pub async fn batch_insert_with_savepoint(
    pool:  &PgPool,
    items: Vec<Item>,
) -> sqlx::Result<()> {
    let mut tx = pool.begin().await?;

    for item in items {
        // 为每个条目创建保存点
        let sp = tx.begin().await?;  // 嵌套事务 = 保存点

        let result = sqlx::query!(
            "INSERT INTO items (id, name) VALUES ($1, $2)",
            item.id,
            item.name
        )
        .execute(&mut *sp)
        .await;

        match result {
            Ok(_)  => sp.commit().await?,    // 提交保存点
            Err(e) => {
                tracing::warn!("跳过失败条目 {}: {e}", item.id);
                // sp drop 时自动回滚到保存点，不影响外层事务
            }
        }
    }

    tx.commit().await
}
```

### 事务辅助宏模式

```rust
// 封装事务逻辑的通用函数
pub async fn with_transaction<F, T>(
    pool: &PgPool,
    f: F,
) -> sqlx::Result<T>
where
    F: for<'c> FnOnce(
        &'c mut sqlx::Transaction<'_, sqlx::Postgres>,
    ) -> futures::future::BoxFuture<'c, sqlx::Result<T>>,
{
    let mut tx = pool.begin().await?;
    match f(&mut tx).await {
        Ok(result) => {
            tx.commit().await?;
            Ok(result)
        }
        Err(e) => {
            tx.rollback().await?;
            Err(e)
        }
    }
}

// 使用
with_transaction(&pool, |tx| {
    Box::pin(async move {
        sqlx::query!("INSERT INTO a VALUES ($1)", 1).execute(&mut **tx).await?;
        sqlx::query!("INSERT INTO b VALUES ($1)", 2).execute(&mut **tx).await?;
        Ok(())
    })
})
.await?;
```

---

## 动态查询

`query!` 宏要求 SQL 在编译期确定，动态查询需要用非宏版本的 `query` / `query_as`。

### QueryBuilder（推荐）

```rust
use sqlx::QueryBuilder;

pub struct UserFilter {
    pub name:      Option<String>,
    pub email:     Option<String>,
    pub is_active: Option<bool>,
    pub page:      u32,
    pub per_page:  u32,
}

pub async fn search_users(
    pool:   &PgPool,
    filter: UserFilter,
) -> sqlx::Result<Vec<User>> {
    let mut builder = QueryBuilder::<sqlx::Postgres>::new(
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at FROM users WHERE 1=1"
    );

    if let Some(name) = &filter.name {
        builder.push(" AND name ILIKE ");
        builder.push_bind(format!("%{name}%"));
    }

    if let Some(email) = &filter.email {
        builder.push(" AND email = ");
        builder.push_bind(email.clone());
    }

    if let Some(is_active) = filter.is_active {
        builder.push(" AND is_active = ");
        builder.push_bind(is_active);
    }

    let offset = ((filter.page.saturating_sub(1)) * filter.per_page) as i64;
    builder
        .push(" ORDER BY created_at DESC LIMIT ")
        .push_bind(filter.per_page as i64)
        .push(" OFFSET ")
        .push_bind(offset);

    builder
        .build_query_as::<User>()
        .fetch_all(pool)
        .await
}
```

### 批量插入（push_values）

```rust
pub async fn batch_insert_users(
    pool:  &PgPool,
    users: Vec<CreateUserDto>,
) -> sqlx::Result<u64> {
    if users.is_empty() {
        return Ok(0);
    }

    let mut builder = QueryBuilder::<sqlx::Postgres>::new(
        "INSERT INTO users (id, name, email, password, created_at, updated_at) "
    );

    builder.push_values(users, |mut row, dto| {
        row.push_bind(Uuid::new_v4())
           .push_bind(dto.name)
           .push_bind(dto.email)
           .push_bind(format!("hashed_{}", dto.password))
           .push_bind(chrono::Utc::now())
           .push_bind(chrono::Utc::now());
    });

    let result = builder.build().execute(pool).await?;
    Ok(result.rows_affected())
}
```

### IN 子句（动态参数列表）

```rust
// sqlx 不支持直接 IN ($1)，需要展开绑定
pub async fn get_users_by_ids(
    pool: &PgPool,
    ids:  &[Uuid],
) -> sqlx::Result<Vec<User>> {
    if ids.is_empty() {
        return Ok(vec![]);
    }

    // 方式一：ANY（PostgreSQL 专用，推荐）
    sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users WHERE id = ANY($1)
        "#,
        ids as &[Uuid],   // 类型标注告诉宏这是数组
    )
    .fetch_all(pool)
    .await
}

// 方式二：QueryBuilder 展开 IN 子句（跨数据库兼容）
pub async fn get_users_by_ids_v2(
    pool: &PgPool,
    ids:  &[Uuid],
) -> sqlx::Result<Vec<User>> {
    let mut builder = QueryBuilder::<sqlx::Postgres>::new(
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at
         FROM users WHERE id IN ("
    );

    let mut separated = builder.separated(", ");
    for id in ids {
        separated.push_bind(*id);
    }
    separated.push_unseparated(")");

    builder.build_query_as::<User>().fetch_all(pool).await
}
```

---

## 流式查询（大数据集）

```rust
use futures::TryStreamExt;

pub async fn process_all_users(pool: &PgPool) -> sqlx::Result<()> {
    // fetch() 返回 Stream，逐行读取，不会一次性加载到内存
    let mut stream = sqlx::query_as!(
        User,
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at
         FROM users ORDER BY created_at"
    )
    .fetch(pool);

    let mut count = 0u64;
    while let Some(user) = stream.try_next().await? {
        // 逐行处理
        process_user(user).await;
        count += 1;
        if count % 1000 == 0 {
            tracing::info!("已处理 {count} 个用户");
        }
    }

    Ok(())
}

// 配合 tokio::spawn 并发处理
pub async fn parallel_process(pool: &PgPool) -> anyhow::Result<()> {
    use futures::StreamExt;

    let stream = sqlx::query_as!(
        User,
        "SELECT id, name, email, avatar_url, is_active, created_at, updated_at FROM users"
    )
    .fetch(pool);

    // 每次最多并发处理 10 条
    stream
        .map_err(anyhow::Error::from)
        .try_for_each_concurrent(10, |user| async move {
            expensive_operation(user).await?;
            Ok(())
        })
        .await
}
```

---

## 类型映射

### PostgreSQL 类型对照

| PostgreSQL 类型 | Rust 类型 | 需要 feature |
|----------------|-----------|-------------|
| `BOOLEAN` | `bool` | — |
| `SMALLINT` | `i16` | — |
| `INTEGER` | `i32` | — |
| `BIGINT` | `i64` | — |
| `REAL` | `f32` | — |
| `DOUBLE PRECISION` | `f64` | — |
| `TEXT` / `VARCHAR` | `String` / `&str` | — |
| `BYTEA` | `Vec<u8>` / `&[u8]` | — |
| `UUID` | `uuid::Uuid` | `uuid` |
| `TIMESTAMPTZ` | `chrono::DateTime<Utc>` | `chrono` |
| `TIMESTAMP` | `chrono::NaiveDateTime` | `chrono` |
| `DATE` | `chrono::NaiveDate` | `chrono` |
| `TIME` | `chrono::NaiveTime` | `chrono` |
| `JSONB` / `JSON` | `serde_json::Value` | `json` |
| `DECIMAL` / `NUMERIC` | `sqlx::types::BigDecimal` | `bigdecimal` |
| `TEXT[]` | `Vec<String>` | — |
| `INTEGER[]` | `Vec<i32>` | — |
| `INET` | `std::net::IpAddr` | `ipnetwork` |

### 自定义枚举类型（PostgreSQL）

```sql
-- 数据库中定义枚举
CREATE TYPE user_role AS ENUM ('admin', 'moderator', 'user');
ALTER TABLE users ADD COLUMN role user_role NOT NULL DEFAULT 'user';
```

```rust
// 方式一：sqlx::Type derive（编译期检查）
#[derive(Debug, Clone, PartialEq, sqlx::Type, serde::Serialize, serde::Deserialize)]
#[sqlx(type_name = "user_role", rename_all = "lowercase")]
pub enum UserRole {
    Admin,
    Moderator,
    User,
}

// 使用
let row = sqlx::query!(
    "SELECT id, role AS \"role: UserRole\" FROM users WHERE id = $1",
    id
)
.fetch_one(pool)
.await?;
println!("{:?}", row.role);
```

```rust
// 方式二：用字符串存储，手动转换（兼容性好）
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub enum UserRole {
    Admin,
    Moderator,
    User,
}

impl std::fmt::Display for UserRole {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            UserRole::Admin     => write!(f, "admin"),
            UserRole::Moderator => write!(f, "moderator"),
            UserRole::User      => write!(f, "user"),
        }
    }
}

impl TryFrom<&str> for UserRole {
    type Error = anyhow::Error;
    fn try_from(s: &str) -> Result<Self, Self::Error> {
        match s {
            "admin"     => Ok(UserRole::Admin),
            "moderator" => Ok(UserRole::Moderator),
            "user"      => Ok(UserRole::User),
            _           => Err(anyhow::anyhow!("未知角色：{s}")),
        }
    }
}
```

### JSONB 字段

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(Debug, Serialize, Deserialize)]
struct UserSettings {
    theme:         String,
    notifications: bool,
    language:      String,
}

// 存储 JSONB
async fn update_settings(
    pool:     &PgPool,
    user_id:  Uuid,
    settings: &UserSettings,
) -> sqlx::Result<()> {
    let json = serde_json::to_value(settings).unwrap();

    sqlx::query!(
        "UPDATE users SET settings = $1 WHERE id = $2",
        json,
        user_id
    )
    .execute(pool)
    .await?;
    Ok(())
}

// 读取 JSONB
async fn get_settings(pool: &PgPool, user_id: Uuid) -> sqlx::Result<UserSettings> {
    let row = sqlx::query!(
        "SELECT settings FROM users WHERE id = $1",
        user_id
    )
    .fetch_one(pool)
    .await?;

    let settings: UserSettings = serde_json::from_value(row.settings.unwrap_or_default())
        .unwrap_or_default();
    Ok(settings)
}
```

### 数组类型

```rust
// 存储文本数组
async fn update_tags(
    pool:    &PgPool,
    post_id: Uuid,
    tags:    &[String],
) -> sqlx::Result<()> {
    sqlx::query!(
        "UPDATE posts SET tags = $1 WHERE id = $2",
        tags,
        post_id
    )
    .execute(pool)
    .await?;
    Ok(())
}

// 读取数组
let row = sqlx::query!(
    "SELECT tags FROM posts WHERE id = $1",
    post_id
)
.fetch_one(pool)
.await?;

let tags: Vec<String> = row.tags.unwrap_or_default();
```

---

## 高级查询

### JOIN 查询

```rust
#[derive(Debug, sqlx::FromRow)]
pub struct PostWithAuthor {
    pub post_id:    Uuid,
    pub title:      String,
    pub content:    String,
    pub author_id:  Uuid,
    pub author_name: String,
    pub created_at: DateTime<Utc>,
}

pub async fn list_posts_with_authors(pool: &PgPool) -> sqlx::Result<Vec<PostWithAuthor>> {
    sqlx::query_as!(
        PostWithAuthor,
        r#"
        SELECT
            p.id         AS post_id,
            p.title,
            p.content,
            u.id         AS author_id,
            u.name       AS author_name,
            p.created_at
        FROM posts p
        INNER JOIN users u ON u.id = p.user_id
        WHERE p.published = true
        ORDER BY p.created_at DESC
        LIMIT 20
        "#
    )
    .fetch_all(pool)
    .await
}
```

### 分页（游标分页）

```rust
// 偏移分页（小数据集）
pub async fn paginate_offset(
    pool:     &PgPool,
    page:     u32,
    per_page: u32,
) -> sqlx::Result<(Vec<User>, i64)> {
    let offset = ((page.saturating_sub(1)) * per_page) as i64;
    let limit  = per_page as i64;

    // 同时获取总数
    let total = sqlx::query_scalar!("SELECT COUNT(*) FROM users WHERE is_active = true")
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

    let users = sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users WHERE is_active = true
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
        "#,
        limit,
        offset,
    )
    .fetch_all(pool)
    .await?;

    Ok((users, total))
}

// 游标分页（大数据集，性能更好）
pub async fn paginate_cursor(
    pool:   &PgPool,
    cursor: Option<DateTime<Utc>>,  // 上一页最后一条的时间
    limit:  u32,
) -> sqlx::Result<Vec<User>> {
    match cursor {
        None => {
            sqlx::query_as!(
                User,
                r#"
                SELECT id, name, email, avatar_url, is_active, created_at, updated_at
                FROM users WHERE is_active = true
                ORDER BY created_at DESC LIMIT $1
                "#,
                limit as i64,
            )
            .fetch_all(pool)
            .await
        }
        Some(cursor_time) => {
            sqlx::query_as!(
                User,
                r#"
                SELECT id, name, email, avatar_url, is_active, created_at, updated_at
                FROM users
                WHERE is_active = true AND created_at < $1
                ORDER BY created_at DESC LIMIT $2
                "#,
                cursor_time,
                limit as i64,
            )
            .fetch_all(pool)
            .await
        }
    }
}
```

### 全文搜索（PostgreSQL）

```rust
pub async fn full_text_search(
    pool:  &PgPool,
    query: &str,
) -> sqlx::Result<Vec<User>> {
    sqlx::query_as!(
        User,
        r#"
        SELECT id, name, email, avatar_url, is_active, created_at, updated_at
        FROM users
        WHERE to_tsvector('simple', name || ' ' || email) @@ plainto_tsquery('simple', $1)
        ORDER BY ts_rank(to_tsvector('simple', name || ' ' || email), plainto_tsquery('simple', $1)) DESC
        LIMIT 20
        "#,
        query
    )
    .fetch_all(pool)
    .await
}
```

---

## Repository 模式

将数据库操作封装为 Repository，使 Handler 与 SQL 解耦，也方便测试。

```rust
// src/repositories/user_repository.rs
use async_trait::async_trait;
use sqlx::PgPool;
use std::sync::Arc;
use uuid::Uuid;

#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> sqlx::Result<Option<User>>;
    async fn find_by_email(&self, email: &str) -> sqlx::Result<Option<User>>;
    async fn list(&self, page: u32, per_page: u32) -> sqlx::Result<Vec<User>>;
    async fn create(&self, dto: CreateUserDto) -> sqlx::Result<User>;
    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> sqlx::Result<Option<User>>;
    async fn delete(&self, id: Uuid) -> sqlx::Result<bool>;
}

pub struct PgUserRepository {
    pool: Arc<PgPool>,
}

impl PgUserRepository {
    pub fn new(pool: Arc<PgPool>) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl UserRepository for PgUserRepository {
    async fn find_by_id(&self, id: Uuid) -> sqlx::Result<Option<User>> {
        sqlx::query_as!(
            User,
            r#"
            SELECT id, name, email, avatar_url, is_active, created_at, updated_at
            FROM users WHERE id = $1
            "#,
            id
        )
        .fetch_optional(self.pool.as_ref())
        .await
    }

    async fn find_by_email(&self, email: &str) -> sqlx::Result<Option<User>> {
        sqlx::query_as!(
            User,
            r#"
            SELECT id, name, email, avatar_url, is_active, created_at, updated_at
            FROM users WHERE email = $1
            "#,
            email
        )
        .fetch_optional(self.pool.as_ref())
        .await
    }

    async fn list(&self, page: u32, per_page: u32) -> sqlx::Result<Vec<User>> {
        let offset = ((page.saturating_sub(1)) * per_page) as i64;
        sqlx::query_as!(
            User,
            r#"
            SELECT id, name, email, avatar_url, is_active, created_at, updated_at
            FROM users WHERE is_active = true
            ORDER BY created_at DESC LIMIT $1 OFFSET $2
            "#,
            per_page as i64,
            offset,
        )
        .fetch_all(self.pool.as_ref())
        .await
    }

    async fn create(&self, dto: CreateUserDto) -> sqlx::Result<User> {
        sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (id, name, email, password, created_at, updated_at)
            VALUES ($1, $2, $3, $4, NOW(), NOW())
            RETURNING id, name, email, avatar_url, is_active, created_at, updated_at
            "#,
            Uuid::new_v4(),
            dto.name,
            dto.email,
            dto.password,
        )
        .fetch_one(self.pool.as_ref())
        .await
    }

    async fn update(&self, id: Uuid, dto: UpdateUserDto) -> sqlx::Result<Option<User>> {
        sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET name = COALESCE($2, name), email = COALESCE($3, email), updated_at = NOW()
            WHERE id = $1 AND is_active = true
            RETURNING id, name, email, avatar_url, is_active, created_at, updated_at
            "#,
            id, dto.name, dto.email,
        )
        .fetch_optional(self.pool.as_ref())
        .await
    }

    async fn delete(&self, id: Uuid) -> sqlx::Result<bool> {
        let r = sqlx::query!("DELETE FROM users WHERE id = $1", id)
            .execute(self.pool.as_ref())
            .await?;
        Ok(r.rows_affected() > 0)
    }
}
```

---

## 与 Axum 集成

```rust
// src/state.rs
use std::sync::Arc;
use crate::repositories::user_repository::UserRepository;

#[derive(Clone)]
pub struct AppState {
    pub user_repo: Arc<dyn UserRepository>,
}

// src/main.rs
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();

    let database_url = std::env::var("DATABASE_URL")?;
    let pool = PgPoolOptions::new()
        .max_connections(20)
        .connect(&database_url)
        .await?;

    // 运行迁移
    sqlx::migrate!("./migrations").run(&pool).await?;

    let pool = Arc::new(pool);
    let state = AppState {
        user_repo: Arc::new(PgUserRepository::new(pool.clone())),
    };

    let app = Router::new()
        .route("/users",     get(list_users_handler).post(create_user_handler))
        .route("/users/:id", get(get_user_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    Ok(())
}

// src/handlers/users.rs
use axum::{extract::{Path, Query, State}, http::StatusCode, response::Json};
use uuid::Uuid;

pub async fn list_users_handler(
    State(state): State<AppState>,
    Query(params): Query<Pagination>,
) -> Result<Json<Vec<User>>, AppError> {
    let users = state.user_repo
        .list(params.page.unwrap_or(1), params.per_page.unwrap_or(20))
        .await?;
    Ok(Json(users))
}

pub async fn get_user_handler(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<User>, AppError> {
    state.user_repo
        .find_by_id(id)
        .await?
        .map(Json)
        .ok_or_else(|| AppError::NotFound(format!("用户 {id} 不存在")))
}

pub async fn create_user_handler(
    State(state): State<AppState>,
    Json(dto): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<User>), AppError> {
    let user = state.user_repo.create(dto.into()).await?;
    Ok((StatusCode::CREATED, Json(user)))
}
```

---

## 测试

### 单元测试（SQLite 内存数据库）

```rust
#[cfg(test)]
mod tests {
    use sqlx::SqlitePool;

    // 创建测试用内存数据库
    async fn setup_test_db() -> SqlitePool {
        let pool = SqlitePool::connect("sqlite::memory:")
            .await
            .expect("无法创建测试数据库");

        // 运行迁移或手动建表
        sqlx::query!(
            r#"
            CREATE TABLE IF NOT EXISTS users (
                id         TEXT PRIMARY KEY,
                name       TEXT NOT NULL,
                email      TEXT NOT NULL UNIQUE,
                created_at TEXT NOT NULL
            )
            "#
        )
        .execute(&pool)
        .await
        .expect("建表失败");

        pool
    }

    #[tokio::test]
    async fn test_create_and_find_user() {
        let pool = setup_test_db().await;

        sqlx::query!(
            "INSERT INTO users (id, name, email, created_at) VALUES (?, ?, ?, ?)",
            "1",
            "Alice",
            "alice@example.com",
            "2024-01-01T00:00:00Z"
        )
        .execute(&pool)
        .await
        .unwrap();

        let user = sqlx::query!("SELECT * FROM users WHERE id = ?", "1")
            .fetch_one(&pool)
            .await
            .unwrap();

        assert_eq!(user.name, "Alice");
        assert_eq!(user.email, "alice@example.com");
    }
}
```

### 集成测试（PostgreSQL 测试数据库）

```rust
// tests/integration_test.rs
use sqlx::{PgPool, postgres::PgPoolOptions};

async fn setup_pg_pool() -> PgPool {
    let url = std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:postgres@localhost:5432/test_db".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&url)
        .await
        .expect("无法连接测试数据库");

    // 清理并重建测试表
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    pool
}

#[tokio::test]
async fn test_user_crud() {
    let pool = setup_pg_pool().await;

    // 每个测试在事务中运行，结束后回滚（隔离测试数据）
    let mut tx = pool.begin().await.unwrap();

    let user = sqlx::query_as!(
        User,
        r#"
        INSERT INTO users (id, name, email, password, created_at, updated_at)
        VALUES ($1, $2, $3, $4, NOW(), NOW())
        RETURNING id, name, email, avatar_url, is_active, created_at, updated_at
        "#,
        Uuid::new_v4(),
        "测试用户",
        "test@example.com",
        "password_hash",
    )
    .fetch_one(&mut *tx)
    .await
    .unwrap();

    assert_eq!(user.name, "测试用户");
    assert!(user.is_active);

    // 测试结束，自动回滚（不污染数据库）
    tx.rollback().await.unwrap();
}
```

---

## 编译期检查与离线模式

### 开发流程

```shell
# 1. 开发时：设置 DATABASE_URL，query! 宏连接数据库做检查
export DATABASE_URL=postgres://user:pass@localhost/mydb

# 2. 编译前生成查询缓存（.sqlx/ 目录）
cargo sqlx prepare

# 3. CI/CD 中：设置离线模式，不需要数据库
export SQLX_OFFLINE=true
cargo build
```

### .sqlx 目录

```
.sqlx/
├── query-abc123.json   # 每个 query! 对应一个缓存文件
├── query-def456.json
└── ...
```

将 `.sqlx/` 目录提交到版本控制，CI 中设置 `SQLX_OFFLINE=true` 即可无数据库编译。

---

## 常见问题

### 字段名冲突（关键字）

```rust
// PostgreSQL 关键字作为列名时需要用引号
let row = sqlx::query!(
    r#"SELECT "type", "order" FROM items WHERE id = $1"#,
    id
)
.fetch_one(pool)
.await?;
```

### Option 字段标注

```rust
// query! 宏对可空列返回 Option，非空列返回直接类型
// 但有时宏无法推断（如表达式结果），需要手动标注

let row = sqlx::query!(
    r#"SELECT COUNT(*) AS "count!" FROM users"#  // ! 表示断言非空
)
.fetch_one(pool)
.await?;
// row.count 类型是 i64，而非 Option<i64>

let row = sqlx::query!(
    r#"SELECT name AS "name?" FROM users WHERE id = $1"#,  // ? 表示可空
    id
)
.fetch_one(pool)
.await?;
// row.name 类型是 Option<String>
```

### 类型覆盖（Type Override）

```rust
// 使用 AS "column_name: Type" 语法强制指定 Rust 类型
let row = sqlx::query!(
    r#"SELECT role AS "role: UserRole" FROM users WHERE id = $1"#,
    id
)
.fetch_one(pool)
.await?;
// row.role 类型是 UserRole 枚举
```

### 连接数耗尽

```rust
// 症状：acquire_timeout 超时，请求堆积
// 原因：持锁时间过长、事务未提交、连接泄漏

// 检查连接池状态
println!("总连接数: {}", pool.size());
println!("空闲连接: {}", pool.num_idle());

// 设置合理的超时
PgPoolOptions::new()
    .max_connections(20)
    .acquire_timeout(Duration::from_secs(3))   // 获取连接超时
    .idle_timeout(Duration::from_secs(300))    // 空闲连接回收
```

---

## 最佳实践

1. **`query!` 宏优先**：能用 `query!` / `query_as!` 就不用运行时字符串拼接，编译期错误远胜运行时错误。

2. **维护 `.sqlx/` 目录**：提交到 Git，CI 中设置 `SQLX_OFFLINE=true`，解耦构建与数据库依赖。

3. **迁移文件只增不改**：已执行的迁移文件绝对不要修改，只能新增迁移文件来修正错误。

4. **Repository 层封装 SQL**：Handler 不直接写 SQL，通过 Repository trait 访问数据，便于测试时替换为 Mock。

5. **事务包裹多步操作**：任何涉及多张表的写操作都应放在事务中，避免部分成功。

6. **测试在事务中运行**：集成测试在事务里操作，测试结束回滚，零污染，无需清理脚本。

7. **用 `ANY($1)` 代替 `IN`**：PostgreSQL 中 `= ANY($1)` 接受数组参数，比动态拼接 IN 子句更安全高效。

8. **大数据集用 `fetch()` 流**：百万级数据避免 `fetch_all`（全部加载到内存），改用 `fetch()` 逐行处理。

9. **合理设置连接池参数**：`max_connections` 不是越大越好，受数据库服务器限制，通常 20-50 足够。
