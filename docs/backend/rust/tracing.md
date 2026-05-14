# Tracing

## 1. 什么是 tracing

`tracing` 是 Tokio 生态的结构化日志与诊断框架，解决传统日志的三大问题：

| 问题           | 传统 log         | tracing                |
| -------------- | ---------------- | ---------------------- |
| 异步上下文丢失 | 无法追踪请求链路 | Span 自动传播上下文    |
| 日志难以查询   | 纯文本字符串     | 结构化键值字段         |
| 性能监控       | 只能打印时间戳   | Span 自带开始/结束时间 |

```
tracing（门面层）
    │
    ▼
Subscriber（后端实现）
    ├── tracing-subscriber（控制台/文件）
    ├── tracing-opentelemetry（分布式追踪）
    └── 自定义 Subscriber
```

---

## 2. 环境准备

### Cargo.toml

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

### 常用扩展库

```toml
[dependencies]
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-appender   = "0.2"   # 异步写文件、日志滚动
tracing-opentelemetry = "0.27" # OpenTelemetry 集成
opentelemetry       = "0.27"
```

---

## 3. 核心概念

### 三个核心抽象

**Span（跨度）**：表示一段有时间范围的操作，可以嵌套

```
请求处理 [Span]
  └── 数据库查询 [Span]
        └── SQL 执行 [Span]
```

**Event（事件）**：某一时刻发生的事情，类似传统日志的一条记录

**Subscriber（订阅者）**：接收 Span 和 Event，决定如何处理（打印、发送到远端等）

### 日志级别

```
ERROR > WARN > INFO > DEBUG > TRACE
```

| 级别    | 用途                     |
| ------- | ------------------------ |
| `ERROR` | 必须处理的错误           |
| `WARN`  | 潜在问题，程序仍可运行   |
| `INFO`  | 关键业务流程节点         |
| `DEBUG` | 开发调试信息             |
| `TRACE` | 极细粒度，生产一般不开启 |

---

## 4. 基础用法：事件（Event）

### 最小示例

```rust
use tracing::{error, warn, info, debug, trace};

fn main() {
    // 初始化默认订阅者（输出到控制台）
    tracing_subscriber::fmt::init();

    error!("发生严重错误");
    warn!("磁盘空间不足 {}%", 10);
    info!("服务启动，端口 {}", 8080);
    debug!("收到请求: {:?}", "GET /");
    trace!("进入函数 handle_request");
}
```

输出示例：

```
2026-05-14T10:00:00.000Z  INFO tracing_demo: 服务启动，端口 8080
2026-05-14T10:00:00.001Z DEBUG tracing_demo: 收到请求: "GET /"
```

### 带结构化字段的事件

```rust
// 键值对字段（推荐）
info!(port = 8080, host = "localhost", "服务启动");

// 调试格式
info!(request = ?req, "收到请求");

// Display 格式
info!(user = %username, "用户登录");

// 表达式字段
let duration_ms = elapsed.as_millis();
info!(duration_ms, status = 200, "请求完成");
```

`?` 使用 `Debug` 格式，`%` 使用 `Display` 格式，不加前缀则字段必须实现 `Value` trait（基础类型均已实现）。

---

## 5. Span：结构化上下文

### 创建和进入 Span

```rust
use tracing::{span, Level, info};

fn handle_request(request_id: &str) {
    // 创建 span
    let span = span!(Level::INFO, "handle_request", request_id);

    // 进入 span（之后的 event 都会附带这个 span 的上下文）
    let _guard = span.enter();

    info!("开始处理");  // 自动关联 handle_request span
    do_work();
    info!("处理完成");

    // _guard drop 时自动退出 span
}
```

### instrument 宏（最常用）

```rust
use tracing::instrument;

#[instrument]
fn process(user_id: u64, action: &str) {
    // 函数参数自动成为 span 字段
    info!("执行操作");
}

// 等价于：
// span!(Level::INFO, "process", user_id, action)
```

### instrument 高级配置

```rust
use tracing::instrument;

#[instrument(
    level = "debug",          // span 级别
    name = "my_process",      // 自定义 span 名称
    skip(password),           // 跳过敏感字段
    skip_all,                 // 跳过所有参数
    fields(                   // 额外添加字段
        request_id = %uuid::Uuid::new_v4(),
        user.id = user_id,
    ),
    err,                      // 自动记录 Err 到 span
    ret,                      // 自动记录返回值
)]
fn login(username: &str, password: &str, user_id: u64) -> Result<(), String> {
    info!("用户尝试登录");
    Ok(())
}
```

### 手动操作 Span 字段

```rust
use tracing::{info_span, Instrument};

let span = info_span!("request", request_id = tracing::field::Empty);

// 稍后填充字段
span.record("request_id", &"req-123");
```

### Span 嵌套

```rust
#[instrument]
async fn handle_request(id: u64) {
    info!("开始处理请求");

    // 子 span 自动继承父 span 上下文
    fetch_from_db(id).await;
}

#[instrument]
async fn fetch_from_db(id: u64) {
    debug!("查询数据库");  // 输出中会看到父子关系
}
```

---

## 6. Subscriber：日志后端

Subscriber 是 tracing 的核心扩展点，决定如何处理 Span 和 Event。

```rust
use tracing::subscriber;

// 设置全局 Subscriber（程序启动时调用一次）
subscriber::set_global_default(my_subscriber)
    .expect("设置 subscriber 失败");
```

### 作用域 Subscriber

```rust
use tracing::subscriber::with_default;

with_default(my_subscriber, || {
    // 只在此闭包内使用 my_subscriber
    info!("只被 my_subscriber 处理");
});
```

---

## 7. tracing-subscriber 详解

### fmt Subscriber（控制台输出）

```rust
use tracing_subscriber::fmt;

// 最简初始化
fmt::init();

// 完整配置
fmt::Subscriber::builder()
    .with_max_level(tracing::Level::DEBUG) // 最大日志级别
    .with_target(true)                     // 显示模块路径
    .with_thread_ids(true)                 // 显示线程 ID
    .with_thread_names(true)               // 显示线程名
    .with_file(true)                       // 显示文件名
    .with_line_number(true)                // 显示行号
    .with_level(true)                      // 显示级别
    .with_ansi(true)                       // 彩色输出
    .compact()                             // 紧凑格式
    .init();
```

### EnvFilter：动态过滤

```rust
use tracing_subscriber::{fmt, EnvFilter};

fmt::Subscriber::builder()
    .with_env_filter(
        EnvFilter::from_default_env()  // 读取 RUST_LOG 环境变量
            .add_directive("my_app=debug".parse().unwrap())
            .add_directive("my_app::db=trace".parse().unwrap())
    )
    .init();
```

```bash
# 环境变量控制日志级别
RUST_LOG=debug cargo run
RUST_LOG=my_app=info,my_app::db=debug cargo run
RUST_LOG=warn,tokio=error cargo run
```

### EnvFilter 语法

```
# 全局级别
RUST_LOG=debug

# 指定 crate
RUST_LOG=my_app=info

# 指定模块
RUST_LOG=my_app::handlers=debug

# 指定 span
RUST_LOG=[request]=info

# 组合
RUST_LOG=warn,my_app=debug,my_app::db=trace
```

### Registry + Layer 组合（推荐生产用法）

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(tracing_subscriber::fmt::layer())
    .init();
```

---

## 8. 字段与结构化日志

### 字段格式速查

```rust
let user_id: u64 = 42;
let user = User { name: "Alice".into() };
let err = std::io::Error::last_os_error();

info!(
    // 基本类型（直接值）
    user_id,
    count = 10,
    latency_ms = 42u64,

    // Display 格式（%）
    addr = %"127.0.0.1:8080",

    // Debug 格式（?）
    user = ?user,
    error = ?err,

    // 布尔
    success = true,

    // 消息（必须是最后一个位置参数）
    "请求处理完成"
);
```

### 在 Span 上动态记录字段

```rust
use tracing::{info_span, field};

let span = info_span!(
    "http_request",
    status = field::Empty,   // 占位符，稍后填充
    latency_ms = field::Empty,
);

let _guard = span.enter();

// ... 处理请求 ...

span.record("status", 200);
span.record("latency_ms", elapsed.as_millis() as u64);
```

---

## 9. 异步代码中的 tracing

### 问题：异步任务间 Span 上下文丢失

```rust
// 错误：enter() 返回的 guard 跨 await 会导致 panic 或上下文丢失
async fn bad_example() {
    let span = info_span!("my_span");
    let _guard = span.enter();
    some_async_fn().await; //  guard 不能跨越 await
}
```

### 正确方式：.instrument()

```rust
use tracing::Instrument;

async fn good_example() {
    let span = info_span!("my_span");

    // 方式一：Future::instrument
    some_async_fn().instrument(span).await;

    // 方式二：#[instrument] 宏（自动处理）
}

#[instrument]  // 推荐：自动处理异步 span 传播
async fn handle_connection(conn_id: u64) {
    info!("处理连接");
    fetch_data().await;  // span 自动传播
    info!("连接处理完成");
}
```

### tokio::spawn 中传播 Span

```rust
use tracing::Instrument;

async fn spawn_with_span() {
    let span = info_span!("parent_task");

    tokio::spawn(
        async move {
            info!("子任务运行中");  // 继承 parent_task span
        }
        .instrument(span),
    );
}
```

---

## 10. 与 log 生态集成

许多库使用 `log` crate，通过桥接层统一输出到 tracing。

```toml
[dependencies]
tracing-log = "0.2"
```

```rust
// 初始化时启用 log 桥接
tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(tracing_subscriber::fmt::layer())
    .init();

// tracing-subscriber 默认已内置 log 桥接
// 或手动初始化：
tracing_log::LogTracer::init().unwrap();
```

之后所有 `log::info!`、`log::error!` 等调用都会路由到 tracing。

---

## 11. 输出到文件

### 滚动日志文件

```toml
[dependencies]
tracing-appender = "0.2"
```

```rust
use tracing_appender::{non_blocking, rolling};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

fn main() {
    // 按天滚动，写入 /var/log/myapp/app.log.2026-05-14
    let file_appender = rolling::daily("/var/log/myapp", "app.log");

    // non_blocking 避免 I/O 阻塞主线程（必须保持 _guard 存活）
    let (non_blocking_writer, _guard) = non_blocking(file_appender);

    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        // 文件层（不带颜色）
        .with(
            tracing_subscriber::fmt::layer()
                .with_writer(non_blocking_writer)
                .with_ansi(false),
        )
        // 控制台层（带颜色）
        .with(tracing_subscriber::fmt::layer())
        .init();

    // _guard 在 main 退出前不能 drop，否则日志会丢失
    tracing::info!("程序启动");

    // ...
}
```

### 滚动策略

```rust
// 按小时滚动
rolling::hourly("logs/", "app.log");

// 按天滚动
rolling::daily("logs/", "app.log");

// 按分钟滚动
rolling::minutely("logs/", "app.log");

// 不滚动（单文件）
rolling::never("logs/", "app.log");
```

---

## 12. JSON 格式输出

JSON 格式便于日志收集系统（ELK、Loki 等）解析。

```toml
[dependencies]
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

tracing_subscriber::registry()
    .with(EnvFilter::from_default_env())
    .with(
        tracing_subscriber::fmt::layer()
            .json()                         // 启用 JSON 格式
            .with_current_span(true)        // 包含当前 span 信息
            .with_span_list(true)           // 包含 span 链路
    )
    .init();
```

输出示例：

```json
{
  "timestamp": "2026-05-14T10:00:00.000Z",
  "level": "INFO",
  "target": "my_app::handlers",
  "span": { "name": "handle_request", "request_id": "req-123" },
  "fields": { "message": "请求完成", "status": 200, "latency_ms": 42 }
}
```

---

## 13. OpenTelemetry 集成

### Cargo.toml

```toml
[dependencies]
tracing-opentelemetry    = "0.27"
opentelemetry            = "0.27"
opentelemetry_sdk        = { version = "0.27", features = ["rt-tokio"] }
opentelemetry-otlp       = { version = "0.27", features = ["tonic"] }
```

### 初始化（发送到 OTLP 收集器）

```rust
use opentelemetry::trace::TracerProvider;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::runtime;
use tracing_opentelemetry::OpenTelemetryLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

async fn init_tracing() -> opentelemetry_sdk::trace::TracerProvider {
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint("http://localhost:4317")
        .build()
        .unwrap();

    let provider = opentelemetry_sdk::trace::TracerProvider::builder()
        .with_batch_exporter(exporter, runtime::Tokio)
        .build();

    let tracer = provider.tracer("my_app");

    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer())
        .with(OpenTelemetryLayer::new(tracer))
        .init();

    provider
}

#[tokio::main]
async fn main() {
    let provider = init_tracing().await;

    run_app().await;

    // 程序退出前刷新缓冲区
    provider.shutdown().unwrap();
}
```

---

## 14. 实战：Tokio + Tonic 全链路追踪

### 完整初始化

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};
use tracing_appender::{non_blocking, rolling};

pub fn init_tracing() -> impl Drop {
    let file_appender = rolling::daily("logs", "app.log");
    let (file_writer, guard) = non_blocking(file_appender);

    tracing_subscriber::registry()
        .with(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info")),
        )
        // 控制台：人类可读格式
        .with(
            tracing_subscriber::fmt::layer()
                .with_target(true)
                .with_level(true),
        )
        // 文件：JSON 格式供日志系统收集
        .with(
            tracing_subscriber::fmt::layer()
                .json()
                .with_writer(file_writer)
                .with_ansi(false),
        )
        .init();

    guard  // 必须保持存活
}
```

### gRPC 服务中使用

```rust
use tonic::{Request, Response, Status};
use tracing::{info, instrument, warn};

pub struct MyService;

#[tonic::async_trait]
impl MyProto for MyService {
    #[instrument(
        skip(self, request),
        fields(
            rpc = "CreateUser",
            request_id = tracing::field::Empty,
        ),
        err
    )]
    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<User>, Status> {
        // 从 metadata 提取 request_id 并记录到 span
        if let Some(id) = request.metadata().get("x-request-id") {
            tracing::Span::current().record(
                "request_id",
                id.to_str().unwrap_or("invalid"),
            );
        }

        let req = request.into_inner();
        info!(username = %req.username, "创建用户");

        if req.username.is_empty() {
            warn!("用户名为空，拒绝请求");
            return Err(Status::invalid_argument("username 不能为空"));
        }

        // 数据库操作自动继承 span 上下文
        let user = save_to_db(&req).await?;

        info!(user_id = user.id, "用户创建成功");
        Ok(Response::new(user))
    }
}

#[instrument(fields(sql = "INSERT INTO users"))]
async fn save_to_db(req: &CreateUserRequest) -> Result<User, Status> {
    debug!("执行数据库写入");
    // ...
}
```

### HTTP 中间件（axum）

```rust
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::Level;

let app = axum::Router::new()
    .route("/", axum::routing::get(handler))
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(
                DefaultMakeSpan::new()
                    .level(Level::INFO)
                    .include_headers(true),
            )
            .on_response(
                DefaultOnResponse::new()
                    .level(Level::INFO)
                    .latency_unit(tower_http::LatencyUnit::Millis),
            ),
    );
```

---

## 15. 常见陷阱

### 陷阱 1：异步代码中 enter() 跨 await

```rust
//  错误：MutexGuard/Guard 跨 await 会 panic（debug 模式）
async fn bad() {
    let span = info_span!("task");
    let _guard = span.enter();
    tokio::time::sleep(Duration::from_secs(1)).await; // panic!
}

// ✓ 正确：用 .instrument() 或 #[instrument]
async fn good() {
    async {
        info!("在 span 内");
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
    .instrument(info_span!("task"))
    .await;
}
```

### 陷阱 2：non_blocking 的 guard 提前 drop

```rust
fn main() {
    let (writer, _guard) = non_blocking(file_appender);
    //             ^^^^^^ 匿名变量！立即 drop，日志全部丢失

    // ✓ 正确：绑定命名变量
    let (writer, _file_guard) = non_blocking(file_appender);

    // ... 程序运行 ...
    // _file_guard 在 main 结束时才 drop，确保日志刷盘
}
```

### 隐患 3：忘记调用 init()

```rust
//  只配置不初始化，什么都不会输出
tracing_subscriber::fmt::Subscriber::builder()
    .with_max_level(Level::DEBUG)
    .build(); // 没有 .init()！

// ✓ 正确
tracing_subscriber::fmt::init();
// 或
tracing_subscriber::fmt::Subscriber::builder()
    .with_max_level(Level::DEBUG)
    .init(); // 必须调用
```

### 陷阱 4：#[instrument] 在 impl 块中需要 Self: Debug

```rust
// 如果 Self 没有实现 Debug，默认会包含 self 字段
#[instrument]
async fn handler(&self, req: Request) {}

// ✓ 跳过 self
#[instrument(skip(self))]
async fn handler(&self, req: Request) {}
```

### 陷阱 5：全局 Subscriber 只能设置一次

```rust
// set_global_default 第二次调用会返回错误
// 测试中每个测试都调用 init() 会 panic

// 测试中使用局部 Subscriber
#[test]
fn test_something() {
    let subscriber = tracing_subscriber::fmt().finish();
    let _guard = tracing::subscriber::set_default(subscriber);
    // 测试代码
}
```

---

## 快速参考

| 需求                    | API                                           |
| ----------------------- | --------------------------------------------- |
| 记录一条日志            | `info!("msg")` / `error!("msg")`              |
| 带字段的日志            | `info!(key = val, "msg")`                     |
| 给函数加 span           | `#[instrument]`                               |
| 给异步 Future 加 span   | `future.instrument(span)`                     |
| 动态填充 span 字段      | `span.record("key", value)`                   |
| 控制台输出初始化        | `tracing_subscriber::fmt::init()`             |
| 环境变量控制级别        | `RUST_LOG=debug`                              |
| 写入滚动日志文件        | `tracing_appender::rolling::daily(...)`       |
| JSON 格式输出           | `.json()` on `fmt::layer()`                   |
| 多层输出（控制台+文件） | `registry().with(layer1).with(layer2).init()` |
| 桥接旧 log crate        | `tracing_subscriber` 默认已内置               |

---
