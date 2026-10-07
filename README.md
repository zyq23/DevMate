# DevMate

DevMate 是一套前后端完整的研发协作助手：把 GitHub 仓库的 Issue、PR、CI 等数据接入本地，再结合项目知识库与工作区源码，交给多个 AI Agent 做分析并给出结论。所有涉及写入的操作只会生成待人工确认的草稿（ActionDraft），系统本身绝不直接改动 GitHub。

## 核心能力

**数据接入**

- 自动拉取仓库的 Issue、PR（含变更文件与评审意见）、CI 工作流运行记录、任务及失败日志；支持定时轮询和 Webhook 两种增量方式。
- 提供工作区工具，可在本地 checkout 内安全地列目录、读文件、搜代码；配套项目索引与代码图谱。

**AI 分析**

- Issue 自动打标：归类、评估优先级与复杂度、建议指派人选、提示疑似重复项、拆出行动清单。
- PR 审查产出概览：变更摘要、高风险点、自查清单、补充测试建议，并标出值得人工细看的文件。
- CI 失败解读：判断失败类别、推断可能根因、给出逐步排查路径及关联上下文。
- 按周期汇总研发活动，生成工程周报。
- 用户在页面上给出负反馈时，可推送到飞书群机器人（可选配置）。

**Agent 体系**

- 对话主 Agent `ChatAgent` 基于模型原生工具调用能力工作，可按需调度工作区检索、记忆、RAG、Issue / PR / CI 查询、周报与安全草稿等工具。
- 复杂任务交给 `WorkflowOrchestrator` 多 Agent 流水线：规划者拆解任务，各领域专用 Agent 分头分析，观察者盯过程，综合者汇总结论，形成闭环。
- Skill Runtime 负责按需装载 `SKILL.md`：用户可以点名指定，也可以按触发条件自动选择；只会给当前任务注入对应指令与受限资源。显式调用会校验允许的工具清单，多 Agent 场景下各专用 Agent 独立装载自己的 Skill；Skill 版本、激活原因、指令摘要与执行校验全部写入 trace。内置 6 个 Skill：`issue-triage`、`pr-review`、`ci-debug`、`weekly-report`、`safety-draft`、`project-investigation`。

**RAG 知识问答**

- 内置完整的知识库问答闭环：建库向导 → 文档解析 → 按段落语义切片 → 向量化 → 向量与关键词混合检索 → 重排 → 召回调参测试 → 问答工作室，最终答案附带编号引用。
- 文档格式覆盖 TXT、Markdown、PDF、DOCX、JSON、CSV 与日志文件；同一文件按 SHA-256 去重。
- 问答分两层检索：调用 Agent 前，`ContextAssembler` 先按用户原始问题查一轮证据注入上下文；推理过程中模型还能再次调用检索工具，改写关键词多轮补查，对照多份结果相互印证后再作答。

**评测**

- 对话质量可量化：以生产配置在隔离会话里真实跑通 ChatAgent 全链路并采集证据，用 Ragas 打分（上下文精确率 / 召回率、忠实度、答案相关性、目标达成率），叠加硬规则与工具调用校验，支持同一基线做回归。

## 技术栈

| 层 | 技术 |
| --- | --- |
| 后端 | Python、FastAPI、Pydantic、SQLAlchemy、PostgreSQL（pgvector）、httpx、LangChain |
| Agent 编排 | 原生工具调用 + 专用分析 Agent + Skill Runtime |
| RAG | Milvus 向量库、混合检索、重排（Rerank） |
| 前端 | Next.js 15、TypeScript、Tailwind CSS、React Query |
| 评测 | Ragas 0.4 Collections API |
| 基础设施 | Docker Compose、`.env` 配置 |

## 架构

```mermaid
flowchart TD
    FE["Next.js 前端"] --> API["FastAPI 后端"]
    API --> DB["PostgreSQL (pgvector)"]
    API --> Milvus["Milvus 向量库"]
    API --> GH["GitHub REST API"]
    API --> LLM["OpenAI 兼容 LLM API"]
    API --> Conv["ChatAgent 对话主 Agent"]
    Conv --> Workflow["多 Agent 编排器"]
    Workflow --> Planner["Planner 规划"]
    Workflow --> Observer["Observer 观察"]
    Workflow --> Synthesis["Synthesis 综合"]
    Conv --> Workspace["工作区工具"]
    Conv --> Memory["记忆 / RAG 工具"]
    Conv --> Issue["Issue 分析 Agent"]
    Conv --> PR["PR 审查 Agent"]
    Conv --> CI["CI 排障 Agent"]
    Conv --> Report["周报 Agent"]
    Conv --> Safety["安全 Agent"]
    Conv --> Skills["Skill 注册中心"]
```

## 环境要求

| 依赖 | 要求 | 说明 |
| --- | --- | --- |
| Python | **≥ 3.10** | 依赖 `langchain 1.x` / `langgraph 1.x` / `pydantic 2.10`，Python 3.8 装不上；建议 3.10–3.12 |
| Node.js | ≥ 18.18 | Next.js 15 要求 |
| Docker | + Compose v2 | 提供 PostgreSQL、etcd、MinIO、Milvus（Redis 为可选 profile） |
| LLM API Key | 必需（核心功能） | 任意 OpenAI 兼容接口；`.env.example` 默认 DeepSeek。Agent 对话、RAG 生成与评测 Judge 都依赖它 |
| GitHub Token | 可选 | 同步真实 GitHub 仓库数据时需要 |

默认端口一览：后端 `8000`、前端 `3000`、PostgreSQL `5432`、MinIO `9000/9001`、Milvus `19530`、etcd `2379`。

## 快速开始

**1. 配置环境变量**

```bash
cp .env.example .env
# 编辑 .env：至少填写 LLM_API_KEY（见下方“关键配置”）
```

**2. 启动基础设施**

```bash
docker compose up -d postgres etcd minio milvus
# 可选组件 Redis：docker compose --profile optional up -d redis
```

PostgreSQL 容器首次启动会自动执行 `scripts/init_pgvector.sql` 启用 pgvector 扩展。Milvus 依赖 etcd + MinIO，等约 30 秒后用 `http://localhost:9091/healthz` 确认健康。

**3. 安装并启动后端**

```bash
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
# 使用默认本地 Qwen Embedding 时，另需安装（含 torch，体积较大）：
# pip install -r requirements-local-embedding.txt
uvicorn app.main:app --reload
```

后端启动时会自动建表，并为老表补齐缺失的列，**不需要手动跑数据库迁移**。默认开启定时同步（`AUTO_SYNC_ENABLED=true`，间隔 300 秒）。

**4. 安装并启动前端**

```bash
cd frontend
npm install
npm run dev
```

**5. 验证**

- 后端健康检查：<http://localhost:8000/health>
- API 文档：<http://localhost:8000/docs>
- 前端：<http://localhost:3000>

## 端口冲突处理（不改源码）

宿主机上已有服务占用了默认端口时，不必改 `docker-compose.yml`，追加一个 `docker-compose.override.yml` 只调整端口映射，再在 `.env` 里同步修改连接地址即可（示例：本机 5432 / 9000 / 9001 已被其他容器占用，3000 被宿主机进程占用）：

```yaml
# docker-compose.override.yml
services:
  postgres:
    ports: ["15432:5432"]
  minio:
    ports: ["19000:9000", "19001:9001"]
```

```bash
# .env 中同步修改
DATABASE_URL=postgresql+psycopg://devflow:devflow@localhost:15432/devflow
```

- MinIO 对外端口可以随意改：Milvus 在容器网络内部通过 `minio:9000` 访问，不受影响。
- 前端换端口：`npm run dev -- -p 3001`。CORS 白名单已包含 3001/3002；开发模式下后端还会放行任意 `localhost:*` 来源。

## 关键配置（`.env`）

| 变量 | 说明 |
| --- | --- |
| `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL` | 对话、Agent、RAG 生成与默认评测 Judge 共用；任何 OpenAI 兼容接口 |
| `GITHUB_TOKEN` | 同步 GitHub 数据；留空时功能受限但可运行 |
| `EMBEDDING_PROVIDER` | 默认 `qwen_local`（本地 `Qwen/Qwen3-Embedding-0.6B`，需装 `requirements-local-embedding.txt`）；也可配 `EMBEDDING_API_KEY` + `EMBEDDING_BASE_URL` 走云端（如百炼 `text-embedding-v4` + `RERANK_BASE_URL` 指向 `qwen3-rerank`）；`deterministic` 仅用于演示/测试 |
| `MILVUS_ENABLED` / `MILVUS_URI` | 默认 `true` / `http://localhost:19530`；Milvus 不可用时 RAG 接口返回 503 `degraded`，应用其余功能照常 |
| `RAGAS_*` | 评测开关、Judge 独立配置（留空回退 `LLM_*`）、五项语义指标的通过门槛 |
| `AUTO_SYNC_*` | 定时同步开关、间隔与单次拉取上限 |
| `REPO_CHECKOUT_DIR` / `UPLOAD_DIR` | 仓库 checkout 与上传文件的本地目录（默认 `.data/` 下） |
| `NEXT_PUBLIC_API_BASE_URL` | 前端访问的后端地址（在根 `.env` 或 `frontend/.env.local` 中设置，默认 `http://localhost:8000`） |

## RAG 知识问答

并非所有私域数据都值得向量化，取舍标准是：是否需要跨来源语义召回、是否会被反复使用。

- 进向量库的是反复引用、需要语义搜索的非结构化知识——项目文档和手工上传的资料，以及散落在各处的文本：Issue 描述、PR 说明与评审意见、失败的 CI 日志、被采纳的长期记忆。
- 源码和配置永远以本地 checkout 为准，走 `workspace.search_code` / `workspace.read_file` 做词法检索；Issue / PR / CI 的状态字段、人员名单、周报这类结构化信息直接查库，不经过向量层。
- 向量检索给出的只是候选证据；涉及实时状态或当前代码的问题，最终仍以权威数据源为准。

RAG API：

```text
POST /api/rag/knowledge-bases
GET  /api/rag/knowledge-bases
GET  /api/rag/knowledge-bases/{repo_id}
POST /api/rag/knowledge-bases/{repo_id}/documents
POST /api/rag/knowledge-bases/{repo_id}/retrieval-tests
GET  /api/rag/{repo_id}/status
POST /api/rag/{repo_id}/search
POST /api/rag/{repo_id}/ask
POST /api/rag/{repo_id}/reindex
```

如果是从旧版本升级上来，请对每个知识库手动触发一次 `POST /api/rag/{repo_id}/reindex`：重建会按新的切片边界清掉历史向量和旧会话 Document；仅更新代码不会触碰已有索引。

### 演示模式（无 Milvus / 无 LLM)

如果暂时没有 Milvus 或模型 Key，又想先看看页面和基本流程，Windows 下可以运行：

```powershell
.\scripts\start_rag.ps1
# 已配置 LLM_API_KEY 时启用模型生成：
.\scripts\start_rag.ps1 -UseConfiguredLLM
```

脚本会绕开 Milvus，改用基于哈希的 deterministic embedding 和启发式重排，并放开演示知识库的 `0.8` 分数阈值，否则只靠关键词分支时所有候选都会被过滤掉。这套模式能走通页面展示、文件上传解析、关系库存储、混合检索的关键词分支和引用渲染，但向量检索状态会如实标为 `degraded`——**它证明不了向量召回和完整 RAG 链路**。要验证那部分，仍需先按"快速开始"把 PostgreSQL、etcd、MinIO 和 Milvus 跑起来。Linux 下的等价做法是设置 `MILVUS_ENABLED=false` 并在代码内选择 deterministic embedding 后启动后端。

### 演示数据

```bash
cd backend
PYTHONPATH=. python ../scripts/seed_demo_data.py
```

会创建示例仓库 `course-demo/devflow-sample-api`，填充 Issue、PR、Review 评论与 CI 运行记录，并索引演示文档。另有 `scripts/sync_github_repo.py` 用于对单个仓库做一次性同步。

## ChatAgent RAGAS 评测

评测入口在前端 `/evals` 页面：选定仓库、评测集与 Top-K 后一键运行。系统会在隔离会话里按生产配置完整执行 ChatAgent（含全部工具调用），先收集检索前的证据，再收集 Agent 追加检索的证据，最后交给 Ragas 打分。每次结果都会落库：评测集哈希、阈值设定、逐题工具轨迹、检索证据、硬规则结果与失败诊断，方便日后对同一基线做回归对比。

```http
POST /api/evals/rag/run
Content-Type: application/json

{
  "repo_id": "<repository-uuid>",
  "run_judge": true,
  "top_k": 5,
  "suite_id": "devflow-real-project-rag-baseline"
}
```

- 提交后立即返回 `queued` 状态和 `eval_id`，评测转入后台执行；用 `GET /api/evals/{eval_id}` 轮询 `result.progress`，看到 `passed`、`failed`、`error` 或 `no_cases` 即为结束。页面会自动轮询，中途刷新或离开都不影响已在后台运行的任务。
- 判定分两层：硬规则必须全部通过，且五项语义指标分别达到 `.env` 里 `RAGAS_*_MIN` 设定的下限。页面上"执行状态"和"质量状态"是分开的两件事——分数不达标属于质量失败，而 Judge 调不通、指标缺失说明这次评测不完整，两种情况都不会被粉饰成通过。
- 固定评测集 JSON 放在 `docs/datas/evals/`（代码中的默认评测集目录，格式见 `backend/app/services/evals/suites.py`：需包含 `name` 与 `cases`）；**当前仓库未附带 `rag_cases.json`，需自行准备**，否则可使用内置的 `auto-document-smoke` 冒烟评测集。

评测失败后不用从头再来，页面支持就地处理：

- 展开单个 Case，对照模型真实回答、参考答案、引用来源、工具调用轨迹，以及系统给出的可执行诊断；
- Judge 超时或个别指标缺失时，点"仅重试异常评分"——复用已落库的回答与证据，不必重跑 ChatAgent;
- 挑两次评测集哈希、Judge 配置、Top-K 完全一致的结果做对比，观察指标回归；
- 可为失败用例生成 `create_issue` 类型的 ActionDraft，连同证据与建议一起进入人工确认流程，不会自动提交到 GitHub。

## 前端页面

| 路径 | 功能 |
| --- | --- |
| `/` | 概览 |
| `/repos` | 仓库管理与同步 |
| `/issues` | Issue 列表与分析 |
| `/pull-requests` | PR 审查 |
| `/ci` | CI 运行与排障 |
| `/workflow-runs` | Workflow Run 详情 |
| `/chat` | Agent 对话 |
| `/knowledge` | 项目知识库 |
| `/rag` | RAG 知识问答工作室 |
| `/reports` | 工程周报 |
| `/evals` | RAGAS 评测 |
| `/action-drafts` | 操作草稿人工确认 |
| `/code-graph` | 代码图谱 |
| `/workspaces` | 工作区浏览 |

## 目录结构

```text
backend/
  app/
    api/routes/        # 18 组 REST 路由（repos/issues/prs/ci/chat/rag/evals/...）
    core/              # 配置、日志、安全（Token 加密）
    db/                # SQLAlchemy 模型与会话
    mcp/               # MCP 客户端接入
    schemas/           # Pydantic Schema
    services/          # agents / github 同步 / rag / evals / skills / 记忆 / 上下文压缩...
    skills/            # 内置 SKILL.md（issue-triage、pr-review、ci-debug 等 6 个）
    tests/             # pytest 测试
  requirements.txt / requirements-local-embedding.txt

frontend/              # Next.js 15 + TypeScript + Tailwind
docs/
  architecture/        # system_design / agent_workflow / database_schema / security
  api/                 # openapi 说明
  examples/            # RAG 示例文档
  datas/evals/         # 评测集目录（需自行放置）
scripts/               # init_pgvector.sql / seed_demo_data.py / sync_github_repo.py / start_rag.ps1
docker-compose.yml     # postgres + etcd + minio + milvus（redis 为 optional profile）
```

## 常见问题

| 现象 | 原因与处理 |
| --- | --- |
| `pip install` 报依赖版本错误 | Python 低于 3.10。用 conda/pyenv 建 3.10+ 环境再安装 |
| 容器启动报端口被占用 | 见上文"端口冲突处理"，用 override 文件改映射 |
| 镜像拉取超时 | 配置镜像加速，或用代理前缀拉取后重新打 tag，例如 `docker pull docker.m.daocloud.io/milvusdb/milvus:v2.5.4` |
| RAG 接口返回 503 `degraded` | Milvus 未启动或未就绪；启动基础设施后重试，其余功能不受影响 |
| 启动时 Embedding 报错 | 默认本地模型需要 `pip install -r requirements-local-embedding.txt`；或改配 `EMBEDDING_API_KEY` 走云端。系统不会静默切换到另一套向量空间 |
| Agent 回复无模型内容 / 全是兜底 | 未配置 `LLM_API_KEY`；对话生成与评测 Judge 都依赖它 |
| 升级后检索结果异常 | 对每个知识库调用一次 `POST /api/rag/{repo_id}/reindex` |

## 更多文档

- [docs/architecture/system_design.md](docs/architecture/system_design.md) — 系统设计
- [docs/architecture/agent_workflow.md](docs/architecture/agent_workflow.md) — Agent 工作流
- [docs/architecture/database_schema.md](docs/architecture/database_schema.md) — 数据库模型
- [docs/architecture/security.md](docs/architecture/security.md) — 安全设计
- [docs/api/openapi.md](docs/api/openapi.md) — API 说明（运行时也可访问 `/docs`）
- [docs/examples/rag_employee_handbook.md](docs/examples/rag_employee_handbook.md) — RAG 示例文档
