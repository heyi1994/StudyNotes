# Tonic

## 1. 什么是 Tonic

Tonic 是基于 Tokio 的 Rust gRPC 框架，特性：

- 基于 HTTP/2 + Protocol Buffers
- 支持全部四种 RPC 模式（一元、服务端流、客户端流、双向流）
- 异步原生，与 Tokio 深度集成
- 支持 TLS、拦截器、反射、健康检查

```
客户端                        服务端
  │                              │
  │  HTTP/2 + Protobuf 二进制    │
  │ ─────────────────────────>  │
  │                              │
  │       响应（流式/一元）       │
  │ <─────────────────────────  │
```

### gRPC 四种 RPC 模式

| 模式             | 请求 | 响应 | 适用场景           |
| ---------------- | ---- | ---- | ------------------ |
| Unary            | 单条 | 单条 | 普通请求响应       |
| Server Streaming | 单条 | 流   | 订阅、大数据推送   |
| Client Streaming | 流   | 单条 | 文件上传、批量写入 |
| Bidirectional    | 流   | 流   | 实时通信、聊天     |

---

## 2. 环境准备

### 安装 protoc

```bash
# macOS
brew install protobuf

# Ubuntu
apt install -y protobuf-compiler

# 验证
protoc --version
```

### Cargo.toml

```toml
[dependencies]
tonic = "0.12"
prost = "0.13"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"

[build-dependencies]
tonic-build = "0.12"
```

### 目录结构

```
my-grpc-project/
├── Cargo.toml
├── build.rs           ← 代码生成配置
├── proto/
│   └── hello.proto    ← Proto 定义
└── src/
    ├── main.rs
    ├── server.rs
    └── client.rs
```

---

## 3. Proto 文件定义

### proto/hello.proto

```protobuf
syntax = "proto3";

package hello;

// 服务定义
service Greeter {
  // 一元 RPC
  rpc SayHello (HelloRequest) returns (HelloResponse);

  // 服务端流
  rpc SayHelloStream (HelloRequest) returns (stream HelloResponse);

  // 客户端流
  rpc SayHelloClientStream (stream HelloRequest) returns (HelloResponse);

  // 双向流
  rpc SayHelloBidi (stream HelloRequest) returns (stream HelloResponse);
}

// 消息定义
message HelloRequest {
  string name = 1;
  int32  count = 2;
}

message HelloResponse {
  string message = 1;
  int64  timestamp = 2;
}
```

### Protobuf 常用类型

| Proto 类型   | Rust 类型       | 说明               |
| ------------ | --------------- | ------------------ |
| `string`     | `String`        |                    |
| `bytes`      | `Vec<u8>`       |                    |
| `bool`       | `bool`          |                    |
| `int32`      | `i32`           |                    |
| `int64`      | `i64`           |                    |
| `uint32`     | `u32`           |                    |
| `float`      | `f32`           |                    |
| `double`     | `f64`           |                    |
| `repeated T` | `Vec<T>`        | 数组               |
| `map<K, V>`  | `HashMap<K, V>` | 字典               |
| `optional T` | `Option<T>`     | 可选字段（proto3） |

---

## 4. 代码生成

### build.rs

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/hello.proto")?;
    Ok(())
}
```

### 高级配置

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        // 为消息派生额外 trait
        .type_attribute("HelloRequest", "#[derive(Hash)]")
        // 生成服务端代码
        .build_server(true)
        // 生成客户端代码
        .build_client(true)
        // 指定输出目录（默认 OUT_DIR）
        .out_dir("src/generated")
        .compile_protos(
            &["proto/hello.proto"],
            &["proto/"],  // include 路径
        )?;
    Ok(())
}
```

### 引入生成代码

```rust
// src/main.rs 或模块中
pub mod hello {
    tonic::include_proto!("hello"); // 对应 package hello
}
```

---

## 5. 一元 RPC（Unary）

### 服务端

```rust
use tonic::{transport::Server, Request, Response, Status};

pub mod hello {
    tonic::include_proto!("hello");
}

use hello::greeter_server::{Greeter, GreeterServer};
use hello::{HelloRequest, HelloResponse};

#[derive(Debug, Default)]
pub struct MyGreeter;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<HelloResponse>, Status> {
        let req = request.into_inner();

        println!("收到请求，name={}", req.name);

        let reply = HelloResponse {
            message: format!("Hello, {}!", req.name),
            timestamp: chrono::Utc::now().timestamp(),
        };

        Ok(Response::new(reply))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "0.0.0.0:50051".parse()?;
    let greeter = MyGreeter::default();

    println!("服务启动，监听 {}", addr);

    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
```

### 客户端

```rust
use hello::greeter_client::GreeterClient;
use hello::HelloRequest;

pub mod hello {
    tonic::include_proto!("hello");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = GreeterClient::connect("http://127.0.0.1:50051").await?;

    let request = tonic::Request::new(HelloRequest {
        name: "World".to_string(),
        count: 1,
    });

    let response = client.say_hello(request).await?;
    println!("响应: {:?}", response.into_inner());

    Ok(())
}
```

### 读取请求元数据

```rust
async fn say_hello(
    &self,
    request: Request<HelloRequest>,
) -> Result<Response<HelloResponse>, Status> {
    // 读取元数据（HTTP header）
    if let Some(token) = request.metadata().get("authorization") {
        println!("token: {:?}", token);
    }

    // 获取客户端地址
    if let Some(addr) = request.remote_addr() {
        println!("客户端地址: {}", addr);
    }

    let inner = request.into_inner();
    // ...
}
```

---

## 6. 服务端流（Server Streaming）

### 服务端

```rust
use tokio_stream::wrappers::ReceiverStream;
use tonic::Status;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    // 返回类型是关联类型，需要在 trait 里声明
    type SayHelloStreamStream = ReceiverStream<Result<HelloResponse, Status>>;

    async fn say_hello_stream(
        &self,
        request: Request<HelloRequest>,
    ) -> Result<Response<Self::SayHelloStreamStream>, Status> {
        let name = request.into_inner().name;
        let (tx, rx) = tokio::sync::mpsc::channel(32);

        // 在独立任务中发送流数据
        tokio::spawn(async move {
            for i in 0..5 {
                let msg = HelloResponse {
                    message: format!("Hello {}, message {}", name, i),
                    timestamp: i,
                };
                if tx.send(Ok(msg)).await.is_err() {
                    // 客户端断开
                    break;
                }
                tokio::time::sleep(tokio::time::Duration::from_millis(500)).await;
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }
}
```

### 客户端

```rust
let mut stream = client
    .say_hello_stream(HelloRequest {
        name: "World".to_string(),
        count: 5,
    })
    .await?
    .into_inner();

// 逐条接收流数据
while let Some(response) = stream.message().await? {
    println!("收到: {}", response.message);
}
```

---

## 7. 客户端流（Client Streaming）

### 服务端

```rust
use tonic::Streaming;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello_client_stream(
        &self,
        request: Request<Streaming<HelloRequest>>,
    ) -> Result<Response<HelloResponse>, Status> {
        let mut stream = request.into_inner();
        let mut names = vec![];

        // 接收所有客户端消息
        while let Some(req) = stream.message().await? {
            names.push(req.name);
        }

        Ok(Response::new(HelloResponse {
            message: format!("Hello: {}", names.join(", ")),
            timestamp: 0,
        }))
    }
}
```

### 客户端

```rust
use tokio_stream::iter;

// 构造请求流
let requests = vec![
    HelloRequest { name: "Alice".to_string(), count: 1 },
    HelloRequest { name: "Bob".to_string(),   count: 2 },
    HelloRequest { name: "Carol".to_string(), count: 3 },
];

let response = client
    .say_hello_client_stream(iter(requests))
    .await?;

println!("响应: {}", response.into_inner().message);
```

---

## 8. 双向流（Bidirectional Streaming）

### 服务端

```rust
use tokio_stream::StreamExt;
use futures::Stream;
use std::pin::Pin;

type BidiStream = Pin<Box<dyn Stream<Item = Result<HelloResponse, Status>> + Send>>;

#[tonic::async_trait]
impl Greeter for MyGreeter {
    type SayHelloBidiStream = BidiStream;

    async fn say_hello_bidi(
        &self,
        request: Request<Streaming<HelloRequest>>,
    ) -> Result<Response<Self::SayHelloBidiStream>, Status> {
        let mut inbound = request.into_inner();

        let outbound = async_stream::try_stream! {
            while let Some(req) = inbound.message().await? {
                yield HelloResponse {
                    message: format!("Echo: {}", req.name),
                    timestamp: req.count as i64,
                };
            }
        };

        Ok(Response::new(Box::pin(outbound)))
    }
}
```

`async_stream` 需要添加依赖：

```toml
[dependencies]
async-stream = "0.3"
```

### 客户端

```rust
use tokio_stream::StreamExt;

let requests = tokio_stream::iter(vec![
    HelloRequest { name: "Alice".to_string(), count: 1 },
    HelloRequest { name: "Bob".to_string(),   count: 2 },
]);

let mut stream = client.say_hello_bidi(requests).await?.into_inner();

while let Some(resp) = stream.next().await {
    match resp {
        Ok(r)  => println!("收到: {}", r.message),
        Err(e) => println!("错误: {}", e),
    }
}
```

---

## 9. 拦截器（Interceptor）

拦截器用于实现认证、日志、追踪等横切关注点。

### 服务端拦截器

```rust
use tonic::{Request, Status};

fn auth_interceptor(req: Request<()>) -> Result<Request<()>, Status> {
    match req.metadata().get("authorization") {
        Some(t) if t == "Bearer secret-token" => Ok(req),
        _ => Err(Status::unauthenticated("无效的 token")),
    }
}

// 应用到服务
Server::builder()
    .add_service(
        GreeterServer::with_interceptor(MyGreeter::default(), auth_interceptor)
    )
    .serve(addr)
    .await?;
```

### 客户端拦截器

```rust
use tonic::service::interceptor;
use tonic::metadata::MetadataValue;

let channel = tonic::transport::Channel::from_static("http://127.0.0.1:50051")
    .connect()
    .await?;

// 自动添加 token
let client = GreeterClient::with_interceptor(channel, |mut req: Request<()>| {
    req.metadata_mut().insert(
        "authorization",
        MetadataValue::from_static("Bearer secret-token"),
    );
    Ok(req)
});
```

### 日志拦截器（服务端）

```rust
fn log_interceptor(req: Request<()>) -> Result<Request<()>, Status> {
    println!(
        "[{}] {} {:?}",
        chrono::Utc::now().format("%Y-%m-%d %H:%M:%S"),
        req.uri(),
        req.remote_addr(),
    );
    Ok(req)
}
```

---

## 10. 认证与 TLS

### 生成自签名证书

```bash
# 生成 CA 私钥和证书
openssl req -x509 -newkey rsa:4096 -keyout ca.key -out ca.crt \
  -days 365 -nodes -subj '/CN=MyCA'

# 生成服务端私钥和 CSR
openssl req -newkey rsa:4096 -keyout server.key -out server.csr \
  -nodes -subj '/CN=localhost'

# 用 CA 签署服务端证书
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365
```

### 服务端启用 TLS

```rust
use tonic::transport::{Server, Identity, ServerTlsConfig};

let cert = tokio::fs::read("server.crt").await?;
let key  = tokio::fs::read("server.key").await?;
let identity = Identity::from_pem(cert, key);

Server::builder()
    .tls_config(ServerTlsConfig::new().identity(identity))?
    .add_service(GreeterServer::new(MyGreeter::default()))
    .serve(addr)
    .await?;
```

### 客户端启用 TLS

```rust
use tonic::transport::{Channel, ClientTlsConfig, Certificate};

let ca_cert = tokio::fs::read("ca.crt").await?;
let ca = Certificate::from_pem(ca_cert);

let tls = ClientTlsConfig::new()
    .ca_certificate(ca)
    .domain_name("localhost");

let channel = Channel::from_static("https://127.0.0.1:50051")
    .tls_config(tls)?
    .connect()
    .await?;

let mut client = GreeterClient::new(channel);
```

---

## 11. 错误处理

### Status 错误码

```rust
use tonic::{Status, Code};

// 常用错误码
Status::ok("成功")
Status::invalid_argument("参数错误")
Status::not_found("资源不存在")
Status::already_exists("已存在")
Status::permission_denied("权限不足")
Status::unauthenticated("未认证")
Status::internal("内部错误")
Status::unavailable("服务不可用")
Status::deadline_exceeded("超时")
Status::resource_exhausted("资源耗尽")
```

### 服务端返回结构化错误

```rust
use tonic::{Status, Code};

async fn say_hello(
    &self,
    request: Request<HelloRequest>,
) -> Result<Response<HelloResponse>, Status> {
    let name = request.into_inner().name;

    if name.is_empty() {
        return Err(Status::invalid_argument("name 不能为空"));
    }

    if name.len() > 50 {
        return Err(Status::invalid_argument(
            format!("name 长度不能超过 50，当前 {}", name.len())
        ));
    }

    Ok(Response::new(HelloResponse {
        message: format!("Hello, {}!", name),
        timestamp: 0,
    }))
}
```

### 客户端处理错误

```rust
match client.say_hello(request).await {
    Ok(response) => println!("成功: {:?}", response.into_inner()),
    Err(status) => {
        match status.code() {
            Code::InvalidArgument => println!("参数错误: {}", status.message()),
            Code::NotFound        => println!("未找到: {}", status.message()),
            Code::Unauthenticated => println!("未认证，请登录"),
            Code::Unavailable     => println!("服务不可用，稍后重试"),
            _                     => println!("未知错误: {}", status),
        }
    }
}
```

### 超时控制

```rust
use std::time::Duration;

// 客户端设置超时
let request = tonic::Request::new(HelloRequest {
    name: "World".to_string(),
    count: 1,
});
// 单次请求超时
request.set_timeout(Duration::from_secs(5));

// 或在 channel 层设置超时
let channel = Channel::from_static("http://127.0.0.1:50051")
    .timeout(Duration::from_secs(10))
    .connect()
    .await?;
```

---

## 12. 反射与健康检查

### 服务反射（支持 grpcurl 等工具）

```toml
[dependencies]
tonic-reflection = "0.12"
```

```rust
use tonic_reflection::server::Builder as ReflectionBuilder;

// 需要在 build.rs 中额外生成描述符
tonic_build::configure()
    .file_descriptor_set_path("proto_descriptor.bin")
    .compile_protos(&["proto/hello.proto"], &["proto/"])?;

// 在 main.rs 中注册反射服务
const DESCRIPTOR: &[u8] = include_bytes!("../proto_descriptor.bin");

let reflection_service = ReflectionBuilder::configure()
    .register_encoded_file_descriptor_set(DESCRIPTOR)
    .build_v1()?;

Server::builder()
    .add_service(reflection_service)
    .add_service(GreeterServer::new(MyGreeter::default()))
    .serve(addr)
    .await?;
```

```bash
# 使用 grpcurl 测试
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext -d '{"name": "World"}' localhost:50051 hello.Greeter/SayHello
```

### 健康检查

```toml
[dependencies]
tonic-health = "0.12"
```

```rust
use tonic_health::server::health_reporter;

let (mut reporter, health_service) = health_reporter();

// 设置服务状态
reporter
    .set_serving::<GreeterServer<MyGreeter>>()
    .await;

Server::builder()
    .add_service(health_service)
    .add_service(GreeterServer::new(MyGreeter::default()))
    .serve(addr)
    .await?;

// 动态修改状态（如检测到依赖不可用时）
reporter.set_not_serving::<GreeterServer<MyGreeter>>().await;
```

---

## 13. 实战：完整用户服务

### proto/user.proto

```protobuf
syntax = "proto3";
package user;

service UserService {
  rpc CreateUser (CreateUserRequest)  returns (User);
  rpc GetUser    (GetUserRequest)     returns (User);
  rpc ListUsers  (ListUsersRequest)   returns (stream User);
  rpc DeleteUser (DeleteUserRequest)  returns (DeleteUserResponse);
}

message User {
  uint64 id         = 1;
  string username   = 2;
  string email      = 3;
  int64  created_at = 4;
}

message CreateUserRequest {
  string username = 1;
  string email    = 2;
}

message GetUserRequest {
  uint64 id = 1;
}

message ListUsersRequest {
  uint32 page_size = 1;
}

message DeleteUserRequest {
  uint64 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}
```

### src/server.rs

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use tokio_stream::wrappers::ReceiverStream;
use tonic::{Request, Response, Status};

pub mod user {
    tonic::include_proto!("user");
}

use user::user_service_server::UserService;
use user::*;

type UserStore = Arc<RwLock<HashMap<u64, User>>>;

pub struct MyUserService {
    store:   UserStore,
    next_id: Arc<tokio::sync::Mutex<u64>>,
}

impl MyUserService {
    pub fn new() -> Self {
        Self {
            store:   Arc::new(RwLock::new(HashMap::new())),
            next_id: Arc::new(tokio::sync::Mutex::new(1)),
        }
    }
}

#[tonic::async_trait]
impl UserService for MyUserService {
    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<User>, Status> {
        let req = request.into_inner();

        if req.username.is_empty() {
            return Err(Status::invalid_argument("username 不能为空"));
        }

        let mut id_lock = self.next_id.lock().await;
        let id = *id_lock;
        *id_lock += 1;
        drop(id_lock);

        let user = User {
            id,
            username: req.username,
            email: req.email,
            created_at: chrono::Utc::now().timestamp(),
        };

        self.store.write().await.insert(id, user.clone());
        Ok(Response::new(user))
    }

    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<User>, Status> {
        let id = request.into_inner().id;
        let store = self.store.read().await;

        store
            .get(&id)
            .cloned()
            .map(Response::new)
            .ok_or_else(|| Status::not_found(format!("用户 {} 不存在", id)))
    }

    type ListUsersStream = ReceiverStream<Result<User, Status>>;

    async fn list_users(
        &self,
        request: Request<ListUsersRequest>,
    ) -> Result<Response<Self::ListUsersStream>, Status> {
        let page_size = request.into_inner().page_size as usize;
        let (tx, rx) = tokio::sync::mpsc::channel(32);
        let store = Arc::clone(&self.store);

        tokio::spawn(async move {
            let users: Vec<User> = store
                .read()
                .await
                .values()
                .take(if page_size > 0 { page_size } else { usize::MAX })
                .cloned()
                .collect();

            for user in users {
                if tx.send(Ok(user)).await.is_err() {
                    break;
                }
            }
        });

        Ok(Response::new(ReceiverStream::new(rx)))
    }

    async fn delete_user(
        &self,
        request: Request<DeleteUserRequest>,
    ) -> Result<Response<DeleteUserResponse>, Status> {
        let id = request.into_inner().id;
        let removed = self.store.write().await.remove(&id).is_some();

        if !removed {
            return Err(Status::not_found(format!("用户 {} 不存在", id)));
        }

        Ok(Response::new(DeleteUserResponse { success: true }))
    }
}
```

### src/main.rs

```rust
mod server;

use server::{user::user_service_server::UserServiceServer, MyUserService};
use tonic::transport::Server;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "0.0.0.0:50051".parse()?;

    println!("UserService 启动，监听 {}", addr);

    Server::builder()
        .add_service(UserServiceServer::new(MyUserService::new()))
        .serve(addr)
        .await?;

    Ok(())
}
```

---

## 14. 常见陷阱

### 陷阱 1：关联类型流必须是 Send

```rust
// 错误：生成的流没有 Send bound
type MyStream = impl Stream<Item = Result<Response, Status>>;

// 正确：显式 Pin<Box<dyn Stream + Send>>
type MyStream = Pin<Box<dyn Stream<Item = Result<Response, Status>> + Send>>;
```

### 陷阱 2：build.rs 未配置导致宏找不到 proto

```rust
// build.rs 必须存在且正确调用
fn main() {
    tonic_build::compile_protos("proto/hello.proto")
        .expect("编译 proto 失败，请确认 protoc 已安装");
}
```

### 陷阱 3：忘记在 Cargo.toml 声明 build-dependencies

```toml
# 注意区分 dependencies 和 build-dependencies
[build-dependencies]
tonic-build = "0.12"  # 只在构建时使用
```

### 陷阱 4：多 proto 文件时包名冲突

```rust
// 不同 proto 包用不同模块名区分
pub mod user {
    tonic::include_proto!("user");
}
pub mod order {
    tonic::include_proto!("order");
}
```

### 陷阱 5：流中 sender drop 时序

```rust
// 确保 tx 在数据发送完毕后才 drop
// 不要在 spawn 外持有 tx，否则流永远不会结束
let (tx, rx) = mpsc::channel(32);

tokio::spawn(async move {
    tx.send(Ok(data)).await.unwrap();
    // tx 在这里自动 drop，触发流结束
});
//  不要在这里再持有 tx 的引用
```

---

## 快速参考

| 需求          | 方式                                                         |
| ------------- | ------------------------------------------------------------ |
| 一元 RPC      | `async fn(&self, Request<T>) -> Result<Response<R>, Status>` |
| 服务端流      | 返回 `ReceiverStream` / `Pin<Box<dyn Stream>>`               |
| 客户端流      | 入参 `Request<Streaming<T>>`                                 |
| 双向流        | 入参流 + 返回流                                              |
| 添加 metadata | `request.metadata_mut().insert(...)`                         |
| 拦截器认证    | `GreeterServer::with_interceptor(...)`                       |
| 错误返回      | `Err(Status::not_found(...))`                                |
| 客户端超时    | `request.set_timeout(Duration::from_secs(5))`                |
| 启用 TLS      | `ServerTlsConfig` / `ClientTlsConfig`                        |
| 服务发现/测试 | `tonic-reflection` + `grpcurl`                               |

