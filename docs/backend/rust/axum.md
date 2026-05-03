# Axum

Axum 是由 Tokio 团队开发的高性能 Rust Web 框架，构建在 `tokio`、`hyper` 和 `tower` 之上。它以零成本抽象、类型安全的提取器（Extractor）体系和与 `tower` 中间件生态的无缝集成著称。

**核心特点：**

| 特点 | 说明 |
|------|------|
| **类型安全提取器** | 路径参数、查询参数、请求体等通过类型系统在编译期验证 |
| **tower 兼容** | 直接复用 `tower` 和 `tower-http` 的所有中间件 |
| **无宏路由** | 纯函数式路由声明，无需过程宏 |
| **hyper 底层** | 基于 `hyper`，性能接近原生 |
| **Tokio 原生** | 完全异步，无运行时开销 |

---

## 安装与项目初始化

```toml
# Cargo.toml
[dependencies]
axum        = "0.7"
tokio       = { version = "1", features = ["full"] }
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
tower       = "0.4"
tower-http  = { version = "0.5", features = ["cors", "trace", "compression-gzip", "fs"] }
tracing     = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow      = "1"
thiserror   = "1"

# 数据库（可选）
sqlx        = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid        = { version = "1", features = ["v4", "serde"] }
chrono      = { version = "0.4", features = ["serde"] }
```

### Hello World

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("监听 http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

---

## 路由

### 基础路由

```rust
use axum::{
    routing::{delete, get, patch, post, put},
    Router,
};

let app = Router::new()
    .route("/",               get(root_handler))
    .route("/users",          get(list_users).post(create_user))
    .route("/users/:id",      get(get_user).put(update_user).delete(delete_user))
    .route("/users/:id/posts", get(list_user_posts))
    .route("/health",         get(|| async { "OK" }));
```

### 嵌套路由

```rust
// 模块化路由，便于拆分大型应用
fn user_routes() -> Router {
    Router::new()
        .route("/",    get(list_users).post(create_user))
        .route("/:id", get(get_user).put(update_user).delete(delete_user))
}

fn post_routes() -> Router {
    Router::new()
        .route("/",    get(list_posts).post(create_post))
        .route("/:id", get(get_post))
}

let app = Router::new()
    .nest("/api/v1/users", user_routes())
    .nest("/api/v1/posts", post_routes());
```

### 路由合并

```rust
// merge：合并两棵路由树（适合模块化拆分）
let api_routes = Router::new()
    .merge(user_routes())
    .merge(post_routes());

let app = Router::new()
    .nest("/api/v1", api_routes)
    .route("/health", get(health_check));
```

### 通配符路由

```rust
use axum::extract::Path;

// 捕获剩余路径段
let app = Router::new()
    .route("/files/*path", get(serve_file));

async fn serve_file(Path(path): Path<String>) -> String {
    format!("请求文件：{path}")
}
```

---

## 提取器（Extractor）

提取器是 Axum 最核心的特性——通过函数参数类型自动从请求中提取数据，编译期保证类型安全。

### Path 参数

```rust
use axum::extract::Path;

// 单个参数
async fn get_user(Path(id): Path<u64>) -> String {
    format!("用户 ID：{id}")
}

// 多个参数（元组）
async fn get_comment(Path((post_id, comment_id)): Path<(u64, u64)>) -> String {
    format!("文章 {post_id}，评论 {comment_id}")
}

// 结构体（字段名需与路由参数名匹配）
#[derive(Deserialize)]
struct PostParams {
    post_id:    u64,
    comment_id: u64,
}
async fn get_comment_v2(Path(params): Path<PostParams>) -> String {
    format!("文章 {}，评论 {}", params.post_id, params.comment_id)
}

// 路由：/posts/:post_id/comments/:comment_id
```

### Query 参数

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page:     Option<u32>,
    per_page: Option<u32>,
    sort:     Option<String>,
}

async fn list_users(Query(params): Query<Pagination>) -> String {
    let page     = params.page.unwrap_or(1);
    let per_page = params.per_page.unwrap_or(20);
    format!("第 {page} 页，每页 {per_page} 条")
}
// GET /users?page=2&per_page=10&sort=name
```

### JSON 请求体

```rust
use axum::{extract::Json, http::StatusCode, response::Json as RespJson};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUserRequest {
    name:  String,
    email: String,
    age:   Option<u32>,
}

#[derive(Serialize)]
struct CreateUserResponse {
    id:    u64,
    name:  String,
    email: String,
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> (StatusCode, RespJson<CreateUserResponse>) {
    let user = CreateUserResponse {
        id:    1,
        name:  payload.name,
        email: payload.email,
    };
    (StatusCode::CREATED, RespJson(user))
}
```

### 表单数据

```rust
use axum::extract::Form;
use serde::Deserialize;

#[derive(Deserialize)]
struct LoginForm {
    username: String,
    password: String,
}

async fn login(Form(form): Form<LoginForm>) -> String {
    format!("用户 {} 登录", form.username)
}
```

### Headers

```rust
use axum::{extract::TypedHeader, headers::Authorization, headers::authorization::Bearer};

async fn protected(
    TypedHeader(auth): TypedHeader<Authorization<Bearer>>,
) -> String {
    format!("Token: {}", auth.token())
}

// 手动读取 Header
use axum::http::HeaderMap;

async fn read_headers(headers: HeaderMap) -> String {
    let ua = headers
        .get("user-agent")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown");
    format!("User-Agent: {ua}")
}
```

### State（共享状态）

```rust
use axum::extract::State;
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
struct AppState {
    db:      Arc<sqlx::PgPool>,
    config:  Arc<Config>,
    counter: Arc<RwLock<u64>>,
}

// 挂载到路由
let state = AppState { /* ... */ };
let app = Router::new()
    .route("/count", get(get_count).post(inc_count))
    .with_state(state);

// 处理函数中使用
async fn get_count(State(state): State<AppState>) -> String {
    let count = state.counter.read().await;
    format!("当前计数：{count}")
}

async fn inc_count(State(state): State<AppState>) -> String {
    let mut count = state.counter.write().await;
    *count += 1;
    format!("计数更新为：{count}")
}
```

### 原始请求体

```rust
use axum::body::Bytes;
use axum::http::Request;
use axum::body::Body;

// 读取原始字节
async fn raw_body(body: Bytes) -> String {
    format!("收到 {} 字节", body.len())
}

// 读取完整请求
async fn full_request(request: Request<Body>) -> String {
    let (parts, body) = request.into_parts();
    let bytes = axum::body::to_bytes(body, usize::MAX).await.unwrap();
    format!("方法：{}，路径：{}", parts.method, parts.uri)
}
```

### 多个提取器组合

```rust
// 处理函数参数顺序有要求：
// 1. 最多一个消费 Body 的提取器（Json/Form/Bytes），且必须放最后
// 2. State 可以放任意位置
// 3. Path/Query/Headers 可以随意顺序

async fn complex_handler(
    State(state):   State<AppState>,
    Path(id):       Path<u64>,
    Query(params):  Query<Pagination>,
    Json(payload):  Json<CreateUserRequest>,  // Body 提取器放最后
) -> impl IntoResponse {
    // ...
}
```

---

## 响应（Response）

### 基本响应类型

```rust
use axum::{
    http::StatusCode,
    response::{Html, IntoResponse, Json, Redirect, Response},
};
use serde_json::json;

// 字符串
async fn text() -> &'static str { "Hello" }

// HTML
async fn html_page() -> Html<&'static str> {
    Html("<h1>Hello, World!</h1>")
}

// JSON
async fn json_resp() -> Json<serde_json::Value> {
    Json(json!({ "status": "ok", "code": 200 }))
}

// 状态码 + 响应体
async fn created() -> (StatusCode, String) {
    (StatusCode::CREATED, "资源已创建".to_string())
}

// 带 Header 的响应
async fn with_header() -> impl IntoResponse {
    (
        StatusCode::OK,
        [("x-custom-header", "value"), ("content-type", "text/plain")],
        "响应内容",
    )
}

// 重定向
async fn redirect() -> Redirect {
    Redirect::to("/new-location")
}

// 永久重定向
async fn permanent_redirect() -> Redirect {
    Redirect::permanent("/new-location")
}
```

### 自定义响应类型

```rust
use axum::{
    http::{header, StatusCode},
    response::{IntoResponse, Response},
};

struct ApiResponse<T> {
    code:    u16,
    message: String,
    data:    Option<T>,
}

impl<T: Serialize> IntoResponse for ApiResponse<T> {
    fn into_response(self) -> Response {
        let body = json!({
            "code":    self.code,
            "message": self.message,
            "data":    self.data,
        });
        (StatusCode::OK, Json(body)).into_response()
    }
}

// 使用
async fn get_user_v2() -> ApiResponse<User> {
    ApiResponse {
        code:    200,
        message: "success".to_string(),
        data:    Some(User { id: 1, name: "Alice".to_string() }),
    }
}
```

### 流式响应

```rust
use axum::response::sse::{Event, Sse};
use futures::stream::{self, Stream};
use std::{convert::Infallible, time::Duration};
use tokio_stream::StreamExt as _;

// Server-Sent Events（SSE）
async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| {
        Event::default().data("心跳")
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(
        axum::response::sse::KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("keep-alive"),
    )
}
```

---

## 错误处理

### 统一错误类型

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Json, Response},
};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("未找到：{0}")]
    NotFound(String),

    #[error("参数无效：{0}")]
    BadRequest(String),

    #[error("未授权")]
    Unauthorized,

    #[error("禁止访问")]
    Forbidden,

    #[error("数据库错误：{0}")]
    Database(#[from] sqlx::Error),

    #[error("内部错误：{0}")]
    Internal(#[from] anyhow::Error),
}

// 实现 IntoResponse，使 AppError 能直接作为 Handler 返回值
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg)    => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::BadRequest(msg)  => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::Unauthorized     => (StatusCode::UNAUTHORIZED, self.to_string()),
            AppError::Forbidden        => (StatusCode::FORBIDDEN, self.to_string()),
            AppError::Database(e)      => {
                tracing::error!("数据库错误：{e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "数据库错误".to_string())
            }
            AppError::Internal(e)      => {
                tracing::error!("内部错误：{e}");
                (StatusCode::INTERNAL_SERVER_ERROR, "服务器内部错误".to_string())
            }
        };

        let body = json!({ "error": message });
        (status, Json(body)).into_response()
    }
}

// Result 别名，简化书写
pub type AppResult<T> = Result<T, AppError>;

// 用法
async fn get_user(Path(id): Path<u64>) -> AppResult<Json<User>> {
    let user = find_user(id).await?;   // sqlx::Error 自动转换为 AppError::Database
    user.ok_or_else(|| AppError::NotFound(format!("用户 {id} 不存在")))
        .map(Json)
}
```

### 提取器错误处理

```rust
use axum::{
    extract::rejection::JsonRejection,
    response::{IntoResponse, Response},
};

// 捕获提取器（如 Json）失败时的错误
async fn create_user(
    payload: Result<Json<CreateUserRequest>, JsonRejection>,
) -> Response {
    match payload {
        Ok(Json(data)) => {
            // 正常处理
            StatusCode::CREATED.into_response()
        }
        Err(err) => {
            let body = json!({ "error": err.to_string() });
            (StatusCode::UNPROCESSABLE_ENTITY, Json(body)).into_response()
        }
    }
}
```

---

## 中间件

Axum 的中间件基于 `tower::Layer`，可以复用整个 `tower-http` 生态。

### 内置中间件（tower-http）

```rust
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
    limit::RequestBodyLimitLayer,
    catch_panic::CatchPanicLayer,
};
use std::time::Duration;

let app = Router::new()
    .route("/", get(handler))
    // 请求追踪日志
    .layer(TraceLayer::new_for_http())
    // CORS
    .layer(
        CorsLayer::new()
            .allow_origin(Any)
            .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
            .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]),
    )
    // 响应压缩
    .layer(CompressionLayer::new())
    // 请求超时
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    // 请求体大小限制（10MB）
    .layer(RequestBodyLimitLayer::new(10 * 1024 * 1024))
    // Panic 捕获（避免线程崩溃）
    .layer(CatchPanicLayer::new());
```

### 自定义中间件（函数式）

```rust
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::Response,
};

// 请求日志中间件
async fn logging_middleware(request: Request, next: Next) -> Response {
    let method = request.method().clone();
    let uri    = request.uri().clone();
    let start  = std::time::Instant::now();

    let response = next.run(request).await;

    let elapsed = start.elapsed();
    tracing::info!("{method} {uri} -> {} ({elapsed:?})", response.status());

    response
}

// 认证中间件
async fn auth_middleware(
    State(state): State<AppState>,
    mut request:  Request,
    next:         Next,
) -> Result<Response, AppError> {
    let token = request
        .headers()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(AppError::Unauthorized)?;

    let user = verify_token(token, &state).await?;

    // 将用户信息注入请求扩展，供后续 Handler 使用
    request.extensions_mut().insert(user);
    Ok(next.run(request).await)
}

// 挂载中间件
let app = Router::new()
    .route("/public",  get(public_handler))
    .route("/private", get(private_handler))
    .route_layer(middleware::from_fn(logging_middleware))
    // 只对需要认证的路由加认证中间件
    .route_layer(middleware::from_fn_with_state(state.clone(), auth_middleware));
```

### 从扩展中读取中间件注入的数据

```rust
use axum::Extension;

// 中间件注入
request.extensions_mut().insert(CurrentUser { id: 1, name: "Alice".to_string() });

// Handler 读取
async fn private_handler(
    Extension(user): Extension<CurrentUser>,
) -> String {
    format!("你好，{}", user.name)
}
```

### 有状态中间件（tower Layer）

```rust
use std::task::{Context, Poll};
use tower::{Layer, Service};
use axum::body::Body;
use axum::http::{Request, Response};
use futures::future::BoxFuture;

// 限流中间件示例（简化版）
#[derive(Clone)]
struct RateLimitLayer {
    requests_per_second: u32,
}

impl<S> Layer<S> for RateLimitLayer {
    type Service = RateLimitService<S>;
    fn layer(&self, inner: S) -> Self::Service {
        RateLimitService {
            inner,
            rps: self.requests_per_second,
        }
    }
}

#[derive(Clone)]
struct RateLimitService<S> {
    inner: S,
    rps:   u32,
}

impl<S> Service<Request<Body>> for RateLimitService<S>
where
    S: Service<Request<Body>, Response = Response<Body>> + Send + 'static,
    S::Future: Send,
{
    type Response = S::Response;
    type Error    = S::Error;
    type Future   = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request<Body>) -> Self::Future {
        let future = self.inner.call(req);
        Box::pin(async move { future.await })
    }
}
```

---

## 状态管理

### 多状态组合

```rust
use axum::extract::State;
use std::sync::Arc;

// 拆分状态，各模块只依赖自己需要的部分
#[derive(Clone)]
struct DatabaseState(Arc<sqlx::PgPool>);

#[derive(Clone)]
struct CacheState(Arc<redis::Client>);

#[derive(Clone)]
struct AppState {
    db:    DatabaseState,
    cache: CacheState,
}

// 也可以用 FromRef 从大状态提取子状态
use axum::extract::FromRef;

impl FromRef<AppState> for DatabaseState {
    fn from_ref(state: &AppState) -> Self {
        state.db.clone()
    }
}

impl FromRef<AppState> for CacheState {
    fn from_ref(state: &AppState) -> Self {
        state.cache.clone()
    }
}

// Handler 只需声明它用到的子状态
async fn db_only_handler(State(db): State<DatabaseState>) { /* ... */ }
async fn cache_only_handler(State(cache): State<CacheState>) { /* ... */ }
```

### 使用 tokio::sync 替代 std::sync

```rust
use tokio::sync::{Mutex, RwLock};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    // 读多写少 -> RwLock
    config: Arc<RwLock<Config>>,
    // 需要独占访问 -> Mutex
    sessions: Arc<Mutex<HashMap<String, Session>>>,
}

async fn read_config(State(state): State<AppState>) -> String {
    let config = state.config.read().await;
    config.some_value.clone()
}

async fn update_session(State(state): State<AppState>) {
    let mut sessions = state.sessions.lock().await;
    sessions.insert("key".to_string(), Session::new());
}
```

---

## 数据库集成（sqlx + PostgreSQL）

### 初始化连接池

```rust
use sqlx::postgres::PgPoolOptions;

async fn create_db_pool(database_url: &str) -> sqlx::PgPool {
    PgPoolOptions::new()
        .max_connections(20)
        .min_connections(2)
        .acquire_timeout(std::time::Duration::from_secs(5))
        .connect(database_url)
        .await
        .expect("数据库连接失败")
}

#[tokio::main]
async fn main() {
    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL 未设置");
    let pool = create_db_pool(&database_url).await;

    let state = AppState { db: Arc::new(pool) };
    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### CRUD 操作

```rust
use sqlx::{FromRow, PgPool};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, FromRow, Serialize, Deserialize, Clone)]
struct User {
    id:         Uuid,
    name:       String,
    email:      String,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

#[derive(Deserialize)]
struct CreateUserDto {
    name:  String,
    email: String,
}

#[derive(Deserialize)]
struct UpdateUserDto {
    name:  Option<String>,
    email: Option<String>,
}

// 查询列表
async fn list_users(
    State(pool): State<Arc<PgPool>>,
    Query(params): Query<Pagination>,
) -> AppResult<Json<Vec<User>>> {
    let offset = (params.page.unwrap_or(1) - 1) * params.per_page.unwrap_or(20);
    let limit  = params.per_page.unwrap_or(20);

    let users = sqlx::query_as!(
        User,
        "SELECT * FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2",
        limit as i64,
        offset as i64
    )
    .fetch_all(pool.as_ref())
    .await?;

    Ok(Json(users))
}

// 查询单个
async fn get_user(
    State(pool): State<Arc<PgPool>>,
    Path(id): Path<Uuid>,
) -> AppResult<Json<User>> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(pool.as_ref())
        .await?
        .ok_or_else(|| AppError::NotFound(format!("用户 {id} 不存在")))?;

    Ok(Json(user))
}

// 创建
async fn create_user(
    State(pool): State<Arc<PgPool>>,
    Json(dto): Json<CreateUserDto>,
) -> AppResult<(StatusCode, Json<User>)> {
    let user = sqlx::query_as!(
        User,
        r#"
        INSERT INTO users (id, name, email, created_at, updated_at)
        VALUES ($1, $2, $3, NOW(), NOW())
        RETURNING *
        "#,
        Uuid::new_v4(),
        dto.name,
        dto.email,
    )
    .fetch_one(pool.as_ref())
    .await?;

    Ok((StatusCode::CREATED, Json(user)))
}

// 更新
async fn update_user(
    State(pool): State<Arc<PgPool>>,
    Path(id): Path<Uuid>,
    Json(dto): Json<UpdateUserDto>,
) -> AppResult<Json<User>> {
    let user = sqlx::query_as!(
        User,
        r#"
        UPDATE users
        SET
            name       = COALESCE($2, name),
            email      = COALESCE($3, email),
            updated_at = NOW()
        WHERE id = $1
        RETURNING *
        "#,
        id,
        dto.name,
        dto.email,
    )
    .fetch_optional(pool.as_ref())
    .await?
    .ok_or_else(|| AppError::NotFound(format!("用户 {id} 不存在")))?;

    Ok(Json(user))
}

// 删除
async fn delete_user(
    State(pool): State<Arc<PgPool>>,
    Path(id): Path<Uuid>,
) -> AppResult<StatusCode> {
    let result = sqlx::query!("DELETE FROM users WHERE id = $1", id)
        .execute(pool.as_ref())
        .await?;

    if result.rows_affected() == 0 {
        Err(AppError::NotFound(format!("用户 {id} 不存在")))
    } else {
        Ok(StatusCode::NO_CONTENT)
    }
}

// 事务
async fn transfer(
    State(pool): State<Arc<PgPool>>,
    Json(req): Json<TransferRequest>,
) -> AppResult<StatusCode> {
    let mut tx = pool.begin().await?;

    sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        req.amount, req.from_id
    )
    .execute(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        req.amount, req.to_id
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(StatusCode::OK)
}
```

---

## JWT 认证

```toml
[dependencies]
jsonwebtoken = "9"
```

```rust
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,   // Subject（用户 ID）
    exp: usize,    // 过期时间（Unix 时间戳）
    iat: usize,    // 签发时间
}

const JWT_SECRET: &[u8] = b"your-secret-key";  // 实际项目从环境变量读取

fn generate_token(user_id: &str) -> Result<String, jsonwebtoken::errors::Error> {
    let now  = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs() as usize;
    let claims = Claims {
        sub: user_id.to_string(),
        iat: now,
        exp: now + 3600 * 24,  // 24 小时后过期
    };
    encode(&Header::default(), &claims, &EncodingKey::from_secret(JWT_SECRET))
}

fn verify_token(token: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
    let data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(JWT_SECRET),
        &Validation::default(),
    )?;
    Ok(data.claims)
}

// 登录接口
#[derive(Deserialize)]
struct LoginRequest { username: String, password: String }

#[derive(Serialize)]
struct LoginResponse { token: String }

async fn login(
    State(pool): State<Arc<PgPool>>,
    Json(req): Json<LoginRequest>,
) -> AppResult<Json<LoginResponse>> {
    // 查询用户并校验密码（实际需要 bcrypt 等哈希）
    let user = sqlx::query!("SELECT id FROM users WHERE name = $1", req.username)
        .fetch_optional(pool.as_ref())
        .await?
        .ok_or(AppError::Unauthorized)?;

    let token = generate_token(&user.id.to_string())
        .map_err(|e| AppError::Internal(e.into()))?;

    Ok(Json(LoginResponse { token }))
}

// JWT 认证中间件
async fn jwt_auth(
    mut request: Request,
    next: Next,
) -> Result<Response, AppError> {
    let token = request
        .headers()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(AppError::Unauthorized)?;

    let claims = verify_token(token).map_err(|_| AppError::Unauthorized)?;
    request.extensions_mut().insert(claims);

    Ok(next.run(request).await)
}
```

---

## 请求验证

```toml
[dependencies]
validator = { version = "0.18", features = ["derive"] }
```

```rust
use validator::{Validate, ValidationError};

#[derive(Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(length(min = 2, max = 50))]
    name: String,

    #[validate(email)]
    email: String,

    #[validate(range(min = 18, max = 120))]
    age: u32,

    #[validate(url)]
    website: Option<String>,

    #[validate(custom = "validate_password")]
    password: String,
}

fn validate_password(password: &str) -> Result<(), ValidationError> {
    if password.len() < 8 {
        return Err(ValidationError::new("密码至少 8 位"));
    }
    if !password.chars().any(|c| c.is_ascii_digit()) {
        return Err(ValidationError::new("密码须包含数字"));
    }
    Ok(())
}

// 自定义 extractor，自动校验
use axum::{
    async_trait,
    extract::{FromRequest, Request},
    http::StatusCode,
};

struct ValidJson<T>(T);

#[async_trait]
impl<S, T> FromRequest<S> for ValidJson<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|err| {
                let body = json!({ "error": err.to_string() });
                (StatusCode::BAD_REQUEST, Json(body))
            })?;

        value.validate().map_err(|err| {
            let body = json!({ "errors": err.to_string() });
            (StatusCode::UNPROCESSABLE_ENTITY, Json(body))
        })?;

        Ok(ValidJson(value))
    }
}

// 使用自定义提取器
async fn create_user(ValidJson(req): ValidJson<CreateUserRequest>) -> StatusCode {
    // req 已通过验证
    StatusCode::CREATED
}
```

---

## 文件上传

```toml
[dependencies]
axum-multipart = "0.6"
# 或使用 multer
multer = "3"
```

```rust
use axum::{
    extract::Multipart,
    http::StatusCode,
};
use std::path::Path;
use tokio::fs;
use tokio::io::AsyncWriteExt;

async fn upload_file(mut multipart: Multipart) -> Result<String, AppError> {
    while let Some(mut field) = multipart.next_field().await.map_err(|e| {
        AppError::BadRequest(format!("解析表单失败：{e}"))
    })? {
        let name     = field.name().unwrap_or("unknown").to_string();
        let filename = field.file_name()
            .ok_or_else(|| AppError::BadRequest("缺少文件名".to_string()))?
            .to_string();

        // 校验文件类型
        let content_type = field.content_type().unwrap_or("application/octet-stream");
        if !["image/jpeg", "image/png", "image/webp"].contains(&content_type) {
            return Err(AppError::BadRequest("仅支持 JPEG/PNG/WebP".to_string()));
        }

        // 保存文件
        let upload_dir = Path::new("uploads");
        fs::create_dir_all(upload_dir).await?;
        let file_path = upload_dir.join(&filename);
        let mut file  = fs::File::create(&file_path).await?;

        let mut total_size = 0usize;
        while let Some(chunk) = field.chunk().await.map_err(|e| {
            AppError::Internal(e.into())
        })? {
            total_size += chunk.len();
            if total_size > 5 * 1024 * 1024 {  // 5MB 限制
                return Err(AppError::BadRequest("文件超过 5MB".to_string()));
            }
            file.write_all(&chunk).await?;
        }
        file.flush().await?;

        tracing::info!("上传成功：{filename}（{total_size} 字节）");
    }

    Ok("上传成功".to_string())
}
```

---

## WebSocket

```rust
use axum::{
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade},
        State,
    },
    response::Response,
};
use futures::{sink::SinkExt, stream::StreamExt};
use std::sync::Arc;
use tokio::sync::broadcast;

// 广播频道：用于聊天室
#[derive(Clone)]
struct WsState {
    tx: broadcast::Sender<String>,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<WsState>>,
) -> Response {
    ws.on_upgrade(|socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: Arc<WsState>) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = state.tx.subscribe();

    // 发送任务：将广播消息发给该客户端
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break;
            }
        }
    });

    // 接收任务：将客户端消息广播出去
    let tx = state.tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            if let Message::Text(text) = msg {
                let _ = tx.send(text);
            }
        }
    });

    // 任意一方断开，取消另一个任务
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }
}

// 路由注册
let (tx, _) = broadcast::channel(100);
let ws_state = Arc::new(WsState { tx });

let app = Router::new()
    .route("/ws", get(ws_handler))
    .with_state(ws_state);
```

---

## 日志与追踪（tracing）

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

fn init_tracing() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| {
            // 默认级别：info，axum 请求日志开启
            "my_app=debug,axum::rejection=trace,tower_http=debug".into()
        }))
        .with(tracing_subscriber::fmt::layer().with_target(false))
        .init();
}

#[tokio::main]
async fn main() {
    init_tracing();

    let app = Router::new()
        .route("/", get(handler))
        .layer(TraceLayer::new_for_http());

    // ...
}

// 在 Handler 中使用
async fn handler() -> &'static str {
    tracing::info!("处理请求");
    tracing::debug!(user_id = 42, action = "read", "用户操作");
    "OK"
}
```

### 请求 ID 追踪

```rust
use tower_http::request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer};
use axum::http::HeaderName;

let x_request_id = HeaderName::from_static("x-request-id");

let app = Router::new()
    .route("/", get(handler))
    .layer(PropagateRequestIdLayer::new(x_request_id.clone()))
    .layer(
        TraceLayer::new_for_http().make_span_with(|request: &Request<_>| {
            let request_id = request
                .headers()
                .get("x-request-id")
                .and_then(|v| v.to_str().ok())
                .unwrap_or("unknown");
            tracing::info_span!("http_request", request_id)
        }),
    )
    .layer(SetRequestIdLayer::new(x_request_id, MakeRequestUuid));
```

---

## 配置管理

```toml
[dependencies]
config = "0.14"
dotenvy = "0.15"
```

```rust
use config::{Config, ConfigError, Environment, File};
use serde::Deserialize;

#[derive(Debug, Deserialize, Clone)]
pub struct AppConfig {
    pub server:   ServerConfig,
    pub database: DatabaseConfig,
    pub jwt:      JwtConfig,
}

#[derive(Debug, Deserialize, Clone)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

#[derive(Debug, Deserialize, Clone)]
pub struct DatabaseConfig {
    pub url:             String,
    pub max_connections: u32,
}

#[derive(Debug, Deserialize, Clone)]
pub struct JwtConfig {
    pub secret:     String,
    pub expires_in: u64,  // 秒
}

impl AppConfig {
    pub fn from_env() -> Result<Self, ConfigError> {
        let config = Config::builder()
            .add_source(File::with_name("config/default"))       // 默认配置
            .add_source(File::with_name("config/local").required(false))  // 本地覆盖
            .add_source(Environment::with_prefix("APP").separator("__")) // 环境变量
            .build()?;

        config.try_deserialize()
    }
}

// config/default.toml
// [server]
// host = "0.0.0.0"
// port = 3000
//
// [database]
// max_connections = 20
```

---

## 完整项目结构

```
my-api/
├── Cargo.toml
├── .env
├── config/
│   ├── default.toml
│   └── local.toml           # gitignore 中排除
├── migrations/              # sqlx 数据库迁移文件
│   └── 001_create_users.sql
└── src/
    ├── main.rs              # 启动入口
    ├── config.rs            # 配置加载
    ├── error.rs             # 统一错误类型
    ├── state.rs             # AppState 定义
    ├── middleware/
    │   ├── mod.rs
    │   ├── auth.rs          # JWT 认证中间件
    │   └── logging.rs       # 日志中间件
    ├── routes/
    │   ├── mod.rs           # 路由汇总
    │   ├── users.rs         # 用户相关路由
    │   └── posts.rs         # 文章相关路由
    ├── handlers/            # 也可与 routes/ 合并
    │   ├── mod.rs
    │   ├── users.rs
    │   └── posts.rs
    ├── models/
    │   ├── mod.rs
    │   ├── user.rs          # User 结构体 + sqlx FromRow
    │   └── post.rs
    └── db/
        ├── mod.rs
        └── users.rs         # 数据库操作封装
```

### main.rs

```rust
mod config;
mod db;
mod error;
mod handlers;
mod middleware;
mod models;
mod routes;
mod state;

use config::AppConfig;
use state::AppState;
use std::sync::Arc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 加载 .env 文件
    dotenvy::dotenv().ok();

    // 初始化日志
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "my_api=debug,tower_http=debug".into()))
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 加载配置
    let config = AppConfig::from_env()?;

    // 初始化数据库
    let pool = db::create_pool(&config.database).await?;

    // 构建状态
    let state = AppState {
        pool: Arc::new(pool),
        config: Arc::new(config.clone()),
    };

    // 构建路由
    let app = routes::create_router(state);

    // 启动服务
    let addr = format!("{}:{}", config.server.host, config.server.port);
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    tracing::info!("服务启动：http://{addr}");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    Ok(())
}

// 优雅关闭：监听 Ctrl+C 和 SIGTERM
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c().await.expect("Ctrl+C 监听失败");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("SIGTERM 监听失败")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c   => tracing::info!("收到 Ctrl+C，开始优雅关闭"),
        _ = terminate => tracing::info!("收到 SIGTERM，开始优雅关闭"),
    }
}
```

---

## 测试

### 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user() {
        let dto = CreateUserDto {
            name:  "Alice".to_string(),
            email: "alice@example.com".to_string(),
        };
        // 直接调用业务函数测试
        let result = validate_user(&dto);
        assert!(result.is_ok());
    }
}
```

### 集成测试（axum-test / tower::ServiceExt）

```toml
[dev-dependencies]
tower         = { version = "0.4", features = ["util"] }
hyper         = { version = "1", features = ["full"] }
http-body-util = "0.1"
```

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::http::{Request, StatusCode};
    use tower::ServiceExt;  // oneshot
    use http_body_util::BodyExt;

    fn test_app() -> Router {
        // 创建测试专用路由（可使用内存 DB）
        let state = AppState {
            pool: Arc::new(create_test_pool().await),
            config: Arc::new(test_config()),
        };
        create_router(state)
    }

    #[tokio::test]
    async fn test_get_users() {
        let app = test_app();

        let response = app
            .oneshot(
                Request::builder()
                    .uri("/api/v1/users")
                    .body(axum::body::Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);

        let body = response.into_body().collect().await.unwrap().to_bytes();
        let users: Vec<User> = serde_json::from_slice(&body).unwrap();
        assert!(users.is_empty());
    }

    #[tokio::test]
    async fn test_create_user() {
        let app = test_app();

        let payload = serde_json::json!({
            "name":  "Alice",
            "email": "alice@example.com"
        });

        let response = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/api/v1/users")
                    .header("content-type", "application/json")
                    .body(axum::body::Body::from(payload.to_string()))
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::CREATED);
    }

    #[tokio::test]
    async fn test_get_user_not_found() {
        let app = test_app();
        let fake_id = Uuid::new_v4();

        let response = app
            .oneshot(
                Request::builder()
                    .uri(format!("/api/v1/users/{fake_id}"))
                    .body(axum::body::Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
    }
}
```

---

## 常见问题

### Handler 返回类型约束

Handler 函数必须满足：
- 所有参数实现 `FromRequest` 或 `FromRequestParts`
- 返回值实现 `IntoResponse`
- 函数本身是 `async fn` 或返回 `Future`
- 所有类型满足 `Send + 'static`

```rust
// 错误：Rc 不是 Send
async fn bad_handler() -> String {
    let rc = std::rc::Rc::new(1);  // 编译错误
    format!("{}", rc)
}

// 正确：用 Arc
async fn good_handler() -> String {
    let arc = std::sync::Arc::new(1);
    format!("{}", arc)
}
```

### 避免在 Handler 中阻塞

```rust
use tokio::task;

// 错误：在异步 Handler 中调用阻塞操作
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(1));  // 阻塞整个线程！
}

// 正确：使用 spawn_blocking 把阻塞操作移到专用线程池
async fn good() {
    task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(1));
    })
    .await
    .unwrap();
}
```

### 提取器顺序

```rust
// Body 只能被消费一次，消费 Body 的提取器（Json/Form/Bytes）必须放最后
async fn handler(
    Path(id):      Path<u64>,      //  不消费 Body
    Query(q):      Query<Params>,  //  不消费 Body
    State(state):  State<AppState>,//  不消费 Body
    Json(payload): Json<CreateDto>,//  消费 Body，放最后
) { }
```

### 共享可变状态的选择

| 场景 | 推荐方案 |
|------|----------|
| 只读配置 | `Arc<Config>` |
| 读多写少 | `Arc<RwLock<T>>` |
| 频繁写 | `Arc<Mutex<T>>` |
| 高并发计数 | `Arc<AtomicU64>` |
| 数据库连接 | `sqlx::PgPool`（内部已是连接池） |

