# Flutter & Dart Agent Skills

Flutter/Dart Agent Skills 是由 Flutter 团队和 Dart 团队官方维护的一套 AI 代理技能库，为 AI 编程助手（Claude Code、Cursor、GitHub Copilot 等）提供标准化的任务蓝图和领域知识，使其能够遵循官方最佳实践完成特定的开发任务。

**与其他 AI 能力的对比：**

| 能力             | 作用                                                   |
| ---------------- | ------------------------------------------------------ |
| **Rules 文件**   | 配置 AI 代理的通用行为规则，作用于所有任务             |
| **MCP Server**   | 提供工具调用能力（运行代码、检查 Widget 树、搜索包等） |
| **Agent Skills** | 为特定任务提供逐步执行的专业指令蓝图                   |

三者协同工作：MCP Server 提供"手"，Rules 文件设定"习惯"，Agent Skills 提供"知识"。

**渐进式加载（Progressive Disclosure）：**

Agent Skills 采用类似 Flutter Deferred Loading 的设计——AI 代理首先只读取技能的元数据（Metadata），只有在实际需要执行某项任务时才加载完整指令，最大程度减少上下文窗口的消耗。

---

## 官方仓库

| 仓库           | 地址               | 维护方       |
| -------------- | ------------------ | ------------ |
| Flutter Skills | `flutter/skills`   | Flutter 团队 |
| Dart Skills    | `dart-lang/skills` | Dart 团队    |

---

## 安装与配置

### 前置要求

安装 `skills` CLI 工具需要 [Node.js](https://nodejs.org/)，技能文件会放入项目的 `.agents/skills/` 目录，兼容大多数主流 AI 代理。

### 安装 Flutter Skills

```bash
# 安装全部 Flutter 技能（推荐）
npx skills add flutter/skills --skill '*' --agent universal

# 只安装特定技能
npx skills add flutter/skills --skill 'flutter-build-responsive-layout' --agent universal
npx skills add flutter/skills --skill 'flutter-setup-declarative-routing' --agent universal
```

### 安装 Dart Skills

```bash
# 安装全部 Dart 技能
npx skills add dart-lang/skills --skill '*' --agent universal

# 只安装特定技能
npx skills add dart-lang/skills --skill 'dart-add-unit-test' --agent universal
```

### 更新技能

```bash
npx skills update
```

### 安装后的目录结构

```
your-flutter-project/
└── .agents/
    └── skills/
        ├── flutter-build-responsive-layout/
        │   └── SKILL.md
        ├── flutter-setup-declarative-routing/
        │   └── SKILL.md
        ├── dart-add-unit-test/
        │   └── SKILL.md
        └── ...
```

### 如何让 AI 代理使用技能

安装后，直接向 AI 代理提问：

```
# 询问有哪些可用技能
"请查看 .agents/skills 目录，告诉我哪些技能可以帮助我完成当前任务？"

# 让代理总结可用能力
"总结一下我已安装的所有技能的功能。"

# 使用特定技能
"使用 flutter-build-responsive-layout 技能，帮我把当前页面改为响应式布局。"
```

---

## Dart & Flutter MCP Server

MCP Server 与 Agent Skills 配合使用，提供工具调用能力。

### 主要能力

| 能力         | 说明                                |
| ------------ | ----------------------------------- |
| 分析代码     | 运行 `dart analyze`，获取错误和警告 |
| 符号解析     | 查找类/函数定义，获取文档注释       |
| Widget 树    | 检查运行中应用的 Widget 层级        |
| 热重载       | 保存修改后自动重载应用              |
| pub.dev 搜索 | 搜索并推荐合适的 Dart 包            |
| 依赖管理     | 添加/更新 pubspec.yaml 中的依赖     |
| 测试运行     | 执行测试并返回结果                  |
| 代码格式化   | 运行 `dart format`                  |

### 配置方式

**Claude Code：**

```bash
claude mcp add --transport stdio dart -- dart mcp-server
```

**Cursor（~/.cursor/mcp.json）：**

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

**VS Code（settings.json）：**

```json
{
  "dart.mcpServer": true
}
```

**Gemini CLI（~/.gemini/settings.json）：**

```json
{
  "mcpServers": {
    "dart": {
      "command": "dart",
      "args": ["mcp-server"]
    }
  }
}
```

---

## AI Rules 文件配置

除 Agent Skills 外，还可以通过 Rules 文件约束 AI 的整体行为风格。

**各编辑器对应文件：**

| 编辑器                   | Rules 文件位置                    | 字数限制  |
| ------------------------ | --------------------------------- | --------- |
| Claude Code              | `CLAUDE.md`                       | 无硬限制  |
| Cursor                   | `AGENTS.md`                       | 无硬限制  |
| GitHub Copilot (VS Code) | `.github/copilot-instructions.md` | ~4k 字符  |
| JetBrains AI (Junie)     | `.junie/guidelines.md`            | 无硬限制  |
| Gemini CLI               | `GEMINI.md`                       | 1M+ Token |
| VS Code                  | `.instructions.md`                | 未知      |

**下载 Flutter 官方 Rules 模板：**

Flutter 团队维护了四个尺寸版本以适配不同的字符限制：

- `rules.md`（完整版）
- `rules_10k.md`（< 10k 字符）
- `rules_4k.md`（< 4k 字符）
- `rules_1k.md`（< 1k 字符）

```bash
# 下载完整版
curl -o CLAUDE.md https://raw.githubusercontent.com/flutter/flutter/refs/heads/main/docs/rules/rules.md
```

---

## 综合使用工作流

将 Agent Skills + MCP Server + Rules 文件结合使用的典型场景：

```
# 1. 项目初始化
npx skills add flutter/skills --skill '*' --agent universal
npx skills add dart-lang/skills --skill '*' --agent universal
claude mcp add --transport stdio dart -- dart mcp-server

# 2. 向 AI 代理提问（Claude Code 示例）
"请查看 .agents/skills 目录，使用 flutter-apply-architecture-best-practices
技能，帮我重构 HomeScreen，按照分层架构拆分 ViewModel 和 Repository。"

"使用 flutter-build-responsive-layout 技能，把 ProductList 改成在平板上
显示两列、手机上显示一列的自适应布局。"

"使用 flutter-add-widget-test 技能，为 LoginForm 编写测试，
覆盖空输入验证和成功登录两个场景。"

"使用 flutter-use-http-package 和 flutter-implement-json-serialization 技能，
帮我实现 /api/products 接口的数据层，包括模型类和 Repository。"
```

---

## 最佳实践

1. **Skills + MCP 结合使用**：Skills 提供知识蓝图，MCP Server 提供工具执行能力，两者结合让 AI 代理既"知道怎么做"又"能实际执行"。

2. **按需安装技能**：不需要全部安装，按项目实际需要选择，减少上下文开销。

3. **明确告知使用技能**：提问时明确指出使用哪个技能（如"使用 flutter-apply-architecture-best-practices 技能"），AI 会加载完整指令而非猜测。

4. **定期更新**：Flutter 和 Dart 团队会持续改进技能内容，定期运行 `npx skills update` 保持最新。

5. **配合 Rules 文件**：Skills 处理具体任务，Rules 文件约束整体代码风格（命名规范、注释规范、禁用模式等），两者互补。

6. **验证 AI 输出**：Skills 提供最佳实践指导，但 AI 生成的代码仍需人工审查，特别是架构决策和安全相关代码。
