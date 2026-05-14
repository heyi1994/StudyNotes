# Protocol Buffers


## 1. 什么是 Protocol Buffers

Protocol Buffers（简称 protobuf）是 Google 开发的**语言无关、平台无关**的结构化数据序列化格式。

### 与其他格式对比

| 维度       | JSON                | XML     | Protobuf            |
| ---------- | ------------------- | ------- | ------------------- |
| 格式       | 文本                | 文本    | 二进制              |
| 可读性     | 好                  | 好      | 差（需工具）        |
| 序列化速度 | 慢                  | 很慢    | 快（3-10x）         |
| 数据体积   | 大                  | 很大    | 小（3-10x）         |
| 类型安全   | 弱                  | 弱      | 强                  |
| Schema     | 可选（JSON Schema） | DTD/XSD | 必须（.proto 文件） |
| 语言支持   | 所有                | 所有    | 主流语言            |
| 跨语言兼容 | 好                  | 好      | 极好                |

### 核心工作流

```
1. 编写 .proto 文件（定义数据结构）
        │
        ▼
2. protoc 编译 .proto 文件
        │
        ▼
3. 生成目标语言代码（.rs / .go / .py / .java ...）
        │
        ▼
4. 在程序中序列化 → 二进制字节
        │
        ▼
5. 在程序中反序列化 ← 二进制字节
```

---

## 2. 安装与环境准备

### 安装 protoc 编译器

```bash
# macOS
brew install protobuf
protoc --version  # libprotoc 27.x

# Ubuntu / Debian
apt-get install -y protobuf-compiler

# Windows（通过 scoop）
scoop install protobuf

# 从源码安装最新版（推荐）
# https://github.com/protocolbuffers/protobuf/releases
# 下载对应平台的预编译二进制
```

### 验证安装

```bash
protoc --version
# 输出：libprotoc 27.0

# 查看帮助
protoc --help
```

### 各语言插件

```bash
# Go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Python（通过 pip）
pip install grpcio-tools

# Rust（通过 tonic-build，无需手动安装插件）
# 在 build.rs 中调用即可
```

---

## 3. 基础语法

### 最简单的 .proto 文件

```protobuf
// 必须声明语法版本（推荐 proto3）
syntax = "proto3";

// 包名（可选，但强烈建议）
package myapp.v1;

// 消息定义
message Person {
  string name  = 1;
  int32  age   = 2;
  string email = 3;
}
```

### 文件结构规范

```protobuf
// 1. 语法声明（必须是第一个非注释行）
syntax = "proto3";

// 2. 包声明
package myapp.v1;

// 3. 选项（可选）
option go_package = "github.com/myorg/myapp/gen/go;myappv1";
option java_package = "com.myorg.myapp.v1";

// 4. 导入（可选）
import "google/protobuf/timestamp.proto";
import "other.proto";

// 5. 枚举定义
enum Status { ... }

// 6. 消息定义
message MyMessage { ... }

// 7. 服务定义（gRPC）
service MyService { ... }
```

### 注释

```protobuf
// 单行注释（推荐用于字段注释）

/*
 * 多行注释
 * 适合文件级说明
 */

message User {
  // 用户唯一标识符，创建后不可修改
  uint64 id = 1;

  // 用户名，3-50 个字符
  string username = 2;
}
```

### 字段编号规则

```protobuf
message Example {
  string field_one   = 1;   // 1-15：使用 1 字节编码（常用字段放这里）
  string field_two   = 2;
  // ...
  string field_15    = 15;  // 1-15 范围结束

  string field_16    = 16;  // 16-2047：使用 2 字节编码
  string field_2047  = 2047;

  // 19000-19999 保留给 protobuf 内部使用，不能使用
  // string bad_field = 19000; //  禁止

  // 最大值
  string max_field   = 536870911; // 2^29 - 1
}
```

**黄金法则**：把最常用的字段分配到 1-15 范围，节省序列化空间。

---

## 4. 标量类型

### 完整类型映射表

| Proto 类型 | 说明                    | Rust      | Go        | Python  | Java         | 默认值  |
| ---------- | ----------------------- | --------- | --------- | ------- | ------------ | ------- |
| `double`   | 64 位浮点               | `f64`     | `float64` | `float` | `double`     | `0`     |
| `float`    | 32 位浮点               | `f32`     | `float32` | `float` | `float`      | `0`     |
| `int32`    | 32 位整数（负数低效）   | `i32`     | `int32`   | `int`   | `int`        | `0`     |
| `int64`    | 64 位整数（负数低效）   | `i64`     | `int64`   | `int`   | `long`       | `0`     |
| `uint32`   | 无符号 32 位整数        | `u32`     | `uint32`  | `int`   | `int`        | `0`     |
| `uint64`   | 无符号 64 位整数        | `u64`     | `uint64`  | `int`   | `long`       | `0`     |
| `sint32`   | 32 位整数（负数高效）   | `i32`     | `int32`   | `int`   | `int`        | `0`     |
| `sint64`   | 64 位整数（负数高效）   | `i64`     | `int64`   | `int`   | `long`       | `0`     |
| `fixed32`  | 固定 4 字节，适合 >2^28 | `u32`     | `uint32`  | `int`   | `int`        | `0`     |
| `fixed64`  | 固定 8 字节，适合 >2^56 | `u64`     | `uint64`  | `int`   | `long`       | `0`     |
| `sfixed32` | 固定 4 字节有符号       | `i32`     | `int32`   | `int`   | `int`        | `0`     |
| `sfixed64` | 固定 8 字节有符号       | `i64`     | `int64`   | `int`   | `long`       | `0`     |
| `bool`     | 布尔值                  | `bool`    | `bool`    | `bool`  | `boolean`    | `false` |
| `string`   | UTF-8 字符串            | `String`  | `string`  | `str`   | `String`     | `""`    |
| `bytes`    | 任意字节序列            | `Vec<u8>` | `[]byte`  | `bytes` | `ByteString` | `[]`    |

### 整数类型选择指南

```protobuf
message NumberGuide {
  // 普通正整数（如 ID、计数）→ uint32/uint64
  uint64 user_id    = 1;
  uint32 page_size  = 2;

  // 可能为负数的整数（如偏移量）→ int32/int64
  int32  offset     = 3;

  // 经常为负数（如温度、差值）→ sint32/sint64（ZigZag 编码，更高效）
  sint32 temperature = 4;
  sint64 balance_delta = 5;

  // 固定范围大整数（如 IPv4 地址、哈希值）→ fixed32/fixed64
  fixed32 ipv4_addr = 6;
  fixed64 hash_value = 7;
}
```

---

## 5. 复合类型

### repeated（数组/列表）

```protobuf
message ShoppingCart {
  repeated string item_names  = 1;  // []string
  repeated int32  quantities  = 2;  // []int32
  repeated Item   items       = 3;  // []Item（嵌套消息）
}

message Item {
  string name  = 1;
  double price = 2;
}
```

**proto3 中 repeated 标量字段默认使用 packed 编码**（更紧凑），等价于：

```protobuf
repeated int32 samples = 1 [packed = true];
```

### map（键值对）

```protobuf
message Config {
  // map<key_type, value_type> 字段名 = 编号;
  map<string, string>  labels     = 1;  // 标签
  map<string, int32>   counters   = 2;  // 计数器
  map<int64,  Profile> profiles   = 3;  // 值可以是消息类型
}
```

**map 的限制：**

- key 类型只能是整数或 string（不能是 float、double、bytes、消息类型）
- map 字段不能是 `repeated`
- map 没有稳定的迭代顺序
- map 在序列化时等价于 `repeated MapFieldEntry`（可以安全地与旧版本互操作）

### oneof（联合类型）

```protobuf
message Notification {
  string title = 1;

  // oneof 中只能设置一个字段
  oneof content {
    string  text_message = 2;
    bytes   image_data   = 3;
    VideoClip video      = 4;
  }
}

message VideoClip {
  string url      = 1;
  uint32 duration = 2;
}
```

**oneof 特性：**

- 设置 oneof 中的一个字段会自动清除其他字段
- 不能在 oneof 中使用 `repeated` 字段
- oneof 本身不能是 `repeated`（但可以通过 `repeated` 消息包含 oneof 实现数组效果）

在 Rust（prost）中生成的代码：

```rust
pub struct Notification {
    pub title: String,
    pub content: Option<notification::Content>,
}

pub mod notification {
    pub enum Content {
        TextMessage(String),
        ImageData(Vec<u8>),
        Video(VideoClip),
    }
}
```

---

## 6. 枚举

### 基础枚举

```protobuf
enum OrderStatus {
  // proto3 要求第一个值必须是 0（作为默认值）
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PROCESSING  = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_DELIVERED   = 4;
  ORDER_STATUS_CANCELLED   = 5;
}

message Order {
  uint64      id     = 1;
  OrderStatus status = 2;  // 默认值：ORDER_STATUS_UNSPECIFIED
}
```

**枚举命名规范（Google 风格）：**

- 枚举类型名：`PascalCase`
- 枚举值：`SCREAMING_SNAKE_CASE`，且以枚举类型名为前缀
- 第一个值（0）命名为 `_UNSPECIFIED` 或 `_UNKNOWN`

### 别名（allow_alias）

```protobuf
enum Direction {
  option allow_alias = true;  // 允许不同名称共享相同编号
  DIRECTION_UNSPECIFIED = 0;
  DIRECTION_NORTH = 1;
  DIRECTION_UP    = 1;  // UP 是 NORTH 的别名
  DIRECTION_SOUTH = 2;
  DIRECTION_DOWN  = 2;  // DOWN 是 SOUTH 的别名
}
```

### 枚举保留值

```protobuf
enum MyEnum {
  reserved 2, 15, 9 to 11;           // 保留编号
  reserved "OLD_VALUE", "DEPRECATED"; // 保留名称

  MY_ENUM_UNSPECIFIED = 0;
  MY_ENUM_FIRST       = 1;
  // MY_ENUM_SECOND = 2;  ❌ 编号 2 已保留
}
```

### 未知枚举值的处理

proto3 中，如果收到未知的枚举值：

- 在支持 proto3 的语言中，保留原始数值
- 可以通过 `number()` 或类似方法获取原始整数值
- 不会导致解析失败（向前兼容）

---

## 7. 消息嵌套与引用

### 嵌套消息定义

```protobuf
message User {
  uint64  id       = 1;
  string  username = 2;
  Address address  = 3;  // 引用外部消息

  // 直接在内部定义（仅此消息使用的结构）
  message Preference {
    string  language    = 1;
    string  timezone    = 2;
    bool    email_notif = 3;
  }

  Preference preference = 4;
}

message Address {
  string street  = 1;
  string city    = 2;
  string country = 3;
  string zipcode = 4;
}
```

引用嵌套消息：

```protobuf
message AnotherMessage {
  // 通过 外层消息.内层消息 引用嵌套消息
  User.Preference pref = 1;
}
```

### 消息作为字段 vs 标量字段的区别

消息类型字段的默认值是 `null`/`nil`（未设置），而不是空消息：

```protobuf
message Response {
  // 如果 error 字段存在，说明有错误；不存在说明成功
  // 这是一种惯用模式
  Error error = 1;
  Data  data  = 2;
}
```

### 循环引用（自引用）

```protobuf
// 树形结构
message TreeNode {
  string           value    = 1;
  repeated TreeNode children = 2;  // 自引用，构建树
}

// 双向链表节点（实际中用 ID 引用更常见）
message LinkedNode {
  uint64 id    = 1;
  string value = 2;
  uint64 next_id = 3;  // 用 ID 而不是直接引用
}
```

---

## 8. 字段修饰符

### proto3 字段规则

```protobuf
message Proto3Example {
  // 普通字段（singular）：有零值时不序列化
  string  name  = 1;
  int32   age   = 2;

  // optional：区分"未设置"和"设置为零值"
  optional int32 score = 3;  // 生成 Option<i32>（Rust）/ *int32（Go）

  // repeated：数组/列表
  repeated string tags = 4;

  // map：键值对
  map<string, int32> metadata = 5;
}
```

### optional 的重要性

```protobuf
message SearchRequest {
  string query = 1;

  // 没有 optional：无法区分"用户传了 0"和"用户没传"
  int32  page = 2;

  // 有 optional：可以判断用户是否传了这个字段
  optional int32 page_size = 3;
}
```

```rust
// Rust 中 optional 字段生成 Option<T>
let req = SearchRequest {
    query: "hello".to_string(),
    page: 0,            // 不知道是"第0页"还是"未设置"
    page_size: None,    // 明确知道未设置
};

if req.page_size.is_none() {
    // 使用默认页大小
}
```

### reserved（保留字段）

用于防止已删除字段的编号/名称被意外复用：

```protobuf
message User {
  reserved 3, 7;              // 保留编号（曾用于 old_field）
  reserved 9 to 11;           // 保留编号范围
  reserved "old_name", "tmp"; // 保留字段名

  uint64 id       = 1;
  string username = 2;
  // 编号 3、7、9、10、11 不可再使用
  string email    = 4;
}
```

---

## 9. Well-Known Types

Google 官方提供的标准消息类型，通过 `import` 引入：

### Timestamp（时间戳）

```protobuf
import "google/protobuf/timestamp.proto";

message Event {
  string                    name       = 1;
  google.protobuf.Timestamp created_at = 2;
  google.protobuf.Timestamp updated_at = 3;
}
```

```rust
// Rust (prost-types)
use prost_types::Timestamp;
use std::time::{SystemTime, UNIX_EPOCH};

let now = SystemTime::now()
    .duration_since(UNIX_EPOCH)
    .unwrap();

let ts = Timestamp {
    seconds: now.as_secs() as i64,
    nanos:   now.subsec_nanos() as i32,
};
```

### Duration（时长）

```protobuf
import "google/protobuf/duration.proto";

message Task {
  string                   name    = 1;
  google.protobuf.Duration timeout = 2;
}
```

### Struct（动态结构）

```protobuf
import "google/protobuf/struct.proto";

message DynamicConfig {
  string              name   = 1;
  google.protobuf.Struct extra = 2;  // 任意 JSON 对象
}
```

### Wrappers（可空基础类型）

```protobuf
import "google/protobuf/wrappers.proto";

message UserProfile {
  // 使用 wrapper 类型表示可空值（区别于 optional）
  google.protobuf.StringValue nickname   = 1;  // nullable string
  google.protobuf.Int32Value  level      = 2;  // nullable int32
  google.protobuf.BoolValue   is_premium = 3;  // nullable bool
}
```

可用 Wrapper 类型：`DoubleValue`、`FloatValue`、`Int64Value`、`UInt64Value`、`Int32Value`、`UInt32Value`、`BoolValue`、`StringValue`、`BytesValue`

### Empty（无参数/无返回值）

```protobuf
import "google/protobuf/empty.proto";

service HealthService {
  // 无需请求体
  rpc Ping (google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

### Any（任意类型）

```protobuf
import "google/protobuf/any.proto";

message Container {
  google.protobuf.Any payload = 1;
}
```

```rust
// Rust 中打包/解包 Any
use prost_types::Any;
use prost::Message;

// 打包
let user = User { id: 1, name: "Alice".into() };
let any = Any::from_msg(&user).unwrap();

// 解包
let user: User = any.to_msg().unwrap();
```

### FieldMask（字段掩码）

用于 PATCH 操作，指定哪些字段需要更新：

```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User                       user       = 1;
  google.protobuf.FieldMask update_mask = 2;
}
```

```
// 只更新 name 和 email
update_mask.paths = ["name", "email"]
```

---

## 10. 服务定义（gRPC）

### 四种 RPC 模式

```protobuf
syntax = "proto3";
package chat.v1;

service ChatService {
  // 一元 RPC：请求-响应
  rpc SendMessage (SendMessageRequest) returns (SendMessageResponse);

  // 服务端流：一个请求，多个响应
  rpc WatchMessages (WatchMessagesRequest) returns (stream Message);

  // 客户端流：多个请求，一个响应
  rpc UploadHistory (stream Message) returns (UploadHistoryResponse);

  // 双向流：多个请求，多个响应
  rpc Chat (stream Message) returns (stream Message);
}

message Message {
  string  id         = 1;
  string  content    = 2;
  string  sender_id  = 3;
  string  room_id    = 4;
  int64   timestamp  = 5;
}

message SendMessageRequest  { Message message = 1; }
message SendMessageResponse { string  msg_id  = 1; }
message WatchMessagesRequest { string room_id = 1; }
message UploadHistoryResponse { uint32 count = 1; bool success = 2; }
```

### 完整的 API 设计示例（用户服务）

```protobuf
syntax = "proto3";
package user.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/field_mask.proto";
import "google/protobuf/empty.proto";

option go_package = "github.com/myorg/user-service/gen/go/user/v1;userv1";

// ── 枚举 ─────────────────────────────────────────
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_MEMBER      = 1;
  USER_ROLE_ADMIN       = 2;
  USER_ROLE_SUPERADMIN  = 3;
}

// ── 核心消息 ──────────────────────────────────────
message User {
  uint64                    id         = 1;
  string                    username   = 2;
  string                    email      = 3;
  UserRole                  role       = 4;
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

// ── 请求/响应 ─────────────────────────────────────
message CreateUserRequest {
  string   username = 1;
  string   email    = 2;
  string   password = 3;
  UserRole role     = 4;
}

message GetUserRequest {
  uint64 id = 1;
}

message UpdateUserRequest {
  User                      user        = 1;
  google.protobuf.FieldMask update_mask = 2;
}

message DeleteUserRequest {
  uint64 id = 1;
}

message ListUsersRequest {
  uint32 page_size  = 1;
  string page_token = 2;
  string filter     = 3;  // 过滤表达式
  string order_by   = 4;  // 排序字段
}

message ListUsersResponse {
  repeated User  users          = 1;
  string         next_page_token = 2;
  uint32         total_count    = 3;
}

// ── 服务 ──────────────────────────────────────────
service UserService {
  rpc CreateUser (CreateUserRequest)            returns (User);
  rpc GetUser    (GetUserRequest)               returns (User);
  rpc UpdateUser (UpdateUserRequest)            returns (User);
  rpc DeleteUser (DeleteUserRequest)            returns (google.protobuf.Empty);
  rpc ListUsers  (ListUsersRequest)             returns (ListUsersResponse);
  rpc WatchUser  (GetUserRequest)               returns (stream User);
}
```

---

## 11. 包与导入

### 包声明

```protobuf
// 包名用于避免命名冲突
// 建议格式：组织名.服务名.版本
package myorg.myservice.v1;
```

### 导入其他 proto 文件

```protobuf
// 导入 Google Well-Known Types
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// 导入同项目的其他 proto
import "common/v1/pagination.proto";
import "user/v1/user.proto";

// 弱导入（即使文件不存在也不报错，通常不需要）
import weak "optional/feature.proto";

// 公开导入（导入此文件的人也能看到被导入的类型）
import public "base_types.proto";
```

### 目录结构最佳实践

```
proto/
├── common/
│   └── v1/
│       ├── pagination.proto
│       ├── error.proto
│       └── timestamp_ext.proto
├── user/
│   └── v1/
│       ├── user.proto
│       └── user_service.proto
├── order/
│   └── v1/
│       ├── order.proto
│       └── order_service.proto
└── buf.yaml   ← 使用 buf 工具时的配置文件
```

### 编译多个 proto 文件

```bash
# 指定 proto 搜索路径（-I 或 --proto_path）
protoc \
  -I ./proto \
  -I $(go env GOPATH)/pkg/mod/google.golang.org/protobuf@v1.34.0 \
  --go_out=./gen/go \
  --go-grpc_out=./gen/go \
  ./proto/user/v1/*.proto \
  ./proto/order/v1/*.proto
```

---

## 12. 选项（Options）

选项不改变语义，但影响代码生成行为。

### 文件级选项

```protobuf
syntax = "proto3";
package myapp.v1;

// Go：生成代码的包路径和包名
option go_package = "github.com/myorg/myapp/gen/go/myapp/v1;myappv1";

// Java：包名
option java_package = "com.myorg.myapp.v1";
option java_outer_classname = "MyAppProto";
option java_multiple_files = true;  // 每个消息生成单独的 Java 文件

// Python：不需要特别选项

// C#：命名空间
option csharp_namespace = "MyOrg.MyApp.V1";

// 优化提示
option optimize_for = SPEED;       // 默认，生成最快的序列化代码
// option optimize_for = CODE_SIZE; // 生成最小的代码
// option optimize_for = LITE_RUNTIME; // 轻量运行时（移动端）
```

### 消息级选项

```protobuf
message MyMessage {
  option deprecated = true;  // 标记整个消息已废弃

  string name = 1;
}
```

### 字段级选项

```protobuf
message Example {
  string old_field = 1 [deprecated = true];  // 标记字段已废弃

  // packed：对 repeated 标量字段控制编码方式
  // proto3 中默认已是 packed = true
  repeated int32 values = 2 [packed = true];

  // retention：控制是否保留到运行时（实验性）
  string runtime_only = 3 [retention = RUNTIME];
}
```

### 自定义选项

```protobuf
import "google/protobuf/descriptor.proto";

// 定义自定义选项
extend google.protobuf.FieldOptions {
  string my_field_option = 50000;
}

message MyMessage {
  string name = 1 [(my_field_option) = "custom_value"];
}
```

---

## 13. 编码原理

### Wire Type（线路类型）

每个字段编码为：`(field_number << 3) | wire_type`

| Wire Type | 值  | 用于                                       |
| --------- | --- | ------------------------------------------ |
| VARINT    | 0   | int32/64, uint32/64, sint32/64, bool, enum |
| I64       | 1   | fixed64, sfixed64, double                  |
| LEN       | 2   | string, bytes, 消息, packed repeated       |
| I32       | 5   | fixed32, sfixed32, float                   |

### Varint 编码（变长整数）

小数值用更少的字节：

```
数值 1   → 0x01                    （1 字节）
数值 127 → 0x7F                    （1 字节）
数值 128 → 0x80 0x01               （2 字节）
数值 300 → 0xAC 0x02               （2 字节）
数值 2^21→ 0x80 0x80 0x80 0x01     （4 字节）
```

**编码规则**：每字节最高位（MSB）为延续标志，剩余 7 位为数据，小端序。

### ZigZag 编码（sint32/sint64）

解决负数编码效率问题：

```
0  → 0
-1 → 1
1  → 2
-2 → 3
2  → 4
-n → 2n - 1
n  → 2n
```

所以 `sint32` 中 -1 只需 1 字节（编码为 1），而 `int32` 中 -1 需要 10 字节（编码为很大的正数）。

### 实际编码示例

```protobuf
message Test {
  int32  field_a = 1;  // 字段编号 1
  string field_b = 2;  // 字段编号 2
}
```

对 `Test { field_a: 150, field_b: "hi" }` 编码：

```
字段 1（field_a = 150）：
  tag:   (1 << 3) | 0 = 0x08（字段1，varint类型）
  value: 150 → 0x96 0x01（varint 编码）
  bytes: 08 96 01

字段 2（field_b = "hi"）：
  tag:   (2 << 3) | 2 = 0x12（字段2，LEN类型）
  len:   2（字符串长度）
  value: 0x68 0x69（"hi" 的 ASCII）
  bytes: 12 02 68 69

完整序列化：08 96 01 12 02 68 69（共 7 字节）
```

对比 JSON：`{"field_a":150,"field_b":"hi"}` = 30 字节

### 消息边界

Protobuf **没有消息边界标记**，序列化后的二进制流无法自我描述长度。在网络传输时需要自行添加长度前缀：

```
[4字节长度][protobuf二进制][4字节长度][protobuf二进制]...
```

gRPC 的分帧格式：

```
[1字节压缩标志][4字节消息长度][消息二进制数据]
```

---

## 14. 向前/向后兼容

### 兼容性规则

**安全操作（不破坏兼容性）：**

```protobuf
// ✓ 添加新字段（旧客户端忽略未知字段）
message User {
  uint64 id       = 1;
  string username = 2;
  string email    = 3;  // 新增字段，旧客户端忽略
}

// ✓ 删除字段（用 reserved 保留编号，防止复用）
message User {
  reserved 2;           // 原来是 username，已删除
  reserved "username";
  uint64 id    = 1;
  string email = 3;
}

// ✓ 重命名字段（编号不变则二进制兼容，但 JSON 格式不兼容）
message User {
  uint64 id         = 1;
  string user_name  = 2;  // 从 username 改为 user_name，二进制格式不变
}
```

**危险操作（破坏兼容性）：**

```protobuf
//  修改字段编号
message User {
  string username = 5;  // 原来是 2，改为 5 → 完全不兼容
}

//  修改字段类型（某些情况下）
message User {
  int64 id = 1;  // 原来是 uint64，改为 int64 → 不兼容
}

// ✓ 但这些类型转换是安全的（wire type 相同）
// int32 ↔ uint32 ↔ int64 ↔ uint64 ↔ bool（都是 VARINT）
// string ↔ bytes（都是 LEN，但语义不同）
// fixed32 ↔ sfixed32 ↔ float（都是 I32）
// fixed64 ↔ sfixed64 ↔ double（都是 I64）

//  重用已删除的字段编号
message User {
  uint64 id = 1;
  // 原来编号 2 是 username（已删除），现在用 2 存 role → 类型冲突！
  UserRole role = 2;  // 危险！
}
```

### API 版本化策略

```
// 推荐：通过包名版本化
package user.v1;  // 稳定版
package user.v2;  // 新版本（有破坏性变更时递增）

// proto 文件路径同步
proto/user/v1/user.proto
proto/user/v2/user.proto  ← 重大变更时创建新版本
```

---

## 15. 在 Rust 中使用（prost）

### Cargo.toml

```toml
[dependencies]
prost       = "0.13"
prost-types = "0.13"  # Well-Known Types
tonic       = "0.12"  # gRPC（可选）
bytes       = "1"     # Bytes 类型支持

[build-dependencies]
tonic-build = "0.12"
# 或纯 prost（不用 gRPC）：
prost-build = "0.13"
```

### build.rs

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 方式一：tonic-build（同时生成 gRPC 代码）
    tonic_build::configure()
        .build_server(true)
        .build_client(true)
        .type_attribute(".", "#[derive(serde::Serialize, serde::Deserialize)]")
        .compile_protos(
            &["proto/user/v1/user.proto"],
            &["proto/"],
        )?;

    // 方式二：纯 prost-build（只生成消息代码）
    // prost_build::compile_protos(
    //     &["proto/user/v1/user.proto"],
    //     &["proto/"],
    // )?;

    Ok(())
}
```

### 引入生成的代码

```rust
// src/lib.rs 或 src/main.rs
pub mod user {
    // 对应 proto 文件中的 package user.v1
    pub mod v1 {
        tonic::include_proto!("user.v1");
    }
}

// 或使用文件包含
pub mod proto {
    include!(concat!(env!("OUT_DIR"), "/user.v1.rs"));
}
```

### 消息序列化与反序列化

```rust
use prost::Message;
use user::v1::User;

fn main() {
    let user = User {
        id:         1,
        username:   "alice".to_string(),
        email:      "alice@example.com".to_string(),
        role:       1,  // USER_ROLE_MEMBER
        created_at: None,
        updated_at: None,
    };

    // 序列化
    let mut buf = Vec::new();
    user.encode(&mut buf).unwrap();
    println!("序列化字节数: {}", buf.len());

    // 序列化到预分配的 bytes::BytesMut
    let mut buf = bytes::BytesMut::with_capacity(user.encoded_len());
    user.encode(&mut buf).unwrap();

    // 反序列化
    let decoded = User::decode(buf.freeze()).unwrap();
    assert_eq!(user.id, decoded.id);

    // 带长度前缀的编解码（流式传输常用）
    let mut buf = Vec::new();
    user.encode_length_delimited(&mut buf).unwrap();
    let decoded = User::decode_length_delimited(bytes::Bytes::from(buf)).unwrap();
}
```

### 处理 oneof 字段

```protobuf
message Notification {
  oneof content {
    string text  = 1;
    bytes  image = 2;
  }
}
```

```rust
use notification::Content;

let notif = Notification {
    content: Some(Content::Text("Hello".to_string())),
};

match notif.content {
    Some(Content::Text(msg))   => println!("文本: {}", msg),
    Some(Content::Image(data)) => println!("图片: {} bytes", data.len()),
    None                       => println!("空消息"),
}
```

### 处理 Timestamp

```rust
use prost_types::Timestamp;
use std::time::{SystemTime, UNIX_EPOCH};

// 当前时间转 Timestamp
fn now_timestamp() -> Timestamp {
    let d = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap();
    Timestamp {
        seconds: d.as_secs() as i64,
        nanos:   d.subsec_nanos() as i32,
    }
}

// Timestamp 转 chrono（需要 chrono 依赖）
fn ts_to_datetime(ts: &Timestamp) -> chrono::DateTime<chrono::Utc> {
    chrono::DateTime::from_timestamp(ts.seconds, ts.nanos as u32)
        .unwrap_or_default()
}
```

### 为生成的消息实现额外 trait

```rust
// build.rs 中为所有消息添加 serde 支持
tonic_build::configure()
    .type_attribute(".", "#[derive(serde::Serialize, serde::Deserialize)]")
    .compile_protos(&["proto/user.proto"], &["proto/"])?;

// 或只为特定消息
tonic_build::configure()
    .type_attribute(
        "user.v1.User",
        "#[derive(serde::Serialize, serde::Deserialize)]",
    )
    .compile_protos(&["proto/user.proto"], &["proto/"])?;
```

---

## 16. 在 Go 中使用

### 生成代码

```bash
protoc \
  --go_out=./gen \
  --go_opt=paths=source_relative \
  --go-grpc_out=./gen \
  --go-grpc_opt=paths=source_relative \
  -I ./proto \
  ./proto/user/v1/user.proto
```

### 消息操作

```go
package main

import (
    "fmt"
    "log"

    "google.golang.org/protobuf/proto"
    userv1 "github.com/myorg/myapp/gen/go/user/v1"
    "google.golang.org/protobuf/types/known/timestamppb"
)

func main() {
    user := &userv1.User{
        Id:        1,
        Username:  "alice",
        Email:     "alice@example.com",
        Role:      userv1.UserRole_USER_ROLE_MEMBER,
        CreatedAt: timestamppb.Now(),
    }

    // 序列化
    data, err := proto.Marshal(user)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("序列化字节数: %d\n", len(data))

    // 反序列化
    decoded := &userv1.User{}
    if err := proto.Unmarshal(data, decoded); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("用户名: %s\n", decoded.Username)

    // 克隆
    cloned := proto.Clone(user).(*userv1.User)

    // 比较（不能用 ==，要用 proto.Equal）
    fmt.Println(proto.Equal(user, cloned)) // true

    // 转 JSON
    // import "google.golang.org/protobuf/encoding/protojson"
    // jsonStr, _ := protojson.Marshal(user)

    // 判断 optional 字段是否设置
    if decoded.UpdatedAt != nil {
        fmt.Println("有更新时间")
    }
}
```

### gRPC 客户端

```go
conn, err := grpc.Dial(
    "localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
defer conn.Close()

client := userv1.NewUserServiceClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
if err != nil {
    log.Fatal(err)
}
fmt.Println(resp.Username)
```

---

## 17. 在 Python 中使用

### 生成代码

```bash
# 方式一：通过 grpc-tools（推荐）
pip install grpcio-tools

python -m grpc_tools.protoc \
  -I ./proto \
  --python_out=./gen \
  --grpc_python_out=./gen \
  ./proto/user/v1/user.proto

# 方式二：使用 buf 工具
buf generate
```

### 消息操作

```python
from gen.user.v1 import user_pb2, user_pb2_grpc
from google.protobuf.timestamp_pb2 import Timestamp
from google.protobuf import json_format
import time

# 创建消息
user = user_pb2.User(
    id=1,
    username="alice",
    email="alice@example.com",
    role=user_pb2.USER_ROLE_MEMBER,
)

# 设置 Timestamp 字段
ts = Timestamp()
ts.FromDatetime(datetime.utcnow())
user.created_at.CopyFrom(ts)

# 序列化
data = user.SerializeToString()
print(f"序列化字节数: {len(data)}")

# 反序列化
decoded = user_pb2.User()
decoded.ParseFromString(data)
print(decoded.username)

# 转 JSON（使用 proto 字段名）
json_str = json_format.MessageToJson(user)

# 从 JSON 解析
json_format.Parse(json_str, user_pb2.User())

# 转 dict
d = json_format.MessageToDict(user)

# 检查字段是否已设置（proto3 optional）
if user.HasField("updated_at"):
    print("有更新时间")
```

### gRPC 客户端

```python
import grpc
from gen.user.v1 import user_pb2, user_pb2_grpc

with grpc.insecure_channel("localhost:50051") as channel:
    stub = user_pb2_grpc.UserServiceStub(channel)

    response = stub.GetUser(
        user_pb2.GetUserRequest(id=1),
        timeout=5,
    )
    print(response.username)
```

---

## 18. Proto2 vs Proto3

| 特性               | Proto2                               | Proto3                               |
| ------------------ | ------------------------------------ | ------------------------------------ |
| 字段规则           | `required` / `optional` / `repeated` | `singular` / `optional` / `repeated` |
| 默认值             | 可自定义 `[default = value]`         | 固定为类型零值                       |
| `required` 字段    | 支持（已不推荐）                     | 不支持                               |
| 未知字段           | 丢弃（protobuf 3.5+ 保留）           | 保留并转发                           |
| Map 类型           | 不支持（需手动定义）                 | 原生支持                             |
| JSON 映射          | 不支持                               | 标准支持                             |
| 扩展（extensions） | 支持                                 | 不支持（用 Any 替代）                |
| 流行程度           | 旧代码库                             | 新项目推荐                           |

**Proto2 示例：**

```protobuf
syntax = "proto2";

message Person {
  required string name  = 1;            // 必须字段（proto3 无此概念）
  optional int32  age   = 2;            // 可选字段
  optional string email = 3 [default = "unknown@example.com"];  // 自定义默认值
  repeated string phones = 4;
}
```

**结论**：新项目一律使用 proto3。proto2 仅在维护旧代码时涉及。

---

## 19. 最佳实践

### 命名规范

```protobuf
// 文件名：小写，下划线分隔
// user_service.proto ✓
// UserService.proto  ✗

// 包名：小写，点分隔，含版本
package myorg.user.v1;

// 消息名：PascalCase
message UserProfile { ... }  // ✓
message user_profile { ... } // ✗

// 字段名：snake_case
string user_name = 1;   // ✓
string userName  = 1;   // ✗

// 枚举类型名：PascalCase
enum UserStatus { ... }

// 枚举值：SCREAMING_SNAKE_CASE，加类型前缀
USER_STATUS_ACTIVE = 1;    // ✓
ACTIVE             = 1;    // ✗（容易冲突）

// 服务名：PascalCase + Service 后缀
service UserService { ... }

// RPC 方法名：PascalCase，动词开头
rpc GetUser    (...)
rpc CreateUser (...)
rpc UpdateUser (...)
rpc DeleteUser (...)
rpc ListUsers  (...)
```

### API 设计规范（参考 Google AIP）

```protobuf
// 1. 标准方法（CRUD）使用标准命名
service BookService {
  rpc GetBook    (GetBookRequest)    returns (Book);
  rpc ListBooks  (ListBooksRequest)  returns (ListBooksResponse);
  rpc CreateBook (CreateBookRequest) returns (Book);
  rpc UpdateBook (UpdateBookRequest) returns (Book);
  rpc DeleteBook (DeleteBookRequest) returns (google.protobuf.Empty);
}

// 2. 列表请求使用分页
message ListBooksRequest {
  string parent     = 1;  // 父资源名称
  int32  page_size  = 2;  // 每页数量，服务端限制最大值
  string page_token = 3;  // 分页 token
  string filter     = 4;  // 过滤器
  string order_by   = 5;  // 排序："-created_at,name"
}

message ListBooksResponse {
  repeated Book books          = 1;
  string        next_page_token = 2;  // 空字符串表示最后一页
  int32         total_size     = 3;   // 总数（可选）
}

// 3. 更新操作使用 FieldMask
message UpdateBookRequest {
  Book                      book        = 1;
  google.protobuf.FieldMask update_mask = 2;
}

// 4. 请求和响应各用独立消息（即使当前为空）
// ✓ 便于后续扩展字段
message DeleteBookRequest {
  string name = 1;
}
// ✗ 不要直接用 Book 作为请求体
```

### 版本化

```
// 重要原则：
// - 不要原地做破坏性变更
// - 增量式演进 v1
// - 有破坏性变更时，创建 v2 包

// proto/book/v1/book.proto  ← 当前稳定版本
// proto/book/v2/book.proto  ← 新版本（有不兼容变更）

// 服务端同时提供 v1 和 v2，给客户端迁移时间
```

### 字段设计原则

```protobuf
message GoodDesign {
  // ✓ 用 string 存储资源名称（而不是 int ID）
  string name = 1;  // "users/123"

  // ✓ 时间用 Timestamp（而不是 int64 时间戳）
  google.protobuf.Timestamp created_at = 2;

  // ✓ 可空字段用 optional
  optional string display_name = 3;

  // ✓ 金额用整数（分）而不是浮点数（避免精度问题）
  int64 price_cents = 4;

  // ✓ 枚举第一个值用 _UNSPECIFIED
  Status status = 5;
}
```

---

## 20. 常见陷阱

### 陷阱 1：proto3 默认值无法区分"未设置"和"零值"

```protobuf
// 问题：无法判断 age 是"未提供"还是"确实是 0"
message Person {
  int32 age = 1;  // 默认值 0
}
```

```protobuf
// 解决方案：使用 optional
message Person {
  optional int32 age = 1;  // 生成 Option<i32>，None = 未设置
}

// 或使用 Wrapper 类型
import "google/protobuf/wrappers.proto";
message Person {
  google.protobuf.Int32Value age = 1;  // null = 未设置
}
```

### 陷阱 2：复用已删除的字段编号

```protobuf
// 版本 1
message User {
  uint64 id       = 1;
  string password = 2;  // 后来删除了这个字段
}

// 版本 2（危险！）
message User {
  uint64 id    = 1;
  string email = 2;  //  复用了编号 2！旧数据会被误读为 email
}

// 正确做法
message User {
  reserved 2;
  reserved "password";

  uint64 id    = 1;
  string email = 3;  // ✓ 使用新编号
}
```

### 陷阱 3：在 map 中使用 float/double 作为 key

```protobuf
//  编译器会报错：float 不能作为 map key
map<float, string> bad_map = 1;

// ✓ 合法的 key 类型
map<string,  string> str_map  = 1;
map<int32,   string> int_map  = 2;
map<uint64,  string> uint_map = 3;
map<bool,    string> bool_map = 4;
```

### 陷阱 4：枚举第一个值不是 0

```protobuf
//  proto3 要求第一个枚举值必须是 0
enum Status {
  STATUS_ACTIVE   = 1;  // 编译错误！
  STATUS_INACTIVE = 2;
}

// ✓ 正确：第一个值为 0，通常用 _UNSPECIFIED
enum Status {
  STATUS_UNSPECIFIED = 0;  // 默认值
  STATUS_ACTIVE      = 1;
  STATUS_INACTIVE    = 2;
}
```

### 陷阱 5：oneof 中使用 repeated

```protobuf
message Bad {
  oneof value {
    repeated string items = 1;  //  oneof 中不能有 repeated 字段
  }
}

// ✓ 用包装消息
message StringList {
  repeated string items = 1;
}

message Good {
  oneof value {
    string     text = 1;
    StringList list = 2;  // ✓
  }
}
```

### 陷阱 6：忽略未知字段

```protobuf
// 场景：服务端升级，新增字段；旧客户端收到响应
// proto3 会保留未知字段，重新序列化时会原样转发
// 这是预期行为，用于在服务间透传字段
// 但不要依赖未知字段的具体值（未解析）
```

### 陷阱 7：JSON 序列化字段名变化

```protobuf
message User {
  string user_name = 1;  // proto 字段名：user_name
}
```

Protobuf JSON 格式将字段名转为 `camelCase`：

- proto 字段 `user_name` → JSON 键 `"userName"`
- proto 字段 `created_at` → JSON 键 `"createdAt"`

如果系统依赖 JSON 格式，注意客户端要使用正确的键名。

---

## 快速参考

### 类型选择速查

| 场景                 | 推荐类型                    |
| -------------------- | --------------------------- |
| 用户 ID / 资源 ID    | `uint64` 或 `string`        |
| 计数 / 数量          | `uint32`                    |
| 偏移量 / 差值        | `sint32` / `sint64`         |
| 金额（避免精度问题） | `int64`（单位：分）         |
| 时间点               | `google.protobuf.Timestamp` |
| 时长                 | `google.protobuf.Duration`  |
| 可空基础类型         | `optional T`                |
| 任意 JSON            | `google.protobuf.Struct`    |
| 动态类型             | `google.protobuf.Any`       |
| PATCH 字段选择       | `google.protobuf.FieldMask` |
| 布尔标志             | `bool`                      |
| 状态 / 类别          | `enum`                      |
| 互斥字段             | `oneof`                     |
| 键值对               | `map<K, V>`                 |

### 编解码命令速查

```bash
# 编译单个文件
protoc -I./proto --go_out=./gen ./proto/user.proto

# 编译所有文件
protoc -I./proto --go_out=./gen $(find ./proto -name "*.proto")

# 解码二进制（调试用）
protoc --decode=myapp.v1.User proto/user.proto < user.bin

# 解码为 JSON
protoc --decode=myapp.v1.User proto/user.proto < user.bin | python3 -c "import sys,json; ..."

# 编码 JSON 为二进制
echo '{"id": 1, "username": "alice"}' | \
  protoc --encode=myapp.v1.User proto/user.proto > user.bin
```


