# 学习笔记

这里记录了前端、后端、移动端、CI/CD、AI 工具等方向的技术学习笔记，持续更新。每篇文档以"能直接用"为目标——核心概念 + 常用命令 + 完整示例，减少查文档时的来回翻找。

---

## 内容导航

### 编程语言

| 文档                                                         | 简介                                           |
| ------------------------------------------------------------ | ---------------------------------------------- |
| [Rust](programming/rust/index.md)                            | 所有权、借用、生命周期、并发、异步、Axum       |
| [clap](programming/rust/clap.md)                             | Rust 命令行解析库，Derive API、子命令、验证、Shell 补全 |
| [build.rs](programming/rust/build_rs.md)                     | Cargo 构建脚本，编译前代码生成、链接库配置、环境变量注入 |
| [TypeScript](programming/typescript/index.md)                | 静态类型系统、工程配置                         |
| [垃圾回收算法](programming/garbage-collection-algorithms.md) | 标记清除、引用计数、分代回收等主流 GC 算法原理 |

---

### 前端

| 文档                                           | 简介                                   |
| ---------------------------------------------- | -------------------------------------- |
| [React](frontend/react/react.md)               | Hooks、状态管理、性能优化              |
| [React Hook](frontend/react/react_hook.md)     | useState、useEffect、useRef、自定义 Hook |
| [React Router](frontend/react/react-router.md) | 声明式路由、嵌套路由、数据加载         |
| [Redux](frontend/react/redux.md)               | Redux Toolkit、异步 Thunk、RTK Query   |
| [Zustand](frontend/react/zustand.md)           | 轻量状态管理、中间件、持久化           |
| [Next.js](frontend/react/nextjs.md)            | App Router、SSR/SSG、Server Components |
| [React Native](frontend/react/react-native.md) | 跨平台移动开发、导航、原生模块         |
| [shadcn/ui](frontend/react/shadcn.md)          | 组件库安装、主题定制、常用组件         |
| [Ant Design](frontend/react/antd.md)           | 企业级 UI 组件、表单、表格             |
| [Sass](frontend/sass.md)                       | 变量、嵌套、Mixin、函数                |
| [CSS Modules](frontend/css-modules.md)         | 局部作用域 CSS、组合、与框架集成        |
| [Tailwind CSS](frontend/tailwindcss.md)        | 原子化 CSS、响应式、暗色模式           |
| [JavaScript ES6~ES2025](frontend/JavaScript-ES6-to-ES2025.md) | ES6 到 ES2025 各版本新特性速查，let/const、模块、异步、Proxy 等 |
| [现代 Web 平台新特性](frontend/Web-Modern-Features.md) | HTML/CSS/JS 平台级新特性，dialog、Container Queries、Web Components 等 |

---

### 后端

| 文档                                    | 简介                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| [Docker](backend/docker.md)             | 镜像、容器、Dockerfile、Compose、网络与存储                  |
| [Axum](backend/rust/axum.md)            | Rust Web 框架，提取器体系、中间件、sqlx 集成、JWT、WebSocket |
| [sqlx](backend/rust/sqlx.md)            | Rust 异步数据库驱动，编译期 SQL 检查、迁移、事务、Repository 模式 |
| [Tokio](backend/rust/tokio.md)          | Rust 异步运行时，Task、Channel、Timer、网络 I/O              |
| [Tonic](backend/rust/tonic.md)          | Rust gRPC 框架，基于 Tokio，Protobuf 集成、双向流           |
| [tracing](backend/rust/tracing.md)      | Rust 结构化日志与诊断框架，Span、事件、订阅者               |
| [Redis](backend/redis.md)               | 内存数据库，String/Hash/List/Set/ZSet、连接池、限流、分布式锁 |
| [SQL](backend/sql.md)                   | 关系型数据库标准语言，DDL/DML/DQL、事务、索引、优化          |
| [Nginx](backend/nginx.md)               | 高性能 HTTP 服务器与反向代理，负载均衡、HTTPS、限流          |
| [MQ](backend/mq.md)                     | 消息队列，异步解耦、发布订阅、RabbitMQ/Kafka 核心概念        |
| [Protobuf](backend/protobuf.md)         | Protocol Buffers，序列化格式、.proto 语法、与 gRPC 集成     |
| [微服务](backend/微服务.md)             | 微服务架构结合 Rust 实践，服务拆分、通信、部署               |

---

### Flutter

| 文档                                                  | 简介                                                                 |
| ----------------------------------------------------- | -------------------------------------------------------------------- |
| [Agent Skills](flutter/agent-skills.md)               | Flutter/Dart 官方 AI 技能库，配合 Claude Code、Cursor 等 AI 助手使用 |
| [Flutter Rust Bridge](flutter/flutter-rust-bridge.md) | Flutter 与 Rust 互调，FFI 绑定代码生成                               |

---

### Android

| 文档                                  | 简介                            |
| ------------------------------------- | ------------------------------- |
| [Jetpack Compose](android/compose.md) | 声明式 UI、状态管理、导航、主题 |

---

### iOS

| 文档                                              | 简介                                                     |
| ------------------------------------------------- | -------------------------------------------------------- |
| [iOS for Flutter 开发者](ios/ios-for-flutter.md) | 证书、描述文件、打包、TestFlight、App Store 上架全流程   |

---

### CI/CD

| 文档                                     | 简介                                                     |
| ---------------------------------------- | -------------------------------------------------------- |
| [GitHub Actions](cicd/github-actions.md) | Workflow 语法、触发器、矩阵构建、可复用工作流            |
| [GitLab CI/CD](cicd/gitlab-cicd.md)      | `.gitlab-ci.yml`、Runner、DAG 依赖、多项目 Pipeline      |
| [Jenkins](cicd/jenkins.md)               | Declarative Pipeline、Shared Library、分布式构建         |
| [Fastlane](cicd/fastlane.md)             | iOS/Android 自动化打包、签名、App Store/Google Play 发布 |

---

### 逆向工程

| 文档                                                   | 简介                                                    |
| ------------------------------------------------------ | ------------------------------------------------------- |
| [Frida](reverse_engineering/frida.md)                  | 动态插桩框架、Java/Native Hook、SSL 绕过、Python 控制端 |
| [JADX](reverse_engineering/jadx.md)                    | APK/DEX 反编译、反混淆、代码搜索、与 Frida 联动         |
| [LSPosed](reverse_engineering/lsposed.md)              | Xposed 模块开发、Hook API、反检测、Kotlin 写法          |
| [主流网络代理与隧道协议](reverse_engineering/agent.md) | HTTP 代理、SOCKS5、隧道协议原理与对比                   |

---

### 游戏开发

| 文档                                               | 简介                                              |
| -------------------------------------------------- | ------------------------------------------------- |
| [Godot 引擎入门](game/godot/godot_tutorial.md)     | Godot 安装、节点/场景系统、2D/3D 基础、项目结构   |
| [GDScript](game/godot/gdscript.md)                 | Godot 专属脚本语言，语法、信号、协程、与引擎集成  |

---

### AI 工具

| 文档                       | 简介                                                     |
| -------------------------- | -------------------------------------------------------- |
| [GitNexus](ai/GitNexus.md) | 代码库知识图谱引擎，零服务器部署，辅助 AI 理解大型代码库 |
