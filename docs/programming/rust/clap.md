# clap — Rust 命令行解析

clap 是 Rust 生态最主流的命令行参数解析库，提供两种 API 风格：**Derive API**（推荐，用宏自动生成）和 **Builder API**（代码构建，灵活但繁琐）。生产项目几乎都选 Derive API。

---

## 安装

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

常用可选 feature：

| feature | 说明 |
|---------|------|
| `derive` | 启用 `#[derive(Parser)]` 宏（强烈推荐） |
| `env` | 支持从环境变量读取参数值 |
| `unicode` | 支持 Unicode 宽字符对齐帮助信息 |
| `color` | 彩色帮助信息（默认启用） |
| `cargo` | 启用 `crate_version!()` 等 Cargo 宏 |

---

## Derive API 快速入门

最小可用示例：

```rust
use clap::Parser;

/// 一个简单的问候工具
#[derive(Parser)]
#[command(version, about)]
struct Cli {
    /// 要问候的名字
    name: String,
}

fn main() {
    let cli = Cli::parse();
    println!("Hello, {}!", cli.name);
}
```

运行效果：

```shell
$ cargo run -- Alice
Hello, Alice!

$ cargo run -- --help
Usage: greet <NAME>

Arguments:
  <NAME>  要问候的名字

Options:
  -h, --help     Print help
  -V, --version  Print version

$ cargo run -- --version
greet 0.1.0
```

---

## 参数类型详解

clap 中的参数分三类：

| 类型 | 示例 | 说明 |
|------|------|------|
| 位置参数（Argument）| `<FILE>` | 按位置顺序接收，无前缀 |
| 选项（Option）| `--output <FILE>` | 有 `--name` 前缀，后跟值 |
| 标志（Flag）| `--verbose` | 有 `--name` 前缀，不跟值，布尔型 |

### 位置参数

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 输入文件路径
    input: String,

    /// 输出文件路径（可选，默认 output.txt）
    #[arg(default_value = "output.txt")]
    output: String,

    /// 处理次数（默认 1）
    #[arg(default_value_t = 1)]
    count: u32,
}
```

### 选项（带值的参数）

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 监听端口
    #[arg(short, long, default_value_t = 8080)]
    port: u16,

    /// 主机地址
    #[arg(short = 'H', long, default_value = "127.0.0.1")]
    host: String,

    /// 最大连接数（可选）
    #[arg(long)]
    max_connections: Option<u32>,
}
```

`short` 自动取字段名首字母作为短选项，也可以用 `short = 'H'` 手动指定。

### 标志（布尔开关）

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 开启详细输出
    #[arg(short, long)]
    verbose: bool,

    /// 不输出任何信息
    #[arg(short, long)]
    quiet: bool,

    /// 详细级别（可重复：-v、-vv、-vvv）
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbosity: u8,
}

fn main() {
    let cli = Cli::parse();
    match cli.verbosity {
        0 => println!("普通模式"),
        1 => println!("详细模式"),
        2 => println!("非常详细"),
        _ => println!("调试模式"),
    }
}
```

### 多值参数

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 要处理的文件列表（可多个）
    #[arg(short, long, num_args = 1..)]
    files: Vec<String>,

    /// 标签（可多次指定：--tag a --tag b）
    #[arg(long)]
    tag: Vec<String>,
}
```

```shell
$ app --files a.txt b.txt c.txt
$ app --tag frontend --tag backend
```

---

## 子命令

类似 `git commit`、`git push` 的多子命令结构：

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(version, about = "一个文件管理工具")]
struct Cli {
    /// 全局详细模式
    #[arg(short, long, global = true)]
    verbose: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// 添加文件
    Add {
        /// 要添加的文件路径
        #[arg(required = true)]
        paths: Vec<String>,

        /// 强制添加（即使已存在）
        #[arg(short, long)]
        force: bool,
    },

    /// 删除文件
    Remove {
        /// 要删除的文件路径
        path: String,

        /// 递归删除目录
        #[arg(short, long)]
        recursive: bool,
    },

    /// 列出文件
    List {
        /// 按时间排序
        #[arg(short, long)]
        time: bool,
    },
}

fn main() {
    let cli = Cli::parse();

    if cli.verbose {
        println!("[verbose] 已启用详细模式");
    }

    match cli.command {
        Commands::Add { paths, force } => {
            for path in &paths {
                if force {
                    println!("强制添加：{path}");
                } else {
                    println!("添加：{path}");
                }
            }
        }
        Commands::Remove { path, recursive } => {
            if recursive {
                println!("递归删除：{path}");
            } else {
                println!("删除：{path}");
            }
        }
        Commands::List { time } => {
            println!("列出文件（按{}排序）", if time { "时间" } else { "名称" });
        }
    }
}
```

运行效果：

```shell
$ app add file1.txt file2.txt --force
$ app remove /tmp/dir --recursive
$ app list --time
$ app --verbose add file.txt
```

### 嵌套子命令

```rust
#[derive(Subcommand)]
enum Commands {
    /// 配置管理
    Config {
        #[command(subcommand)]
        action: ConfigAction,
    },
}

#[derive(Subcommand)]
enum ConfigAction {
    /// 获取配置
    Get { key: String },
    /// 设置配置
    Set { key: String, value: String },
}
```

---

## 枚举参数

使用 `ValueEnum` 限制参数只能取特定值：

```rust
use clap::{Parser, ValueEnum};

#[derive(Debug, Clone, ValueEnum)]
enum OutputFormat {
    Json,
    Yaml,
    Table,
    Csv,
}

#[derive(Parser)]
struct Cli {
    /// 输出格式
    #[arg(short, long, value_enum, default_value_t = OutputFormat::Table)]
    format: OutputFormat,

    /// 日志级别
    #[arg(long, value_enum, default_value_t = LogLevel::Info)]
    log_level: LogLevel,
}

#[derive(Debug, Clone, ValueEnum)]
enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}

fn main() {
    let cli = Cli::parse();
    match cli.format {
        OutputFormat::Json  => println!("输出 JSON"),
        OutputFormat::Yaml  => println!("输出 YAML"),
        OutputFormat::Table => println!("输出表格"),
        OutputFormat::Csv   => println!("输出 CSV"),
    }
}
```

```shell
$ app --format json
$ app --format table
$ app --format invalid   # 报错，提示可用值
```

---

## 环境变量

需要开启 `env` feature：

```toml
clap = { version = "4", features = ["derive", "env"] }
```

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// 数据库连接 URL（也可用 DATABASE_URL 环境变量）
    #[arg(long, env = "DATABASE_URL")]
    database_url: String,

    /// API 密钥（也可用 API_KEY 环境变量）
    #[arg(long, env = "API_KEY", hide_env_values = true)]  // 隐藏环境变量值，不在帮助中显示
    api_key: String,

    /// 端口（优先命令行，其次环境变量，最后默认值）
    #[arg(short, long, env = "PORT", default_value_t = 3000)]
    port: u16,
}
```

```shell
$ DATABASE_URL=postgres://localhost/mydb API_KEY=secret app
$ app --database-url postgres://localhost/mydb --api-key secret
```

---

## 参数验证

### 内置验证

```rust
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// 端口号（1–65535）
    #[arg(short, long, default_value_t = 8080, value_parser = clap::value_parser!(u16).range(1..=65535))]
    port: u16,

    /// 必须存在的文件
    #[arg(value_hint = clap::ValueHint::FilePath)]
    input: PathBuf,

    /// 线程数（1–32）
    #[arg(long, default_value_t = 4, value_parser = clap::value_parser!(u8).range(1..=32))]
    threads: u8,
}
```

### 自定义验证函数

```rust
use clap::Parser;

fn validate_port(s: &str) -> Result<u16, String> {
    let port: u16 = s.parse().map_err(|_| format!("'{s}' 不是有效端口号"))?;
    if port < 1024 {
        Err(format!("端口 {port} 是特权端口，请使用 1024 以上"))
    } else {
        Ok(port)
    }
}

fn validate_email(s: &str) -> Result<String, String> {
    if s.contains('@') && s.contains('.') {
        Ok(s.to_string())
    } else {
        Err(format!("'{s}' 不是有效的邮箱格式"))
    }
}

#[derive(Parser)]
struct Cli {
    #[arg(short, long, value_parser = validate_port)]
    port: u16,

    #[arg(long, value_parser = validate_email)]
    email: String,
}
```

### 互斥参数

```rust
use clap::Parser;

#[derive(Parser)]
#[command(group(
    clap::ArgGroup::new("output_mode")
        .required(true)          // 必须选一个
        .args(["json", "yaml", "text"]),
))]
struct Cli {
    /// 以 JSON 格式输出
    #[arg(long)]
    json: bool,

    /// 以 YAML 格式输出
    #[arg(long)]
    yaml: bool,

    /// 以文本格式输出
    #[arg(long)]
    text: bool,
}
```

---

## #[arg] 属性速查

```rust
#[derive(Parser)]
struct Cli {
    // 基础
    #[arg(short)]                         // 自动取首字母作为短选项 -x
    #[arg(short = 'x')]                   // 手动指定短选项
    #[arg(long)]                          // 自动取字段名作为长选项 --field-name
    #[arg(long = "my-option")]            // 手动指定长选项名
    #[arg(short, long)]                   // 同时有短选项和长选项

    // 文档（作为帮助信息显示）
    /// 这是帮助信息                       // /// 注释自动作为 help

    // 默认值
    #[arg(default_value = "hello")]       // 字符串默认值
    #[arg(default_value_t = 42)]          // 表达式默认值（类型需实现 Display）
    #[arg(default_missing_value = "42")]  // 指定选项但没给值时的默认值

    // 必填 / 可选
    #[arg(required = true)]               // 强制必填（Vec 默认不必填）
    #[arg(required = false)]              // 可选

    // 值数量
    #[arg(num_args = 1)]                  // 恰好 1 个值（默认）
    #[arg(num_args = 0..=1)]              // 0 或 1 个值（等效 Option）
    #[arg(num_args = 1..)]                // 至少 1 个
    #[arg(num_args = 2..=5)]              // 2 到 5 个

    // 动作
    #[arg(action = clap::ArgAction::Append)]   // 每次指定追加一个值到 Vec
    #[arg(action = clap::ArgAction::Count)]    // 计数（-vvv = 3）
    #[arg(action = clap::ArgAction::SetTrue)]  // 出现时设为 true
    #[arg(action = clap::ArgAction::SetFalse)] // 出现时设为 false

    // 验证
    #[arg(value_parser = clap::value_parser!(u16).range(1..=65535))]
    #[arg(value_parser = my_validate_fn)]

    // 枚举
    #[arg(value_enum)]                    // 配合 ValueEnum 使用

    // 显示控制
    #[arg(hide = true)]                   // 在帮助中隐藏
    #[arg(hide_short_help = true)]        // 只在 --help 中显示，-h 中隐藏
    #[arg(hide_env_values = true)]        // 隐藏环境变量的值

    // 提示
    #[arg(value_hint = clap::ValueHint::FilePath)]   // Shell 补全提示：文件路径
    #[arg(value_hint = clap::ValueHint::DirPath)]    // Shell 补全提示：目录路径
    #[arg(value_hint = clap::ValueHint::Username)]   // Shell 补全提示：用户名

    // 别名
    #[arg(alias = "colour")]              // 为选项设置别名

    // 全局（对所有子命令生效）
    #[arg(global = true)]

    // 名称冲突时覆盖
    #[arg(overrides_with = "other_field")]

    // 环境变量
    #[arg(env = "MY_VAR")]
    x: String,
}
```

---

## #[command] 属性速查

```rust
#[derive(Parser)]
#[command(
    name = "myapp",                         // 程序名（默认取 Cargo.toml 的 name）
    version = "1.2.3",                      // 版本号（或用 version 自动读取）
    version,                                // 自动读取 Cargo.toml 的 version
    author = "Alice <alice@example.com>",   // 作者
    about = "简短描述",                      // -h 中的简短描述
    long_about = "详细描述...",              // --help 中的详细描述
    after_help = "更多信息见 https://...",   // 帮助信息末尾追加
    before_help = "使用前请注意：...",       // 帮助信息开头追加
    propagate_version = true,               // 将版本传播到子命令
    subcommand_required = true,             // 强制要求指定子命令
    arg_required_else_help = true,          // 无参数时显示帮助（常用）
    disable_help_flag = true,               // 禁用 -h/--help
    disable_version_flag = true,            // 禁用 -V/--version
    color = clap::ColorChoice::Always,      // 颜色策略
    styles = ...,                           // 自定义样式
)]
struct Cli { }
```

---

## 完整实战示例

一个类似 `grep` 的文本搜索工具：

```rust
use clap::{Parser, ValueEnum, ArgGroup};
use std::path::PathBuf;

/// 文本搜索工具
#[derive(Parser)]
#[command(
    version,
    about = "在文件中搜索文本",
    long_about = "一个简单的文本搜索工具，支持正则表达式和多文件搜索。",
    arg_required_else_help = true,
)]
struct Cli {
    /// 搜索模式（支持正则表达式）
    pattern: String,

    /// 要搜索的文件（可多个）
    #[arg(required = true)]
    files: Vec<PathBuf>,

    /// 忽略大小写
    #[arg(short = 'i', long)]
    ignore_case: bool,

    /// 只输出包含匹配的文件名
    #[arg(short = 'l', long)]
    files_with_matches: bool,

    /// 只输出匹配行数
    #[arg(short = 'c', long)]
    count: bool,

    /// 输出前 N 行上下文
    #[arg(short = 'B', long, value_name = "NUM", default_value_t = 0)]
    before_context: usize,

    /// 输出后 N 行上下文
    #[arg(short = 'A', long, value_name = "NUM", default_value_t = 0)]
    after_context: usize,

    /// 输出格式
    #[arg(long, value_enum, default_value_t = OutputFormat::Text)]
    format: OutputFormat,

    /// 排除文件（glob 模式）
    #[arg(long, value_name = "PATTERN")]
    exclude: Vec<String>,

    /// 最大匹配数
    #[arg(short = 'm', long)]
    max_count: Option<usize>,

    /// 详细输出（可重复 -v/-vv 增加级别）
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
}

#[derive(Debug, Clone, ValueEnum)]
enum OutputFormat {
    Text,
    Json,
    Csv,
}

fn main() {
    let cli = Cli::parse();

    if cli.verbose >= 2 {
        eprintln!("[debug] 搜索模式：{}", cli.pattern);
        eprintln!("[debug] 文件列表：{:?}", cli.files);
    }

    for file in &cli.files {
        if !file.exists() {
            eprintln!("文件不存在：{}", file.display());
            continue;
        }

        let content = match std::fs::read_to_string(file) {
            Ok(c) => c,
            Err(e) => {
                eprintln!("读取失败：{} — {e}", file.display());
                continue;
            }
        };

        let pattern = if cli.ignore_case {
            cli.pattern.to_lowercase()
        } else {
            cli.pattern.clone()
        };

        let matched_lines: Vec<(usize, &str)> = content
            .lines()
            .enumerate()
            .filter(|(_, line)| {
                let text = if cli.ignore_case { line.to_lowercase() } else { line.to_string() };
                text.contains(&pattern)
            })
            .collect();

        if cli.files_with_matches {
            if !matched_lines.is_empty() {
                println!("{}", file.display());
            }
        } else if cli.count {
            println!("{}:{}", file.display(), matched_lines.len());
        } else {
            for (lineno, line) in &matched_lines {
                println!("{}:{}:{}", file.display(), lineno + 1, line);
            }
        }
    }
}
```

---

## 编程式调用（测试用）

`parse_from` 可以在测试中模拟命令行输入，无需真实 `std::env::args`：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_default_port() {
        let cli = Cli::parse_from(["app"]);
        assert_eq!(cli.port, 8080);
    }

    #[test]
    fn test_custom_port() {
        let cli = Cli::parse_from(["app", "--port", "3000"]);
        assert_eq!(cli.port, 3000);
    }

    #[test]
    fn test_verbose_flag() {
        let cli = Cli::parse_from(["app", "-v", "-v"]);
        assert_eq!(cli.verbosity, 2);
    }

    #[test]
    fn test_invalid_port() {
        // 验证非法参数确实会报错
        let result = Cli::try_parse_from(["app", "--port", "99999"]);
        assert!(result.is_err());
    }

    #[test]
    fn test_subcommand() {
        let cli = Cli::parse_from(["app", "add", "file.txt", "--force"]);
        match cli.command {
            Commands::Add { paths, force } => {
                assert_eq!(paths, vec!["file.txt"]);
                assert!(force);
            }
            _ => panic!("期望 Add 子命令"),
        }
    }
}
```

---

## Builder API

不使用 derive 宏，手动构建（灵活但冗长，适合动态生成 CLI）：

```rust
use clap::{Arg, ArgAction, Command};

fn main() {
    let matches = Command::new("myapp")
        .version("1.0")
        .about("Builder API 示例")
        .arg(
            Arg::new("port")
                .short('p')
                .long("port")
                .default_value("8080")
                .value_parser(clap::value_parser!(u16))
                .help("监听端口"),
        )
        .arg(
            Arg::new("verbose")
                .short('v')
                .long("verbose")
                .action(ArgAction::Count)
                .help("详细模式"),
        )
        .subcommand(
            Command::new("serve")
                .about("启动服务器")
                .arg(Arg::new("host").long("host").default_value("localhost")),
        )
        .get_matches();

    let port = *matches.get_one::<u16>("port").unwrap();
    let verbosity = matches.get_count("verbose");

    println!("端口：{port}，详细级别：{verbosity}");

    if let Some(sub) = matches.subcommand_matches("serve") {
        let host = sub.get_one::<String>("host").unwrap();
        println!("启动服务器：{host}:{port}");
    }
}
```

---

## Shell 补全生成

clap 可以生成 Bash、Zsh、Fish、PowerShell 等 Shell 的自动补全脚本：

```toml
[build-dependencies]
clap_complete = "4"
```

**build.rs：**

```rust
use clap::CommandFactory;
use clap_complete::{generate_to, shells::Bash};
use std::env;

include!("src/cli.rs");   // 引入 Cli 结构体

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();
    let mut cmd = Cli::command();
    let path = generate_to(Bash, &mut cmd, "myapp", &out_dir).unwrap();
    println!("cargo:warning=补全脚本生成到：{path:?}");
}
```

或者在运行时通过子命令生成：

```rust
use clap::{CommandFactory, Parser};
use clap_complete::{generate, Shell};

#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(clap::Subcommand)]
enum Commands {
    /// 生成 Shell 补全脚本
    Completions {
        /// 目标 Shell
        #[arg(value_enum)]
        shell: Shell,
    },
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Some(Commands::Completions { shell }) => {
            let mut cmd = Cli::command();
            generate(shell, &mut cmd, "myapp", &mut std::io::stdout());
        }
        None => {
            println!("正常运行");
        }
    }
}
```

```shell
# 生成并加载 Bash 补全
$ myapp completions bash > ~/.bash_completion.d/myapp

# 生成 Zsh 补全
$ myapp completions zsh > ~/.zsh/completions/_myapp
```

---

## 自定义帮助信息样式

clap 4 支持通过 `Styles` API 自定义帮助信息颜色和样式：

```rust
use clap::{builder::Styles, Parser};
use clap::builder::styling::{AnsiColor, Effects};

const STYLES: Styles = Styles::styled()
    .header(AnsiColor::Yellow.on_default().effects(Effects::BOLD))
    .usage(AnsiColor::Yellow.on_default().effects(Effects::BOLD))
    .literal(AnsiColor::Cyan.on_default().effects(Effects::BOLD))
    .placeholder(AnsiColor::Green.on_default());

#[derive(Parser)]
#[command(styles = STYLES)]
struct Cli {
    name: String,
}
```

---

## 常见模式总结

| 需求 | 写法 |
|------|------|
| 可选参数 | `field: Option<String>` |
| 有默认值的参数 | `#[arg(default_value_t = 42)] field: u32` |
| 布尔标志 | `#[arg(short, long)] verbose: bool` |
| 可重复标志（计数）| `#[arg(short, long, action = ArgAction::Count)] verbosity: u8` |
| 多值选项 | `#[arg(long, num_args = 1..)] tags: Vec<String>` |
| 枚举限制 | `#[arg(value_enum)] format: OutputFormat`（需 `#[derive(ValueEnum)]`）|
| 环境变量 | `#[arg(env = "MY_VAR")] key: String` |
| 互斥参数 | `#[command(group(ArgGroup::new("g").args(["a", "b"])))]` |
| 子命令 | `#[command(subcommand)] cmd: Commands`（需 `#[derive(Subcommand)]`）|
| 参数验证 | `#[arg(value_parser = my_fn)]` 或 `value_parser!(u16).range(1..=1000)` |
| 无参数显示帮助 | `#[command(arg_required_else_help = true)]` |
| 全局参数 | `#[arg(global = true)] verbose: bool` |
| Shell 补全提示 | `#[arg(value_hint = ValueHint::FilePath)]` |
| 测试中解析 | `Cli::parse_from(["app", "--flag"])` |
| 测试错误情况 | `Cli::try_parse_from(["app", "--bad"])` |
