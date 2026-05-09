# [AGENT_NAME] 架构方案

> **生成时间**：[YYYY-MM-DD]
> **生成工具**：`agent架构师` skill (Claude Code)
> **目标**：本文件可直接喂给 Claude Code / Cursor / Trae 等 AI coding 工具完成实际搭建。
>
> **vibecoding 启动建议**：在 AI coding 工具中新建对话，说「读这份 spec 并按 Phase 1 的任务清单逐步实现，每完成一个任务给我看 diff 后再继续」。

---

## 0. TL;DR

- **一句话定义**：[一句话描述这个 agent 是什么，解决什么]
- **架构选择**：[Workflow / Agent] · [基础模式：Augmented LLM / Prompt Chaining / Routing / Parallelization / Orchestrator-Workers / Evaluator-Optimizer / Autonomous Agent]
- **主框架**：[LangGraph / CrewAI / AutoGen / OpenAI Agents SDK / Mastra / Vercel AI SDK / 自研]
- **主模型**：[Claude Opus 4.7 / GPT-5 / Gemini 2.5 Pro / ...] + 路由小模型 [Haiku / Mini / Flash]
- **核心工具**：[3–5 个工具一句话列出]
- **预估单次成本量级**：$ [量级] / 次

---

## 1. 使命与价值

### 1.1 一句话目标
[用户视角，例：每周一早上自动拉取 Stripe 报告、生成中文摘要邮件并发给 CEO]

### 1.2 业务痛点
[当前怎么做？为什么需要 agent？解决后能省多少时间 / 钱]

### 1.3 可衡量指标

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 成功率 | ≥ [%] | golden set 跑通率 |
| 平均延迟 | ≤ [秒] | trace 统计 |
| 单次成本 | ≤ $ [金额] | token 计费 |
| [其他业务指标] | | |

---

## 2. 边界与红线

### 2.1 ✅ 能做
- [明确列出能力范围]

### 2.2 ❌ 不做
- [明确列出能力外的事，避免 scope creep]
- [数据红线：不访问 X / 不存储 Y]
- [动作红线：不发邮件给外部 / 不删除文件 / 不动钱]

### 2.3 🚨 失败兜底
- [模型挂了：fallback 到 X]
- [工具调用失败：retry 3 次 → 报错 → 通知人工]
- [confidence 低：转人工接管]

---

## 3. 架构设计

### 3.1 预诊结论：Workflow vs Agent
**结论**：[Workflow / Agent]
**理由**：
- 任务路径是否事先编排？[是 / 否]
- 是否需要模型动态决定下一步？[是 / 否]
- 失败成本是否值得不确定性？[是 / 否]

### 3.2 基础架构模式
**选择**：[模式名]（Anthropic Building Effective Agents, Dec 2024）
**理由**：[为什么这个模式最匹配]

### 3.3 推理范式
**选择**：[ReAct / Plan-and-Execute / Self-Refine / ...]
**理由**：[为什么选这个]

### 3.4 决策循环图

```
[ASCII 或 Mermaid 图，展示 Input → Reason → Act → Observe 循环或多 agent 拓扑]

例：

  ┌──────────┐
  │  User    │
  │  Input   │
  └─────┬────┘
        ▼
  ┌──────────┐    ┌──────────┐
  │  Plan    │───▶│  Tools   │
  │  (LLM)   │◀───│  Result  │
  └─────┬────┘    └──────────┘
        ▼
  ┌──────────┐
  │  Output  │
  └──────────┘
```

### 3.5 多 Agent 拓扑（如适用）
**拓扑类型**：[Sequential / Hierarchical / Concurrent / Network / Swarm / Lead-Subagent / N/A]
**Agent 列表**：

| Agent 名 | 职责 | 模型 | 工具集 |
|---------|------|------|-------|
| [agent-1] | | | |
| [agent-2] | | | |

---

## 4. 工具集（Tools）

| 工具名 | 用途 | 何时调用 | 何时不调用 | 来源 |
|--------|------|---------|----------|------|
| [tool_1] | [一句话] | [触发条件] | [禁止条件] | MCP server / 自建 |
| [tool_2] | | | | |
| [tool_3] | | | | |

### 4.1 MCP Server 复用
- [优先复用的 MCP server 列表，例：filesystem、github、postgres、slack]

### 4.2 自建工具实现指引
对每个自建工具，给出函数签名 + 一段实现伪代码：

```python
# tool_1: [描述]
def tool_1(arg1: str, arg2: int) -> dict:
    """
    [docstring：何时调用、参数语义、返回值结构]
    """
    # 实现要点：
    # 1. [步骤]
    # 2. [步骤]
    # 3. [错误处理]
    pass
```

---

## 5. 记忆系统

| 层 | 是否启用 | 内容 | 存储方案 | 检索策略 |
|----|---------|------|---------|---------|
| **Working Memory** | ✅ | 当前会话 | 上下文窗口 | 滑动窗口 + summary |
| **Episodic Memory** | [✅/❌] | [内容] | [Mem0 / Letta / Zep / 自建] | [向量 + 时序] |
| **Semantic Memory** | [✅/❌] | [内容] | [Qdrant / pgvector / ...] | [BM25 + 向量 + Rerank] |
| **Procedural Memory** | [✅/❌] | [内容] | [skills 目录 / DB] | [按任务类型查] |

### 5.1 数据 Schema

```python
# 示例：episodic memory schema
class Episode:
    id: str
    user_id: str
    timestamp: datetime
    summary: str          # LLM 生成的摘要，用于检索
    raw_messages: list    # 原始对话
    embedding: list[float]
    metadata: dict
```

---

## 6. 框架与技术栈

### 6.1 主框架
**选定**：[LangGraph / CrewAI / ...]
**版本**：[X.Y.Z]
**理由**：[为什么不是其他框架]

### 6.2 依赖清单

**Python 项目**（`requirements.txt` / `pyproject.toml`）：
```
# 核心
langgraph>=0.2.0
langchain>=0.3.0
anthropic>=0.40.0  # 或 openai / google-generativeai

# 记忆
mem0ai>=0.1.0      # 或 letta / zep-python

# 向量库
qdrant-client>=1.10.0
sentence-transformers>=3.0.0

# 工具
mcp>=0.1.0
httpx>=0.27.0

# 观测
langsmith>=0.1.0   # 或 langfuse / arize-phoenix

# Eval
pytest>=8.0.0
pytest-asyncio>=0.24.0
```

**TypeScript 项目**（`package.json`）：
```json
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.30.0",
    "@modelcontextprotocol/sdk": "^0.5.0",
    "ai": "^4.0.0",
    "@mastra/core": "^0.5.0",
    "langfuse": "^3.0.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "vitest": "^2.0.0",
    "tsx": "^4.0.0"
  }
}
```

### 6.3 模型路由策略

| 任务类型 | 模型 | 理由 |
|---------|------|------|
| 简单分类 / 路由 | [Haiku 4.5 / GPT-4o-mini / Gemini Flash] | 便宜快速 |
| 复杂推理 / 决策 | [Opus 4.7 / GPT-5 / Gemini Pro] | 质量优先 |
| 嵌入 | [voyage-3 / openai text-embedding-3-large / bge-m3] | 检索质量 |

### 6.4 可观测性
**选定**：[LangSmith / Langfuse / Phoenix / Braintrust]
**接入点**：所有 LLM 调用、所有工具调用、记忆读写

### 6.5 沙箱（如适用）
**方案**：[E2B / Modal / Daytona / 无]
**用途**：[隔离代码执行 / 文件操作 / ...]

---

## 7. 项目结构

```
[agent-name]/
├── README.md
├── .env.example                # API keys、配置模板
├── pyproject.toml              # 或 package.json
├── src/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── main.py             # 主 agent 入口
│   │   ├── graph.py            # LangGraph 状态图（如用）
│   │   ├── prompts/
│   │   │   ├── system.md       # 主 system prompt
│   │   │   └── ...
│   │   ├── tools/
│   │   │   ├── __init__.py
│   │   │   ├── tool_1.py
│   │   │   └── tool_2.py
│   │   ├── memory/
│   │   │   ├── __init__.py
│   │   │   └── store.py
│   │   └── guardrails/
│   │       └── filters.py
│   └── eval/
│       ├── golden_set.jsonl    # eval 数据
│       ├── judge.py            # LLM-as-judge
│       └── run_eval.py         # 自动跑 eval
├── tests/
│   ├── test_tools.py           # 单元测试
│   └── test_agent.py
├── scripts/
│   └── trace_replay.py         # 生产 trace 离线重跑
└── docs/
    └── runbook.md              # 运维手册
```

---

## 8. 关键文件代码骨架

### 8.1 主 Agent 入口（`src/agent/main.py` 示例）

```python
# 用 LangGraph 的最小骨架，按需替换
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
from langchain_anthropic import ChatAnthropic

class AgentState(TypedDict):
    messages: list
    next_action: str | None

def reason_node(state: AgentState) -> AgentState:
    """LLM 决定下一步"""
    # TODO: 调 LLM，决定调哪个工具
    pass

def act_node(state: AgentState) -> AgentState:
    """执行工具调用"""
    # TODO: 根据 next_action 调对应 tool
    pass

def should_continue(state: AgentState) -> str:
    """循环条件"""
    return "act" if state.get("next_action") else END

graph = StateGraph(AgentState)
graph.add_node("reason", reason_node)
graph.add_node("act", act_node)
graph.set_entry_point("reason")
graph.add_conditional_edges("reason", should_continue, {"act": "act", END: END})
graph.add_edge("act", "reason")

agent = graph.compile()
```

### 8.2 System Prompt（`src/agent/prompts/system.md`）

```markdown
# [AGENT_NAME] System Prompt

你是 [角色定位]。

## 目标
[一句话目标]

## 可用工具
[工具列表 + 何时用]

## 行为准则
- [准则 1]
- [准则 2]
- 遇到不确定的情况：[行为]
- 危险动作前：[行为，例：先确认]

## 输出格式
[结构化输出要求]
```

### 8.3 Eval 跑批（`src/eval/run_eval.py`）

```python
import json
from pathlib import Path
from src.agent.main import agent
from src.eval.judge import judge_response

def run_eval():
    golden = [json.loads(l) for l in Path("src/eval/golden_set.jsonl").open()]
    results = []
    for case in golden:
        output = agent.invoke({"messages": [case["input"]]})
        score = judge_response(case, output)
        results.append({"id": case["id"], "score": score})
    pass_rate = sum(r["score"]["pass"] for r in results) / len(results)
    print(f"Pass rate: {pass_rate:.1%}")
    return results

if __name__ == "__main__":
    run_eval()
```

---

## 9. Eval 方案

### 9.1 Golden Set（`src/eval/golden_set.jsonl`）

至少 5 条，建议 10–30 条覆盖：
- 主路径（happy path）3–5 条
- 边缘案例（歧义、长尾）2–3 条
- 对抗性输入（注入、越权）2–3 条
- 已知 bug 复现 1–2 条

格式示例：
```jsonl
{"id": "happy-1", "input": "...", "expected": {"contains": ["..."], "tools_called": ["..."], "format": "json"}}
{"id": "edge-1", "input": "...", "expected": {...}}
```

### 9.2 Judge Prompt（`src/eval/judge.py` 内嵌）

评分维度：
- **正确性**（必须）：输出是否准确解决了任务
- **相关性**：是否答非所问
- **安全性**：是否违反红线
- **效率**：是否调用了多余工具

### 9.3 何时跑 Eval
- 改 prompt 后
- 升级模型后
- 新增工具后
- 每周定时跑一次（防止上游模型悄悄变化）

---

## 10. 生产化清单

启动前必须打勾：

- [ ] **Token 经济**
  - [ ] 启用 prompt caching（system prompt 静态部分放前缀）
  - [ ] 设置单次任务 token 上限（防止递归爆炸）
  - [ ] 对话长上下文有压缩策略
- [ ] **延迟**
  - [ ] 流式输出已启用
  - [ ] 工具调用支持并行（如多个独立调用）
- [ ] **可靠性**
  - [ ] LLM 调用有 retry + 指数退避
  - [ ] 关键步骤 checkpoint（LangGraph）或可重入设计
  - [ ] 模型供应商挂掉的降级路径
- [ ] **安全**
  - [ ] 输入 PII 脱敏
  - [ ] 工具白名单机制
  - [ ] 危险动作前 HITL 审批
  - [ ] Prompt injection 防护（输入扫描或专用工具）
- [ ] **可观测性**
  - [ ] 所有 LLM 调用进 trace
  - [ ] 所有工具调用进 trace
  - [ ] 错误率告警
  - [ ] 成本仪表盘
- [ ] **沙箱**（如有代码执行）
  - [ ] 代码在 E2B / 容器中跑
  - [ ] 网络白名单
- [ ] **版本化**
  - [ ] Prompt 进 git
  - [ ] 模型版本锁定（不用 latest）
  - [ ] 回滚预案文档化

---

## 11. 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|---------|
| [风险 1] | 高 | 中 | [措施] |
| [风险 2] | 中 | 高 | [措施] |
| 模型供应商 API 限流 | 高 | 低 | 多供应商 fallback |
| Token 成本超预算 | 中 | 中 | 任务级 token 上限 + 告警 |
| Prompt injection | 高 | 中 | 输入扫描 + 输出过滤 |

---

## 12. 实施 Roadmap（vibe-codable 任务清单）

> **使用方法**：把 AI coding 工具切到这份 spec 所在目录，让它按以下任务清单逐项实现。
> 每个任务都标注了「**验收标准**」和「**预计单文件量级**」，方便 AI 按粒度执行。

### Phase 1：最小可跑骨架（v0.1）

**任务 1.1**：创建项目脚手架
- 验收：`pyproject.toml` 生效、`pip install -e .` 通过、目录结构按 §7
- 量级：S（2–3 文件）

**任务 1.2**：实现主 agent 入口（基于 §8.1 骨架）
- 验收：`python -m src.agent.main "hello"` 能跑通最简单的输入并返回输出
- 量级：M（3–5 文件）

**任务 1.3**：实现 1 个核心工具（先实现 `[tool_1]`）
- 验收：单元测试 `tests/test_tools.py::test_tool_1` 通过
- 量级：S（1–2 文件）

**任务 1.4**：写 5 条 golden set + 跑通 eval 脚本
- 验收：`python -m src.eval.run_eval` 输出 pass_rate
- 量级：M（2–3 文件）

**Checkpoint 1**：基础 agent 能完成主流程的 1 个 happy path 用例

---

### Phase 2：功能完整（v0.5）

**任务 2.1**：实现剩余工具（`[tool_2]`、`[tool_3]`）
- 验收：每个工具都有单元测试通过
- 量级：M

**任务 2.2**：接入记忆系统（按 §5）
- 验收：跨会话能召回历史信息
- 量级：M

**任务 2.3**：接入可观测性（LangSmith / Langfuse）
- 验收：dashboard 能看到 trace
- 量级：S

**任务 2.4**：扩充 golden set 到 15+ 条（含边缘 / 对抗）
- 验收：eval 脚本跑通，pass_rate ≥ [目标]
- 量级：S

**Checkpoint 2**：所有 happy path + 主要边缘案例通过

---

### Phase 3：生产就绪（v1.0）

**任务 3.1**：完成 §10 生产化清单全部打勾
- 验收：清单逐项验证
- 量级：L（多文件）

**任务 3.2**：HITL 审批流（如有危险动作）
- 验收：危险动作前能阻塞等审批
- 量级：M

**任务 3.3**：成本与告警监控
- 验收：超阈值有告警
- 量级：S

**任务 3.4**：编写 `docs/runbook.md`（运维手册：常见问题、回滚步骤）
- 验收：新人能照着排查
- 量级：S

**Checkpoint 3**：可上线，并有完整运维文档

---

## 13. 后续迭代方向

- [ ] [v1.5 想加的能力]
- [ ] [v2.0 想加的能力]
- [ ] [想做但不在 MVP 范围的事]

---

## 附录 A：本架构所采用的关键设计原则

- **Simplicity First**（Anthropic, Dec 2024）：能用 workflow 不用 agent
- **Eval First**：先有 golden set 再写代码
- **Observability First**：第一天就接 trace
- **Cost Awareness**：每个新增能力都问 token 涨多少

## 附录 B：参考资料

- Anthropic, "Building Effective Agents", Dec 2024 — https://www.anthropic.com/engineering/building-effective-agents
- Andrew Ng, "Agentic Design Patterns", deeplearning.ai
- Model Context Protocol — https://modelcontextprotocol.io
- [本项目用到的具体框架文档链接]
