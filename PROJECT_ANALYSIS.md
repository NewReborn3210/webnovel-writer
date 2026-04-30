# Webnovel Writer 项目分析

> 生成时间：2026-05-01
> 当前版本：v6.0.0（Story System Phase 5）
> 协议：GPL v3
> 仓库：[lingfengQAQ/webnovel-writer](https://github.com/lingfengQAQ/webnovel-writer)

---

## 一、项目定位

`Webnovel Writer` 是一个基于 **Claude Code Plugin Marketplace** 发布的长篇网文创作系统。

核心目标用一句话概括：**让 AI 写长篇小说时不乱编、不忘事**。

通过"合同驱动 + 章节提交 + 多 Agent 协作 + RAG 检索"四件套，让模型在连载几百章的过程中保持人设、设定、伏笔、节奏的一致性。

---

## 二、技术栈与运行形态

| 维度 | 选型 |
|------|------|
| 宿主 | Claude Code（通过插件 Marketplace 安装） |
| 语言 | Python ≥ 3.10 |
| 核心依赖 | `aiohttp`、`filelock`、`pydantic` |
| 测试 | `pytest` + `pytest-asyncio` + `pytest-timeout` + `pytest-cov` |
| 数据存储 | JSON（state/合同/事件）+ SQLite（`index.db` / `vectors.db`）|
| 检索 | 向量（OpenAI 兼容 Embedding）+ BM25 + RRF + Rerank（Jina）|
| 默认 Embedding | `Qwen/Qwen3-Embedding-8B`（ModelScope）|
| 默认 Reranker | `jina-reranker-v3` |
| 前端面板 | Vite 构建的纯前端（已预构建，无需 npm build）|
| 服务端面板 | Python（dashboard 模块，FastAPI/aiohttp 风格）|

运行形态分为三种：

1. **Skill 命令**：`/webnovel-init`、`/webnovel-write` 等，在 Claude Code 中调用。
2. **统一 CLI**：所有命令最终都进 [webnovel-writer/scripts/webnovel.py](webnovel-writer/scripts/webnovel.py)。
3. **Dashboard**：只读可视化面板（[webnovel-writer/dashboard/](webnovel-writer/dashboard/)）。

---

## 三、目录结构总览

```text
webnovel-writer/                        # 仓库根
├── README.md                           # 用户文档入口
├── requirements.txt                    # 顶层依赖聚合
├── pytest.ini / sitecustomize.py       # 测试与运行时兼容
├── docs/                               # 文档中心
│   ├── architecture/                   # 系统架构与现状诊断
│   ├── guides/                         # commands / rag / genres
│   ├── operations/                     # 运维 + 插件发版
│   ├── memory/                         # 长期记忆架构
│   ├── research/                       # 论文与方案调研
│   └── superpowers/                    # spec/plans 设计文档
└── webnovel-writer/                    # 插件实际内容（CLAUDE_PLUGIN_ROOT 指向这里）
    ├── agents/                         # 3 个 Agent 定义（context / data / reviewer）
    ├── skills/                         # 7 个 Skill 命令定义
    ├── scripts/                        # Python CLI 与数据链
    │   ├── webnovel.py                 # 统一 CLI 入口
    │   ├── data_modules/               # 60+ 模块的数据/合同/索引/记忆核心
    │   └── tests/                      # 单元测试
    ├── dashboard/                      # 只读可视化面板（Python + Vite 前端）
    ├── references/                     # 题材画像、追读力分类法等参考资料
    ├── templates/                      # 36 个题材模板 + 金手指模板
    └── genres/                         # 6 个精调题材的细粒度配置
```

代码规模（粗略）：

- Python 脚本：约 **147 个 .py 文件**（含 60+ 测试文件）
- `data_modules/` 是核心：**40+ 模块**（合同、提交、索引、记忆、RAG、健康检查等）
- 题材模板：**36 个 `.md`**（涵盖玄幻、言情、悬疑、现实等主流网文方向）

---

## 四、核心架构

### 4.1 分层模型

```text
┌──────────────────────────────────────────────────────┐
│                Claude Code（宿主）                    │
├──────────────────────────────────────────────────────┤
│ Skills (7):                                          │
│   init / plan / write / review / query / learn /     │
│   dashboard                                          │
├──────────────────────────────────────────────────────┤
│ Agents (3):                                          │
│   Context Agent (读) / Data Agent (写) /             │
│   Reviewer (审，含六维)                               │
├──────────────────────────────────────────────────────┤
│ Data Layer:                                          │
│   state.json / index.db (SQLite) / vectors.db        │
├──────────────────────────────────────────────────────┤
│ Story System（合同·提交·事件）：                      │
│   .story-system/ —— Phase 5 已成主真源                │
└──────────────────────────────────────────────────────┘
```

### 4.2 三大设计原则（防幻觉三定律）

| 定律 | 含义 | 执行者 |
|------|------|--------|
| **大纲即法律** | 严守大纲，不擅自发挥 | Context Agent 强制注入大纲上下文 |
| **设定即物理** | 严守设定，不自相矛盾 | Reviewer Agent 一致性审查 |
| **发明需识别** | 新实体必须入库 | Data Agent 自动提取 + 消歧入库 |

### 4.3 Strand Weave 节奏系统

以三股线（Quest 主线 / Fire 感情 / Constellation 世界观）按 6:2:2 编织，并设置断档红线：

- Quest 连续 ≤ 5 章
- Fire 断档 ≤ 10 章
- Constellation 断档 ≤ 15 章

### 4.4 Story System（合同驱动体系）

`.story-system/` 是**写前真源 + 写后事实**唯一入口，分五阶段递进：

| Phase | 内容 | 产物 |
|-------|------|------|
| 1 | 合同种子 | `MASTER_SETTING.json` + 章节合同 + 反模式配置 |
| 2 | 合同优先运行时 | `volumes/`、`reviews/` + 写前校验 |
| 3 | 章节提交链 | `commits/chapter_XXX.commit.json` + 多投影写入 |
| 4 | 事件审计链 | `events/*.events.json` + 修订提案 + 覆写账本 |
| 5（当前）| 旧链路降级 | `.story-system/` 主真源；`.webnovel/*` 降级为投影/read-model；`references/genre-profiles.md` 降级为 fallback-only |

数据流向：

```text
story-system --persist
   └─ 写入合同种子
story-system --emit-runtime-contracts --chapter N
   └─ 生成运行时合同 + 写前校验
chapter-commit --chapter N
   └─ 提交 accepted commit + 触发 projection writers
        ├─ state_projection_writer        → state.json
        ├─ index_projection_writer        → index.db
        ├─ summary_projection_writer      → summaries/
        ├─ memory_projection_writer       → memory_scratchpad.json
        └─ vector_projection_writer       → vectors.db
story-events --chapter N / --health
   └─ 事件审计与健康检查
preflight / dashboard
   └─ runtime health 第一观察点
```

注意：Phase 4 的事件路由仅做声明式激活，**实际写入入口仍是 `ChapterCommitService.apply_projections()`**，避免双投影循环。

---

## 五、Agent 与 Skill

### Agents（[webnovel-writer/agents/](webnovel-writer/agents/)）

| Agent | 文件 | 职责 |
|-------|------|------|
| Context Agent | [context-agent.md](webnovel-writer/agents/context-agent.md) | 写前构建"创作任务书"，注入上下文/约束/追读力策略 |
| Data Agent | [data-agent.md](webnovel-writer/agents/data-agent.md) | 写后从正文提取 commit artifacts，驱动各 projection writer |
| Reviewer | [reviewer.md](webnovel-writer/agents/reviewer.md) | 六维审查：爽点/一致性/节奏/OOC/连贯性/追读力 |

另含 [deconstruction-agent.md](webnovel-writer/agents/deconstruction-agent.md)（拆文 Agent）与 [evals/](webnovel-writer/agents/evals/) 评测集。

### Skills（[webnovel-writer/skills/](webnovel-writer/skills/)）

| 命令 | 功能 |
|------|------|
| `/webnovel-init` | 初始化项目、生成设定集与状态文件 |
| `/webnovel-plan [卷号]` | 生成卷级规划与章节大纲 |
| `/webnovel-write [章号]` | 完整章节创作流（task → 起草 → 审查 → 润色 → 落盘）|
| `/webnovel-review [范围]` | 多维质量审查 |
| `/webnovel-query [关键词]` | 查询角色/伏笔/节奏/状态 |
| `/webnovel-learn [内容]` | 沉淀写作模式到项目记忆 |
| `/webnovel-dashboard` | 启动只读可视化面板 |

---

## 六、统一 CLI（[scripts/webnovel.py](webnovel-writer/scripts/webnovel.py)）

调用模板：

```bash
python -X utf8 "${CLAUDE_PLUGIN_ROOT}/scripts/webnovel.py" \
       --project-root "${PROJECT_ROOT}" <子命令> [参数]
```

子命令分组：

| 分组 | 子命令 |
|------|--------|
| **基础** | `where`、`preflight`、`use` |
| **数据链** | `index`、`state`、`rag`、`entity`、`context`、`style`、`migrate` |
| **运维** | `status`、`update-state`、`backup`、`archive`、`extract-context` |
| **长期记忆** | `memory stats / query / dump / conflicts / bootstrap / update` |
| **Story System** | `story-system`、`chapter-commit`、`story-events`、`memory-contract`、`review-pipeline` |

---

## 七、数据链核心模块（[scripts/data_modules/](webnovel-writer/scripts/data_modules/)）

按职责分组：

| 类别 | 关键模块 |
|------|---------|
| **配置/客户端** | `config.py`、`api_client.py`、`rag_adapter.py` |
| **Story Contracts** | `story_contract_schema.py`、`story_contracts.py`、`story_system_engine.py`、`runtime_contract_builder.py`、`prewrite_validator.py`、`genre_profile_builder.py`、`genre_aliases.py` |
| **Chapter Commit** | `chapter_commit_service.py`、`schemas.py`、`amend_proposal_schema.py`、`override_ledger_service.py` |
| **Events** | `story_event_schema.py`、`event_log_store.py`、`event_projection_router.py` |
| **Projection Writers** | `state_projection_writer.py`、`index_projection_writer.py`、`summary_projection_writer.py`、`memory_projection_writer.py`、`vector_projection_writer.py` |
| **Index（SQLite）** | `index_manager.py` + `index_chapter_mixin.py` / `index_entity_mixin.py` / `index_reading_mixin.py` / `index_debt_mixin.py` / `index_observability_mixin.py` |
| **State** | `state_manager.py`、`sql_state_manager.py`、`state_validator.py`、`migrate_state_to_sqlite.py` |
| **Context/Query** | `context_manager.py`、`context_ranker.py`、`context_weights.py`、`query_router.py`、`knowledge_query.py` |
| **Entity** | `entity_linker.py`、`placeholder_scanner.py` |
| **Memory（长期记忆）** | `memory/` 子包：`bootstrap`、`budget`、`compactor`、`orchestrator`、`schema`、`store`、`writer` + `memory_contract.py` / `memory_contract_adapter.py` |
| **Health/Observability** | `story_runtime_health.py`、`story_runtime_sources.py`、`observability.py` |
| **Misc** | `style_sampler.py`、`writing_guidance_builder.py`、`review_schema.py`、`cli_args.py`、`cli_output.py` |

---

## 八、RAG 与配置

检索流程：

```text
查询 → QueryRouter(auto)
       ├─ vector  / bm25 / hybrid / graph_hybrid
       └─ RRF 融合 + Rerank → Top-K
```

- 默认 `auto`：向量优先，失败回退 BM25
- `graph_hybrid`：叠加实体图谱关联

环境变量加载优先级：

1. 进程环境变量
2. `${PROJECT_ROOT}/.env`
3. `~/.claude/webnovel-writer/.env`

最小 `.env`：

```bash
EMBED_BASE_URL=https://api-inference.modelscope.cn/v1
EMBED_MODEL=Qwen/Qwen3-Embedding-8B
EMBED_API_KEY=...

RERANK_BASE_URL=https://api.jina.ai/v1
RERANK_MODEL=jina-reranker-v3
RERANK_API_KEY=...
```

---

## 九、题材模板系统

- **36 个内置题材模板**：[webnovel-writer/templates/genres/](webnovel-writer/templates/genres/)
- **6 个精调题材目录**：[webnovel-writer/genres/](webnovel-writer/genres/)（`xuanhuan`、`dog-blood-romance`、`period-drama`、`realistic`、`rules-mystery`、`zhihu-short`）
- 支持别名映射（如"玄幻修真" → "修仙"）
- 支持复合题材（最多 2 个，主辅 7:3，分隔符 `+ / 、 与`）

题材模板内容会在 `/webnovel-init` 时自动注入到 `设定集/世界观.md`。

---

## 十、目录层级（运行时）

| 层级 | 含义 | 示例 |
|------|------|------|
| `WORKSPACE_ROOT` | Claude Code 工作区 | `D:\wk\novels` |
| `.claude/` | 工作区配置 + 当前书指针 (`.webnovel-current-project`) | — |
| `PROJECT_ROOT` | 单本书项目根 | `D:\wk\novels\凡人资本论` |
| `CLAUDE_PLUGIN_ROOT` | 插件缓存目录（Marketplace 管理） | — |

书项目目录：

```text
project-root/
├── .webnovel/         # 投影/read-model：state.json / index.db / vectors.db / summaries/ / backups/ / archive/
├── .story-system/     # 主真源：MASTER_SETTING.json + chapters/ + volumes/ + reviews/ + commits/ + events/
├── 正文/
├── 大纲/
├── 设定集/
└── 审查报告/
```

---

## 十一、版本演进

| 版本 | 主要变化 |
|------|----------|
| **v6.0.0（当前）** | Story System 全链路上线（合同种子 + 运行时合同 + 章节提交 + 事件审计）|
| v5.5.5 | 长期记忆闭环：写前注入 + 写后沉淀，新增 `memory` 运维命令 |
| v5.5.4 | 写作链提示词强约束，统一中文化审查文案 |
| v5.5.3 | 统一 `preflight` 预检命令，修 Windows 终端编码 |
| v5.5.2 | 大纲章节名同步到正文文件名 |
| v5.5.1 | 修复卷级大纲上下文提取 |
| v5.5.0 | 新增只读 Dashboard，支持实时刷新 |
| v5.4.4 | 接入 Plugin Marketplace 安装机制 |
| v5.4.3 | 增强 RAG 智能上下文（`auto/graph_hybrid` → BM25 回退）|
| v5.3 | 引入追读力系统（Hook / Cool-point / 微兑现 / 债务追踪）|

---

## 十二、文档与研究资料

- 架构：[docs/architecture/overview.md](docs/architecture/overview.md)、[story-system-phase5.md](docs/architecture/story-system-phase5.md)、[current-system-diagnosis.md](docs/architecture/current-system-diagnosis.md)
- 指南：[commands.md](docs/guides/commands.md)、[rag-and-config.md](docs/guides/rag-and-config.md)、[genres.md](docs/guides/genres.md)
- 运维：[operations.md](docs/operations/operations.md)、[plugin-release.md](docs/operations/plugin-release.md)
- 记忆：[long-term-memory-architecture-v2.md](docs/memory/long-term-memory-architecture-v2.md)
- 研究：[long-term-memory-research-report.md](docs/research/long-term-memory-research-report.md)、[storyteller-paper-summary.md](docs/research/storyteller-paper-summary.md)、[2026-04-14-ui-ux-pro-max-skill-architecture-research.md](docs/research/2026-04-14-ui-ux-pro-max-skill-architecture-research.md)
- Specs/Plans：[docs/superpowers/](docs/superpowers/) 含 v6 迁移、Phase1 清理等设计文档

---

## 十三、亮点与设计风格总结

1. **合同驱动 + 事件溯源**：`.story-system/` 把"写前约束"和"写后事实"都用 JSON Schema 固化，避免模型自由发挥导致的设定漂移；通过 `chapter-commit` 单一入口扇出多个 projection writer，符合事件溯源模式。
2. **真源 vs 投影分层**：`.webnovel/*` 与 SQLite 索引被明确降级为 read-model，可随时按 commit 重建，简化故障恢复。
3. **多 Agent 拆分职责**：读/写/审三 Agent 隔离，单 Agent 不承担过多上下文，便于 Claude 模型稳定输出。
4. **追读力系统（Phase 5.3 起）**：Hook / Cool-point / 微兑现 / 债务追踪，把"网文爽点工程化"的产业经验固化为可量化指标（[references/reading-power-taxonomy.md](webnovel-writer/references/reading-power-taxonomy.md)）。
5. **检索弹性**：RAG 链 `auto` 模式自带 BM25 回退，无 API Key 也能跑，降低使用门槛。
6. **本地化运维**：CLI 全中文文案、强制 `-X utf8`、Windows 终端编码兼容、文件锁并发保护。
7. **文档完整度高**：`docs/` 分 architecture / guides / operations / memory / research / superpowers 六类，含外部论文调研（STORYTELLER）和长期记忆架构 v2，体现"边研究边落地"的工程节奏。

---

## 十四、潜在改进方向（基于阅读观察）

> 以下为读码归纳，非项目官方 roadmap，仅供参考。

- `data_modules/` 模块数量已超 40，可考虑按 `contracts/`、`commits/`、`projections/`、`memory/`、`rag/` 等子包进一步分层，降低顶层目录密度。
- `index_manager.py` 通过多个 `*_mixin.py` 拼装，单类承担的职责较广，未来若要做并发优化或多书共享缓存，建议拆为组合式 service。
- 顶层 `requirements.txt` 仅做聚合（`-r` 转发），但运行时核心依赖只有 3 个（`aiohttp`、`filelock`、`pydantic`）；若后续引入向量本地化或 BM25 库，需注意保留"零重型依赖"的优势。
- Dashboard 前端已预构建在 `dashboard/frontend/dist/`，源代码在 `src/`，发版流程依赖手工同步——可结合 [docs/operations/plugin-release.md](docs/operations/plugin-release.md) 增加 CI 校验。

---

*本文档由项目代码与 `docs/` 自动归纳生成，作为新成员/外部协作者快速上手的索引。*
