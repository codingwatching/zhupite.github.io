---
layout: post
title: "AgentShield：在 MCP 和 AI Agent 扩展发布前做离线安全扫描"
categories: [sec]
description: "介绍 aiconnai/agentshield 的静态扫描、GitHub Action、VS Code 修复和实验性运行时 Guard，帮助团队在安装或发布 Agent 工具前发现命令注入、凭证泄露与 SSRF 等风险。"
tags:
  - AgentShield
  - MCP 安全
  - AI Agent 安全
  - Rust
  - 静态分析
---

> **项目辨析**：本文介绍的是 [aiconnai/agentshield](https://github.com/aiconnai/agentshield)，不是其他同名的 AgentShield 项目。官方仓库当前 README 标注版本为 `v1.0.1`，核心定位是离线优先的 Rust 安全扫描引擎。

## 一句话结论

AgentShield 适合放在 MCP Server、AI Agent 工具仓库和 Agent 配置进入团队或生产环境之前：它在本地分析源码、工具描述、依赖和配置，识别命令注入、凭证外泄、SSRF、不安全文件访问、动态代码执行、提示注入面和依赖风险，并可输出 SARIF 接入 GitHub Code Scanning。

它的稳定能力是**发布前静态扫描和策略评估**；运行时 Guard 和 MCP Proxy 目前仍是实验性功能，不能把它当作已经成熟的在线 Agent 防火墙或沙箱替代品。

## AgentShield 解决什么问题

MCP Server 和 Agent 扩展通常同时具备网络访问、文件操作、命令执行、依赖安装和外部数据读取能力。传统 SAST 工具可以覆盖部分代码问题，但未必理解 MCP 工具描述、Agent 配置、提示注入面和工具调用链。

AgentShield 将这些对象放到同一份扫描报告中，主要检查：

- **命令执行**：Shell、子进程和动态命令拼接。
- **凭证泄露**：环境变量、密钥文件、Token 和外发请求之间的关联。
- **SSRF 与元数据访问**：包括云实例元数据服务等高风险目标。
- **文件访问**：不安全路径、越权读取和敏感文件访问。
- **动态代码执行**：危险的解释执行和下载后执行链。
- **不安全反序列化**：例如不安全的 YAML 反序列化器。
- **提示注入面**：工具响应、日志、外部内容和 Agent 指令混用的风险。
- **依赖卫生**：未固定版本、危险依赖和供应链相关问题。
- **MCP 配置**：Server Manifest、工具名称、工具 Schema 和传输配置。
- **路径间数据流**：在 Python、TypeScript 等代码中跨函数跟踪不可信输入到危险 Sink 的流向。

官方 README 当前列出 **37 个内置上下文规则**，并支持通过 YAML 声明自定义规则。规则数量和覆盖范围会随版本变化，实际项目应以运行时 `agentshield list-rules` 的结果为准。

## 支持的对象和开发框架

| 对象或生态 | 扫描内容 |
|---|---|
| MCP Server | Manifest、工具 Schema、Python/TypeScript/JavaScript 源码、依赖和来源信息 |
| OpenClaw Skills | Skill 配置、指令文件和关联扩展面 |
| Hermes Agent | 配置、`mcp_servers`、`.hermes.md`、Skill 树和可选 MCP Manifest |
| CrewAI | Python 工具项目、依赖元数据和导入关系 |
| LangChain / LangGraph | 依赖、导入、工具实现和 `langgraph.json` |
| GPT Actions | Action 或 OpenAPI 风格的工具接口 |
| Cursor Rules | Cursor 规则文件和 Agent 指导内容 |
| 其他工具型 Agent | Browser Use、FastMCP、GitHub MCP Server、Playwright MCP 等工作流中的工具代码 |

语言支持并不等同于“所有语言都做同等深度的源码分析”。当前项目将 Python、TypeScript/TSX 和 Shell 列为主要扫描对象；官方 README 明确说明，Go、Ruby、Java 和 Rust 源文件在当前版本中不做执行级解析或统一执行面分析，可以与其他扫描器组合使用。

## 安装方式

### 方式一：下载官方安装脚本

macOS 和 Linux 可以使用项目 README 给出的安装命令：

```bash
curl -fsSL https://aiconnai.github.io/agentshield/install.sh | sh
```

生产环境不建议盲目执行远程脚本。更稳妥的做法是先下载脚本、审查内容，再在固定版本或经过校验的环境中运行。

### 方式二：Homebrew

```bash
brew tap aiconnai/tap
brew install agentshield
```

### 方式三：Cargo 从指定 Release 构建

项目官方 README 给出的版本安装命令是：

```bash
cargo install --git https://github.com/aiconnai/agentshield \
  --tag v1.0.1 --features full --force
```

`full` 特性会包含 Python、TypeScript、运行时代理和实验性 Runtime Guard。需要注意，AgentShield 是 Rust 项目，安装 Cargo/Rust 工具链后才能使用此方式。

### 方式四：从源码构建

```bash
git clone https://github.com/aiconnai/agentshield.git
cd agentshield
cargo build --features full --release
./target/release/agentshield scan /path/to/agent-extension
```

Windows 用户可以使用对应的 `.exe` 文件路径。官方 Release 页面提供 `x86_64-pc-windows-msvc` 等平台制品，优先使用与目标平台匹配的已发布二进制，可以减少本地编译依赖。

## 第一次扫描

进入待扫描的 Agent 或 MCP 项目目录，先运行：

```bash
agentshield quickstart
```

`quickstart` 会生成首次使用所需的配置建议、推荐 CI 设置、执行第一次扫描并解释结果。

也可以直接扫描当前目录：

```bash
agentshield scan . --ignore-tests --fail-on high --explain
```

参数含义如下：

| 参数 | 用途 |
|---|---|
| `.` | 扫描当前目录 |
| `--ignore-tests` | 忽略测试目录和测试文件，减少测试样例造成的噪声 |
| `--fail-on high` | 出现 High 或更高等级问题时让策略检查失败 |
| `--explain` | 输出扫描门禁、覆盖范围、置信度、分组发现和后续建议 |

建议第一次不要直接使用过于严格的门禁阻断所有提交，先用 `--explain` 理解项目发现，再决定哪些等级进入 CI 阻断策略。

## 选择报告格式

AgentShield 支持以下输出格式：

```bash
# 默认控制台输出
agentshield scan . --format console

# JSON，供脚本或其他平台处理
agentshield scan . --format json --output agentshield.json

# SARIF，接入 GitHub Code Scanning
agentshield scan . --format sarif --output results.sarif

# 独立 HTML 报告，便于分享和人工查看
agentshield scan . --format html --output report.html
```

格式选择建议：

| 场景 | 推荐格式 |
|---|---|
| 本地开发 | Console + `--explain` |
| 自动化解析 | JSON |
| GitHub 安全页 | SARIF |
| 安全评审和离线分享 | HTML |

## 接入 GitHub Actions

AgentShield 官方 README 和 `docs/mcp-security-scanner.md` 提供了以下工作流示例：

```yaml
name: Agent Security

on: [push, pull_request]

permissions:
  actions: read
  contents: read
  security-events: write

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: aiconnai/agentshield@main
        with:
          path: '.'
          fail-on: 'high'
          ignore-tests: true
          upload-sarif: true
```

在组织级生产流水线中，建议进一步固定 Action 到经过审核的版本或 Commit SHA，而不是长期直接跟随 `main`。同时确认仓库确实需要 `security-events: write`，避免为了上传 SARIF 而授予多余权限。

如果希望同时启用通用代码和密钥扫描，可以使用：

```bash
agentshield ci install --suite
```

官方说明该模式会生成包含 CodeQL、Gitleaks、Semgrep CE 和 AgentShield 的 GitHub Actions 工作流。实际启用前仍应检查生成文件中的 Action 版本、权限和触发范围。

## 配置项目策略

在项目根目录执行：

```bash
agentshield init
```

这会生成 `.agentshield.toml` 起始配置。官方示例中，策略可以设置扫描门禁和路径过滤：

```toml
[scan]
ignore_tests = true
include = ["src/**", "server/**"]
exclude = ["docs/**", "examples/**"]

[policy]
fail_on = "high"
```

路径过滤需要结合 `--explain` 检查实际生效范围。当 `include` 和 `exclude` 同时匹配时，官方文档说明 `exclude` 优先。

### 自定义声明式规则

将自定义规则放入 `.agentshield/rules/`，例如：

```yaml
id: "ORG-002"
name: "Banned Deprecated Library"
description: "Disallow telnetlib in agent tools"
severity: "medium"
attack_category: "supply_chain"
match:
  banned_dependencies:
    - name: "telnetlib"
      reason: "Insecure unencrypted remote protocol"
  tool_name_regex: "^telnet_"
```

再运行：

```bash
agentshield scan . --explain
```

自定义规则应当先在非阻断模式验证命中范围，避免规则过宽导致整个 Agent 仓库无法提交。

## 处理已知发现：Baseline 和 Suppression

### Baseline

已有安全问题的老项目不必一次性修完所有历史结果，可以先生成 Baseline：

```bash
agentshield scan --write-baseline .agentshield-baseline.json
agentshield scan --baseline .agentshield-baseline.json --explain
agentshield ci install --baseline .agentshield-baseline.json
```

Baseline 的作用是记录已知发现，让 CI 更关注新引入的问题。Baseline 文件本身应当纳入代码审查，避免有人通过重新生成文件把新增问题全部隐藏。

### Suppression

对于确认是误报、且有明确业务理由的发现，可以按 fingerprint 抑制：

```bash
agentshield suppress <fingerprint> \
  --reason "该工具只接收固定枚举值，已通过额外校验"

agentshield list-suppressions
```

抑制项应当设置负责人、理由和可选过期时间。不要用 suppression 代替修复真实的命令注入、凭证泄露或 SSRF 风险。

## 自动修复

AgentShield 的 `fix` 命令用于修复部分确定性问题，例如不安全反序列化器和未固定依赖版本。先预览差异：

```bash
agentshield fix . --dry-run
```

确认变更范围后再执行：

```bash
agentshield fix .
```

官方 README 还提供 VS Code 扩展，支持在编辑器中显示诊断，并通过灯泡操作执行部分一键修复。自动修复后仍应运行测试和重新扫描，因为安全工具不能替代业务语义验证。

## 运行时 Guard：当前应如何理解

AgentShield 的稳定契约是离线静态扫描和策略评估。`runtime-guard` 是可选特性：

```bash
cargo run --features runtime-guard -- guard --stdin
```

它可以读取一个 Runtime Event JSON，返回 `allow`、`warn` 或 `block` 判定。官方文档规定：`allow` 和 `warn` 返回退出码 0，`block` 返回退出码 3；无效 JSON、超过 1 MiB 的输入和不支持的调用会 fail-closed。

构建完整特性：

```bash
cargo build --features full --release
```

### MCP Proxy Guard

实验性的 MCP Proxy 位于 Agent Host 和 MCP Server 之间，检查 `tools/call`：

```text
Agent Host → AgentShield Proxy → MCP Server
                 ↓
          allow / warn / block
```

它只在工具请求进入下游 Server 前做策略判断，不执行工具逻辑。允许或警告的请求会继续转发；阻断请求不会到达下游 Server，并返回安全的 JSON-RPC 错误。使用时需要明确：

- Proxy Guard 仍是实验性实现。
- 兼容性和运行时保证尚未达到静态扫描的稳定程度。
- 它不是完整沙箱，也不能替代网络隔离、凭证最小化和人工审批。
- Runtime Event、工具参数、路径和 URL 可能包含敏感数据，日志前必须完成脱敏。

## 实际部署建议

### 发布前

1. 在 MCP Server 和 Agent 扩展仓库中执行 `agentshield scan`。
2. 检查 High/Critical 发现的证据、文件位置和数据流。
3. 对外部可控输入与 Shell、文件、网络 Sink 的链路做人工复核。
4. 生成 SARIF 并接入 Pull Request 检查。
5. 为已确认的历史问题建立 Baseline，而不是降低全部规则等级。

### 安装第三方 MCP Server 前

1. 固定仓库 Commit、Release 或镜像摘要。
2. 扫描 Manifest、工具 Schema、源码和锁文件。
3. 检查它需要的环境变量、文件目录和网络目的地。
4. 审查是否存在动态下载、任意 Shell、广泛文件读取和外发请求。
5. 使用最小权限的运行身份，在隔离环境中做首次运行。

### 进入生产前

1. 将 Agent 身份与开发者身份分离。
2. 使用任务范围内的短时凭证。
3. 对公网出站、云元数据、内部 API 和高影响写操作设置独立策略。
4. 对提示注入和工具响应投毒做对抗测试。
5. 将扫描结果、工具调用、审批和最终变更通过会话 ID 关联到审计系统。

## 项目边界

| 能力 | AgentShield 可以做什么 | 不能替代什么 |
|---|---|---|
| 静态扫描 | 在运行前发现代码、配置、依赖和工具 Schema 风险 | 不能证明运行时绝对安全 |
| SARIF/CI | 让风险进入 Pull Request 和 Code Scanning | 不能替团队决定风险是否可接受 |
| 自动修复 | 修复部分确定性的模式问题 | 不能替代测试和人工代码审查 |
| Runtime Guard | 实验性地评估运行时事件或 MCP 调用 | 不能替代完整沙箱、IAM 和网络防火墙 |
| 离线运行 | 源码可以留在本地，减少上传第三方服务 | 不代表运行中的 Agent 不会访问外部网络 |
| 自定义规则 | 用组织规则补充内置检测 | 规则质量仍由组织维护 |

## 与同名项目的辨析

GitHub 上存在多个名为 AgentShield 的仓库。本文目标项目是：

- **仓库**：`aiconnai/agentshield`
- **Crate**：`agent-shield`
- **主要语言**：Rust
- **许可证**：MIT OR Apache-2.0
- **核心能力**：离线静态扫描、策略评估、SARIF 输出、可选实验性运行时 Guard

站内已有文章提到的 `Yassin-H-Rassul / AgentShield` 是另一个项目，不能将其功能、版本、许可证或使用方式套用到本文介绍的仓库上。

## 项目状态

截至 2026-09-08 核验：

- 官方仓库 Star：18
- Fork：2
- 仓库公开 Commit：342（GitHub 页面数据，动态指标）
- 当前 Release：`v1.0.1`，2026-08-19 发布
- License：MIT OR Apache-2.0
- 默认分支：`main`
- 仓库创建时间：2026-02-14

GitHub API 在本次核验时触发了限流，因此 Star、Fork 和 Commit 数量以 GitHub 页面提取结果与搜索结果为准；版本、许可证、目录和命令则通过官方仓库页面、Release 页面和源码克隆交叉核对。项目主页和 Release 下载页可能随时间变化，使用时应以官方仓库为准。

## 结语

AgentShield 的实际价值在于把 Agent 特有的安全检查前移到“安装和发布之前”：先扫描 MCP Server、工具实现、Agent 配置和依赖，再决定是否允许进入 CI、编辑器或生产环境。

最推荐的使用方式不是单独依赖它，而是将它放进一条分层流水线：AgentShield 负责 Agent/MCP 语义相关的静态检查，CodeQL、Gitleaks、Semgrep 和依赖扫描器负责通用代码与供应链问题，运行时再用最小权限、网络控制、沙箱和审批策略限制实际影响范围。

## 参考资料

- [AgentShield 官方 GitHub 仓库](https://github.com/aiconnai/agentshield)
- [AgentShield v1.0.1 Release](https://github.com/aiconnai/agentshield/releases/tag/v1.0.1)
- [MCP Security Scanner 文档](https://github.com/aiconnai/agentshield/blob/main/docs/mcp-security-scanner.md)
- [Runtime Guard 文档](https://github.com/aiconnai/agentshield/blob/main/docs/RUNTIME_GUARD.md)
- [AgentShield LICENSE](https://github.com/aiconnai/agentshield/blob/main/LICENSE)
- [AgentShield Cargo.toml](https://github.com/aiconnai/agentshield/blob/main/Cargo.toml)
- [AgentShield 官方项目主页](https://aiconnai.github.io/agentshield/)

来源字段：2026-09-08｜aiconnai/agentshield 官方仓库及 Release｜https://github.com/aiconnai/agentshield｜介绍 AgentShield 的离线静态扫描、MCP/Agent 框架适配、GitHub Action、自动修复、SARIF 输出和实验性运行时 Guard｜适合在 Agent 工具安装、代码提交和发布前建立安全门禁，但不能替代运行时隔离、最小权限、网络控制和人工审批。
