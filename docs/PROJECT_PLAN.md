# GenStudio：面向 AI 应用后端实习的 AIGC 平台项目计划

> **文档性质：Living Document（持续更新）**  
> 这不是一次性方案，而是整个项目的“项目总纲 + Codex 工作上下文 + 学习路线 + 进度记录”。  
> 后续每完成一个里程碑、修改一个技术决策、遇到一个重要问题，都更新本文件。  
> 建议仓库建立后将本文件放到：`docs/PROJECT_PLAN.md`。

---

## 0. 项目当前状态

**项目名：** GenStudio  
**当前阶段：** Phase 0 — Bootstrap / Issue #1 准备  
**当前目标：** 建立一个可以长期演进、能够用于实习投递和面试讲解的 AIGC 应用后端项目。  
**第一目标岗位：** AI 应用后端开发实习生，尤其是类似剪映 / CapCut / 即梦 / Dreamina 一类 AIGC 产品团队。  

### 当前优先级

1. 不追求“功能多”，先追求“架构清楚、每一步能解释”。
2. 不追求“AI 模型很高级”，先把后端工程能力做扎实。
3. 不允许一次性让 AI 生成整个项目。
4. 每个重要模块必须经过：
   - 需求理解
   - 设计
   - 实现
   - 测试
   - Debug
   - 文档
   - Git 提交
5. 最终目标不是“项目能跑”，而是：
   - 我能解释它为什么这样设计；
   - 我能修改它；
   - 我能排查它；
   - 我能在面试中围绕它讨论工程问题。

---

# 1. 为什么要做这个项目

## 1.1 背景

目标岗位通常要求以下能力：

- 熟练掌握至少一门后端语言：Go / Java / Python / C++；
- 有实际工程开发能力；
- 熟悉数据库、缓存、网络、并发、API 设计；
- 理解 AI / LLM / 多模态 / AIGC 基本原理；
- 能将 AI 能力转化为真实产品能力；
- 有 Agent、SFT、LoRA、RL、生成模型经验属于加分项；
- 有独立项目、GitHub、开源贡献、完整工程经验会明显提高说服力。

问题在于：

> 单独学习 Go、Redis、PostgreSQL、Docker、Diffusion、ComfyUI、Agent 等知识，容易形成大量零散知识，却无法证明“能做工程”。

因此本项目采用 **“目标岗位反向设计项目”** 的方式：

```text
目标岗位 JD
   ↓
拆出核心能力
   ↓
设计一个真实 AIGC 产品后端
   ↓
在做项目过程中补知识
   ↓
形成 GitHub 可验证成果
   ↓
用于简历 / 面试 / 开源贡献
```

---

## 1.2 项目最终想证明什么

GenStudio 最终要证明的不是：

> “我会调用一个 Stable Diffusion API。”

而是：

> “我能设计并实现一个面向真实 AIGC 业务的后端系统。”

需要覆盖的能力包括：

```text
Go 工程开发
├── HTTP API
├── 项目分层
├── Context
├── 并发
├── 错误处理
├── 日志
└── 测试

数据层
├── PostgreSQL
├── 数据模型
├── SQL
├── Transaction
└── Migration

缓存 / 异步任务
├── Redis
├── Queue
├── Worker
├── Retry
├── Timeout
├── Idempotency
└── Dead Letter Queue

AIGC
├── Text-to-Image
├── Image-to-Image
├── Diffusion / DiT 基础
├── ComfyUI API
├── Model Provider
└── LoRA / Model Routing（后期）

生产化
├── Docker Compose
├── MinIO
├── SSE / WebSocket
├── Rate Limit
├── Quota
├── Metrics
├── CI
└── Benchmark
```

---

# 2. 项目定位

## 2.1 一句话介绍

**GenStudio is a production-oriented AIGC generation platform built with Go, supporting asynchronous generation jobs, model providers, real-time progress tracking, object storage and scalable workers.**

中文：

> GenStudio 是一个使用 Go 实现的生产型 AIGC 生成平台，支持异步生成任务、AI 模型调用、实时任务状态、对象存储和可扩展 Worker 架构。

---

## 2.2 项目不是做什么

以下内容暂时不是项目初期目标：

- 不从零训练大模型；
- 不自己实现 Diffusion；
- 不追求复杂前端；
- 不追求一开始就支持几十种模型；
- 不追求微服务；
- 不追求 Kubernetes；
- 不追求“为了技术而技术”。

原则：

> 只有当前规模出现真实问题以后，才引入更复杂的技术。

---

# 3. 产品视角：我们到底在做什么

用户进入 GenStudio 后可以：

1. 输入 Prompt；
2. 选择生成模型；
3. 创建图片生成任务；
4. 后端立即返回 `task_id`；
5. 用户可以查看任务状态；
6. Worker 异步执行图片生成；
7. 用户实时看到进度；
8. 生成结果存入对象存储；
9. 用户查看自己的历史生成记录；
10. 失败任务可以重试。

后期扩展：

- img2img；
- LoRA；
- Prompt Enhancement；
- 多模型路由；
- 多 GPU Worker；
- 用户额度；
- 优先级队列；
- 任务取消；
- 生成参数模板。

---

# 4. 总体架构

目标架构：

```text
                    ┌─────────────────┐
                    │   Web Client    │
                    │ Vue / React     │
                    └────────┬────────┘
                             │
                         HTTP / SSE
                             │
                             ▼
                    ┌─────────────────┐
                    │    Go API       │
                    │   GenStudio     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        PostgreSQL         Redis           MinIO
        用户/任务/历史     Cache/Queue     图片/资产
                              │
                              ▼
                       Generation Queue
                              │
                              ▼
                       ┌──────────────┐
                       │    Worker    │
                       └──────┬───────┘
                              │
                   ┌──────────┴──────────┐
                   ▼                     ▼
               ComfyUI              AI Provider
                   │                     │
                   ▼                     ▼
            SDXL / FLUX / ...      其他推理服务
```

后期：

```text
                  Scheduler
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Worker A    Worker B    Worker C
       GPU 0       GPU 1       GPU 2
```

---

# 5. 技术选型

## 5.1 后端语言：Go

原因：

- 与目标后端岗位高度匹配；
- 适合高并发 API 服务；
- goroutine / channel / context 很适合任务系统；
- 编译部署简单；
- 能明显体现后端工程能力。

Go 是本项目的 **主语言**。

---

## 5.2 AI 推理：第一阶段不自己写 Python 模型服务

第一版：

```text
Go Backend
    ↓ HTTP
ComfyUI API
    ↓
SDXL / FLUX
```

理由：

- 先集中训练后端能力；
- 模型服务和业务服务解耦；
- 避免一开始同时学习过多 Python 推理细节。

后期再增加：

```text
Go Backend
    ↓
Python FastAPI Inference Service
    ↓
Diffusers / PyTorch
```

---

## 5.3 PostgreSQL

负责：

- 用户；
- 生成任务；
- 生成历史；
- 模型信息；
- 资产元数据；
- 状态持久化。

需要学习：

- Schema 设计；
- Index；
- Transaction；
- Migration；
- EXPLAIN；
- SQL 优化基础。

---

## 5.4 Redis

负责：

- 异步任务队列；
- Cache；
- Rate Limit；
- 短期任务状态；
- Worker 心跳（后期）。

项目会利用 Redis 学习：

- List / Stream 等结构；
- Consumer；
- Retry；
- Duplicate delivery；
- Idempotency。

---

## 5.5 MinIO

负责存储：

- 图片；
- 缩略图；
- 后续可能的视频或生成资产。

目的：

理解：

> 数据库应该保存资产元数据，而大文件通常进入对象存储。

---

## 5.6 Docker Compose

本地开发最终通过：

```bash
docker compose up
```

启动：

- PostgreSQL
- Redis
- MinIO
- Go API
- Worker

后期可加入：

- ComfyUI
- Prometheus
- Grafana

---

# 6. Go 工程结构

初始结构：

```text
genstudio/
├── cmd/
│   ├── server/
│   │   └── main.go
│   └── worker/
│       └── main.go
│
├── internal/
│   ├── config/
│   ├── handler/
│   ├── service/
│   ├── repository/
│   ├── model/
│   ├── queue/
│   ├── worker/
│   ├── provider/
│   └── storage/
│
├── migrations/
├── deployments/
│   └── docker-compose.yml
├── docs/
├── .env.example
├── Makefile
├── go.mod
└── README.md
```

### 各目录职责

#### `handler`

HTTP 世界：

- 解析参数；
- 参数校验；
- 调用 Service；
- 转换 HTTP Response；
- HTTP Status Code。

不允许把核心业务逻辑塞进 Handler。

#### `service`

业务逻辑：

例如：

> 创建 generation 时应该做什么？

可能包括：

1. 校验用户；
2. 校验 model；
3. 创建 Task；
4. 写数据库；
5. 投递 Queue；
6. 返回 Task。

#### `repository`

持久化：

- INSERT；
- SELECT；
- UPDATE；
- Transaction。

Service 不应该直接到处写 SQL。

#### `model`

核心业务数据结构。

#### `queue`

消息队列抽象。

#### `worker`

消费生成任务。

#### `provider`

AI Provider 抽象，例如：

```text
ComfyUIProvider
LocalAIProvider
MockProvider
```

#### `storage`

对象存储抽象，例如：

```text
MinIOStorage
LocalStorage
```

---

# 7. 第一版领域模型

核心实体：

## GenerationTask

```text
id
user_id
prompt
negative_prompt
model
width
height
status
progress
result_url
error_message
created_at
started_at
finished_at
```

状态机：

```text
PENDING
   ↓
RUNNING
   ├──→ SUCCEEDED
   ├──→ FAILED
   └──→ CANCELED
```

未来可能加入：

```text
RETRYING
TIMEOUT
```

---

# 8. API 规划

## Phase 1

```http
GET /health
```

返回：

```json
{
  "status": "ok"
}
```

---

## Phase 2

创建任务：

```http
POST /api/v1/generations
```

示例：

```json
{
  "prompt": "A futuristic city at sunset",
  "model": "flux-dev",
  "width": 1024,
  "height": 1024
}
```

返回：

```json
{
  "task_id": "01J...",
  "status": "pending"
}
```

查询任务：

```http
GET /api/v1/generations/:id
```

历史记录：

```http
GET /api/v1/generations
```

取消：

```http
POST /api/v1/generations/:id/cancel
```

---

# 9. 项目路线

---

## Phase 0 — Bootstrap

目标：

建立规范工程骨架。

内容：

- GitHub repository；
- Go module；
- 目录结构；
- README；
- `.env.example`；
- Makefile；
- Docker Compose；
- `/health`；
- graceful shutdown；
- 基础 config。

验收：

```text
[ ] go run ./cmd/server 可以启动
[ ] GET /health → 200
[ ] Ctrl+C 可以优雅退出
[ ] PostgreSQL 可启动
[ ] Redis 可启动
[ ] README 能让别人成功启动项目
```

---

## Phase 1 — Database

目标：

正式引入 PostgreSQL。

内容：

- DB connection pool；
- Migration；
- generation_tasks 表；
- Repository；
- 基础 CRUD；
- Transaction 基础。

验收：

```text
[ ] migration 可执行
[ ] 可创建 GenerationTask
[ ] 可查询 GenerationTask
[ ] 数据可以持久化
```

---

## Phase 2 — Generation API

目标：

形成第一条完整业务链路：

```text
HTTP
 ↓
Handler
 ↓
Service
 ↓
Repository
 ↓
PostgreSQL
```

API：

```text
POST /generations
GET /generations/:id
GET /generations
```

此时还不真实生成图片。

Provider 使用：

```text
MockProvider
```

---

## Phase 3 — Redis Queue

目标：

把生成任务从同步模型变成异步任务。

流程：

```text
POST /generations
      ↓
写 DB
      ↓
Push Queue
      ↓
立即返回 task_id
```

学习：

- Queue；
- Producer；
- Consumer；
- at-least-once；
- 消息重复。

---

## Phase 4 — Worker

新增独立：

```text
cmd/worker
```

Worker：

```text
Pop Task
   ↓
PENDING → RUNNING
   ↓
Mock Generation
   ↓
RUNNING → SUCCEEDED
```

必须考虑：

- Worker crash；
- 重复任务；
- 状态冲突；
- graceful shutdown。

---

## Phase 5 — Real AIGC Provider

接入：

```text
ComfyUI
```

Go 通过 HTTP 调用。

学习：

- workflow；
- prompt；
- execution；
- polling / callback；
- 错误处理；
- timeout。

目标：

真实生成第一张图片。

---

## Phase 6 — Object Storage

接入 MinIO。

流程：

```text
ComfyUI
 ↓
生成结果
 ↓
Worker
 ↓
MinIO
 ↓
保存 URL
 ↓
PostgreSQL
```

---

## Phase 7 — Realtime Progress

使用：

```text
SSE
```

优先 SSE 而不是直接 WebSocket。

原因：

生成任务主要是：

```text
Server → Client
```

单向推送。

事件：

```text
queued
running
progress
succeeded
failed
```

---

## Phase 8 — Reliability

这是项目含金量的重要阶段。

加入：

### Retry

例如：

```text
max_attempts = 3
```

### Timeout

模型超时后任务失败。

### Idempotency

避免同一任务因为消息重复被生成两次。

### Dead Letter Queue

反复失败任务进入 DLQ。

### Rate Limit

防止单用户无限请求。

### Quota

例如：

```text
100 generations / day
```

---

## Phase 9 — Multi Worker / Scheduler

目标：

模拟真实 GPU 集群。

Worker 注册：

```text
worker_id
gpu
available_memory
supported_models
status
heartbeat
```

Scheduler：

```text
Generation Task
      ↓
选择合适 Worker
      ↓
执行
```

可以讨论：

- least loaded；
- model affinity；
- GPU memory；
- queue length。

这是非常适合面试讨论的模块。

---

## Phase 10 — Productionization

加入：

- Structured Logging；
- Metrics；
- Prometheus；
- Grafana；
- tracing 基础；
- integration test；
- load test；
- GitHub Actions；
- Docker Image；
- Benchmark。

---

# 10. 学习路线与项目路线绑定

原则：

> 不单独“学完再做”，而是做到哪里，学到哪里。

## Phase 0

学习：

- Go project layout；
- HTTP server；
- Context；
- signal；
- graceful shutdown；
- env config。

## Phase 1

学习：

- PostgreSQL；
- SQL；
- schema；
- migration；
- indexes。

## Phase 3–4

学习：

- Redis；
- Queue；
- Worker；
- goroutine；
- concurrency；
- delivery semantics。

## Phase 5

学习：

- Diffusion；
- SDXL / FLUX；
- ComfyUI workflow；
- AIGC inference pipeline。

## Phase 7

学习：

- HTTP streaming；
- SSE；
- WebSocket 对比。

## Phase 8

学习：

- retry；
- exponential backoff；
- idempotency；
- distributed systems failure。

## Phase 9

学习：

- distributed worker；
- heartbeat；
- scheduling；
- load balancing。

---

# 11. Git / GitHub 工作流

禁止：

```text
commit: update
commit: fix
commit: test
```

推荐 Conventional Commits：

```text
feat:
fix:
refactor:
test:
docs:
chore:
```

例：

```text
chore: initialize project structure
feat: add health check endpoint
feat: persist generation tasks
feat: enqueue generation jobs
fix: prevent duplicated task execution
test: add generation service tests
```

---

# 12. GitHub Issue 规划

初步 Issue 列表：

```text
#1  Bootstrap backend service
#2  Add PostgreSQL integration
#3  Define generation task model
#4  Implement generation REST API
#5  Add Redis job queue
#6  Implement generation worker
#7  Add task state machine
#8  Integrate ComfyUI provider
#9  Store generated assets in MinIO
#10 Add SSE task progress
#11 Add task retry policy
#12 Add timeout handling
#13 Add idempotent job execution
#14 Add rate limiting
#15 Add task cancellation
#16 Add worker heartbeat
#17 Add multi-worker scheduler
#18 Add metrics and monitoring
#19 Add CI pipeline
#20 Write production-ready documentation
```

每个 Issue 必须至少包含：

```text
Background
Goal
Non-goals
Design
Acceptance Criteria
Testing
```

---

# 13. Codex 的角色

Codex 是：

- Pair Programmer；
- Code Reviewer；
- Debug Assistant；
- Documentation Assistant。

Codex 不是：

> 一键项目生成器。

---

# 14. 给 Codex 的长期工作指令

将以下内容作为 Codex 项目上下文：

---

## Codex Working Agreement

You are assisting me in building **GenStudio**, a production-oriented AIGC backend platform written primarily in Go.

### Project goals

The goal is not merely to produce working code.  
The project is intended to train and demonstrate real backend engineering skills for AI application backend internship interviews.

Therefore:

1. Do not generate the entire project at once.
2. Work only on the current GitHub Issue / milestone.
3. Before making architectural changes, explain:
   - what problem is being solved;
   - why the change is needed;
   - what alternatives exist;
   - what trade-offs are involved.
4. Prefer the simplest architecture that satisfies the current requirements.
5. Do not introduce Redis, Kafka, microservices, Kubernetes, abstractions, design patterns, or dependencies unless they solve a concrete current problem.
6. Keep HTTP, business logic, persistence, queue logic and AI-provider logic separated.
7. Write code that is understandable to a student learning production backend engineering.
8. When generating code:
   - explain the role of each new file;
   - explain important APIs and Go concepts;
   - point out concurrency or lifecycle risks;
   - provide tests where appropriate.
9. When debugging:
   - first reproduce / identify the failure;
   - form hypotheses;
   - collect evidence;
   - only then modify code.
10. Never silently rewrite large portions of working code.
11. Prefer incremental commits.
12. At the end of each task, summarize:
   - what changed;
   - why;
   - how to test it;
   - what was learned;
   - remaining risks.

### Architecture principles

Current target architecture:

```text
Client
  ↓
Go API
  ↓
Service
  ↓
Repository / Queue / Provider
  ↓
PostgreSQL / Redis / ComfyUI / MinIO
```

Primary language: Go.

The model inference layer should initially be treated as an external service.  
The backend should interact with it through a Provider interface.

### Engineering priorities

Priority order:

1. Correctness
2. Understandability
3. Testability
4. Reliability
5. Performance
6. Sophistication

Do not optimize prematurely.

### Learning requirement

If a task uses a concept that I may not understand, explain it before or while implementing it.

Examples:

- `context.Context`
- graceful shutdown
- DB connection pool
- transaction
- Redis stream
- idempotency
- retries
- delivery semantics
- SSE
- worker heartbeat

The purpose is to make me capable of explaining the code independently in an interview.

---

# 15. 每次让 Codex 开始任务时使用的模板

```text
Read docs/PROJECT_PLAN.md first.

We are currently working on:

Issue: <issue>

Before writing code:

1. Inspect the existing repository.
2. Explain the current relevant architecture.
3. Explain what this issue needs to change.
4. Propose the smallest reasonable implementation.
5. List the files you expect to modify.

Do not implement until the plan is clear.

After implementation:

1. Run relevant tests.
2. Explain each important change.
3. Point out anything I should understand.
4. Suggest a commit message.
5. Update progress notes if necessary.
```

---

# 16. 面试价值

完成 Phase 4 后，可以讨论：

- REST API；
- PostgreSQL；
- Redis；
- async job；
- worker；
- Go concurrency。

完成 Phase 7 后，可以讨论：

- AIGC integration；
- object storage；
- SSE；
- real-time product experience。

完成 Phase 8 后，可以讨论：

- failure handling；
- retry；
- idempotency；
- timeout；
- rate limiting。

完成 Phase 9 后，可以讨论：

- distributed workers；
- scheduling；
- heartbeat；
- horizontal scaling。

最终理想简历描述：

> **GenStudio — AIGC Generation Backend Platform**
>
> - Built a Go-based asynchronous AIGC generation service supporting text-to-image workloads.
> - Designed PostgreSQL-backed task persistence and Redis-based asynchronous job processing.
> - Implemented independent generation workers and integrated ComfyUI/FLUX inference services.
> - Added object storage, SSE-based realtime task progress, retry, timeout and idempotent execution.
> - Extended the system toward multi-worker scheduling and production observability.

注意：只有真正实现的内容才能写进简历。

---

# 17. 参考开源项目的方式

本项目可以参考，但禁止照搬：

## AIdea Server

重点看：

- Go 工程结构；
- service；
- repo；
- Redis；
- queue；
- storage；
- AI API 组织。

## ComfyUI

重点看：

- workflow；
- execution；
- generation task；
- model backend。

## AI Horde

重点看：

- distributed worker；
- queue；
- scheduling；
- fault tolerance。

## InvokeAI

重点看：

- AIGC 产品功能；
- asset management；
- workflow；
- 用户体验。

## LocalAI

重点看：

- Go AI gateway；
- provider abstraction；
- multi-model backend。

原则：

```text
阅读
 ↓
理解问题
 ↓
理解设计
 ↓
自己重新设计
 ↓
实现
```

而不是：

```text
复制代码
 ↓
改名字
```

---

# 18. 文档持续更新规则

每次完成一个 Issue 后，本文件至少更新：

## Current Status

当前 Phase / Issue。

## Completed

已经完成什么。

## Decisions

新的技术决策以及原因。

## Problems Encountered

真实遇到的问题。

## Lessons Learned

真正学到的工程知识。

## Next Step

唯一明确的下一步。

---

# 19. Progress Log

## 2026-10-08

### Completed

- 明确项目目标：面向 AI 应用后端实习。
- 确定项目方向：生产型 AIGC Generation Backend。
- 确定主语言：Go。
- 确定初步技术栈：
  - Go
  - PostgreSQL
  - Redis
  - MinIO
  - Docker Compose
  - ComfyUI / FLUX
- 完成总体项目规划。
- 创建 GitHub 仓库：`Hahhhy/genstudio`。
- 在 WSL Ubuntu 中完成仓库 clone。
- 初始化 Go module：`github.com/hahhhy/genstudio`。
- 当前 Go 版本：`go1.26.5 linux/amd64`。
- 建立首批项目路径与占位文件。
- 完成首个提交：`1c0b124 chore: initialize project structure`。
- 已推送 `main` 分支到 GitHub。

### Engineering Note

Git 不会跟踪空目录。因此本地已经创建的：

```text
internal/config
internal/handler
internal/service
internal/repository
internal/model
migrations
```

在目录中没有文件以前不会出现在 GitHub，也不会出现在首个 commit 中。这是 Git 的正常行为，不需要为了“显示目录”而随意加入无意义占位文件；等对应模块真正开始实现时自然进入版本控制。

### Current

**Phase 0 — Bootstrap / Issue #1**

当前远程仓库已经具备最小骨架，`README.md`、`cmd/server/main.go` 等目前为空。

下一步：

```text
将 docs/PROJECT_PLAN.md 纳入仓库
↓
实现最小 Go HTTP Server
↓
实现 GET /health
↓
理解并实现 graceful shutdown
↓
补基础配置
↓
Docker Compose 启动 PostgreSQL + Redis
↓
补 README 启动说明
```

### Not Started

- `/health`
- graceful shutdown
- PostgreSQL / Redis Docker Compose
- config loading
- README 启动说明
- tests

---

# 20. 下一步任务

当前唯一任务：

> **完成 Issue #1：Bootstrap backend service。**

不要提前实现：

- Redis Queue；
- Worker；
- ComfyUI；
- MinIO；
- 用户系统；
- 前端。

完成仓库初始化后，再进入 Issue #1。

---

# 21. 最终原则

整个项目始终遵守：

```text
不是：
“AI 帮我写了一个项目”

而是：
“我借助 AI，完成了一个我真正理解的工程项目”
```

每一次技术选择都尽量做到：

```text
知道为什么
↓
知道怎么实现
↓
知道可能怎么坏
↓
知道怎么测试
↓
知道还有什么替代方案
```

这才是 GenStudio 最重要的产出。
