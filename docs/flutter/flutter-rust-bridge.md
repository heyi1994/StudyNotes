# Flutter Rust Bridge

flutter_rust_bridge（FRB）是连接 Flutter/Dart 与 Rust 的代码生成工具。它自动生成 FFI 绑定代码，让你可以在 Flutter 中直接调用 Rust 函数，享受 Rust 的高性能和内存安全。当前主流版本为 **v2**。

**适用场景：**

- CPU 密集型计算（图像处理、加解密、音视频编解码）
- 复用已有 Rust 库
- 需要跨平台（Android / iOS / Desktop / Web）的原生性能
- 替代平台通道（Platform Channel）的繁琐 FFI 代码

---

## 环境准备

### 安装 Rust

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup update stable

# Android 编译目标
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android i686-linux-android

# iOS 编译目标
rustup target add aarch64-apple-ios x86_64-apple-ios aarch64-apple-ios-sim

# macOS / Linux / Windows Desktop
rustup target add x86_64-apple-darwin aarch64-apple-darwin    # macOS
```

### 安装 FRB CLI

```shell
cargo install flutter_rust_bridge_codegen

# Android 交叉编译工具
cargo install cargo-ndk

# iOS 多架构打包
cargo install cargo-lipo
```

### 安装 Flutter 侧依赖

```yaml
# pubspec.yaml
dependencies:
  flutter_rust_bridge: ^2.0.0

dev_dependencies:
  ffigen: ^9.0.0
```

---

## 创建项目

### 方式一：从模板创建（推荐）

```shell
flutter_rust_bridge_codegen create my_app
cd my_app
```

生成的项目结构：

```
my_app/
├── lib/
│   ├── main.dart
│   └── src/
│       └── rust/              # 自动生成的 Dart 绑定代码（勿手动修改）
│           ├── api/
│           │   └── simple.dart
│           ├── frb_generated.dart
│           └── frb_generated.io.dart
├── rust/                      # Rust 核心逻辑（你的代码写这里）
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs
│       └── api/
│           └── simple.rs
├── android/
├── ios/
└── pubspec.yaml
```

### 方式二：添加到已有 Flutter 项目

```shell
cd your_flutter_project
flutter_rust_bridge_codegen integrate
```

---

## 代码生成工作流

```
1. 编写 Rust 代码（rust/src/api/）
        ↓
2. 运行代码生成
   flutter_rust_bridge_codegen generate
        ↓
3. 自动生成 Dart 绑定（lib/src/rust/）
        ↓
4. 在 Flutter 中调用
```

开发时开启监听模式，保存 Rust 文件后自动重新生成：

```shell
flutter_rust_bridge_codegen generate --watch
```

---

## 基础类型映射

| Rust 类型 | Dart 类型 | 说明 |
|-----------|-----------|------|
| `i8` / `i16` / `i32` | `int` | |
| `i64` | `int` | Dart int 是 64 位 |
| `u8` / `u16` / `u32` | `int` | |
| `u64` | `BigInt` | 超出 Dart int 范围 |
| `f32` / `f64` | `double` | |
| `bool` | `bool` | |
| `String` | `String` | |
| `&str` | `String` | |
| `Vec<u8>` | `Uint8List` | 字节数组 |
| `Vec<T>` | `List<T>` | |
| `[T; N]` | `List<T>` | 固定长度数组 |
| `Option<T>` | `T?` | 可空类型 |
| `HashMap<K, V>` | `Map<K, V>` | |
| `(A, B)` | `(A, B)` | Dart Record |
| `()` | `void` | |

---

## 基础函数

### 普通函数

```rust
// rust/src/api/simple.rs
use flutter_rust_bridge::frb;

/// 简单加法
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

/// 字符串处理
pub fn greet(name: String) -> String {
    format!("你好，{}！", name)
}

/// 处理可选值
pub fn double_if_positive(value: i32) -> Option<i32> {
    if value > 0 { Some(value * 2) } else { None }
}

/// 处理字节数组
pub fn reverse_bytes(data: Vec<u8>) -> Vec<u8> {
    data.into_iter().rev().collect()
}
```

```dart
// Dart 调用
import 'package:my_app/src/rust/api/simple.dart';
import 'package:my_app/src/rust/frb_generated.dart';

void main() async {
  await RustLib.init();  // 初始化，只需调用一次
  runApp(const MyApp());
}

// 在组件中调用
final result = await add(a: 1, b: 2);           // 3
final greeting = await greet(name: "张三");      // "你好，张三！"
final doubled = await doubleIfPositive(value: 5); // 10
final reversed = await reverseBytes(data: Uint8List.fromList([1, 2, 3]));
```

### 异步函数

FRB v2 中，所有 Rust 函数在 Dart 侧都是异步的（返回 `Future`）。若 Rust 函数本身也是异步的，使用 `async`：

```rust
// rust/src/api/network.rs
use flutter_rust_bridge::frb;

/// 异步网络请求（使用 tokio）
pub async fn fetch_data(url: String) -> Result<String, String> {
    let body = reqwest::get(&url)
        .await
        .map_err(|e| e.to_string())?
        .text()
        .await
        .map_err(|e| e.to_string())?;
    Ok(body)
}

/// 异步文件读取
pub async fn read_file(path: String) -> Result<Vec<u8>, String> {
    tokio::fs::read(&path)
        .await
        .map_err(|e| e.to_string())
}
```

```rust
// Cargo.toml
[dependencies]
flutter_rust_bridge = "2"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
```

```dart
// Dart 调用
try {
  final html = await fetchData(url: 'https://example.com');
  print(html);
} catch (e) {
  print('请求失败：$e');
}
```

---

## 结构体（Struct）

### 基础结构体

```rust
// rust/src/api/models.rs
use flutter_rust_bridge::frb;

/// FRB 会为此结构体生成对应的 Dart 类
pub struct User {
    pub id: u32,
    pub name: String,
    pub email: Option<String>,
    pub age: u8,
    pub active: bool,
}

pub fn create_user(name: String, age: u8) -> User {
    User {
        id: rand::random(),
        name,
        email: None,
        age,
        active: true,
    }
}

pub fn get_users() -> Vec<User> {
    vec![
        User { id: 1, name: "张三".into(), email: Some("zhang@example.com".into()), age: 25, active: true },
        User { id: 2, name: "李四".into(), email: None, age: 30, active: false },
    ]
}
```

```dart
// 自动生成的 Dart 类（无需手写）
// class User {
//   final int id;
//   final String name;
//   final String? email;
//   final int age;
//   final bool active;
// }

final user = await createUser(name: "张三", age: 25);
print(user.name);    // "张三"
print(user.email);   // null

final users = await getUsers();
for (final u in users) {
  print('${u.id}: ${u.name}');
}
```

### 结构体方法（impl）

使用 `#[frb]` 属性暴露方法：

```rust
pub struct Counter {
    pub count: i32,
    pub step: i32,
}

impl Counter {
    /// 构造函数（Dart 侧使用 Counter.newWithStep()）
    #[frb(sync)]
    pub fn new_with_step(step: i32) -> Counter {
        Counter { count: 0, step }
    }

    /// 实例方法
    pub fn increment(&mut self) {
        self.count += self.step;
    }

    pub fn decrement(&mut self) {
        self.count -= self.step;
    }

    pub fn reset(&mut self) {
        self.count = 0;
    }

    #[frb(sync)]
    pub fn value(&self) -> i32 {
        self.count
    }
}
```

```dart
final counter = Counter.newWithStep(step: 5);
await counter.increment();
await counter.increment();
print(counter.value());  // 10
await counter.reset();
print(counter.value());  // 0
```

---

## 枚举（Enum）

### 简单枚举

```rust
pub enum Direction {
    North,
    South,
    East,
    West,
}

pub fn describe_direction(dir: Direction) -> String {
    match dir {
        Direction::North => "向北走".to_string(),
        Direction::South => "向南走".to_string(),
        Direction::East  => "向东走".to_string(),
        Direction::West  => "向西走".to_string(),
    }
}
```

```dart
final desc = await describeDirection(dir: Direction.north);
print(desc);  // "向北走"
```

### 带数据的枚举（ADT）

```rust
pub enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { base: f64, height: f64 },
}

pub fn area(shape: Shape) -> f64 {
    match shape {
        Shape::Circle { radius }            => std::f64::consts::PI * radius * radius,
        Shape::Rectangle { width, height }  => width * height,
        Shape::Triangle { base, height }    => 0.5 * base * height,
    }
}
```

```dart
final circleArea = await area(
  shape: Shape_Circle(radius: 5.0),
);
final rectArea = await area(
  shape: Shape_Rectangle(width: 4.0, height: 6.0),
);
print(circleArea);  // 78.53981633974483
```

---

## 错误处理

### anyhow::Result（推荐）

```rust
// Cargo.toml
// anyhow = "1"

use anyhow::Result;

pub fn parse_number(s: String) -> Result<i32> {
    let n = s.trim().parse::<i32>()?;
    if n < 0 {
        anyhow::bail!("数字不能为负数：{}", n);
    }
    Ok(n)
}

pub fn read_config(path: String) -> Result<String> {
    let content = std::fs::read_to_string(&path)
        .map_err(|e| anyhow::anyhow!("读取文件 {} 失败：{}", path, e))?;
    Ok(content)
}
```

```dart
// Rust 的 anyhow::Error 在 Dart 侧抛出为异常
try {
  final n = await parseNumber(s: "42");
  print(n);  // 42
} on AnyhowException catch (e) {
  print('解析失败：${e.message}');
}

// 或用 try-catch
try {
  final config = await readConfig(path: '/etc/app.conf');
} catch (e) {
  print(e);
}
```

### 自定义错误枚举

```rust
use flutter_rust_bridge::frb;

#[derive(Debug)]
pub enum AppError {
    NotFound { resource: String },
    PermissionDenied { action: String },
    NetworkError { message: String },
    ParseError { input: String, reason: String },
}

impl std::fmt::Display for AppError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AppError::NotFound { resource }         => write!(f, "未找到：{}", resource),
            AppError::PermissionDenied { action }   => write!(f, "无权限：{}", action),
            AppError::NetworkError { message }      => write!(f, "网络错误：{}", message),
            AppError::ParseError { input, reason }  => write!(f, "解析 '{}' 失败：{}", input, reason),
        }
    }
}

pub fn find_user(id: u32) -> Result<User, AppError> {
    if id == 0 {
        return Err(AppError::NotFound { resource: format!("用户 {}", id) });
    }
    Ok(User { id, name: "张三".into(), email: None, age: 25, active: true })
}
```

```dart
try {
  final user = await findUser(id: 0);
} on AppError_NotFound catch (e) {
  print('未找到：${e.resource}');
} on AppError_PermissionDenied catch (e) {
  print('无权限：${e.action}');
} on AppError catch (e) {
  print('其他错误：$e');
}
```

---

## 流（Stream）

`StreamSink` 让 Rust 向 Dart 持续推送数据，类似 Dart 的 `Stream`：

```rust
use flutter_rust_bridge::frb;

/// 进度推送
pub async fn download_file(
    url: String,
    sink: StreamSink<f64>,  // 推送下载进度 0.0 ~ 1.0
) -> Result<Vec<u8>, String> {
    let response = reqwest::get(&url).await.map_err(|e| e.to_string())?;
    let total = response.content_length().unwrap_or(0);
    let mut downloaded = 0u64;
    let mut bytes = Vec::new();

    let mut stream = response.bytes_stream();
    while let Some(chunk) = stream.next().await {
        let chunk = chunk.map_err(|e| e.to_string())?;
        downloaded += chunk.len() as u64;
        bytes.extend_from_slice(&chunk);

        if total > 0 {
            let progress = downloaded as f64 / total as f64;
            sink.add(progress);  // 推送进度
        }
    }
    Ok(bytes)
}

/// 持续推送传感器数据
pub fn subscribe_sensor(sink: StreamSink<SensorData>) {
    std::thread::spawn(move || {
        loop {
            let data = read_sensor();
            if sink.add(data).is_err() {
                break;  // Dart 侧关闭 Stream 时退出
            }
            std::thread::sleep(std::time::Duration::from_millis(100));
        }
    });
}

pub struct SensorData {
    pub timestamp: i64,
    pub temperature: f32,
    pub humidity: f32,
}
```

```dart
// 监听下载进度
final stream = downloadFile(url: 'https://example.com/file.zip');

stream.listen(
  (progress) => print('下载进度：${(progress * 100).toStringAsFixed(1)}%'),
  onError: (e) => print('下载失败：$e'),
  onDone: () => print('下载完成'),
);

// 传感器数据流
late StreamSubscription subscription;

@override
void initState() {
  super.initState();
  subscription = subscribeSensor().listen((data) {
    setState(() {
      temperature = data.temperature;
      humidity = data.humidity;
    });
  });
}

@override
void dispose() {
  subscription.cancel();  // 取消订阅，Rust 侧感知到后退出循环
  super.dispose();
}
```

---

## 不透明类型（Opaque Types）

当 Rust 类型不需要（或无法）直接映射到 Dart 时，使用 `#[frb(opaque)]` 将其作为不透明句柄传递：

```rust
use flutter_rust_bridge::frb;

/// 数据库连接（Dart 只持有句柄，不关心内部结构）
#[frb(opaque)]
pub struct Database {
    conn: sqlite::Connection,
    path: String,
}

impl Database {
    pub fn open(path: String) -> Result<Database, String> {
        let conn = sqlite::open(&path).map_err(|e| e.to_string())?;
        Ok(Database { conn, path })
    }

    pub fn execute(&self, sql: String) -> Result<(), String> {
        self.conn.execute(&sql).map_err(|e| e.to_string())
    }

    pub fn query_users(&self) -> Result<Vec<User>, String> {
        // 查询逻辑
        todo!()
    }

    pub fn close(self) {
        // 消耗 self，自动关闭连接
    }
}
```

```dart
// Dart 侧 Database 是一个不透明对象，无法访问内部字段
final db = await Database.open(path: '/data/app.db');

await db.execute(sql: 'CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY)');
final users = await db.queryUsers();

// 不再需要时释放
db.dispose();  // FRB 自动管理内存
```

---

## `#[frb]` 常用属性

```rust
use flutter_rust_bridge::frb;

/// 同步函数（无需 await，直接返回值）
/// 注意：只适合非阻塞的轻量操作
#[frb(sync)]
pub fn add_sync(a: i32, b: i32) -> i32 {
    a + b
}

/// 忽略此函数，不生成 Dart 绑定
#[frb(ignore)]
pub fn internal_helper() {}

/// 初始化函数（应用启动时自动调用）
#[frb(init)]
pub fn init_app() {
    // 初始化日志、全局状态等
    env_logger::init();
}

/// 指定 Dart 侧的函数名
#[frb(dart_code = "
  static Future<int> fromString(String s) => parseIntFromString(s: s);
")]
pub fn parse_int_from_string(s: String) -> Result<i32, String> {
    s.trim().parse().map_err(|e: std::num::ParseIntError| e.to_string())
}
```

```dart
// 同步调用（不需要 await）
final sum = addSync(a: 1, b: 2);  // 3

// 非同步调用
final sum2 = await add(a: 1, b: 2);  // 3
```

---

## 在 Rust 中调用 Dart 回调

```rust
use flutter_rust_bridge::DartFnFuture;

/// 接收 Dart 函数作为回调
pub async fn process_with_callback(
    data: Vec<u8>,
    on_progress: impl Fn(f64) -> DartFnFuture<()>,
) -> Vec<u8> {
    let total = data.len();
    let mut result = Vec::new();

    for (i, chunk) in data.chunks(1024).enumerate() {
        // 处理数据块
        result.extend_from_slice(chunk);

        let progress = (i + 1) as f64 / (total / 1024 + 1) as f64;
        on_progress(progress).await;  // 调用 Dart 回调
    }
    result
}
```

```dart
final result = await processWithCallback(
  data: largeData,
  onProgress: (progress) async {
    setState(() => _progress = progress);
  },
);
```

---

## 多线程与并发

```rust
use std::sync::{Arc, Mutex};
use flutter_rust_bridge::frb;

/// 线程安全的共享状态
#[frb(opaque)]
pub struct SharedState {
    inner: Arc<Mutex<StateInner>>,
}

struct StateInner {
    data: Vec<String>,
    count: u32,
}

impl SharedState {
    pub fn new() -> SharedState {
        SharedState {
            inner: Arc::new(Mutex::new(StateInner { data: vec![], count: 0 })),
        }
    }

    pub fn add(&self, item: String) {
        let mut inner = self.inner.lock().unwrap();
        inner.data.push(item);
        inner.count += 1;
    }

    pub fn get_count(&self) -> u32 {
        self.inner.lock().unwrap().count
    }

    pub fn get_all(&self) -> Vec<String> {
        self.inner.lock().unwrap().data.clone()
    }
}

/// CPU 密集型并行计算
pub async fn parallel_process(items: Vec<String>) -> Vec<String> {
    use rayon::prelude::*;

    // rayon 自动并行，不阻塞 Dart 线程
    tokio::task::spawn_blocking(move || {
        items.par_iter()
            .map(|s| s.to_uppercase())
            .collect()
    }).await.unwrap()
}
```

---

## 平台配置

### Android

```groovy
// android/app/build.gradle
android {
    defaultConfig {
        // 最低 API 21
        minSdkVersion 21
    }
}
```

```shell
# 构建 Android .so 库
cargo ndk -t armeabi-v7a -t arm64-v8a -t x86_64 -o ../android/app/src/main/jniLibs build --release
```

FRB 的 Gradle 插件会自动处理，通常无需手动运行上述命令，直接 `flutter run` 即可。

### iOS

```shell
# 构建 iOS 静态库
cargo lipo --release --targets aarch64-apple-ios x86_64-apple-ios

# 或使用 FRB 提供的脚本
flutter_rust_bridge_codegen build-web   # 仅 Web
```

```ruby
# ios/Podfile
platform :ios, '12.0'
```

### macOS Desktop

```shell
# 在 macOS 上直接运行
flutter run -d macos
```

### Web（实验性）

```shell
# 安装 wasm-pack
cargo install wasm-pack

# FRB v2 支持 Web，但需要额外配置
flutter_rust_bridge_codegen generate --web
```

---

## 完整示例：图片处理器

```rust
// rust/src/api/image_processor.rs
use flutter_rust_bridge::frb;
use image::{DynamicImage, ImageFormat};
use std::io::Cursor;

pub struct ImageInfo {
    pub width: u32,
    pub height: u32,
    pub format: String,
    pub size_bytes: usize,
}

/// 获取图片信息
pub fn get_image_info(bytes: Vec<u8>) -> Result<ImageInfo, String> {
    let img = image::load_from_memory(&bytes).map_err(|e| e.to_string())?;
    Ok(ImageInfo {
        width: img.width(),
        height: img.height(),
        format: "JPEG".to_string(),
        size_bytes: bytes.len(),
    })
}

/// 压缩图片
pub async fn compress_image(
    bytes: Vec<u8>,
    quality: u8,
    max_width: Option<u32>,
    sink: StreamSink<f32>,   // 推送进度
) -> Result<Vec<u8>, String> {
    tokio::task::spawn_blocking(move || {
        sink.add(0.1);
        let img = image::load_from_memory(&bytes).map_err(|e| e.to_string())?;
        sink.add(0.4);

        let img = if let Some(max_w) = max_width {
            if img.width() > max_w {
                img.resize(max_w, u32::MAX, image::imageops::FilterType::Lanczos3)
            } else {
                img
            }
        } else {
            img
        };

        sink.add(0.7);

        let mut output = Cursor::new(Vec::new());
        img.write_to(&mut output, ImageFormat::Jpeg).map_err(|e| e.to_string())?;
        sink.add(1.0);

        Ok(output.into_inner())
    }).await.map_err(|e| e.to_string())?
}

/// 生成缩略图
pub fn generate_thumbnail(bytes: Vec<u8>, size: u32) -> Result<Vec<u8>, String> {
    let img = image::load_from_memory(&bytes).map_err(|e| e.to_string())?;
    let thumb = img.thumbnail(size, size);

    let mut output = Cursor::new(Vec::new());
    thumb.write_to(&mut output, ImageFormat::Jpeg).map_err(|e| e.to_string())?;
    Ok(output.into_inner())
}
```

```dart
// lib/pages/image_page.dart
class ImageProcessorPage extends StatefulWidget { ... }

class _ImageProcessorPageState extends State<ImageProcessorPage> {
  double _progress = 0;
  Uint8List? _compressed;

  Future<void> _pickAndCompress() async {
    final picker = ImagePicker();
    final file = await picker.pickImage(source: ImageSource.gallery);
    if (file == null) return;

    final bytes = await file.readAsBytes();

    // 获取图片信息
    final info = await getImageInfo(bytes: bytes);
    print('${info.width}x${info.height}, ${info.sizeBytes} bytes');

    // 压缩（带进度）
    final stream = compressImage(
      bytes: bytes,
      quality: 80,
      maxWidth: 1080,
    );

    Uint8List? result;
    await for (final event in stream) {
      // StreamSink 的最后一个事件是 Result
      if (event is RustStreamEvent_Progress) {
        setState(() => _progress = event.field0);
      }
    }

    // 生成缩略图
    final thumb = await generateThumbnail(bytes: bytes, size: 200);
    setState(() => _compressed = Uint8List.fromList(thumb));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          LinearProgressIndicator(value: _progress),
          if (_compressed != null) Image.memory(_compressed!),
          ElevatedButton(onPressed: _pickAndCompress, child: const Text('选择并压缩')),
        ],
      ),
    );
  }
}
```

---

## 调试与常见问题

### 日志输出

```rust
// Cargo.toml
// log = "0.4"
// env_logger = "0.10"  (仅 Android/Desktop)
// oslog = "0.2"        (仅 iOS)

#[frb(init)]
pub fn init_app() {
    #[cfg(target_os = "android")]
    android_logger::init_once(
        android_logger::Config::default().with_min_level(log::Level::Debug),
    );

    #[cfg(target_os = "ios")]
    oslog::OsLogger::new("com.example.app").init().ok();

    log::info!("Rust 初始化完成");
}
```

### 常见报错

```
# 1. 找不到动态库
# 原因：未编译 Rust 或 .so/.dylib 文件路径不对
# 解决：flutter clean && flutter run

# 2. 代码生成失败
# 原因：Rust 代码有编译错误
# 解决：先 cargo build 确保 Rust 代码本身能编译

# 3. 类型不支持
# 原因：使用了 FRB 不支持的 Rust 类型
# 解决：用 #[frb(opaque)] 包装，或转换为支持的类型

# 4. 函数签名冲突
# 原因：两个模块有同名函数
# 解决：在 Dart 侧会自动加模块前缀，检查生成的代码
```

### 性能调优

```rust
// 避免不必要的数据拷贝：大块数据用引用
// 但注意 FRB 的生命周期限制，通常只能传 owned 值

// CPU 密集型任务放在 spawn_blocking 中
pub async fn heavy_computation(data: Vec<f64>) -> Vec<f64> {
    tokio::task::spawn_blocking(move || {
        data.iter().map(|x| x.sqrt()).collect()
    }).await.unwrap()
}

// 批量处理优于单次处理（减少 FFI 跨越次数）
// ✗ 在循环中逐个调用 Rust 函数
for item in items {
    process_item(item: item);
}
// ✓ 一次传入全部数据
process_items(items: items);
```

---

## 与 Platform Channel 对比

| | flutter_rust_bridge | Platform Channel |
|---|---|---|
| 语言 | Rust | Kotlin / Swift / Java / ObjC |
| 类型安全 | 自动生成，完全类型安全 | 手动维护，容易出错 |
| 性能 | 直接 FFI，几乎无开销 | 序列化开销较大 |
| 代码量 | 极少样板代码 | 需要大量胶水代码 |
| 调试 | Rust 工具链（lldb/gdb） | 原生 IDE |
| 跨平台 | 一份 Rust 代码全平台 | 每个平台单独实现 |
| 生态 | crates.io（丰富） | 各平台原生生态 |
| 适用场景 | 计算密集、跨平台逻辑 | 访问平台专有 API |
