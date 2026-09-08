---
layout: post
title: "ArgusWatch AI 使用指南：自建一个面向 MSSP 的威胁情报平台"
categories: [tool]
description: "介绍 ArgusWatch AI 的功能边界、部署准备、情报源配置、Dashboard 使用和日常运维，帮助安全团队快速判断是否适合自建。"
tags:
  - ArgusWatch
  - 威胁情报
  - MSSP
  - Docker Compose
  - Ollama
---

> 本文根据微信公众号文章《33500行代码，47个情报源，这个AI威胁情报平台开源了》整理，并以 ArgusWatch AI 官方仓库文档为准。官方仓库 README 当前标注版本为 `v16.4.7`。本文中的命令和配置来自项目官方文件；由于本次写作环境未安装 Docker，未将部署命令描述为本地实测结果。

## 一句话介绍

[ArgusWatch AI](https://github.com/3sk1nt4n/arguswatch-ai) 是一个面向 MSSP 和安全运营团队的自托管威胁情报平台：它通过 Docker Compose 启动一组服务，汇聚公开和商业威胁情报源，将 IOC 与客户资产关联，再在 Dashboard 中展示发现、威胁演员、攻击活动、暴露面和修复动作。

如果你的需求是“把多个情报源集中起来，并按客户或资产查看关联结果”，它值得试用；如果只是偶尔查询一个 IP、域名或文件哈希，直接使用专门的 IOC 查询服务会更轻量。

## 它能做什么

根据官方 README，ArgusWatch AI 主要提供以下能力：

- **威胁情报采集**：内置多个 Collector，对接 NVD、CISA KEV、MITRE ATT&CK、ThreatFox、Feodo、OpenPhish、URLhaus、MalwareBazaar、crt.sh 等来源；部分高级来源需要 API Key。
- **IOC 识别与匹配**：从原始情报中识别多种 IOC 类型，并通过域名、子域名、IP、CIDR、关键词、技术栈、相似域名和云资产等策略关联客户。
- **客户与资产管理**：为客户登记域名、IP、邮箱域、技术栈等资产，随后将新采集的情报与这些资产关联。
- **发现与证据链**：在 Findings 页面查看关联发现、严重性、客户、原始证据和来源链接。
- **威胁演员与 Campaign**：查看威胁组织、攻击活动以及多个发现之间的关联。
- **暴露面评分**：在 Exposure 页面查看客户维度的暴露面评分和组成项。
- **修复动作**：根据 IOC 类型生成修复步骤、SLA、所需证据和责任角色。
- **本地或云端 AI**：默认可以使用 Ollama；也可以在 Dashboard 中切换到 Claude、GPT 或 Gemini，前提是配置对应 API Key。
- **多租户部署**：项目定位包含 MSSP 场景，数据库配置中使用 PostgreSQL 的 Row Level Security 进行租户隔离。

这些能力中，Collector、正则识别、匹配策略和评分属于确定性工程；AI 主要用于 IOC 调查路径、自然语言查询和分析文字生成。项目仓库提供了 [AGENTIC-AI-HONEST-ASSESSMENT.md](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/AGENTIC-AI-HONEST-ASSESSMENT.md)，可以用来了解哪些功能确实需要 LLM，哪些功能使用规则就足够。

## 部署前准备

官方 README 给出的基本前提是：

| 项目 | 要求 |
|---|---|
| 容器环境 | Docker Desktop 或 Rancher Desktop |
| 操作系统 | Windows、macOS 或 Linux，具体以 Docker 环境为准 |
| 内存 | 8GB 以上 |
| 磁盘 | 约 20GB |
| AI | 推荐原生安装 Ollama；也可以使用 Compose 中的 Ollama 容器 |
| 必填配置 | PostgreSQL 密码、JWT Secret、管理员密码 |

### Windows 安装 Docker Desktop

项目 README 建议 Windows 使用 Docker Desktop，并启用 WSL 2 后端。Docker Desktop 的官方下载地址是：

<https://www.docker.com/products/docker-desktop/>

### 安装 Ollama

项目推荐在宿主机原生安装 Ollama，这样更容易使用 GPU。Windows 官方 README 给出的安装命令是：

```powershell
winget install Ollama.Ollama
```

安装后拉取项目默认模型：

```powershell
ollama pull qwen3:8b
```

检查模型是否能够正常运行：

```powershell
ollama run qwen3:8b "say hello" --verbose
```

项目 README 将 `eval rate` 大于 40 tokens/s 作为 GPU 工作的参考，将低于 10 tokens/s 视为 CPU 推理参考。这个数值会受到硬件、模型版本和运行环境影响，适合用来做粗略判断，不应当当作固定性能承诺。

如果显存有限，可以在 `.env` 中改用项目列出的轻量模型，例如：

```dotenv
OLLAMA_MODEL=qwen3:4b
```

## 四步启动 ArgusWatch

以下步骤来自仓库根目录的 `README.md`、`.env.example` 和 `docker-compose.yml`。

### 1. 克隆仓库

```bash
git clone https://github.com/3sk1nt4n/arguswatch-ai.git
cd arguswatch-ai
```

### 2. 创建环境配置

```bash
cp .env.example .env
```

Windows 用户也可以直接复制文件：

```powershell
Copy-Item .env.example .env
```

至少要修改以下配置。不要把真实密码和 API Key 提交到 Git 仓库：

```dotenv
OLLAMA_URL=http://host.docker.internal:11434
OLLAMA_MODEL=qwen3:8b

POSTGRES_USER=arguswatch
POSTGRES_PASSWORD=请替换为强密码
POSTGRES_DB=arguswatch

AUTH_DISABLED=true
JWT_SECRET_KEY=请替换为随机字符串
ADMIN_USER=admin
ADMIN_PASSWORD=请替换为强密码
```

`AUTH_DISABLED=true` 适合本地试用，但不建议直接用于暴露在公网的生产环境。正式部署前应重新检查认证配置、反向代理、TLS 和访问控制。

### 3. 按需填写情报源 API Key

ArgusWatch 可以在没有任何高级 API Key 的情况下先使用部分公开来源。需要更广覆盖时，在 `.env` 中填写对应 Key，例如：

```dotenv
GITHUB_TOKEN=
OTX_API_KEY=
URLSCAN_API_KEY=
CENSYS_API_ID=
CENSYS_API_SECRET=
ABUSEIPDB_API_KEY=
VIRUSTOTAL_API_KEY=
SHODAN_API_KEY=
HIBP_API_KEY=
```

不要为了“启用更多 Collector”一次性收集所有 Key。建议先启用公开来源和自己已有账号的来源，确认数据质量、请求配额和合规范围后再增加配置。

### 4. 启动服务

```bash
docker compose up -d --build
```

首次启动会构建服务、初始化数据库并启动后台任务。根据官方 Compose 文件，核心服务包括：

- Backend：FastAPI API 和业务处理，容器端口 8000。
- Intel Proxy：情报源采集和 IOC 处理，容器端口 9000。
- PostgreSQL：数据存储，宿主机映射端口 5433。
- Redis：Celery 消息队列和缓存，宿主机映射端口 6380。
- Ollama：本地模型服务，宿主机映射端口 11435。
- Recon Engine：子域名、DNS 和证书相关发现。
- Celery Worker：后台处理任务。
- Celery Beat：定时任务。
- Nginx：边缘入口。
- Prometheus：指标和健康监控。

项目当前 Compose 文件将 Backend 映射到 `localhost:7777`：

```text
Dashboard: http://localhost:7777
API Docs:  http://localhost:7777/docs
```

项目还定义了 Nginx 入口：

```text
HTTP:       http://localhost:8080
HTTPS:      https://localhost:9443
Prometheus: http://localhost:9091
```

以实际 `docker compose ps` 输出为准。如果端口被占用，应修改 Compose 文件的宿主机映射端口，而不是修改容器内部端口。

## 启动后怎么用

### 1. 先检查服务状态

```bash
docker compose ps
```

查看 Backend 日志：

```bash
docker logs arguswatch-backend --tail 50
```

查看情报采集服务日志：

```bash
docker logs arguswatch-intel-proxy --tail 50
```

查看 Ollama 日志：

```bash
docker logs arguswatch-ollama --tail 20
```

确认服务通过健康检查后，再打开：

<http://localhost:7777>

### 2. 在 Customers 中登记客户

ArgusWatch 的主要使用顺序不是先搜索 IOC，而是先建立客户和资产范围：

1. 打开 **Customers** 页面。
2. 创建客户并填写行业等基本信息。
3. 添加客户域名、IP、邮箱域或其他资产。
4. 运行客户 Onboarding。
5. 等待资产发现、情报匹配和暴露面计算完成。
6. 返回 Overview、Findings 和 Exposure 页面查看结果。

项目提供了 `POST /api/customers/onboard` 作为一站式客户接入接口。如果使用 API 而不是 Dashboard，可以先打开 `/docs` 查看当前版本的 OpenAPI 参数定义，不建议凭旧文章或二手教程猜测请求体。

### 3. 查看 Overview

Overview 适合快速判断当前环境的总体情况，官方 Dashboard 页面包括：

- Threat Pressure Index。
- 严重性分布。
- Detection 时间线。
- IOC 类型分布。
- 客户和发现数量。

这里适合做每日巡检，但不应把总量直接当作风险结论。继续下钻到 Findings 查看具体客户、证据和来源。

### 4. 查看 Findings 和证据

Findings 页面是日常使用的核心页面。建议按以下顺序查看每条发现：

1. 确认关联客户和资产是否正确。
2. 查看 IOC 类型和原始情报。
3. 打开证据来源，例如 NVD、VirusTotal 或其他 Collector 提供的链接。
4. 查看匹配策略和匹配依据。
5. 判断是否是真实暴露、误报或需要人工复核。
6. 进入 Remediations 查看后续处理步骤。

项目强调每条发现都要能够回到来源和匹配依据。实际运营时，仍应由分析师确认资产归属和业务影响，不能只依据 AI 生成的文字自动关闭或升级告警。

### 5. 使用 AI Bar 和 Chat

AI Bar 用于从一个 IOC 出发进行调查。根据项目的官方说明，它会先识别 IOC，再检查相关情报源，随后由模型决定是否继续查询客户发现、暴露面、关联邮箱、威胁演员、暗网或修复记录，最后生成分析摘要。

Chat Agent 用于用自然语言查询实时数据库，例如查询某个客户的高危发现、某类 IOC 的数量或近期相关威胁。使用本地 Ollama 时，第一次 AI 请求可能需要等待模型加载；如果模型没有 GPU 加速，响应时间会明显增加。

在 Dashboard 顶部切换模型前，先确认对应配置已经写入 `.env`：

```dotenv
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GOOGLE_AI_API_KEY=
```

本地模型适合处理对隐私敏感、调用量较大的分析；云端模型适合临时提升响应速度或推理能力。无论使用哪种模型，都要检查其输入是否包含不应外发的客户数据、凭证或内部资产信息。

### 6. 查看 Exposure、Actors 和 Campaigns

- **Exposure**：查看客户暴露面评分及各维度组成，用来确定优先处理对象。
- **Actors**：查看威胁演员、所属国家、TTP 等信息。
- **Campaigns**：查看多个发现是否被归为同一个攻击活动。
- **Dark Web**：查看勒索软件、Paste 和其他暗网相关结果。
- **Remediations**：查看技术处置步骤、SLA、状态和责任角色。

这些页面适合做调查上下文补充，最终处置仍应结合资产负责人、漏洞验证、日志和业务影响进行判断。

## 常用运维命令

```bash
# 启动
docker compose up -d

# 代码或镜像变化后重新构建
docker compose up -d --build

# 查看服务状态
docker compose ps

# 停止服务，但保留数据
docker compose down

# 查看日志
docker logs arguswatch-backend -f --tail=50
docker logs arguswatch-intel-proxy -f --tail=50
docker logs arguswatch-ollama -f --tail=20

# 重新构建 Backend
docker compose build --no-cache backend && docker compose up -d

# 清空数据并重新部署，谨慎使用
docker compose down -v && docker compose up -d --build
```

最后一条命令会删除 Compose 数据卷，适合实验环境重置，不适合生产环境日常维护。

## API 快速入口

官方 README 列出了部分常用接口：

| 接口 | 用途 |
|---|---|
| `POST /api/customers/onboard` | 一站式接入客户 |
| `POST /api/match-intel-all` | 执行情报与客户资产匹配 |
| `POST /api/ai-triage?limit=5` | 批量执行 AI 分诊 |
| `POST /api/collect-all` | 触发 Collector |
| `GET /api/findings` | 查询发现 |
| `GET /api/findings/{id}` | 查询发现详情和证据链 |
| `GET /api/detections` | 查询原始检测结果 |
| `GET /api/actors` | 查询威胁演员 |
| `GET /api/collectors/status` | 查看 Collector 状态和 IOC 数量 |
| `GET /api/settings/ai` | 查看 AI 提供商状态 |
| `POST /api/settings/active-provider` | 切换 AI 提供商 |

不同版本的请求字段可能变化。使用前请访问 `http://localhost:7777/docs`，以当前运行版本的 OpenAPI 文档为准。

## 适合谁，不适合谁

### 适合

- 需要自建威胁情报工作台的 MSSP。
- 希望将多个公开情报源和已有商业情报源统一管理的安全团队。
- 需要按客户、资产和租户隔离查看发现的服务商。
- 想让 AI 辅助 IOC 调查和报告撰写，但仍保留人工判断环节的 SOC 团队。
- 希望研究一个完整 Docker Compose 安全平台的人。

### 不太适合

- 只需要临时查询 IOC 的个人用户。
- 没有 Docker、数据库和基础运维能力的团队。
- 需要厂商正式 SLA、商业支持和成熟合规交付的生产环境。
- 希望开箱即用获得全部高级情报源的人：不少 Collector 需要单独申请 API Key，且受供应商配额约束。
- 不能接受初始数据采集、模型下载和多服务维护成本的环境。

## 使用前的三个检查

1. **先确认数据合规**：公开情报源、暗网采集、客户资产扫描和第三方 API 的使用范围需要符合组织政策与当地法律。
2. **先用测试客户验证误报**：项目预置或示例中的客户资产不等于你的授权范围，接入真实客户前应使用自有、明确授权的资产进行验证。
3. **先锁定版本和备份策略**：项目 README、Compose 文件和模型配置会变化；升级前保存 `.env`、数据库卷和自定义配置，避免直接使用 `down -v`。

## 项目状态与许可证

截至本文核验时：

- 官方仓库：<https://github.com/3sk1nt4n/arguswatch-ai>
- README 标注版本：`v16.4.7`
- 主要语言：Python
- 默认分支：`main`
- GitHub API 返回许可证：MIT License
- GitHub API 返回仓库 Star 数：68（动态数据，核验时间为 2026-09-08）
- GitHub API 返回 Fork 数：17（动态数据，核验时间为 2026-09-08）
- 仓库创建时间：2026-03-05

Star、Fork、版本和 Collector 数量都可能随仓库更新变化，使用时应以官方仓库当前内容为准。

## 结语

ArgusWatch AI 的价值不只是把若干情报源放在一个页面中，而是把“采集—匹配—发现—调查—评分—修复”的工作流串起来，并为 MSSP 场景提供客户和租户维度的组织方式。

实际落地时，建议先从最小配置开始：原生 Ollama、公开情报源、一个明确授权的测试客户，先验证 Dashboard、匹配结果和误报处理流程，再逐步接入 VirusTotal、Shodan、Censys 等需要 Key 的服务。这样比一开始启用全部 Collector，更容易判断项目是否真正适合你的运营流程。

## 参考资料

- [微信公众号原文：33500行代码，47个情报源，这个AI威胁情报平台开源了](https://mp.weixin.qq.com/s/7JJV4fdCG2mH55xm3YPceQ)
- [ArgusWatch AI 官方 GitHub 仓库](https://github.com/3sk1nt4n/arguswatch-ai)
- [官方 README](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/README.md)
- [官方环境变量示例](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/.env.example)
- [官方 Docker Compose 配置](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/docker-compose.yml)
- [AI 能力诚实评估](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/AGENTIC-AI-HONEST-ASSESSMENT.md)
- [MIT License](https://github.com/3sk1nt4n/arguswatch-ai/blob/main/LICENSE)

来源字段：2026-09-08｜微信公众号文章、ArgusWatch AI 官方 GitHub｜<https://mp.weixin.qq.com/s/7JJV4fdCG2mH55xm3YPceQ>｜介绍 ArgusWatch AI 的定位、部署和使用流程｜适合需要自建威胁情报工作台的 MSSP 与 SOC 团队，但应先验证数据合规、误报率和运维成本。
