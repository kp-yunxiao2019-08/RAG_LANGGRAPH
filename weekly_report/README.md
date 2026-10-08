# Weekly Report HITL Dashboard

基于 **LangGraph + LangChain + FastAPI + SQLite** 的多 Agent 周报生成系统。
AI 自动采集多源工作数据 → 清洗/写作/自审 → 人工审批（HITL）→ 分发最终周报。

---

## 一、系统角色（Actor）

| 角色 | 对应节点 | 职责 |
|------|---------|------|
| **PM / 拆解者** (Planner) | `planner` | 解析用户输入，拆出采集计划（要采哪些源、时间范围） |
| **采集员** (Collector) | `collector` | 并行调用 Git / 飞书 / Calendar 数据源 API |
| **清洗工** (Cleaner) | `cleaner` | 去重、归一化、过滤无关条目 |
| **写手** (Writer) | `writer` | 把清洗后素材写成结构化周报草稿 |
| **审稿人** (Critic) | `critic` | LLM-as-Judge 自评草稿质量，决定是否返工 |
| **审批人** (HITL) | `human_gate` | 人工审阅 / 通过 / 打回（生产级中断介入） |
| **排版/分发** (Assembler + Distributor) | `assembler`, `distributor` | 组装最终 Markdown 周报，保存到 reports 表 |

---

## 二、快速开始

### 环境要求

- Python 3.10+
- pip

### 安装 & 运行（后端 + 前端 Dashboard）

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 启动 HITL API 服务（终端 A）
uvicorn hitl.api:app --port 8000 --reload
# 首次启动自动初始化 SQLite DB（weekly_report.sqlite）

# 3. 启动前端 Dashboard（终端 B）
cd frontend
python -m http.server 3000
# 浏览器打开 http://localhost:3000
```

### CLI 模式（可选，不需要前端）

```bash
# 纯内存快速演示（不涉及 DB，最快验证）
python main.py --mode mock --week 2026-W09

# 真实 HITL 闭环（submit → 挂起 → 模拟审批 → resume）
python main.py --mode hitl --week 2026-W09 --user alice

# 等价于 "uvicorn hitl.api:app --port 8000"
python main.py --mode server
```

### 演示账号

| 用户名 | 密码 | 说明 |
|--------|------|------|
| alice | alice123 | 默认员工 |
| bob   | bob123  | 可选 |

---

## 三、目录结构

```
weekly_report_full_v3/
├── README.md                  # 本文件
├── CHANGES.md                 # 版本变更日志
├── requirements.txt           # Python 依赖
├── .env.example               # 环境变量模板（LangSmith 等）
├── .env                       # 实际环境变量（gitignore）
├── weekly_report.sqlite       # SQLite DB（首次启动自动生成）
│
├── config/                    # 配置层
│   ├── settings.py            # 全局配置（DB 路径、阈值等）
│   └── prompts.py             # LLM prompt 模板
│
├── state/                     # Graph State 定义
│   └── report_state.py        # ReportState (TypedDict) —— 整条状态机的契约
│
├── graph/                     # LangGraph 核心
│   ├── nodes.py               # 所有节点函数（planner / collector / critic / human_gate ...）
│   ├── edges.py               # conditional edge 路由（should_revise / after_hitl）
│   └── graph.py               # build_graph + SqliteSaver checkpointer + resume_graph
│
├── tools/                     # 数据源 Tool 抽象
│   ├── base.py                # Tool 抽象基类 + _get_tmp_data 工厂
│   ├── github_tool.py         # GitHub Commit/PR 采集
│   ├── feishu_tool.py         # 飞书文档/任务采集
│   ├── calendar_tool.py       # 日历/会议采集
│   └── registry.py            # run_plan() —— asyncio.gather 并发执行
│
├── hitl/                      # ✅ HITL 闭环核心（本项目生产级模块）
│   ├── __init__.py
│   ├── api.py                 # FastAPI 审批 API（10 个端点）
│   ├── auth.py                # JWT 认证 + DEMO_USERS
│   ├── models.py              # Pydantic 请求/响应模型
│   ├── runner.py              # submit_and_wait() —— 跑 graph → 挂起 → 落库
│   └── notify.py              # 通知（console / feishu / wecom）
│
├── model/                     # 数据持久层
│   ├── store.py               # SQLite CRUD（approval_tasks + reports）
│   └── temp_data.py           # 演示用临时数据
│
├── frontend/                  # ✅ 纯 Vanilla JS 审批 Dashboard
│   ├── index.html             # 骨架（登录页 + 主 Dashboard + 审批 Modal + Toast）
│   ├── app.js                 # 全部交互逻辑（已加详细教学注释）
│   └── style.css              # 样式（Flex / Grid / 动画）
│
├── mock_api/                  # （遗留）独立 Mock 后端
│   └── server.py              # 旧版数据源模拟接口（已被 tool.collect tmp_data 替代）
│
├── utils/
│   └── gen_token.py           # 生成密码 hash 的小工具
│
├── evaluation/                # LangSmith 评估套件
│   ├── dataset.py
│   ├── evaluators.py
│   └── run_eval.py
│
├── main.py                    # CLI 入口（--mode mock/hitl/server）
└── test_*.py                  # pytest 测试套件
```

---

## 四、架构：状态机 (StateGraph)

```
entry
  ↓
planner      —— 拆出采集计划
  ↓
collector    —— 并行采集（GitHub / 飞书 / Calendar）
  ↓
cleaner      —— 去重归一化
  ↓
writer       —— 生成草稿 draft
  ↓
critic       —— 自评打分 critic_score（LLM Judge）
  ↓ (conditional: should_revise)
  ├─ "revise"   → writer   (仅首轮自审循环，人工介入后禁用)
  ├─ "human"    ↓          (首轮 force / 低分 / 达上限 / 人工打回后自动走这条)
  └─ "approve"  ↓          (score=1.0 满分)
human_gate   —— interrupt 挂起 → SQLite 落库 → 等待 HTTP 审批
  ↓ (POST /api/approve 恢复)
after_hitl   —— 路由：approved="true" → "assemble" / "false" → "revise"
  ↓ "assemble"
assembler    —— 组装最终 final_report（附 hitl_comments 完整审阅意见历史）
  ↓
distributor  —— save_report() 写入 reports 表
  ↓
END
```

### 关键 Graph 设计

- **Checkpointer**：`SqliteSaver`（langgraph-checkpoint-sqlite），跨 HTTP 请求持久化状态
- **Interrupt**：`human_gate` 节点调用 `interrupt()` 挂起，`Command(resume=...)` 恢复
- **State 持久化**：graph.get_state(config).values 在挂起时被序列化为 `state_snapshot` 存进 `approval_tasks` 表
- **人工打回后禁止自动 revise 循环**：`should_revise` 在 `human_reject_count > 0` 时直接返回 `"human"`（除非 critic 给满分 1.0），让人工在每次改写后亲自过目
- **hitl_comments 累积**：`ReportState.hitl_comments` 使用 `operator.add` reducer，每次人工打回的批注被 append 进 list，全程不丢失
- **安全上限保护**：`max_human_rejects=5` / `max_iterations=10` / `max_revise_per_reject=5`（均可在 initial state 覆盖），达上限自动 `force_passthrough` 放行

---

## 五、API 接口（hitl/api.py）

所有端点前缀 `/api`，JWT Bearer Token 认证（除了 `/api/auth/login` 和 `/health`）。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/auth/login` | 登录，返回 JWT |
| POST | `/api/submit` | **提交新周报生成任务**（内部跑 graph → 挂起 → 落库） |
| GET  | `/api/approvals_pending?user_id=alice` | 待审批列表（SQL: `status IN ('pending','pending_review','revising')`） |
| GET  | `/api/approvals_all?user_id=alice` | 该用户的全部审批任务 |
| GET  | `/api/approvals?thread_id=xxx` | 单个审批任务详情（含完整 state_snapshot） |
| POST | `/api/approve` | **通过/打回**（resolve_task + resume_graph 自动恢复） |
| GET  | `/api/reports/history?user_id=alice` | 已完成周报历史 |
| POST | `/api/reports` | 手动保存周报（调试用，生产由 distributor 自动保存） |
| POST | `/api/resume` | 手动恢复 graph（调试用） |
| GET  | `/health` | 健康检查 |

### 核心请求体示例

**POST /api/submit**
```json
{
  "week": "2026-W09",           // ISO 周格式 "YYYY-Www"
  "user_id": "alice",
  "force_score": 0.3,            // Critic 打分（0.3 → 强制走 human_gate；1.0 → 直接通过）
  "interrupt_approve": true      // 是否强制挂起（演示中断介入）
}
```

> `force_score` 是演示用开关：设为 0.3 让 Critic 始终打低分 → `should_revise` 返回 `"human"` → 触发 human_gate interrupt。
> 设为 1.0 则 Critic 直接打高分 → `should_revise` 返回 `"approve"` → 跳过人工审批。

**POST /api/approve**
```json
{
  "thread_id": "weekly-alice-2026-W09-1726951234567",
  "approved": "true",            // 注意是字符串 "true" / "false"（不是 JSON bool）
  "comments": "内容很完整，指标清晰"
}
```

> 前端传 `approved: "true"`（字符串），后端 Pydantic 声明为 `str`，SQLite 存 TEXT。

---

## 六、数据模型

### SQLite 表结构（`weekly_report.sqlite`）

#### approval_tasks — 审批任务

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增 |
| thread_id | TEXT UNIQUE | Graph 线程 ID（唯一） |
| week | TEXT | 周次，如 "2026-W09" |
| user_id | TEXT | 员工 ID |
| node | TEXT | 当前挂起节点（通常是 `"human_gate"`） |
| state_snapshot | TEXT | graph.get_state().values 序列化的完整 state（含 draft / critic_score / hitl_comments ...） |
| **status** | TEXT | `"pending"` / `"revising"` / `"pending_review"` / `"approved"` / `"force_passed"`（达打回上限系统强制放行） |
| approved | TEXT | `"true"` / `"false"` / `"force_passthrough"` / NULL（尚未裁决） |
| hitl_comments | TEXT | **DB 层**仅存最新一条批注（人类可读）；**完整历史累积在 `state_snapshot.hitl_comments`**（`operator.add` reducer，list 类型） |
| human_reject_count | INTEGER | 累计人工打回次数（每次打回 +1，达 `max_human_rejects` 触发 force_passthrough） |
| revise_count | INTEGER | Writer 改写轮次（每次 writer 消费 approved="false" +1） |
| created_at | TEXT | 创建时间（CURRENT_TIMESTAMP） |
| updated_at | TEXT | 更新时间（CURRENT_TIMESTAMP） |
| resolved_at | TEXT | 裁决时间 |

**status 筛选逻辑**（`get_pending_tasks`）：
```sql
WHERE status IN ('pending', 'pending_review', 'revising')
```
approved / done 的任务不在待审批列表中。

#### reports — 最终周报

| 字段 | 类型 | 说明 |
|------|------|------|
| id | INTEGER PK | 自增 |
| thread_id | TEXT UNIQUE | 唯一 |
| user_id | TEXT | 员工 ID |
| week | TEXT | 周次 |
| content | TEXT | 最终周报完整内容 |
| score | REAL | Critic 最终打分 |
| approved | TEXT | 审批结果 |
| created_at | TEXT | 创建时间 |
| updated_at | TEXT | 更新时间 |

**唯一性**：两张表都对 `thread_id` 加了 `UNIQUE` 约束，`ON CONFLICT DO UPDATE` 保证重试/重入不会产生重复记录。

---

## 七、数据状态流转

```
┌─ 首次提交 ─────────────────────────────────────────────────────────────┐
│                                                                        │
│  POST /api/submit {week, user_id, force_score, interrupt_approve}      │
│    ↓                                                                   │
│  计算 start_date / end_date（ISO 周周一/周日）塞进 initial state        │
│    ↓                                                                   │
│  graph.invoke(initial) → planner → collector → cleaner → writer        │
│    → critic 打低分（force_score=0.3）→ should_revise: "human"           │
│    ↓                                                                   │
│  human_gate_node:                                                      │
│    - create_or_reset_task() → DB: status=pending                       │
│    - interrupt() 挂起 → 返回 {thread_id, status: "pending"}            │
│                                                                        │
├─ 人工审批通过 ─────────────────────────────────────────────────────────┤
│                                                                        │
│  POST /api/approve {thread_id, approved: "true", comments}            │
│    ↓                                                                   │
│  resolve_task() → DB: status=approved                                  │
│    ↓                                                                   │
│  resume_graph(thread_id)：                                             │
│    - Command(resume={"approved": True, "comments": "..."})             │
│    - human_gate_node 重入 → approved="true" → return {}                │
│    - after_hitl → "assemble"                                           │
│    - assembler：把 hitl_comments 作为"审阅意见"章节附到 final_report   │
│    - distributor：save_report() → reports 表                           │
│    - update_final_snapshot() → DB 写回最终 state_snapshot              │
│    → END ✅                                                             │
│                                                                        │
├─ 人工审批打回 ─────────────────────────────────────────────────────────┤
│                                                                        │
│  POST /api/approve {thread_id, approved: "false", comments}           │
│    ↓                                                                   │
│  resolve_task() → DB: status=revising, human_reject_count+=1           │
│    ↓                                                                   │
│  resume_graph(thread_id)：                                             │
│    - Command(resume={"approved": "false", "hitl_comments": "..."})     │
│    - human_gate_node 重入 → approved="false" → mark_revising()         │
│      → return {"approved":"false", "human_reject_count": count+1}     │
│      → hitl_comments via Command 被 append 进 state（operator.add）   │
│    - after_hitl → "revise"                                             │
│    - writer_node：hitl_comments 作为修改指导 → revise_count+=1         │
│      消费 approved="false" → 清 approved=''                            │
│    - critic 重新打分                                                    │
│    ↓ 第二轮 should_revise（关键保护！）：                               │
│      human_reject_count > 0 → **强制走 "human"**（禁止自动 revise）   │
│      score ≥ 1.0（罕见满分） → 才会直接 "approve"                     │
│    ↓                                                                   │
│    human_gate 再次 interrupt → mark_pending_review()                   │
│      → DB: status=pending_review, **同步 human_reject_count**          │
│      → 再次 interrupt → 等二审（hitl_comments 已完整累加 ✅）         │
│                                                                        │
├─ 二审通过 → 同"通过"路径 → END                                        │
├─ 二审再打回 → 再循环（human_reject_count 累加，hitl_comments 累加）    │
├─ 打回次数达 max_human_rejects=5 → force_passthrough → END             │
└────────────────────────────────────────────────────────────────────────┘
```

### Tool.collect 的 debug_interrupt 快捷路径

当 `interrupt_approve=True`（前端勾选"强制挂起"）时，所有 Tool 的 `collect()` 方法会跳过真实 HTTP 采集：

```python
# tools/base.py → _get_tmp_data()
# tools/github_tool.py / feishu_tool.py / calendar_tool.py → collect()
debug_interrupt = state.get('interrupt_approve', False)
if debug_interrupt:
    tmp_data = _get_tmp_data(user, self.name, start, end)  # 直接返回 mock 数据
    return tmp_data
```

这意味着即使没有后端 Mock API，整个 Graph 也能跑通全链路（方便本地演示）。

### start_date / end_date 自动填充

`POST /api/submit` 时，`api.py` 根据 `week` 字段（如 `"2026-W09"`）自动算出 ISO 周的起止日期塞进 state：

```python
import datetime as _dt
year_str, _, week_str = req.week.partition("-W")
monday  = _dt.date.fromisocalendar(int(year_str), int(week_str), 1)
sunday  = _dt.date.fromisocalendar(int(year_str), int(week_str), 7)
# "2026-W09" → start_date="2026-02-23", end_date="2026-03-01"
```

### should_revise 路由规则（edges.py）

| 条件 | 返回 | 目标 | 说明 |
|------|------|------|------|
| `human_reject_count > 0` 且 `critic_score < 1.0` | `"human"` | human_gate | **核心保护**：人工打回后禁用自动 revise 循环，每次改写都让人工亲自过目 |
| `human_reject_count > 0` 且 `critic_score >= 1.0` | `"approve"` | assembler | 罕见：打回后 LLM 突然给满分，直接通过 |
| `interrupt_approve=True` 且 `human_reject_count=0` | `"human"` | human_gate | 首轮强制挂起（演示用） |
| `approved` ∈ `("true", "force_passthrough")` | `"approve"` | assembler | 已裁决通过 / 系统放行 |
| `iteration_count >= max_iterations` | `"human"` | human_gate | 自审循环达上限，交人工 |
| `critic_score >= 1.0` | `"approve"` | assembler | 首轮自审满分 |
| `critic_score >= 0.5` | `"revise"` | writer | 首轮中高分，继续自审返工 |
| 其他（score < 0.5） | `"human"` | human_gate | 首轮低分，交人工 |

> **设计意图**：`human_reject_count > 0` 的保护分支优先级最高，是本次修复 hitl_comments 丢失问题的关键——之前人工打回后 graph 会偷偷跑几轮 writer→critic 循环烧穿 max_iterations，直接 force_passthrough 结束，后续打回的 comments 永远进不了 state。

---

## 八、前端 Dashboard

| Tab | 功能 |
|-----|------|
| **提交新任务** | 选员工 / 填周次 / 设 force_score / 勾选强制挂起 → 一键提交 |
| **待审批** | 列表展示 pending / revising / pending_review 任务，点卡片打开审批详情 |
| **周报历史** | 展示 reports 表中已完成的最终周报（含完整审阅意见） |
| **Graph 状态流** | 可视化显示 graph 节点进度 + 完整 state_snapshot JSON |

前端技术栈：**纯 Vanilla JS + HTML + CSS**，无构建工具，无框架依赖。所有交互代码（app.js）已加**教学型中文注释**，覆盖 DOM API / async-await / 事件监听 / fetch API / ES6+ 等基础语法。

---

## 九、运行架构一览

```
┌──────────────────────┐     ┌─────────────────────────────────┐
│  前端 Dashboard       │     │  FastAPI (hitl/api.py)           │
│  (Vanilla JS, :3000) │◄──►│  ┌─────────────────────────────┐ │
│                      │     │  │ POST /api/auth/login        │ │
│  login  ─────────────┼────►│  │ POST /api/submit            │ │
│  submit ─────────────┼────►│  │ GET  /api/approvals_pending │ │
│  approve─────────────┼────►│  │ POST /api/approve ─────────┼─┼──┐
│  history┼─────────────┼────►│  │ GET  /api/reports/history  │ │  │
│  flow   ─────────────┼────►│  └─────────────────────────────┘ │  │
└──────────────────────┘     └──────────────┬──────────────────┘  │
                                             │                     │
                                    ┌────────▼────────┐           │
                                    │ SQLite          │           │
                                    │ weekly_report   │           │
                                    │ .sqlite         │           │
                                    └────────┬────────┘           │
                                             │                     │
                                    ┌────────▼────────┐           │
                                    │ LangGraph       │           │
                                    │ SqliteSaver     │           │
                                    │ Checkpointer    │◄──────────┘
                                    │ (thread_id)     │
                                    └─────────────────┘
```

---

## 十、测试

```bash
# 快速跑全部测试
python run_all_tests.py

# 或用 pytest
pytest test_hitl_sqlite.py -v          # SQLite 持久化 + HITL 闭环
pytest test_hitl.py -v                 # 旧版 HITL 测试
pytest test_api.py -v                  # API 端点测试
pytest test_scenarios.py -v            # 端到端场景
pytest test_v3.py -v                   # V3 回归
```

---

## 十一、依赖说明

| 包 | 版本 | 用途 |
|----|------|------|
| langgraph | ≥0.2.0 | 状态机核心 |
| langgraph-checkpoint-sqlite | ≥2.0 | SQLite checkpointer（跨请求持久化） |
| langchain-openai | ≥0.1.0 | LLM 调用 |
| langsmith | ≥0.1.0 | 可观测性（可选） |
| fastapi | ≥0.110 | 审批 API 框架 |
| uvicorn | ≥0.27 | ASGI server |
| httpx | ≥0.27 | HTTP 客户端（真实数据源采集） |
| pydantic | ≥2.0 | 数据模型验证 |
| bcrypt | ≥4.0 | 密码 hash |
| python-dotenv | ≥1.0 | .env 文件加载 |

> ⚠️ SQLite checkpointer：如果 langgraph-checkpoint-sqlite 版本冲突，可降级为 `MemorySaver`（仅内存，重启丢失状态）。
