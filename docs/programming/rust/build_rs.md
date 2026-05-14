# Rust build.rs

## 1. 什么是 build.rs

`build.rs` 是 Cargo 在**编译主代码之前**自动运行的构建脚本，可以：

- 检测目标平台、OS、CPU 架构
- 生成 Rust 源代码（proto、模板、绑定等）
- 链接本地 C/C++ 库
- 编译 C/C++ 源文件并静态链接
- 向编译器传递条件编译标志
- 嵌入版本号、构建时间等元信息

### 执行顺序

```
cargo build
    │
    ▼
编译 build.rs（作为独立二进制）
    │
    ▼
运行 build.rs 输出的 cargo: 指令
    │
    ▼
编译主代码（src/）
```

### 文件位置

```
my-project/
├── Cargo.toml
├── build.rs        ← 默认位置，无需配置
└── src/
    └── main.rs
```

自定义路径（Cargo.toml）：

```toml
[package]
name    = "my-project"
version = "0.1.0"
build   = "scripts/build.rs"   # 自定义 build.rs 路径
```

禁用构建脚本：

```toml
[package]
build = false
```

---

## 2. 基础用法

### 最小示例

```rust
// build.rs
fn main() {
    println!("cargo:warning=构建脚本正在运行");
}
```

### build.rs 的依赖声明

build.rs 的依赖在 `[build-dependencies]` 中声明，与主代码的依赖**完全独立**：

```toml
[build-dependencies]
cc          = "1"           # 编译 C/C++ 代码
tonic-build = "0.12"        # 生成 gRPC 代码
prost-build = "0.13"        # 生成 protobuf 代码
bindgen     = "0.69"        # 生成 C 绑定

[dependencies]
# 主代码依赖，build.rs 不能使用这些
tokio = { version = "1", features = ["full"] }
```

---

## 3. Cargo 指令（cargo:）

build.rs 通过向标准输出打印特定格式的指令与 Cargo 通信。

### 完整指令列表

#### cargo:rustc-link-lib — 链接库

```rust
// 链接动态库（默认）
println!("cargo:rustc-link-lib=ssl");          // -lssl
println!("cargo:rustc-link-lib=dylib=ssl");    // 显式动态库

// 链接静态库
println!("cargo:rustc-link-lib=static=mylib"); // -Bstatic -lmylib

// 链接框架（macOS）
println!("cargo:rustc-link-lib=framework=CoreFoundation");
```

#### cargo:rustc-link-search — 库搜索路径

```rust
// 添加库搜索路径
println!("cargo:rustc-link-search=/usr/local/lib");
println!("cargo:rustc-link-search=native=/path/to/libs");

// 路径类型
// native   → 本地库（默认）
// framework → macOS framework
// all      → 所有类型
println!("cargo:rustc-link-search=all=/opt/custom/lib");
```

#### cargo:rustc-cfg — 条件编译标志

```rust
// 设置 cfg 标志，主代码中用 #[cfg(feature_x)] 判断
println!("cargo:rustc-cfg=feature_x");
println!("cargo:rustc-cfg=has_avx2");

// 带值的 cfg
println!("cargo:rustc-cfg=database=\"postgres\"");
```

#### cargo:rustc-env — 设置编译期环境变量

```rust
// 主代码中用 env!("MY_VAR") 读取
println!("cargo:rustc-env=MY_VAR=hello");
println!("cargo:rustc-env=BUILD_TIME=2026-05-14T10:00:00Z");
println!("cargo:rustc-env=GIT_HASH=abc123def");
```

#### cargo:rerun-if-changed — 控制重新构建

```rust
// 只有这些文件变化时才重新运行 build.rs
println!("cargo:rerun-if-changed=build.rs");
println!("cargo:rerun-if-changed=proto/user.proto");
println!("cargo:rerun-if-changed=src/ffi/mylib.h");

// 监听整个目录（目录内任何文件变化都触发）
println!("cargo:rerun-if-changed=proto/");
```

#### cargo:rerun-if-env-changed — 环境变量变化时重新构建

```rust
// 指定环境变量变化时重新运行
println!("cargo:rerun-if-env-changed=CC");
println!("cargo:rerun-if-env-changed=MY_LIB_PATH");
```

#### cargo:warning — 输出警告

```rust
println!("cargo:warning=正在使用实验性功能");
println!("cargo:warning=找不到系统库，将使用内置版本");
```

#### cargo:rustc-flags — 传递链接器标志

```rust
println!("cargo:rustc-flags=-L /usr/local/lib -l ssl");
```

#### cargo:rustc-link-arg — 传递链接参数

```rust
// 传递给链接器的参数
println!("cargo:rustc-link-arg=-Wl,--gc-sections");
println!("cargo:rustc-link-arg=-Wl,-rpath,/usr/local/lib");
```

#### cargo:metadata — 自定义元数据（供父包脚本读取）

```rust
// 作为依赖库时，父包的 build.rs 可以通过 DEP_xxx_yyy 读取
println!("cargo:version=1.2.3");
println!("cargo:include=/path/to/headers");
```

---

## 4. 环境变量

Cargo 运行 build.rs 时会注入大量环境变量。

### 构建目标信息

```rust
fn main() {
    // 目标三元组，如 "x86_64-unknown-linux-gnu"
    let target = std::env::var("TARGET").unwrap();

    // 目标架构，如 "x86_64" / "aarch64" / "wasm32"
    let arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();

    // 目标操作系统，如 "linux" / "macos" / "windows" / "none"
    let os = std::env::var("CARGO_CFG_TARGET_OS").unwrap();

    // 目标环境，如 "gnu" / "msvc" / "musl"
    let env = std::env::var("CARGO_CFG_TARGET_ENV").unwrap();

    // 目标指针宽度，如 "64" / "32"
    let pointer_width = std::env::var("CARGO_CFG_TARGET_POINTER_WIDTH").unwrap();

    // 目标字节序，如 "little" / "big"
    let endian = std::env::var("CARGO_CFG_TARGET_ENDIAN").unwrap();

    // 目标操作系统系列，如 "unix" / "windows"
    let family = std::env::var("CARGO_CFG_TARGET_FAMILY").unwrap_or_default();

    println!("目标：{target}，架构：{arch}，OS：{os}");
}
```

### 构建模式信息

```rust
fn main() {
    // "debug" 或 "release"
    let profile = std::env::var("PROFILE").unwrap();

    // 优化级别："0"/"1"/"2"/"3"/"s"/"z"
    let opt_level = std::env::var("OPT_LEVEL").unwrap();

    // 是否开启 debug 信息："0"/"1"/"2"
    let debug = std::env::var("DEBUG").unwrap();

    // 并行编译的 job 数
    let num_jobs = std::env::var("NUM_JOBS").unwrap();

    if profile == "release" {
        println!("cargo:rustc-cfg=release_build");
    }
}
```

### 路径信息

```rust
fn main() {
    // Cargo 输出目录（生成文件放这里）
    let out_dir = std::env::var("OUT_DIR").unwrap();

    // 包的 Cargo.toml 所在目录
    let manifest_dir = std::env::var("CARGO_MANIFEST_DIR").unwrap();

    // 工作区根目录
    let workspace_dir = std::env::var("CARGO_WORKSPACE_DIR")
        .unwrap_or_else(|_| manifest_dir.clone());

    println!("OUT_DIR: {out_dir}");
    println!("MANIFEST_DIR: {manifest_dir}");
}
```

### 包信息

```rust
fn main() {
    let name    = std::env::var("CARGO_PKG_NAME").unwrap();
    let version = std::env::var("CARGO_PKG_VERSION").unwrap();
    let authors = std::env::var("CARGO_PKG_AUTHORS").unwrap();
    let desc    = std::env::var("CARGO_PKG_DESCRIPTION").unwrap();
    let repo    = std::env::var("CARGO_PKG_REPOSITORY").unwrap();

    // 版本各部分
    let major = std::env::var("CARGO_PKG_VERSION_MAJOR").unwrap();
    let minor = std::env::var("CARGO_PKG_VERSION_MINOR").unwrap();
    let patch = std::env::var("CARGO_PKG_VERSION_PATCH").unwrap();

    println!("cargo:rustc-env=PKG_VERSION={version}");
}
```

### Feature 标志

```rust
fn main() {
    // 检查是否启用了某个 feature
    // 格式：CARGO_FEATURE_<大写FEATURE名>（连字符替换为下划线）
    if std::env::var("CARGO_FEATURE_ASYNC").is_ok() {
        println!("cargo:rustc-cfg=has_async");
    }

    if std::env::var("CARGO_FEATURE_TLS").is_ok() {
        // 启用了 tls feature，链接 openssl
        println!("cargo:rustc-link-lib=ssl");
        println!("cargo:rustc-link-lib=crypto");
    }
}
```

### 依赖包元数据

```rust
fn main() {
    // 读取名为 "mylib" 的依赖包在其 build.rs 中发布的元数据
    // 对应：println!("cargo:version=1.2.3"); 在 mylib 的 build.rs 中
    if let Ok(ver) = std::env::var("DEP_MYLIB_VERSION") {
        println!("mylib 版本: {ver}");
    }
    if let Ok(inc) = std::env::var("DEP_MYLIB_INCLUDE") {
        println!("mylib 头文件路径: {inc}");
    }
}
```

---

## 5. 条件编译

### 根据平台生成不同代码

```rust
fn main() {
    let os   = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    let arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();

    match os.as_str() {
        "linux" => {
            println!("cargo:rustc-cfg=platform_linux");
            println!("cargo:rustc-link-lib=pthread");
        }
        "macos" => {
            println!("cargo:rustc-cfg=platform_macos");
            println!("cargo:rustc-link-lib=framework=CoreFoundation");
            println!("cargo:rustc-link-lib=framework=Security");
        }
        "windows" => {
            println!("cargo:rustc-cfg=platform_windows");
            println!("cargo:rustc-link-lib=ws2_32");
            println!("cargo:rustc-link-lib=userenv");
        }
        _ => {}
    }

    if arch == "x86_64" || arch == "aarch64" {
        println!("cargo:rustc-cfg=has_64bit");
    }
}
```

### 在主代码中使用

```rust
// src/main.rs

#[cfg(platform_linux)]
fn get_memory_info() -> u64 {
    // Linux 实现
    0
}

#[cfg(platform_macos)]
fn get_memory_info() -> u64 {
    // macOS 实现
    0
}

#[cfg(platform_windows)]
fn get_memory_info() -> u64 {
    // Windows 实现
    0
}

// 使用带值的 cfg
#[cfg(database = "postgres")]
fn connect() { /* postgres 实现 */ }

#[cfg(database = "sqlite")]
fn connect() { /* sqlite 实现 */ }
```

### 检测系统能力

```rust
use std::process::Command;

fn main() {
    // 检测是否支持 AVX2 指令集（x86_64）
    let arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap_or_default();
    if arch == "x86_64" {
        if is_avx2_available() {
            println!("cargo:rustc-cfg=has_avx2");
        }
    }

    // 检测系统库是否存在
    if pkg_config_exists("openssl") {
        println!("cargo:rustc-cfg=has_openssl");
    }
}

fn is_avx2_available() -> bool {
    // 编译一个小程序测试
    let test = "
        #include <immintrin.h>
        int main() { __m256i x = _mm256_setzero_si256(); return 0; }
    ";
    compile_test_c(test)
}

fn pkg_config_exists(lib: &str) -> bool {
    Command::new("pkg-config")
        .args(["--exists", lib])
        .status()
        .map(|s| s.success())
        .unwrap_or(false)
}

fn compile_test_c(_src: &str) -> bool {
    // 简化示例
    true
}
```

---

## 6. 链接本地 C/C++ 库

### 链接已安装的系统库

```rust
fn main() {
    // 链接系统中的 libssl 和 libcrypto
    println!("cargo:rustc-link-lib=ssl");
    println!("cargo:rustc-link-lib=crypto");

    // 如果库不在标准路径，需要指定搜索路径
    if let Ok(lib_dir) = std::env::var("OPENSSL_LIB_DIR") {
        println!("cargo:rustc-link-search=native={lib_dir}");
    }

    // 重新构建时机控制
    println!("cargo:rerun-if-env-changed=OPENSSL_LIB_DIR");
}
```

### 使用 pkg-config 自动探测

```toml
[build-dependencies]
pkg-config = "0.3"
```

```rust
fn main() {
    // 自动找到 openssl 的库路径和版本
    pkg_config::Config::new()
        .atleast_version("1.1")
        .probe("openssl")
        .unwrap();
    // 自动输出：
    // cargo:rustc-link-lib=ssl
    // cargo:rustc-link-lib=crypto
    // cargo:rustc-link-search=native=/usr/lib/x86_64-linux-gnu
    // cargo:include=/usr/include/openssl
}
```

### 链接静态库

```rust
use std::path::PathBuf;

fn main() {
    let lib_dir = PathBuf::from(
        std::env::var("MYLIB_DIR").unwrap_or_else(|_| "/usr/local".to_string())
    );

    // 静态链接
    println!(
        "cargo:rustc-link-search=native={}",
        lib_dir.join("lib").display()
    );
    println!("cargo:rustc-link-lib=static=mylib");

    // 静态库一般需要同时链接它依赖的库
    println!("cargo:rustc-link-lib=pthread");
    println!("cargo:rustc-link-lib=m");      // libm（数学库）
}
```

---

## 7. 用 cc crate 编译 C/C++ 代码

`cc` 是最常用的编译 C/C++ 源文件的工具。

```toml
[build-dependencies]
cc = "1"
```

### 编译单个 C 文件

```rust
fn main() {
    cc::Build::new()
        .file("src/ffi/helper.c")
        .compile("helper");
    // 自动输出：cargo:rustc-link-lib=static=helper
}
```

### 编译多个文件并配置

```rust
fn main() {
    cc::Build::new()
        // 源文件
        .files([
            "src/ffi/util.c",
            "src/ffi/codec.c",
            "src/ffi/parser.c",
        ])
        // 头文件搜索路径
        .include("src/ffi/include")
        .include("/usr/local/include")
        // 编译器标志
        .flag("-O2")
        .flag("-fPIC")
        .flag_if_supported("-Wno-unused-parameter")
        // 预处理宏定义
        .define("MY_FEATURE", "1")
        .define("VERSION", "\"1.2.3\"")
        // 是否显示编译警告
        .warnings(false)
        // 优化级别（继承 Cargo profile）
        .opt_level(2)
        .compile("mylib");
}
```

### 编译 C++ 代码

```rust
fn main() {
    cc::Build::new()
        .cpp(true)                  // 启用 C++ 模式
        .file("src/ffi/engine.cpp")
        .file("src/ffi/renderer.cpp")
        .flag_if_supported("-std=c++17")
        .flag_if_supported("-stdlib=libc++")  // macOS
        .compile("engine");

    // C++ 标准库链接
    let target_os = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    if target_os == "macos" {
        println!("cargo:rustc-link-lib=c++");
    } else {
        println!("cargo:rustc-link-lib=stdc++");
    }
}
```

### 平台差异处理

```rust
fn main() {
    let mut build = cc::Build::new();
    build.file("src/ffi/common.c");

    let target_os = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    match target_os.as_str() {
        "linux" => {
            build.file("src/ffi/platform_linux.c");
            build.define("PLATFORM_LINUX", None);
        }
        "macos" => {
            build.file("src/ffi/platform_macos.c");
            build.define("PLATFORM_MACOS", None);
        }
        "windows" => {
            build.file("src/ffi/platform_win.c");
            build.define("PLATFORM_WINDOWS", None);
        }
        _ => {}
    }

    build.compile("platform_lib");
}
```

---

## 8. 代码生成

### 生成 Rust 源文件

生成的文件必须放在 `OUT_DIR` 中，通过 `include!` 宏引入：

```rust
// build.rs
use std::fs;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    // 生成一个 Rust 文件
    let code = r#"
pub const VERSION: &str = "1.2.3";
pub const BUILD_TIME: &str = "2026-05-14T10:00:00Z";

pub fn platform() -> &'static str {
    env!("CARGO_CFG_TARGET_OS")
}
"#;

    fs::write(out_dir.join("generated.rs"), code).unwrap();

    println!("cargo:rerun-if-changed=build.rs");
}
```

```rust
// src/main.rs
include!(concat!(env!("OUT_DIR"), "/generated.rs"));

fn main() {
    println!("版本: {}", VERSION);
    println!("构建时间: {}", BUILD_TIME);
}
```

### 嵌入 Git 信息

```rust
// build.rs
use std::process::Command;

fn main() {
    // 获取 git commit hash
    let git_hash = Command::new("git")
        .args(["rev-parse", "--short", "HEAD"])
        .output()
        .ok()
        .and_then(|o| String::from_utf8(o.stdout).ok())
        .unwrap_or_else(|| "unknown".to_string());

    // 获取 git branch
    let git_branch = Command::new("git")
        .args(["rev-parse", "--abbrev-ref", "HEAD"])
        .output()
        .ok()
        .and_then(|o| String::from_utf8(o.stdout).ok())
        .unwrap_or_else(|| "unknown".to_string());

    println!("cargo:rustc-env=GIT_HASH={}", git_hash.trim());
    println!("cargo:rustc-env=GIT_BRANCH={}", git_branch.trim());

    // git 状态变化时重新构建
    println!("cargo:rerun-if-changed=.git/HEAD");
    println!("cargo:rerun-if-changed=.git/refs");
}
```

```rust
// src/main.rs
fn main() {
    println!("Git Hash:   {}", env!("GIT_HASH"));
    println!("Git Branch: {}", env!("GIT_BRANCH"));
}
```

### 从模板生成代码

```rust
// build.rs
use std::fs;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());
    let manifest_dir = PathBuf::from(std::env::var("CARGO_MANIFEST_DIR").unwrap());

    // 读取错误码配置文件（如 JSON/TOML）
    let codes_path = manifest_dir.join("src/error_codes.json");
    let codes_json = fs::read_to_string(&codes_path).unwrap();
    let codes: Vec<serde_json::Value> = serde_json::from_str(&codes_json).unwrap();

    // 生成错误码枚举
    let mut code = String::from("pub enum ErrorCode {\n");
    for item in &codes {
        let name  = item["name"].as_str().unwrap();
        let value = item["code"].as_u64().unwrap();
        code.push_str(&format!("    {} = {},\n", name, value));
    }
    code.push_str("}\n");

    fs::write(out_dir.join("error_codes.rs"), code).unwrap();

    println!("cargo:rerun-if-changed=src/error_codes.json");
    println!("cargo:rerun-if-changed=build.rs");
}
```

### 生成 FFI 绑定（bindgen）

```toml
[build-dependencies]
bindgen = "0.69"
```

```rust
// build.rs
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    // 从 C 头文件自动生成 Rust FFI 绑定
    bindgen::Builder::default()
        .header("src/ffi/mylib.h")
        // 只生成指定前缀的绑定
        .allowlist_function("mylib_.*")
        .allowlist_type("MyLib.*")
        .allowlist_var("MYLIB_.*")
        // 生成的类型派生这些 trait
        .derive_debug(true)
        .derive_default(true)
        // 输出文件
        .generate()
        .expect("bindgen 生成失败")
        .write_to_file(out_dir.join("bindings.rs"))
        .expect("写入绑定文件失败");

    println!("cargo:rerun-if-changed=src/ffi/mylib.h");
    println!("cargo:rustc-link-lib=mylib");
}
```

```rust
// src/ffi.rs
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]
#![allow(dead_code)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

---

## 9. 生成 Proto 文件（tonic / prost）

### 基础用法

```toml
[build-dependencies]
tonic-build = "0.12"
```

```rust
// build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/hello.proto")?;
    Ok(())
}
```

### 完整配置

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        // 是否生成服务端代码
        .build_server(true)
        // 是否生成客户端代码
        .build_client(true)
        // 为所有生成的消息添加额外 derive
        .type_attribute(".", "#[derive(serde::Serialize, serde::Deserialize)]")
        // 只为特定消息添加
        .type_attribute(
            "user.v1.User",
            "#[derive(Hash, Eq)]",
        )
        // 为字段添加属性
        .field_attribute(
            "user.v1.User.password",
            "#[serde(skip_serializing)]",
        )
        // 生成文件描述符（供 gRPC 反射使用）
        .file_descriptor_set_path(
            std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap())
                .join("proto_descriptor.bin"),
        )
        // 指定编译的 proto 文件和 include 路径
        .compile_protos(
            &[
                "proto/user/v1/user.proto",
                "proto/order/v1/order.proto",
            ],
            &["proto/"],
        )?;

    // 告诉 Cargo：proto 文件变化时重新构建
    println!("cargo:rerun-if-changed=proto/");

    Ok(())
}
```

### 引入生成的代码

```rust
// src/proto/mod.rs

// 方式一：tonic::include_proto! 宏（推荐）
pub mod user {
    tonic::include_proto!("user.v1");
}

// 方式二：手动 include!
pub mod order {
    include!(concat!(env!("OUT_DIR"), "/order.v1.rs"));
}

// 引入文件描述符（用于 gRPC 反射）
pub const FILE_DESCRIPTOR_SET: &[u8] =
    include_bytes!(concat!(env!("OUT_DIR"), "/proto_descriptor.bin"));
```

---

## 10. 读取外部文件与目录

### 正确构造文件路径

```rust
use std::path::PathBuf;
use std::fs;

fn main() {
    let manifest_dir = PathBuf::from(std::env::var("CARGO_MANIFEST_DIR").unwrap());
    let out_dir      = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    // 读取项目根目录的文件
    let config_path = manifest_dir.join("config/default.toml");
    let config = fs::read_to_string(&config_path)
        .expect("读取配置文件失败");

    // 写入生成文件到 OUT_DIR
    let out_path = out_dir.join("config_embed.rs");
    let code = format!(
        r#"pub const DEFAULT_CONFIG: &str = r#"{}"#;"#,
        config
    );
    fs::write(out_path, code).unwrap();

    println!("cargo:rerun-if-changed={}", config_path.display());
}
```

### 遍历目录生成代码

```rust
use std::fs;
use std::path::PathBuf;

fn main() {
    let manifest_dir = PathBuf::from(std::env::var("CARGO_MANIFEST_DIR").unwrap());
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    let sql_dir = manifest_dir.join("migrations");
    let mut migrations = vec![];

    // 遍历 SQL 文件
    for entry in fs::read_dir(&sql_dir).unwrap() {
        let entry = entry.unwrap();
        let path = entry.path();
        if path.extension().and_then(|e| e.to_str()) == Some("sql") {
            let name = path.file_stem().unwrap().to_str().unwrap().to_string();
            let sql  = fs::read_to_string(&path).unwrap();
            migrations.push((name, sql));
            println!("cargo:rerun-if-changed={}", path.display());
        }
    }

    // 按文件名排序
    migrations.sort_by(|a, b| a.0.cmp(&b.0));

    // 生成迁移数组
    let mut code = String::from(
        "pub static MIGRATIONS: &[(&str, &str)] = &[\n"
    );
    for (name, sql) in &migrations {
        code.push_str(&format!(
            "    ({:?}, {:?}),\n",
            name, sql
        ));
    }
    code.push_str("];\n");

    fs::write(out_dir.join("migrations.rs"), code).unwrap();
    println!("cargo:rerun-if-changed={}", sql_dir.display());
}
```

---

## 11. 缓存与重新构建控制

### 默认行为

如果 build.rs **没有输出任何 `rerun-if-changed` 指令**，Cargo 会在以下情况重新运行：
- 任何源文件变化
- 任何环境变量变化
- 每次 `cargo build`（某些版本）

这会导致频繁的不必要重新构建。

### 精确控制重新构建

```rust
fn main() {
    // ✓ 明确声明监听的文件
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=proto/user.proto");
    println!("cargo:rerun-if-changed=src/ffi/mylib.h");
    println!("cargo:rerun-if-changed=src/ffi/helper.c");

    // ✓ 监听环境变量
    println!("cargo:rerun-if-env-changed=OPENSSL_DIR");
    println!("cargo:rerun-if-env-changed=PKG_CONFIG_PATH");

    // 其他构建逻辑...
}
```

### 只监听目录（目录内任何文件变化都触发）

```rust
fn main() {
    // 监听整个 proto 目录
    println!("cargo:rerun-if-changed=proto/");
    println!("cargo:rerun-if-changed=migrations/");

    // 注意：目录路径末尾加 / 监听的是目录内容
    // 不加 / 监听的是目录本身的元数据
}
```

### 缓存生成结果（避免重复生成）

```rust
use std::path::PathBuf;
use std::fs;

fn main() {
    let out_dir = PathBuf::from(std::env::var("OUT_DIR").unwrap());
    let output  = out_dir.join("generated.rs");

    // 检查输入文件的修改时间
    let input_modified = fs::metadata("src/schema.json")
        .and_then(|m| m.modified())
        .ok();
    let output_modified = fs::metadata(&output)
        .and_then(|m| m.modified())
        .ok();

    let needs_regen = match (input_modified, output_modified) {
        (Some(i), Some(o)) => i > o,
        _ => true,
    };

    if needs_regen {
        generate_code(&output);
    }

    println!("cargo:rerun-if-changed=src/schema.json");
    println!("cargo:rerun-if-changed=build.rs");
}

fn generate_code(output: &PathBuf) {
    // 生成代码逻辑...
    fs::write(output, "// 生成的代码").unwrap();
}
```

---

## 12. 与主代码通信

### 通过环境变量传递（编译期常量）

```rust
// build.rs
fn main() {
    println!("cargo:rustc-env=APP_VERSION=1.2.3");
    println!("cargo:rustc-env=BUILD_PROFILE={}", 
        std::env::var("PROFILE").unwrap());
}
```

```rust
// src/main.rs
const APP_VERSION: &str = env!("APP_VERSION");
const BUILD_PROFILE: &str = env!("BUILD_PROFILE");

fn main() {
    println!("版本: {}", APP_VERSION);
    // 运行时也可用 option_env! 处理可能不存在的变量
    let git_hash = option_env!("GIT_HASH").unwrap_or("unknown");
    println!("Git: {}", git_hash);
}
```

### 通过 include! 传递代码

```rust
// build.rs
use std::path::PathBuf;

fn main() {
    let out = PathBuf::from(std::env::var("OUT_DIR").unwrap());

    std::fs::write(
        out.join("constants.rs"),
        format!(
            "pub const MAX_CONNECTIONS: usize = {};\n\
             pub const SERVER_NAME: &str = {:?};\n",
            num_cpus(),
            hostname(),
        ),
    ).unwrap();
}

fn num_cpus() -> usize {
    std::thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(4)
}

fn hostname() -> String {
    std::env::var("HOSTNAME").unwrap_or_else(|_| "localhost".to_string())
}
```

```rust
// src/config.rs
include!(concat!(env!("OUT_DIR"), "/constants.rs"));
```

### 通过 cfg 传递布尔标志

```rust
// build.rs
fn main() {
    // 检测是否有 AVX2 支持
    println!("cargo:rustc-cfg=simd_avx2");
    
    // 检测操作系统特性
    let os = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    if os == "linux" {
        println!("cargo:rustc-cfg=has_epoll");
    }
}
```

```rust
// src/io.rs
#[cfg(has_epoll)]
mod epoll_impl { /* ... */ }

#[cfg(not(has_epoll))]
mod fallback_impl { /* ... */ }

#[cfg(simd_avx2)]
fn fast_hash(data: &[u8]) -> u64 {
    // AVX2 加速版本
    todo!()
}
```

---

## 13. 实战示例

### 示例一：完整的 gRPC 服务构建脚本

```rust
// build.rs
use std::path::PathBuf;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let out_dir = PathBuf::from(std::env::var("OUT_DIR")?);

    // 生成 proto 代码 + 文件描述符
    tonic_build::configure()
        .build_server(true)
        .build_client(true)
        .file_descriptor_set_path(out_dir.join("grpc_descriptor.bin"))
        .type_attribute(
            ".",
            "#[derive(serde::Serialize, serde::Deserialize)]",
        )
        .compile_protos(
            &[
                "proto/user/v1/user_service.proto",
                "proto/health/v1/health.proto",
            ],
            &["proto/"],
        )?;

    // 嵌入 git 信息
    embed_git_info();

    // 嵌入构建时间
    let build_time = chrono::Utc::now().to_rfc3339();
    println!("cargo:rustc-env=BUILD_TIME={build_time}");

    // 监听文件变化
    println!("cargo:rerun-if-changed=proto/");
    println!("cargo:rerun-if-changed=build.rs");
    println!("cargo:rerun-if-changed=.git/HEAD");

    Ok(())
}

fn embed_git_info() {
    let hash = run_git(&["rev-parse", "--short", "HEAD"])
        .unwrap_or_else(|| "unknown".to_string());
    let branch = run_git(&["rev-parse", "--abbrev-ref", "HEAD"])
        .unwrap_or_else(|| "unknown".to_string());
    let dirty = run_git(&["status", "--porcelain"])
        .map(|s| !s.trim().is_empty())
        .unwrap_or(false);

    println!("cargo:rustc-env=GIT_HASH={}", hash.trim());
    println!("cargo:rustc-env=GIT_BRANCH={}", branch.trim());
    println!("cargo:rustc-env=GIT_DIRTY={}", dirty);
}

fn run_git(args: &[&str]) -> Option<String> {
    std::process::Command::new("git")
        .args(args)
        .output()
        .ok()
        .filter(|o| o.status.success())
        .and_then(|o| String::from_utf8(o.stdout).ok())
}
```

### 示例二：混合 C 代码的 Rust 库

```rust
// build.rs
fn main() {
    let target_os   = std::env::var("CARGO_CFG_TARGET_OS").unwrap();
    let target_arch = std::env::var("CARGO_CFG_TARGET_ARCH").unwrap();

    // 编译核心 C 代码
    let mut build = cc::Build::new();
    build
        .file("native/core.c")
        .file("native/codec.c")
        .include("native/include")
        .define("RUST_FFI", "1");

    // 平台特定代码
    match target_os.as_str() {
        "linux" => {
            build.file("native/platform/linux.c");
            println!("cargo:rustc-link-lib=pthread");
            println!("cargo:rustc-link-lib=rt");
        }
        "macos" => {
            build.file("native/platform/macos.c");
            println!("cargo:rustc-link-lib=framework=CoreFoundation");
        }
        "windows" => {
            build.file("native/platform/windows.c");
            println!("cargo:rustc-link-lib=ws2_32");
        }
        _ => {}
    }

    // 架构优化
    if target_arch == "x86_64" {
        build.file("native/arch/x86_64.c");
        build.flag_if_supported("-mavx2");
        println!("cargo:rustc-cfg=has_x86_64_opt");
    } else if target_arch == "aarch64" {
        build.file("native/arch/aarch64.c");
        println!("cargo:rustc-cfg=has_aarch64_opt");
    }

    build.compile("mycore");

    // 生成 FFI 绑定
    let out = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindgen::Builder::default()
        .header("native/include/mycore.h")
        .allowlist_function("mycore_.*")
        .generate()
        .unwrap()
        .write_to_file(out.join("mycore_bindings.rs"))
        .unwrap();

    println!("cargo:rerun-if-changed=native/");
    println!("cargo:rerun-if-changed=build.rs");
}
```

### 示例三：嵌入式开发（no_std）构建脚本

```rust
// build.rs（适用于裸机/嵌入式）
use std::path::PathBuf;

fn main() {
    let manifest_dir = PathBuf::from(std::env::var("CARGO_MANIFEST_DIR").unwrap());

    // 链接脚本（告诉链接器内存布局）
    println!(
        "cargo:rustc-link-search={}",
        manifest_dir.display()
    );
    println!("cargo:rustc-link-arg=-Tmemory.x");
    println!("cargo:rustc-link-arg=-Tlink.x");

    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rerun-if-changed=link.x");
    println!("cargo:rerun-if-changed=build.rs");
}
```

---

## 14. 调试 build.rs

### 查看 build.rs 的输出

```bash
# 显示构建脚本的输出（包括 cargo: 指令和 stderr）
cargo build -vv 2>&1 | grep -A 50 "Running.*build-script"

# 只查看警告
cargo build 2>&1 | grep "warning:"
```

### 在 build.rs 中打印调试信息

```rust
fn main() {
    // 警告会显示在 cargo build 输出中
    println!("cargo:warning=调试: OUT_DIR = {}",
        std::env::var("OUT_DIR").unwrap_or_default());

    // 普通 eprintln! 输出到 stderr（-vv 模式下可见）
    eprintln!("[build.rs] 正在处理...");
}
```

### 查看生成的文件

```bash
# 找到 OUT_DIR 路径
cargo build -vv 2>&1 | grep "OUT_DIR"
# 输出类似：OUT_DIR = /path/to/target/debug/build/myapp-xxx/out

# 查看生成的文件
ls target/debug/build/myapp-*/out/
```

### 临时禁用 build.rs 缓存强制重新构建

```bash
# 删除 build 脚本的 fingerprint，强制重新运行
cargo clean -p mypackage
cargo build

# 或者修改 build.rs 的任意内容（触发重新编译）
touch build.rs && cargo build
```

### 常用调试模式

```rust
fn main() {
    // 将所有环境变量打印出来（仅调试时使用）
    if std::env::var("BUILD_DEBUG").is_ok() {
        for (key, val) in std::env::vars() {
            if key.starts_with("CARGO") || key.starts_with("TARGET") {
                eprintln!("[build.rs] {key} = {val}");
            }
        }
    }
}
```

```bash
BUILD_DEBUG=1 cargo build -vv
```

---

## 15. 常见陷阱

### 陷阱 1：路径使用相对路径

```rust
// ❌ 相对路径在 build.rs 中不可靠（工作目录不固定）
let content = std::fs::read_to_string("config.toml").unwrap();

// ✓ 使用 CARGO_MANIFEST_DIR 构造绝对路径
let manifest = std::env::var("CARGO_MANIFEST_DIR").unwrap();
let content = std::fs::read_to_string(
    std::path::Path::new(&manifest).join("config.toml")
).unwrap();
```

### 陷阱 2：生成文件放在 src/ 目录

```rust
// ❌ 不要将生成文件放在 src/ 或项目其他目录（污染源码树，影响 git）
std::fs::write("src/generated.rs", code).unwrap();

// ✓ 必须放在 OUT_DIR
let out = std::env::var("OUT_DIR").unwrap();
std::fs::write(format!("{out}/generated.rs"), code).unwrap();
```

### 陷阱 3：忘记声明 rerun-if-changed

```rust
// ❌ 没有声明：每次 cargo build 都会重新运行（很慢）
fn main() {
    tonic_build::compile_protos("proto/hello.proto").unwrap();
}

// ✓ 明确声明监听的文件
fn main() {
    tonic_build::compile_protos("proto/hello.proto").unwrap();
    println!("cargo:rerun-if-changed=proto/hello.proto");
    println!("cargo:rerun-if-changed=build.rs");
}
```

### 陷阱 4：build.rs 中使用 [dependencies] 中的 crate

```toml
# ❌ build.rs 不能使用 [dependencies] 中的 crate
[dependencies]
serde = "1"

# ✓ build.rs 专用依赖放在 [build-dependencies]
[build-dependencies]
serde = "1"
serde_json = "1"
```

### 陷阱 5：panic 导致构建失败时信息不清晰

```rust
// ❌ unwrap 在 build.rs 中 panic 时错误信息难以定位
let val = std::env::var("MY_VAR").unwrap();

// ✓ 提供清晰的错误信息
let val = std::env::var("MY_VAR")
    .expect("请设置环境变量 MY_VAR，例如：export MY_VAR=/path/to/lib");

// 或者返回 Result
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let val = std::env::var("MY_VAR")
        .map_err(|_| "缺少环境变量 MY_VAR")?;
    Ok(())
}
```

### 陷阱 6：Windows 路径反斜杠问题

```rust
// ❌ 在 Windows 上反斜杠可能引起问题
println!("cargo:rustc-link-search=C:\\libs\\mylib");

// ✓ 使用 Path 自动处理路径分隔符
let path = std::path::Path::new("C:\\libs\\mylib");
println!("cargo:rustc-link-search={}", path.display());

// 或者直接用正斜杠（Windows 链接器通常也接受）
println!("cargo:rustc-link-search=native=/usr/local/lib");
```

---

## 快速参考

### 常用指令速查

```rust
// 链接库
println!("cargo:rustc-link-lib=ssl");
println!("cargo:rustc-link-lib=static=mylib");

// 库搜索路径
println!("cargo:rustc-link-search=native=/usr/local/lib");

// 条件编译标志
println!("cargo:rustc-cfg=has_feature_x");

// 编译期环境变量
println!("cargo:rustc-env=MY_KEY=value");

// 重新构建控制
println!("cargo:rerun-if-changed=build.rs");
println!("cargo:rerun-if-env-changed=MY_ENV");

// 警告输出
println!("cargo:warning=提示信息");
```

### 常用环境变量速查

```rust
std::env::var("OUT_DIR")              // 生成文件输出目录
std::env::var("CARGO_MANIFEST_DIR")   // Cargo.toml 所在目录
std::env::var("PROFILE")              // "debug" / "release"
std::env::var("TARGET")               // 目标三元组
std::env::var("CARGO_CFG_TARGET_OS")  // 目标 OS
std::env::var("CARGO_CFG_TARGET_ARCH")// 目标架构
std::env::var("CARGO_PKG_VERSION")    // 包版本号
std::env::var("CARGO_FEATURE_<NAME>") // Feature 是否启用
```

