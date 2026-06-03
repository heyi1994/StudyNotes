# AOSP构建 Rust 模块

> 参考：[Android 官方文档 — Building Rust Modules](https://source.android.com/docs/setup/build/rust/building-rust-modules/overview?hl=zh-cn)
>
> 本文档面向在 **AOSP（Android Open Source Project）平台代码**中编写 Rust 的开发者，介绍如何使用 Soong 构建系统（`Android.bp`）来声明、编译、测试 Rust 模块。
>
> 注意：这里说的是**平台层 Rust**（系统服务、HAL、守护进程、库等），构建工具是 Soong / `Android.bp`，**不是** Cargo。Android 平台不使用 `Cargo.toml` 进行平台内构建（外部 crate 通过 `cargo_embargo` 转换成 `Android.bp`）。

---

## 1. 为什么在 Android 中使用 Rust

Android 平台历史上大量使用 C/C++ 编写底层代码，但内存安全漏洞（缓冲区溢出、释放后使用 UAF、空指针解引用等）长期占据 Android 高危安全漏洞的大头。Rust 在保持 **可媲美 C/C++ 性能**（无 GC、零成本抽象）的同时，提供了**编译期内存安全保证**。

Rust 的关键特性：

| 特性                   | 说明                                                                   |
| ---------------------- | ---------------------------------------------------------------------- |
| **内存安全**           | 借用检查器（borrow checker）在编译期消除大部分 UAF、悬垂指针、数据竞争 |
| **安全的并发**         | 类型系统保证线程间数据共享安全（`Send` / `Sync`）                      |
| **表现力强的类型系统** | 枚举、模式匹配、trait、泛型，便于表达不变量                            |
| **编译时检查**         | 大量错误在编译期暴露，而非运行时                                       |
| **内置测试框架**       | `#[test]` 无需第三方框架即可写单元测试                                 |
| **强制错误处理**       | `Result` / `Option`，配合 `?` 运算符显式处理错误                       |
| **初始化要求**         | 变量必须初始化后才能使用                                               |
| **整数处理安全**       | 默认 debug 下整数溢出 panic，release 下回绕但有显式 API                |

---

## 2. Soong 构建系统与 Android.bp 基础

AOSP 使用 **Soong** 构建系统，模块定义写在名为 `Android.bp` 的蓝图（Blueprint）文件中。

`Android.bp` 是一种**声明式**配置文件，语法类似 JSON，但带模块类型：

```bp
// 注释使用 // 或 /* */
模块类型 {
    属性名: 值,
    列表属性: ["a", "b", "c"],
    布尔属性: true,
}
```

特点：

- **没有条件语句、没有循环**（保持可解析、可静态分析）。复杂逻辑通过 `defaults`、`soong_config_variables` 实现。
- 每个目录可以有一个 `Android.bp`，描述该目录下的所有模块。
- 模块名（`name`）在整个构建中**全局唯一**。

---

## 3. Rust 模块类型总览

| 模块类型             | 用途                                       | 输出                |
| -------------------- | ------------------------------------------ | ------------------- |
| `rust_binary`        | 设备上的可执行文件                         | 可执行二进制        |
| `rust_binary_host`   | 主机（构建机器）上运行的可执行文件         | host 可执行二进制   |
| `rust_library`       | Rust 库（**推荐**，同时产出 rlib + dylib） | `.rlib` 和 `.dylib` |
| `rust_library_rlib`  | 仅产出 rlib 静态变体                       | `.rlib`             |
| `rust_library_dylib` | 仅产出 dylib 动态变体                      | `.dylib`            |
| `rust_library_host`  | 主机库                                     | host 库             |
| `rust_ffi`           | 供 C/C++ 调用的库（产出 static + shared）  | `.a` 和 `.so`       |
| `rust_ffi_shared`    | 仅产出 C 共享库                            | `.so`               |
| `rust_ffi_static`    | 仅产出 C 静态库                            | `.a`                |
| `rust_proc_macro`    | 过程宏（编译期运行的宏）                   | proc-macro crate    |
| `rust_test`          | 使用 Rust 测试框架的测试                   | 测试二进制          |
| `rust_test_host`     | 主机端测试                                 | host 测试二进制     |
| `rust_fuzz`          | 基于 libFuzzer 的模糊测试                  | fuzzer 二进制       |
| `rust_bindgen`       | 由 C/C++ 头文件自动生成 Rust FFI 绑定      | 生成的 Rust 源      |
| `rust_protobuf`      | 由 `.proto` 生成 Rust 代码                 | 生成的 Rust 源      |
| `rust_defaults`      | 复用属性的“模板”（本身不产出任何东西）     | 无                  |

---

## 4. 通用属性（所有 Rust 模块共享）

以下属性几乎适用于所有 Rust 模块：

| 属性                 | 类型       | 说明                                                                                                       |
| -------------------- | ---------- | ---------------------------------------------------------------------------------------------------------- |
| `name`               | string     | **必填**，模块全局唯一标识符                                                                               |
| `srcs`               | 字符串列表 | 入口源文件。通常二进制是 `main.rs`，库是 `lib.rs`。**只列入口文件**，其余模块由 Rust 自身的 `mod` 系统发现 |
| `crate_name`         | string     | crate 名称，**对生成库的模块必填**，必须与源码中 `use` 的名称一致                                          |
| `edition`            | string     | Rust 版本，如 `"2015"`、`"2018"`、`"2021"`。默认为 `2018`                                                  |
| `rustlibs`           | 列表       | **首选**的 Rust 库依赖。让 Soong 自动选择 rlib/dylib 关联方式                                              |
| `rlibs`              | 列表       | 强制以 rlib 形式链接的 Rust 依赖                                                                           |
| `shared_libs`        | 列表       | 依赖的 C/C++ 共享库（`.so`）                                                                               |
| `static_libs`        | 列表       | 依赖的 C/C++ 静态库（`.a`）                                                                                |
| `proc_macros`        | 列表       | 依赖的 `rust_proc_macro` 模块                                                                              |
| `features`           | 列表       | 启用的 Cargo features（传给 `--cfg feature="..."`）                                                        |
| `cfgs`               | 列表       | 额外的 `--cfg` 编译标志                                                                                    |
| `flags`              | 列表       | 传给 `rustc` 的额外标志                                                                                    |
| `ld_flags`           | 列表       | 传给链接器的额外标志                                                                                       |
| `lints`              | string     | lint 集合（`"default"` / `"android"` / `"vendor"` / `"none"`）                                             |
| `clippy_lints`       | string     | Clippy lint 集合                                                                                           |
| `apex_available`     | 列表       | 该模块可被哪些 APEX 包含                                                                                   |
| `vendor` / `product` | bool       | 是否构建到 vendor / product 分区                                                                           |
| `host_supported`     | bool       | 是否同时支持 host 构建                                                                                     |
| `defaults`           | 列表       | 引用的 `rust_defaults` 模块                                                                                |

> **关于 `srcs`**：和 C/C++ 不同，Rust 模块通常**只需列出 crate 根文件**（`lib.rs` 或 `main.rs`）。编译器会顺着 `mod xxx;` 声明找到其余文件。

---

## 5. 二进制模块 rust_binary

生成在设备上运行的可执行文件。

### 基础示例

```bp
rust_binary {
    name: "hello_rust",
    srcs: ["src/hello_rust.rs"],
}
```

主机版本：

```bp
rust_binary_host {
    name: "hello_rust_host",
    srcs: ["src/hello_rust.rs"],
}
```

### 关键属性

| 属性                | 说明                                                                                                                                     |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `static_executable` | 生成完全静态的二进制，并自动把 `prefer_rlib` 设为 `true`。**完全静态仅对 Bionic 目标有效**；非 Bionic 目标仍会动态链接 `libc` 和 `libdl` |
| `prefer_rlib`       | 改变 rustlib 的链接方式，为设备目标优先使用 `rlib`，并以 `rlib` 形式链接 `libstd`。这是 **host 目标的默认行为**                          |

### 链接行为

- **设备目标（device）**：默认**动态链接 `libstd`**，并优先从 `rustlibs` 选择 `dylib` 变体（除非启用 `prefer_rlib`）。需要完全静态时用 `static_executable`。
- **主机目标（host）**：默认**静态链接 `libstd`**，并优先选择 `rlib` 变体。

---

## 6. 库模块 rust_library 与变体

### rust_library（推荐）

同时产出两个变体：`rlib`（静态 Rust 库）和 `dylib`（动态 Rust 库）。这是 AOSP 中编写 Rust 库的**推荐方式**，因为它能同时满足下游对两种链接方式的需求。

```bp
rust_library {
    name: "libfoo",
    crate_name: "foo",
    srcs: ["src/lib.rs"],
    rustlibs: ["libbar"],
}
```

> 注意：`name` 通常是 `lib<crate_name>`，而 `crate_name` 是不带 `lib` 前缀的 `foo`。源码里 `use foo::...;`。

### 单变体版本

| 模块                 | 产出       | 注意                                             |
| -------------------- | ---------- | ------------------------------------------------ |
| `rust_library_rlib`  | 仅 `rlib`  | 单变体可能无法保证与依赖方的 `rustlibs` 属性兼容 |
| `rust_library_dylib` | 仅 `dylib` | 同上                                             |

一般**优先用 `rust_library`**，把变体选择权交给构建系统，除非你明确知道只需要某一种。

### 关键属性

| 属性         | 说明                                                                        |
| ------------ | --------------------------------------------------------------------------- |
| `crate_name` | **必填**。建立 crate 名与输出文件名的关系                                   |
| `stem`       | 控制输出库文件名。必须遵循 Rust 编译器要求的 `lib<crate_name><suffix>` 格式 |

### 链接行为

- 设备上的 Rust 库**默认动态链接 `libstd`**，host 模块**默认静态链接**。
- 依赖的链接方式跟随**根模块（顶层二进制/库）**的链接偏好。

---

## 7. FFI 库 rust_ffi（与 C/C++ 互操作）

当你需要让 **C/C++ 代码调用 Rust**（或反过来）时，使用 `rust_ffi` 系列。它们产出符合 C ABI 的库，可被 Soong 的 `cc_*` 模块作为 `shared_libs` / `static_libs` 依赖。

| 模块              | 产出                                          |
| ----------------- | --------------------------------------------- |
| `rust_ffi`        | C 静态库（`.a`）+ C 共享库（`.so`），两种变体 |
| `rust_ffi_shared` | 仅 C 共享库（`.so`）                          |
| `rust_ffi_static` | 仅 C 静态库（`.a`）                           |

```bp
rust_ffi {
    name: "libfoo_ffi",
    crate_name: "foo_ffi",
    srcs: ["src/lib.rs"],
    // 让依赖此库的 cc 模块能找到对应头文件
    export_include_dirs: ["include"],
}
```

### 关键属性

| 属性                  | 说明                                       |
| --------------------- | ------------------------------------------ |
| `crate_name`          | **必填**                                   |
| `stem`                | 控制输出文件名（`lib<crate_name>` 格式）   |
| `export_include_dirs` | 提供给依赖方 `cc` 模块的头文件相对路径目录 |

Rust 源代码侧需要用 `#[no_mangle]` 和 `extern "C"` 暴露 C ABI 函数：

```rust
#[no_mangle]
pub extern "C" fn foo_add(a: i32, b: i32) -> i32 {
    a + b
}
```

---

## 8. 过程宏 rust_proc_macro

过程宏（procedural macro）是在**编译期**运行、操作 token 流的特殊 crate（如自定义 `#[derive(...)]`）。

```bp
rust_proc_macro {
    name: "libfoo_macros",
    crate_name: "foo_macros",
    srcs: ["src/lib.rs"],
}
```

在使用方模块中通过 `proc_macros` 属性引用（**不是** `rustlibs`）：

```bp
rust_library {
    name: "libfoo",
    crate_name: "foo",
    srcs: ["src/lib.rs"],
    proc_macros: ["libfoo_macros"],
}
```

---

## 9. 测试模块 rust_test

`rust_test` 使用 Rust 内置测试框架构建自动化测试。它依靠 rustc 的 `--test` 标志，把标有 `#[test]` 属性的函数编译为测试。

### 基础示例

```bp
rust_test {
    name: "libfoo_inline_tests",
    srcs: ["src/lib.rs"],
    test_suites: ["general-tests"],
    auto_gen_config: true,
    rustlibs: ["libfoo"],
}
```

### 关键属性

| 属性              | 说明                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `test_harness`    | 是否启用测试框架，默认 `true`。设为 `false` 时不向 rustc 传 `--test`，用于自定义测试主程序 |
| `test_suites`     | 该测试归属的测试套件，如 `"general-tests"`、`"device-tests"`                               |
| `auto_gen_config` | 是否自动生成测试配置（`AndroidTest.xml`），通常设 `true`                                   |

`rust_test` **继承自 `rust_binary`**，因此 `rust_binary` 的属性以及所有通用属性都可用。

### TEST_MAPPING（可选，用于 pre-commit / 提交前测试）

在目录下放一个 `TEST_MAPPING` 文件，把测试加入 presubmit，可在代码合入前自动运行：

```json
{
  "presubmit": [
    {
      "name": "libfoo_inline_tests"
    }
  ]
}
```

> `rust_test_host` 模块默认会在提交前运行单元测试，除非显式设置 `unit_tests: false`。

### 运行测试

```bash
atest libfoo_inline_tests
```

---

## 10. 模糊测试模块 rust_fuzz

`rust_fuzz` 基于 **libFuzzer** 构建覆盖率引导的模糊测试，用于自动探测 panic / 崩溃 / 内存问题。

```bp
rust_fuzz {
    name: "libfoo_fuzzer",
    srcs: ["fuzz/fuzz_target.rs"],
    rustlibs: ["libfoo"],
}
```

fuzz target 源码使用 `libfuzzer-sys` 的 `fuzz_target!` 宏：

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    // 用 data 驱动被测代码，让其尝试 panic / 崩溃
    let _ = foo::parse(data);
});
```

`rust_fuzz` 继承 `rust_binary` 的属性，并接入 Android 的模糊测试基础设施（如持续模糊测试 ClusterFuzz）。

---

## 11. 代码生成：rust_bindgen 与 rust_protobuf

### rust_bindgen — 从 C/C++ 头文件生成 Rust 绑定

当 Rust 需要调用现有 C/C++ 库时，用 `bindgen` 自动从头文件生成不安全（`unsafe`）的 Rust FFI 绑定。

```bp
rust_bindgen {
    name: "libbuzz_bindgen",
    crate_name: "buzz_bindgen",
    // bindgen 的入口包装头文件
    wrapper_src: "bindings.h",
    // 提供符号实现的 C 库
    shared_libs: ["libbuzz"],
    // 传给 bindgen 的标志，例如只允许某些函数/类型
    bindgen_flags: ["--allowlist-function=buzz_.*"],
    // 传给底层 clang 的标志
    cflags: ["-DSOME_MACRO"],
}
```

| 属性                                          | 说明                                            |
| --------------------------------------------- | ----------------------------------------------- |
| `wrapper_src`                                 | 入口头文件，`#include` 需要生成绑定的所有头     |
| `crate_name`                                  | 生成绑定 crate 的名称                           |
| `bindgen_flags`                               | 传给 `bindgen` 的参数（如 allowlist/blocklist） |
| `cflags`                                      | 传给 clang 的编译标志                           |
| `shared_libs` / `static_libs` / `header_libs` | 提供头文件与符号的 C/C++ 库                     |

生成的模块可像普通库一样在 `rustlibs` 中引用。

### rust_protobuf — 从 .proto 生成 Rust 代码

```bp
rust_protobuf {
    name: "libfoo_proto",
    crate_name: "foo_proto",
    protos: ["src/foo.proto"],
    source_stem: "foo_proto",
}
```

| 属性          | 说明                                |
| ------------- | ----------------------------------- |
| `protos`      | `.proto` 源文件列表                 |
| `crate_name`  | 生成的 crate 名                     |
| `source_stem` | 生成的 `.rs` 源文件名（不含扩展名） |
| `grpc_protos` | 需要生成 gRPC 服务代码的 proto      |

生成的 crate 同样通过 `rustlibs` 被其它模块引用。

---

## 12. 依赖管理与链接行为

### 依赖属性的选择

| 属性          | 用于                  | 建议                                 |
| ------------- | --------------------- | ------------------------------------ |
| `rustlibs`    | 依赖其它 Rust 库      | **首选**，让 Soong 自动选 rlib/dylib |
| `rlibs`       | 强制 rlib 链接        | 仅在确有需要时使用                   |
| `proc_macros` | 依赖过程宏            | 过程宏必须用它                       |
| `shared_libs` | 依赖 C/C++ `.so`      | 与 C/C++ 互操作                      |
| `static_libs` | 依赖 C/C++ `.a`       | 与 C/C++ 互操作                      |
| `header_libs` | 仅需头文件的 C/C++ 库 | 配合 bindgen                         |

> **最佳实践**：优先使用 `rustlibs`，把 rlib/dylib 的关联决策交给构建系统，避免变体不兼容问题。

### 链接行为总结

| 目标           | libstd 链接   | rustlibs 偏好变体                                 |
| -------------- | ------------- | ------------------------------------------------- |
| 设备（device） | 动态（dylib） | dylib（除非 `prefer_rlib` / `static_executable`） |
| 主机（host）   | 静态（rlib）  | rlib                                              |

- 依赖项的链接方式由**根模块**（顶层 binary/库）的链接偏好统一决定，以避免同一 crate 出现多份实例。

---

## 13. 使用 rust_defaults 减少重复

当库模块和其测试模块需要共享一组属性（源文件、依赖等）时，用 `rust_defaults` 抽出公共部分，避免重复并防止漂移。

```bp
rust_defaults {
    name: "libfoo_defaults",
    srcs: ["src/lib.rs"],
    crate_name: "foo",
    rustlibs: [
        "libbar",
        "liblog_rust",
    ],
    edition: "2021",
}

rust_library {
    name: "libfoo",
    defaults: ["libfoo_defaults"],
}

rust_test {
    name: "libfoo_inline_tests",
    defaults: ["libfoo_defaults"],
    test_suites: ["general-tests"],
    auto_gen_config: true,
}
```

`rust_defaults` 模块本身不会产出任何构建产物，只作为属性模板被 `defaults` 引用。

---

## 14. Hello World 完整示例

目录结构：

```
external/rust/hello_rust/
├── Android.bp
└── src/
    └── main.rs
```

`src/main.rs`：

```rust
//! Hello World 示例二进制
fn main() {
    println!("Hello from Rust on Android!");
}
```

`Android.bp`：

```bp
rust_binary {
    name: "hello_rust",
    srcs: ["src/main.rs"],
    // 可选：指定 Rust edition
    edition: "2021",
}
```

构建并推送到设备运行：

```bash
# 1. 配置构建环境
source build/envsetup.sh
lunch aosp_arm64-trunk_staging-userdebug   # 选择你的目标

# 2. 构建单个模块
m hello_rust

# 3. 推送到设备并运行
adb push $OUT/system/bin/hello_rust /data/local/tmp/
adb shell /data/local/tmp/hello_rust
# 输出: Hello from Rust on Android!
```

带库的示例（库 + 二进制 + 测试）：

```
hello_rust_lib/
├── Android.bp
└── src/
    ├── lib.rs
    └── main.rs
```

`Android.bp`：

```bp
rust_library {
    name: "libgreeting",
    crate_name: "greeting",
    srcs: ["src/lib.rs"],
}

rust_binary {
    name: "hello_rust_with_lib",
    srcs: ["src/main.rs"],
    rustlibs: ["libgreeting"],
}

rust_test {
    name: "libgreeting_tests",
    srcs: ["src/lib.rs"],
    rustlibs: ["libgreeting"],
    test_suites: ["general-tests"],
    auto_gen_config: true,
}
```

`src/lib.rs`：

```rust
//! 一个简单的问候库
pub fn greeting(name: &str) -> String {
    format!("Hello, {}!", name)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_greeting() {
        assert_eq!(greeting("Android"), "Hello, Android!");
    }
}
```

`src/main.rs`：

```rust
use greeting::greeting;

fn main() {
    println!("{}", greeting("Rust"));
}
```

---

## 15. 构建与运行命令速查

| 操作           | 命令                                                                       |
| -------------- | -------------------------------------------------------------------------- |
| 初始化构建环境 | `source build/envsetup.sh`                                                 |
| 选择构建目标   | `lunch <target>`（如 `aosp_arm64-trunk_staging-userdebug`）                |
| 构建单个模块   | `m <module_name>`                                                          |
| 构建全部       | `m`                                                                        |
| 运行测试       | `atest <test_module>`                                                      |
| 查看模块信息   | `m <module> --no-skip-soong-tests`，或查询 `out/soong/module-actions.json` |
| 推送二进制     | `adb push $OUT/system/bin/<bin> /data/local/tmp/`                          |
| 查看 host 产物 | `$ANDROID_HOST_OUT/bin/<bin>`                                              |

---

## 16. 最佳实践与常见陷阱

**推荐做法**

- 库优先使用 `rust_library`（双变体），依赖优先使用 `rustlibs`，让 Soong 自动决策。
- `crate_name` 不带 `lib` 前缀；`name` 通常写成 `lib<crate_name>`。
- `srcs` 只列入口（`lib.rs` / `main.rs`），其余靠 `mod` 发现。
- 用 `rust_defaults` 在库与其测试之间共享属性。
- 编写单元测试 `#[cfg(test)] mod tests`，并加入 `TEST_MAPPING` 的 presubmit。
- FFI 库记得用 `export_include_dirs` 暴露头文件，Rust 侧用 `#[no_mangle]` + `extern "C"`。

  **常见陷阱**

- 忘记设置 `crate_name`（生成库的模块**必填**）。
- 误用单变体 `rust_library_rlib` / `_dylib` 导致与下游 `rustlibs` 变体不兼容——除非确有需要，否则用 `rust_library`。
- 把过程宏放进 `rustlibs`——过程宏必须用 `proc_macros`。
- 期望 `static_executable` 在非 Bionic 目标上完全静态——实际上 `libc`/`libdl` 仍会动态链接。
- `name` 不唯一导致构建冲突——模块名是全局命名空间。
- 把外部 crate 的 `Cargo.toml` 直接用于平台构建——平台内构建只认 `Android.bp`（外部 crate 用 `cargo_embargo` 转换）。

---

## 17. Android Rust 模式（惯用法与互操作）

这一节汇总在 AOSP 中编写 Rust 时反复出现的**惯用模式**，重点是日志、AIDL（含异步）、以及 Rust 与 C / C++ / Java 三种语言的互操作。

### 17.1 Android 日志记录

Rust 代码通过 `liblogger` + `liblog_rust` 接入 Android 日志系统。它会**自动选择后端**：在设备上使用 `android_logger`（写入 logcat），在主机上使用 `env_logger`（写入 stderr）。

`Android.bp`：

```bp
rust_binary {
    name: "logging_test",
    srcs: ["src/main.rs"],
    rustlibs: [
        "liblogger",
        "liblog_rust",
    ],
}
```

`src/main.rs`：

```rust
use log::{debug, error, LevelFilter};

fn main() {
    // 在程序启动时初始化一次即可
    let _init_success = logger::init(
        logger::Config::default()
            .with_tag_on_device("mytag")          // logcat 中的 tag
            .with_max_level(LevelFilter::Trace),  // 日志级别
    );
    debug!("This is a debug message.");
    error!("Something went wrong!");
}
```

之后即可在任意位置使用 `log` crate 的宏：`trace!` / `debug!` / `info!` / `warn!` / `error!`。

---

### 17.2 Rust AIDL（Binder 服务）

AIDL 接口可以生成 Rust 后端，从而用 Rust 实现/调用 Binder 服务。流程分三步：**定义 `.aidl` → 在 `aidl_interface` 中启用 rust 后端 → 在 `rustlibs` 中引用生成库**（生成库名为 `<aidl_interface 名>-rust`）。

**第 1 步：定义接口** `IRemoteService.aidl`

```java
// IRemoteService.aidl
package com.example.android;

interface IRemoteService {
    int getPid();
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
}
```

**第 2 步：在 `Android.bp` 中启用 Rust 后端**

```bp
aidl_interface {
    name: "com.example.android.remoteservice",
    srcs: [ "aidl/com/example/android/*.aidl", ],
    unstable: true,
    backend: {
        rust: {
            enabled: true,
        },
    },
}
```

**第 3 步：在库中引用生成的 `-rust` 库**

```bp
rust_library {
    name: "libmyservice",
    srcs: ["src/lib.rs"],
    crate_name: "myservice",
    rustlibs: [
        "com.example.android.remoteservice-rust",
        "libbinder_rs",
    ],
}
```

**服务端实现** `src/lib.rs`：

```rust
use com_example_android_remoteservice::aidl::com::example::android::{
  IRemoteService::{BnRemoteService, IRemoteService}
};
use binder::{
    BinderFeatures, Interface, Result as BinderResult, Strong,
};

pub struct MyService;

impl Interface for MyService {}

impl IRemoteService for MyService {
    fn getPid(&self) -> BinderResult<i32> {
        Ok(42)
    }

    fn basicTypes(&self, _: i32, _: i64, _: bool, _: f32, _: f64, _: &str) -> BinderResult<()> {
        Ok(())
    }
}
```

**注册并启动服务**（二进制 `main.rs`）：

```rust
use myservice::MyService;

fn main() {
    let my_service = MyService;
    let my_service_binder = BnRemoteService::new_binder(
        my_service,
        BinderFeatures::default(),
    );
    binder::add_service("myservice", my_service_binder.as_binder())
        .expect("Failed to register service?");
    // 进入 binder 线程池，阻塞等待请求
    binder::ProcessState::join_thread_pool()
}
```

---

### 17.3 异步 Rust AIDL（结合 Tokio）

生成的 Rust 后端同时提供**异步**接口：服务端 trait `IRemoteServiceAsyncServer`，客户端 trait `IRemoteServiceAsync<T>`。配合 `tokio` 运行时使用。

> ⚠️ 注意：**不能在 `block_on` 内再调用 `block_on`**，否则会 panic。因此服务端用 `task::block_in_place` 包裹 `join_thread_pool`，把阻塞调用交给独立线程。

**异步服务端实现：**

```rust
use com_example_android_remoteservice::aidl::com::example::android::IRemoteService::{
        BnRemoteService, IRemoteServiceAsyncServer,
    };
use binder::{BinderFeatures, Interface, Result as BinderResult};

pub struct MyAsyncService;

impl Interface for MyAsyncService {}

#[async_trait]
impl IRemoteServiceAsyncServer for MyAsyncService {
    async fn getPid(&self) -> BinderResult<i32> {
        Ok(42)
    }

    async fn basicTypes(&self, _: i32, _: i64, _: bool, _: f32, _: f64,_: &str,) -> BinderResult<()> {
        Ok(())
    }
}
```

**异步服务端启动：**

```rust
#[tokio::main(flavor = "multi_thread", worker_threads = 2)]
async fn main() {
    binder::ProcessState::start_thread_pool();

    let my_service = MyAsyncService;
    let my_service_binder = BnRemoteService::new_async_binder(
        my_service,
        TokioRuntime(Handle::current()),
        BinderFeatures::default(),
    );

    binder::add_service("myservice", my_service_binder.as_binder())
        .expect("Failed to register service?");

    // 用 block_in_place 避免在 tokio 运行时内阻塞工作线程
    task::block_in_place(move || {
        binder::ProcessState::join_thread_pool();
    });
}
```

**异步客户端实现：**

```rust
use com_example_android_remoteservice::aidl::com::example::android::IRemoteService::IRemoteServiceAsync;
use binder_tokio::Tokio;

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let binder_service = binder_tokio::wait_for_interface::<dyn IRemoteServiceAsync<Tokio>>("myservice");

    let my_client = binder_service.await.expect("Cannot find Remote Service");

    let result = my_client.getPid().await;

    match result {
        Err(err) => panic!("Cannot get the process id from Remote Service {:?}", err),
        Ok(p_id) => println!("PID = {}", p_id),
    }
}
```

---

### 17.4 从 C 调用 Rust

把 Rust 函数以 C ABI 导出（`#[no_mangle]` + `pub extern fn`），用 `rust_ffi` 构建并通过 `export_include_dirs` 暴露头文件，C 端用 `cc_binary` + `shared_libs` 依赖。

**Rust 库** `libsimple_c_printer.rs`：

```rust
#[no_mangle]
pub extern fn print_c_hello_rust() {
    println!("Hello Rust!");
}
```

**C 头文件** `simple_printer.h`（需开发者手写，与导出函数签名匹配）：

```c
#ifndef SIMPLE_PRINTER_H
#define SIMPLE_PRINTER_H

void print_c_hello_rust();

#endif
```

**Rust 库的 `Android.bp`：**

```bp
rust_ffi {
    name: "libsimple_c_printer",
    crate_name: "simple_c_printer",
    srcs: ["libsimple_c_printer.rs"],
    export_include_dirs: ["."],
}
```

**C 调用方** `main.c`：

```c
#include "simple_printer.h"

int main() {
  print_c_hello_rust();
  return 0;
}
```

**C 二进制的 `Android.bp`：**

```bp
cc_binary {
    name: "c_hello_rust",
    srcs: ["main.c"],
    shared_libs: ["libsimple_c_printer"],
}
```

---

### 17.5 Rust ↔ Java 互操作（JNI）

通过 `jni` crate 实现，**无需在构建期做代码生成**（区别于 CXX）。Rust 侧编译为 `rust_ffi_shared`（产出 `.so`），Java 侧通过 `required` 或 `uses_libs` 把该 `.so` 拉入运行时依赖。

**Rust 侧 `Android.bp`：**

```bp
rust_ffi_shared {
    name: "libhello_jni",
    crate_name: "hello_jni",
    srcs: ["src/lib.rs"],
    rustlibs: ["libjni"],
}
```

**Java 库依赖该共享库（两种写法）：**

```bp
// 写法一：运行时需要该原生库
java_library {
    name: "libhelloworld",
    // [...]
    required: ["libhellorust"],
    // [...]
}
```

```bp
// 写法二：作为 uses_libs
java_library {
    name: "libhelloworld",
    // [...]
    uses_libs: ["libhellorust"],
    // [...]
}
```

Rust 侧需按 JNI 命名约定导出函数（如 `Java_<包名>_<类名>_<方法名>`），并使用 `jni` crate 的 `JNIEnv`、`JObject` 等类型。

---

### 17.6 Rust ↔ C++ 互操作（CXX）

[CXX](https://cxx.rs/) 提供**安全**的 Rust/C++ 双向桥接。它依赖 `cxxbridge` 工具在构建期从 `#[cxx::bridge]` 模块生成 C++ 胶水代码与头文件，需要用 `genrule` 显式声明这些生成步骤。

整体装配关系：

```
lib.rs (#[cxx::bridge])
   ├─ genrule(cxxbridge)        → libcxx_test_cxx_generated.cc   (生成源)
   ├─ genrule(cxxbridge --header)→ lib.rs.h                       (生成头)
   └─ cxx-bridge-header                                           (CXX 运行时头)
                ↓
   cc_library_static (libcxx_test_cpp)  ← 你手写的 cxx_test.cpp + 上述生成物
                ↓ static_libs
   rust_binary (cxx_test)  ← lib.rs + libcxx
```

**C++ 库与代码生成的 `Android.bp`：**

```bp
cc_library_static {
    name: "libcxx_test_cpp",
    srcs: ["cxx_test.cpp"],
    generated_headers: [
        "cxx-bridge-header",
        "libcxx_test_bridge_header"
    ],
    generated_sources: ["libcxx_test_bridge_code"],
}

genrule {
    name: "libcxx_test_bridge_code",
    tools: ["cxxbridge"],
    cmd: "$(location cxxbridge) $(in) > $(out)",
    srcs: ["lib.rs"],
    out: ["libcxx_test_cxx_generated.cc"],
}

genrule {
    name: "libcxx_test_bridge_header",
    tools: ["cxxbridge"],
    cmd: "$(location cxxbridge) $(in) --header > $(out)",
    srcs: ["lib.rs"],
    out: ["lib.rs.h"],
}
```

**Rust 二进制的 `Android.bp`：**

```bp
rust_binary {
    name: "cxx_test",
    srcs: ["lib.rs"],
    rustlibs: ["libcxx"],
    static_libs: ["libcxx_test_cpp"],
}
```

**C++ 头文件** `cxx_test.hpp`：

```cpp
#pragma once

#include "rust/cxx.h"
#include "lib.rs.h"

int greet(rust::Str greetee);
```

**C++ 源文件** `cxx_test.cpp`：

```cpp
#include "cxx_test.hpp"
#include "lib.rs.h"

#include <iostream>

int greet(rust::Str greetee) {
  std::cout << "Hello, " << greetee << std::endl;
  return get_num();
}
```

**Rust 源文件** `lib.rs`：

```rust
#[cxx::bridge]
mod ffi {
    // 声明从 C++ 引入、可在 Rust 中调用的项
    unsafe extern "C++" {
        include!("cxx_test.hpp");
        fn greet(greetee: &str) -> i32;
    }
    // 声明从 Rust 导出、可在 C++ 中调用的项
    extern "Rust" {
        fn get_num() -> i32;
    }
}

fn main() {
    let result = ffi::greet("world");
    println!("C++ returned {}", result);
}

fn get_num() -> i32 {
    return 42;
}
```

> CXX 与原始 `rust_ffi` 的区别：`rust_ffi` 是裸 C ABI、需要手写头文件且全程 `unsafe`；CXX 用 `#[cxx::bridge]` 描述接口，自动生成两侧胶水代码并提供 `rust::Str`、`rust::Vec` 等安全共享类型，更适合复杂的 C++ 互操作。

---

### 17.7 三种互操作方式对比

| 场景                      | 推荐方式                 | 构建模块                                                   | 是否需代码生成       |
| ------------------------- | ------------------------ | ---------------------------------------------------------- | -------------------- |
| C 调用 Rust（简单 C ABI） | `#[no_mangle] extern fn` | `rust_ffi` / `rust_ffi_shared`                             | 否（手写头文件）     |
| Rust 调用 C/C++ 头文件    | `bindgen`                | `rust_bindgen`                                             | 是（构建期生成绑定） |
| Rust ↔ C++（复杂、安全）  | CXX                      | `genrule(cxxbridge)` + `cc_library_static` + `rust_binary` | 是（cxxbridge 生成） |
| Rust ↔ Java               | `jni` crate              | `rust_ffi_shared`                                          | 否                   |
| Rust ↔ Binder 服务        | AIDL rust 后端           | `aidl_interface` + `rustlibs`                              | 是（aidl 生成）      |
