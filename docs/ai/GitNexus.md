# GitNexus

## 简介

GitNexus 是一个零服务器的代码智能引擎，可以将任何代码库转换为 AI 可理解的知识图谱。它通过索引代码库中的每个依赖、调用链、集群和执行流程，帮助 AI 智能体深度理解代码结构。

### 主要特性

- **零服务器架构** - 完全在浏览器端运行的客户端知识图谱构建工具
- **深度代码理解** - 索引依赖关系、调用链、符号和执行流程
- **AI 智能体集成** - 内置图 RAG 智能体，支持 Claude Code、Cursor、Windsurf 等
- **交互式知识图谱** - 可视化代码结构和关系
- **快速索引** - 智能增量更新机制

---

## 快速开始

### 方式一：CLI 模式（推荐）

#### 1. 安装

**全局安装：**

```bash
npm install -g gitnexus
```

**或使用 npx（无需安装）：**

```bash
npx gitnexus --version
```

#### 2. 配置编辑器（一次性操作）

在任何目录下运行：

```bash
npx gitnexus setup
```

此命令会：

- 自动检测已安装的 AI 编辑器
- 配置 MCP（Model Context Protocol）
- 使 GitNexus MCP 服务器在所有项目中可用

支持的编辑器：

- Claude Code（最深度集成：MCP + 技能 + Hook）
- Cursor
- Windsurf
- OpenCode

#### 3. 索引代码库

进入你的项目目录：

```bash
cd /path/to/your/project
npx gitnexus analyze
```

这一条命令会完成：

- 索引整个代码库
- 安装智能体技能文件
- 注册 Claude Code hooks
- 创建 `AGENTS.md` 和 `CLAUDE.md` 上下文文件

---

### 方式二：Web 模式（浏览器端）

1. 访问 GitNexus Web 界面
2. 拖入 GitHub 仓库 URL 或 ZIP 文件
3. 等待浏览器完成索引
4. 使用内置的图 RAG 智能体进行代码探索

---

## 核心命令详解

### analyze - 索引仓库

**基础用法：**

```bash
gitnexus analyze [path]
```

**常用选项：**

| 选项                | 说明                                    |
| ------------------- | --------------------------------------- |
| `--force`           | 强制完整重建索引                        |
| `--skip-embeddings` | 跳过 embedding 生成（更快）             |
| `--embeddings`      | 启用 embedding 生成（更慢，搜索更准确） |
| `--skills`          | 从检测到的社区生成仓库特定的技能文件    |
| `--skip-agents-md`  | 保留自定义的 AGENTS.md/CLAUDE.md 内容   |
| `--verbose`         | 显示详细日志，包括跳过的文件            |

**示例：**

```bash
# 快速索引（跳过 embeddings）
npx gitnexus analyze --skip-embeddings

# 强制完全重建索引
npx gitnexus analyze --force

# 生成技能文件
npx gitnexus analyze --skills

# 索引指定路径
npx gitnexus analyze /path/to/repo
```

### setup - 配置编辑器

```bash
gitnexus setup
```

一次性配置命令，自动为所有支持的 AI 编辑器写入正确的全局 MCP 配置。

### mcp - 启动 MCP 服务器

```bash
gitnexus mcp
```

为所有已索引的仓库启动 MCP 服务器（stdio 模式）。

### list - 列出已索引的仓库

```bash
gitnexus list
```

显示所有已建立索引的代码库。

### status - 查看索引状态

```bash
gitnexus status
```

显示当前仓库的索引状态和统计信息。

### clean - 删除索引

```bash
gitnexus clean
```

删除当前仓库的索引数据。

---

## 使用技巧

### 1. 内存管理

对于大型仓库（如 Linux 内核），GitNexus 会自动分配最多 8GB 堆内存。如需自定义：

```bash
NODE_OPTIONS="--max-old-space-size=16384" npx gitnexus analyze
```

### 2. 增量更新

GitNexus 支持智能增量更新。再次运行 `analyze` 命令只会更新发生变化的部分：

```bash
npx gitnexus analyze
```

### 3. 跳过 Embeddings 加速

如果不需要语义搜索功能，可以跳过 embeddings 生成：

```bash
npx gitnexus analyze --skip-embeddings
```

### 4. 自定义 AGENTS.md

如果你手动编辑了 `AGENTS.md` 或 `CLAUDE.md`，使用 `--skip-agents-md` 保留修改：

```bash
npx gitnexus analyze --skip-agents-md
```

---

## 与 AI 编辑器集成

### Claude Code 集成（最佳体验）

GitNexus 为 Claude Code 提供最深度的集成：

1. **MCP 工具** - 通过 MCP 协议访问知识图谱
2. **智能体技能** - 自动安装仓库特定的技能
3. **PreToolUse Hooks** - 在工具调用前提供上下文

**使用流程：**

```bash
# 1. 配置（仅首次）
npx gitnexus setup

# 2. 索引项目
cd /path/to/your/project
npx gitnexus analyze

# 3. 在 Claude Code 中打开项目
# AI 将自动获得代码库的深度理解能力
```

### Cursor / Windsurf 集成

通过 MCP 协议集成：

```bash
# 配置
npx gitnexus setup

# 索引
npx gitnexus analyze
```

---

## 索引输出文件

索引完成后，会在项目根目录生成：

### AGENTS.md

包含面向 AI 智能体的代码库概览，包括：

- 项目架构说明
- 关键模块和依赖
- 代码约定和模式

### CLAUDE.md

Claude Code 专用的上下文文件，包含：

- 代码库结构
- 重要符号索引
- 执行流程图

### .gitnexus/

索引数据目录（自动管理，无需手动修改）

---

## 最佳实践

### 1. 项目初始化时索引

在克隆新项目后立即索引：

```bash
git clone https://github.com/username/repo.git
cd repo
npx gitnexus analyze
```

### 2. 定期更新索引

在重大代码变更后重新索引：

```bash
git pull
npx gitnexus analyze
```

### 3. 针对大型项目优化

对于超大型项目：

```bash
# 跳过 embeddings，仅索引结构
npx gitnexus analyze --skip-embeddings --verbose
```

### 4. 生成技能文件

让 AI 更好地理解项目特定的模式：

```bash
npx gitnexus analyze --skills
```

---

## 常见问题

### Q: 索引需要多长时间？

A: 取决于项目大小：

- 小型项目（< 100 文件）：几秒钟
- 中型项目（1000 文件）：1-2 分钟
- 大型项目（10000+ 文件）：5-15 分钟

### Q: 支持哪些编程语言？

A: GitNexus 支持主流编程语言，包括：

- JavaScript/TypeScript
- Python
- Java
- Go
- Rust
- C/C++
- 等等

### Q: 索引数据存储在哪里？

A: 索引数据存储在项目的 `.gitnexus/` 目录中，可以安全地添加到 `.gitignore`。

### Q: 如何删除索引？

A: 使用 clean 命令：

```bash
gitnexus clean
```

### Q: MCP 配置失败怎么办？

A: 重新运行 setup 命令：

```bash
npx gitnexus setup --force
```

---

## 资源链接

- **GitHub 仓库**: https://github.com/abhigyanpatwari/GitNexus
- **NPM 包**: https://www.npmjs.com/package/gitnexus
- **官方文档**: https://www.mintlify.com/abhigyanpatwari/GitNexus

---

## 总结

GitNexus 通过将代码库转换为 AI 可理解的知识图谱，显著提升了 AI 编程助手的代码理解能力。

**核心工作流：**

```bash
# 1. 一次性配置
npx gitnexus setup

# 2. 每个项目索引一次
cd /path/to/project
npx gitnexus analyze

# 3. 在 AI 编辑器中享受增强的代码理解能力
```

开始使用 GitNexus，让 AI 真正理解你的代码！
