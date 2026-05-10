# AI Agent 架构知识库（2026 Edition）

> **用途**：本文件是 `agent架构师` skill 的深度知识参考。在执行 13 步流程时，按需查阅对应章节给用户引用与解释。
> **维护**：每季度回访 Anthropic / OpenAI / Google 的 agent 工程博客与 LangChain / CrewAI 的 release notes，将新模式 / 新协议增补到对应章节。

---

## 0. 三阶段演进与 Harness Engineering（2025 统摄共识）

### 0.1 三阶段演进

2023–2025 的 AI 工程经历了三次明显的重心迁移，三者是**包含关系**而非替代：

| 阶段 | 解决的核心问题 | 关键工具 / 范式 | 代表时期 |
|------|---------------|----------------|---------|
| **Prompt Engineering** | 模型有没有听懂任务 | 角色设定、few-shot、CoT、约束输出 | 2023 |
| **Context Engineering** | 模型有没有拿到足够且正确的信息 | RAG、上下文裁剪、长文压缩、progressive disclosure（agent skills） | 2024 |
| **Harness Engineering** | 模型在真实执行中能不能持续做对 | 工具系统 / 编排 / 状态 / 评估 / 约束-恢复 的整套外壳 | 2025+ |

```
┌─────────────────────────────────────────────┐
│              Harness Engineering            │
│  ┌────────────────────────────────────┐    │
│  │       Context Engineering          │    │
│  │  ┌────────────────────────┐        │    │
│  │  │   Prompt Engineering   │        │    │
│  │  └────────────────────────┘        │    │
│  └────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

> 当任务还是单轮生成 → Prompt 是关键；
> 当任务依赖外部知识 → Context 是关键；
> 当任务进入长链路、可执行、低容错 → Harness 几乎不可避免。

### 0.2 Harness Engineering 是什么

LangChain 的工程定义：

```
agent = model + harness
harness = agent − model
```

Harness 原意为「缰绳 / 马具 / 约束装置」。放到 AI 系统里，它是一个 agent 系统中除模型本身外几乎所有决定能否稳定交付的东西。

### 0.3 为什么 Prompt / Context 解决不了长程执行

Prompt 优化的是**意图的表达**；Context 优化的是**信息的供给**。两者主要解决「输入侧」的问题。

但在长链路任务中，模型开始连续行动，会出现新一类问题：

- **计划做得很好，执行跑偏** —— 调错工具、误读返回、慢慢偏航而系统没发现；
- **自评失真** —— 模型自己干完自己打分，倾向乐观；
- **上下文焦虑** —— 上下文越来越满，模型开始着急收尾、丢细节；
- **失败是常态** —— 搜索不准、API 超时、文档格式混乱、工具误用，没有恢复机制就只能从头来。

→ 这些问题 prompt 和 context 都管不到。需要一套外圈的「监督 + 约束 + 纠偏」系统，这就是 Harness。

### 0.4 决定上限 vs 决定落地

> 真正决定能力**上限**的可能是模型，但真正决定能否**落地**、能否**稳定交付**的就是 Harness。

LangChain 在自家智能体上做过实测：模型不变，仅改造 harness，就把榜单排名从 30 开外杀到前五（详见 §13）。

### 0.5 一手出处

- LangChain, "The Rise of Context Engineering", 2025 — https://www.langchain.com/blog/the-rise-of-context-engineering
- Anthropic, "How we built our multi-agent research system", 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system
- Anthropic, "Building Effective Agents", Dec 2024 — https://www.anthropic.com/engineering/building-effective-agents

---

## 1. Workflow vs Agent（最重要的判别）

| 维度 | Workflow（工作流） | Agent（智能体） |
|------|-------------------|----------------|
| 控制流 | 由代码 / 图预定义 | 由 LLM 自主决定 |
| 可预测性 | 高 | 低 |
| 调试难度 | 容易 | 困难 |
| Token 成本 | 可预算 | 可能爆炸（递归 / 循环） |
| 上线难度 | 低 | 高 |
| 适用场景 | 路径可枚举的任务 | 开放式、不可预先编排的任务 |
| 典型例子 | 文档摘要管线、客服分流、ETL | SWE 助手、深度研究、Computer Use |

**判断准则（Anthropic, Dec 2024）**：
- 任务路径能否事先编排？
- 是否需要模型动态决定下一步？
- 失败成本是否值得 agent 的不确定性？

> ⚠️ 90% 的「我要做个 agent」需求其实用 workflow 就能更稳更便宜地解决。

---

## 2. 六大基础架构模式（Anthropic, Dec 2024）

| 模式 | 触发条件 | 典型应用 | 复杂度 |
|------|---------|---------|--------|
| **Augmented LLM** | 单次 LLM + 检索 + 工具 + 记忆 | 知识问答助手、智能搜索 | ★ |
| **Prompt Chaining** | 任务可线性拆解为子步 | 文案生成 → 翻译 → 润色 | ★ |
| **Routing** | 输入异质，需分流处理 | 客服意图分类 → 专门处理器 | ★★ |
| **Parallelization** | 子任务可独立并行 | 多角度评分（sectioning）、多 LLM 投票（voting） | ★★ |
| **Orchestrator-Workers** | 子任务在运行时才能确定 | 跨文件代码重构、复杂搜索、多源研究 | ★★★ |
| **Evaluator-Optimizer** | 输出需明确反馈循环改进 | 文学翻译、复杂检索结果筛选 | ★★★ |
| **Autonomous Agent** | 开放式、不可预测路径 | SWE 助手、Computer Use、深度研究 | ★★★★ |

**升级路径**：从 ★ 到 ★★★★，永远先尝试上一级是否够用。

### 2.1 模式选择决策树

```
任务可一步搞定？
├── 是 → Augmented LLM
└── 否 → 任务步骤事先确定？
    ├── 是 → 步骤间需要分流？
    │   ├── 是 → Routing
    │   └── 否 → 步骤可并行？
    │       ├── 是 → Parallelization
    │       └── 否 → Prompt Chaining
    └── 否 → 步骤需运行时决定？
        ├── 子任务有限可枚举 → Orchestrator-Workers
        ├── 输出需迭代改进 → Evaluator-Optimizer
        └── 完全开放式 → Autonomous Agent
```

---

## 3. 四大 Agentic 设计能力（Andrew Ng）

| 能力 | 核心 | 示例 | 何时用 |
|------|------|------|-------|
| **Reflection（反思）** | 自我评估输出 → 改进 | 让模型评判自己的代码再修订 | 输出质量敏感、可迭代改进 |
| **Tool Use（工具使用）** | 调用 API / 函数 / 检索 | 搜索、计算、文件操作、外部 API | 几乎所有 agent 都需要 |
| **Planning（规划）** | 多步任务拆解、子目标管理 | Plan-and-Execute、ReAct 长任务 | 步骤多、需动态调整 |
| **Multi-Agent（多智能体协作）** | 分工与角色化协同 | 写作者 + 评审者、研究者团队 | 单 agent 上下文/能力受限 |

> 💡 这四种能力可以叠加：一个 SWE agent 通常同时具备规划、工具使用、反思，并通过子 agent 协作。

---

## 4. 推理范式（Reasoning Paradigms）

| 范式 | 核心 | 何时用 | 出处 |
|------|------|-------|------|
| **ReAct** | Thought → Action → Observation 循环 | 通用 agent 默认起点 | Yao et al. 2022 |
| **Reflexion** | 失败后自我反思并重试，记入 episodic memory | 需从错误学习的迭代任务 | Shinn et al. 2023 |
| **Tree of Thoughts (ToT)** | 多路径探索 + 投票 / 剪枝 | 解空间需广度搜索的问题 | Yao et al. 2023 |
| **LATS** (Language Agent Tree Search) | MCTS + 反思 | 需高质量结果且能承受 token 开销 | Zhou et al. 2023 |
| **Plan-and-Execute** | 先规划完整路径再执行 | 步骤较多、可预先规划 | LangChain, 2023 |
| **Self-Refine** | 同一 LLM 生成 → 评判 → 修订 | 写作、代码生成等可迭代输出 | Madaan et al. 2023 |
| **Inference-Time Scaling**（o1 类） | 让模型在响应前"思考更久" | 数学、复杂推理；用 extended thinking 控制 | OpenAI o1 (2024), Claude extended thinking |

---

## 5. 记忆系统四层架构

| 类型 | 存储位置 | 内容 | 例 | 工具 |
|------|---------|------|------|------|
| **Working Memory（工作记忆）** | 上下文窗口 | 当前会话 / 任务状态 | 最近 N 轮对话 | 上下文压缩、conversation summary |
| **Episodic Memory（情景记忆）** | 时序事件库 | 具体过往交互 | "上次用户提到偏好 X" | Mem0、Letta、Zep |
| **Semantic Memory（语义记忆）** | 知识图 / 向量库 | 领域知识、事实 | 公司产品手册 | Qdrant、Pinecone、pgvector |
| **Procedural Memory（程序记忆）** | 提示词 / 技能库 | 如何做某类事 | "处理退款的标准流程" | Skills folder、System prompt 模板 |

### 5.1 主流工具栈

- **专用记忆服务**：Mem0、Letta（前 MemGPT）、Zep、Cognee
- **向量数据库**：Qdrant、Weaviate、Pinecone、pgvector、Milvus、Chroma
- **混合检索**：BM25 + 向量 + Rerank（Cohere Rerank / Jina Rerank / Voyage Rerank）
- **嵌入模型**：OpenAI text-embedding-3、Voyage、Jina、bge-m3、Cohere embed v3

### 5.2 设计准则

> 不要把所有信息都丢进上下文。先问「这条信息要存多久？被检索的频率？」再选层。
>
> - 一次性 → 不存
> - 当前会话需要 → working
> - 跨会话回忆 → episodic
> - 通用知识 → semantic
> - 技能复用 → procedural

---

## 6. 工具与协议生态（2026 关键升级）

### 6.1 Function Calling / Tool Use
模型原生工具调用接口。OpenAI、Anthropic、Gemini、Qwen 等主流模型均支持。优先用模型原生接口而不是 prompt hack。

### 6.2 Model Context Protocol (MCP)
- **定位**：Anthropic 主导、跨厂商工具/数据源标准。被誉为「AI 应用的 USB-C」。
- **支持**：Claude、ChatGPT、Cursor、VS Code (GitHub Copilot)、Windsurf、Trae 等主流客户端均已支持。
- **生态**：GitHub、Slack、Notion、Postgres、Filesystem、Brave Search、Puppeteer、Sentry 等都有官方或社区 MCP server。
- **建议**：优先复用 MCP server 而不是自写 wrapper。

### 6.3 Agent-to-Agent (A2A)
- **定位**：Google 主导的跨 agent 通信协议，解决多厂商 agent 互操作问题。
- **状态**：2025 年发布，2026 年逐步落地。可关注但生产慎用。

### 6.4 Computer Use / Browser Use
- **Claude Computer Use**（Anthropic, 2024）：通过截图 + 坐标控制 GUI
- **OpenAI Operator**（2025）：浏览器内自主操作
- **browser-use**（开源）：基于 Playwright 的 agent 浏览器控制库
- **Stagehand**（Browserbase）：AI 友好的浏览器自动化框架

### 6.5 Code Sandboxing
- **E2B**：Firecracker microVM 沙箱
- **Modal**：serverless 代码执行
- **Daytona**：开发环境隔离
- **Riza**：轻量 isolate 沙箱
- **Pyodide**：浏览器内 Python

### 6.6 结构化输出
- OpenAI Structured Outputs（JSON Schema 强保证）
- Anthropic Tool Use（schema-driven）
- Pydantic Schema + LLM
- Outlines（开源约束生成）

### 6.7 Harness 6 层架构（统摄视图）

把 §0 的 Harness Engineering 落到工程结构上，分 6 层。这是本知识库其他章节的**实现层映射表**：每一层对应本文档的具体节、对应 skill 的 13 步、并指向常见反模式。

| 层 | 解决的核心问题 | 关键设计准则 | 本文档实现细节 | 13 步映射 | 常见反模式 |
|---|---|---|---|---|---|
| **L1 上下文（Context）** | 模型在正确的信息边界内思考 | 角色目标定义、信息裁剪与分层、progressive disclosure（agent.md 当目录页） | §5（记忆四层 working/episodic/semantic/procedural）、§6.6 结构化输出 | 步骤 2-3, 7 | agent.md 大杂烩、把所有信息塞 system prompt |
| **L2 工具系统（Tools）** | 给什么 / 何时调 / 结果怎么回写 | 最少工具集、明确触发条件（when to use / NOT use）、工具结果筛选回填而非原样塞回 | §6.1 Function Calling、§6.2 MCP、§6.3 A2A、§6.4 Computer Use | 步骤 6 | 工具描述模糊、无调用预算上限 |
| **L3 执行编排（Orchestration）** | 步骤怎么串、跑偏怎么办 | 推理范式选型、状态机 / 图、任务轨道（理解→补信息→分析→输出→检查→重试）、必要时拆 sub-agent | §2 六大模式、§4 推理范式、§7 多 agent 拓扑 | 步骤 1, 4-5, 8 | 一上来就上多 agent、想到哪做到哪 |
| **L4 记忆与状态（Memory & State）** | 任务状态 / 会话中间结果 / 长期记忆 三者必须分清 | working / episodic / semantic / procedural 分层；checkpoint 可恢复 | §5 记忆系统四层架构、§9.3 checkpoint | 步骤 7 | 记忆库无限制增长、混在一个池子 |
| **L5 评估与观测（Eval & Observability）** | 不让 agent「自我感觉良好」→ 独立验收 + 持续 trace | 生产-验收分离（planner/generator/evaluator）、golden set、judge 校准、trace 全覆盖 | §8 评估与可观测性 | 步骤 10, 12 | 自评失真、没有 trace 就上线 |
| **L6 约束-校验-恢复（Guardrails & Recovery）** | 失败是常态而非例外 | 红线白名单、输出前后校验、retry / circuit breaker / fallback、context reset、HITL 高风险动作审批 | §9.3 可靠性、§9.4 安全护栏、§9.5 HITL | 步骤 3, 11, 13 | 无失败兜底、危险动作没 HITL |

#### 6.7.1 关键设计准则一览

- **Progressive Disclosure（渐进式披露）**：不要一次性把所有能力 / 规范暴露给模型。`agent.md` 应该是目录页 + 索引，详细规范拆到子文档（架构 / 设计 / 执行 / 质量 / 安全），agent 触发时再按需加载。这与 Claude agent skills 的设计哲学同源。
- **Context Reset > Context Compaction**：长程任务中，光把上下文压短不能消除模型的「负担感」（着急收尾、丢细节）。Anthropic 的解法：换一个干净的新 agent 接力，把关键状态作为交接物传递 —— 像内存泄漏后重启进程而不是反复清缓存。
- **Production-Validation Split（生产-验收分离）**：让生成 agent 自己给自己打分会偏乐观。把 planner（拆需求）/ generator（实现）/ evaluator（独立验收，能真实操作页面看结果）三角拆开。
- **Environment-Driven Design（环境驱动设计）**：agent 出问题时，第一反应不是调 prompt 让它「更小心」，而是问「环境里缺什么结构性能力」（缺工具 / 缺反馈链路 / 缺独立验证 / 任务粒度太大 / 看不到自己的工作结果）。OpenAI 的 100% agent 编写代码项目的核心方法论。
- **反馈链路闭环**：agent 必须能看到自己工作的真实结果（浏览器截图 / 日志 / 指标 / eval 跑分），否则就是闭眼提交。

#### 6.7.2 Harness 6 层与单层升级路径

不是一开始就把 6 层都堆满。按需启用：

```
最小 harness：L1 + L2 + L3（基础 ReAct）
+ 跨会话需求 → 启用 L4
+ 进入生产 → 必须启用 L5
+ 涉及外部副作用 / 长程任务 → 必须启用 L6
```

---

## 7. 多 Agent 拓扑

| 拓扑 | 描述 | 适用 | 示例框架 |
|------|------|------|---------|
| **Sequential（流水线）** | A → B → C 线性传递 | 写作管线、ETL、文档处理 | 任意框架 |
| **Hierarchical（层级）** | Manager + Workers，主管派发 | 规划+执行分离、复杂任务分解 | LangGraph、CrewAI |
| **Concurrent（并发）** | 并行扇出 + 聚合 | 多观点辩论、多源检索、并行评分 | LangGraph、AutoGen |
| **Network / Mesh（网状）** | 任意 agent 互联 | 复杂协商、动态协作 | AutoGen |
| **Swarm / Handoff（动态移交）** | 控制权杖在 agent 间动态传递 | 客服分流、多领域助手 | OpenAI Swarm / Agents SDK |
| **Lead-Subagent（主副）** | 主 agent 启动子 agent 并行 | Claude Code、Devin 风格 | Anthropic SDK + 自研 |

### 7.1 拓扑选择准则

> 多 agent 不是免费午餐——通信成本、状态一致性、调试难度都会指数上升。
>
> **顺序**：单 agent + 工具 → 单 agent + sub-agent → 多 agent。
>
> 拆 agent 的合理理由：
> - 上下文隔离（避免污染）
> - 能力分工（不同模型 / 不同 prompt）
> - 并行加速（独立子任务）
>
> 不合理的理由：
> - 「显得高级」
> - 「框架建议这么写」

---

## 8. 评估与可观测性

### 8.1 离线评估
- **Golden Set**：5–50 个真实输入 + 期望输出，每次改 prompt/模型都跑
- **LLM-as-Judge**：用强模型评判弱模型输出。**注意**：需校准（人工标注一批先测 judge 一致性）、避免 judge 偏见（位置偏见、自我偏好）
- **单元 eval**：针对单个工具调用、单个子任务的精确断言（pytest 风格）

### 8.2 在线评估
- A/B 测试、用户反馈（thumbs up/down）
- 含糊响应检测（"I'm not sure" 等）
- 实时 trace 抽样人工标注
- Production replay（生产 trace 离线重跑新版本）

### 8.3 主流基准
| 基准 | 测什么 | 适合 |
|------|-------|------|
| **SWE-Bench** | 真实代码修复 | SWE agent |
| **GAIA** | 通用助手任务 | 通用 agent |
| **AgentBench** | 多领域 agent 能力 | 综合评测 |
| **WebArena / VisualWebArena** | 网页操作 | Browser agent |
| **OSWorld** | 桌面 OS 操作 | Computer Use agent |
| **τ-Bench** | 工具调用与对话 | Function calling agent |
| **MMAU** | Multi-domain agent | 通用 agent 综合 |

### 8.4 观测工具
| 工具 | 强项 |
|------|------|
| **LangSmith** | LangChain 生态深度集成 |
| **Langfuse** | 开源、自托管友好 |
| **Arize Phoenix** | 开源、ML observability 老牌 |
| **Helicone** | proxy 模式、易接入 |
| **Braintrust** | eval-driven、prompt 实验 |
| **W&B Weave** | Weights & Biases 集成 |

---

## 9. 生产化关键议题

### 9.1 Token 经济学
- **Prompt Caching**：Anthropic（5 分钟 TTL，最高 90% 折扣）/ OpenAI（自动）/ Gemini（context caching）
- **上下文压缩**：长会话定期 summarize、重要信息提取
- **结构化输出**：减少冗余 token
- **模型分级路由**：小模型预筛 + 大模型决策

### 9.2 延迟优化
- **Streaming**：SSE 输出
- **并行化工具调用**：模型支持 parallel tool calls
- **推测解码**：小模型起草 + 大模型验证
- **预热**：常用 prompt 预先 cache

### 9.3 可靠性
- **Retry with backoff**：指数退避
- **Circuit Breaker**：失败率超阈值熔断
- **Checkpoint / Resume**：LangGraph 原生支持，长任务必备
- **降级路径**：模型挂了 → 换备用模型 / fallback 文案

### 9.4 安全护栏
- **输入过滤**：PII 脱敏、prompt injection 检测
- **输出过滤**：敏感信息脱敏、毒性检测
- **能力边界**：工具白名单、动作白名单
- **专用工具**：Lakera Guard、Llama Guard、Anthropic prompt injection 检测、ProtectAI

### 9.5 HITL（Human-in-the-Loop）
- **高风险动作**前人工审批断点（删除、转账、外发邮件、合同签署）
- **不确定性升级**：模型自评 confidence 低 → 转人工
- 实现：LangGraph interrupt、自定义审批队列

### 9.6 沙箱
- **代码执行**：E2B / Modal / Daytona
- **文件系统隔离**：chroot / 容器
- **网络白名单**：限制可访问域名

### 9.7 多模型路由
- 简单任务：Haiku 4.5 / GPT-4o-mini / Gemini Flash
- 复杂任务：Opus 4.x / GPT-5 / Gemini Pro
- 路由策略：规则 / 分类器 / 路由 LLM

### 9.8 版本化
- **Prompt 版本控制**：PromptLayer、Langfuse Prompt Management、自建 git
- **模型版本锁定**：避免供应商悄悄升级影响行为
- **回滚预案**：上线即准备一键回滚

---

## 10. 框架选型矩阵

| 框架 | 强项 | 何时选 | 何时不选 |
|------|------|-------|---------|
| **LangGraph** | 有状态图、checkpoint、HITL、流式 | 复杂分支 / 长任务 / 审批断点 / 需 resume | 简单一次性 chain |
| **CrewAI** | 角色化多 agent、易上手、Python | 角色清晰的小队协作（PM+开发+测试） | 需细粒度状态控制 |
| **AutoGen (Microsoft)** | 对话式多 agent、研究友好 | 多 agent 辩论、可定制 group chat | 生产级稳定性敏感 |
| **OpenAI Agents SDK** (Swarm 演进) | 轻量 handoff、官方支持 | OpenAI 生态轻量 agent / 客服分流 | 需跨厂商 / 复杂状态机 |
| **Google ADK + A2A** | 跨厂商互操作、Gemini 优化 | 异构 agent 互联、多供应商策略 | 单 agent 简单场景 |
| **Anthropic SDK + 自研** | 完全可控、贴合 Claude 能力（Computer Use、extended thinking） | 团队具备 LLM 工程能力 / 需深度定制 | 团队希望少写脚手架 |
| **PydanticAI** | 类型安全、Python 后端友好 | 严肃 Python 后端集成 | 复杂多 agent 编排 |
| **Mastra** | TypeScript 全栈 | Node 工程团队 / TS 优先 | Python 数据科学栈 |
| **Vercel AI SDK** | 前端流式 + UI 组件 | Next.js / React 应用 | 复杂后端编排 |
| **LlamaIndex Agents** | RAG 强项 | 知识密集型 agent | 复杂工具/多 agent 编排 |

### 10.1 精简决策树

```
你的需求是？
├── 单 agent + 简单工具调用
│   └── 直接用模型 SDK + Function Calling（最少抽象）
├── 需要分支 / 状态 / 审批断点 / 长任务
│   └── LangGraph
├── 角色化多 agent 协作
│   ├── 快速原型 → CrewAI
│   └── 研究 / 辩论 → AutoGen
├── 跨厂商 / 异构 agent 互联
│   └── A2A 协议 + 各家 SDK
├── 全栈 TypeScript / Node
│   ├── 后端 → Mastra
│   └── 前端 → Vercel AI SDK
├── 知识密集型（大量文档检索）
│   └── LlamaIndex Agents
└── 团队有 LLM 工程能力且需深度定制
    └── 模型原生 SDK + 自研编排（参考 Anthropic / Claude Code 模式）
```

---

## 11. 反模式与红旗

### 11.1 架构层面

❌ **过度设计型**：
- 一上来就上多 agent / LangGraph 复杂图（YAGNI）
- 把 agent 当万能锤（适合 workflow 的硬要 agent）
- 加冗余的反思 / 评判循环（每次调用涨 3–5x token）

❌ **盲飞型**：
- 没有 eval 就开始迭代 prompt（等于闭眼调参）
- 没有 trace / observability 就上线（出问题完全摸黑）
- 不算 token 成本（递归 agent 可能一次烧 100x）

❌ **上下文滥用型**：
- 把所有上下文塞进 system prompt（应分层：常驻 / 动态 / 检索）
- 让记忆库无限制增长（变成「数据沼泽」，检索失效）
- 每轮都重建上下文（不利用 prompt caching）

❌ **工具失控型**：
- 工具描述模糊导致模型乱调
- 没有工具调用预算上限（死循环烧钱）
- 危险工具（删除、转账、外发邮件）没有 HITL 审批

❌ **架构脆弱型**：
- 没有失败兜底与人工接管路径
- 单点 LLM 故障无降级
- 无版本化（改 prompt 出问题无法回滚）

### 11.2 Harness 层面（一线公司踩过的最新坑）

❌ **上下文焦虑型（Context Anxiety）**
- **症状**：长程任务中模型快装满上下文时开始着急收尾、丢细节、甚至自动「找借口结束」。
- **错误做法**：只做 context compaction（压缩历史）。压缩后变短了，但模型感受到的「负担」并未真消失。
- **正确做法**：context reset —— 换一个干净的新 agent 接力，把已确定的关键状态作为交接物传给新 agent。像内存泄漏后重启进程，而不是反复清缓存。

❌ **自评失真型（Self-Evaluation Bias）**
- **症状**：让生成 agent 自己评自己的输出，pass rate 很高但生产环境一塌糊涂。在设计 / 体验 / 完整度等无标准答案的任务上尤其严重。
- **错误做法**：在同一个 agent 里加「请你检查刚才的输出」一段。
- **正确做法**：planner（拆需求）/ generator（实现）/ evaluator（独立验收）三角分离。evaluator 必须是独立 agent，且能真实操作页面 / 调真实接口看实际结果，不只是抽象审查。

❌ **「更努力」幻觉（Try Harder Fallacy）**
- **症状**：agent 跑偏时反复调 prompt 让它「更小心」「更仔细」「请务必……」，没用。
- **错误做法**：prompt 越写越长、越来越多「请记住 / 请确保 / 请注意」。
- **正确做法**：问「环境里缺什么结构性能力」—— 缺工具？缺反馈链路？缺独立验证？任务粒度太大？agent 看不到自己工作的结果？修复几乎从来不是「更努力」，而是补齐缺失的 harness 层。

❌ **agent.md 大杂烩（Agent Doc Dump）**
- **症状**：把所有规范、框架、约定、SOP 一次性塞进 system prompt 或 `agent.md`，agent 表现反而更糊涂。
- **错误做法**：写一个几千行的 `agent.md` 全塞进上下文。
- **正确做法**：`agent.md` 当目录页 + 索引，只保留最少的核心元信息；详细规范拆到架构文档 / 设计文档 / 执行计划 / 质量评分 / 安全规则等子文档里，agent 触发时按需加载（progressive disclosure）。这与 Claude agent skills 的设计哲学同源。

---

## 12. 一线公司 Harness 实践案例

> 这些案例的价值不是细节本身，而是它们暴露的**可迁移设计原则**。

### 12.1 LangChain：Harness 决定上限

- **背景**：在某 agent 榜单上，LangChain 自家智能体排名 30 开外。
- **关键变更**：底层模型完全不变，只改造和迭代 harness（编排 / 工具集 / 评估闭环 / 错误恢复）。
- **结果**：榜单排名跃升至前五。
- **启示**：模型决定能力天花板，但 harness 决定能不能摸到天花板。在抱怨「模型不够强」之前，先问「harness 是不是没做完」。
- **出处**：LangChain blog "The Rise of Context Engineering", 2025 — https://www.langchain.com/blog/the-rise-of-context-engineering

### 12.2 Anthropic：Context Reset + 生产-验收分离

**问题一：上下文焦虑**

- 长程自主任务中，模型上下文越满，行为越异常（着急收尾、丢细节、甚至「假装完成」）。
- 业界常用方案：context compaction（压缩历史）。
- Anthropic 实测发现这不够：压缩后变短了，但「负担感」没消失。
- 解法：**context reset** —— 换一个干净的新 agent 接力，把关键状态作为结构化交接物传递。类比：内存泄漏后重启进程，而不是反复清缓存。

**问题二：自评失真**

- 让生成 agent 自己给自己打分会偏乐观，尤其在设计 / 体验 / 产品完整度类任务。
- 解法：**planner / generator / evaluator 三角分离**。
  - **Planner**：把模糊需求扩展成完整规格
  - **Generator**：逐步实现
  - **Evaluator**：像 QA 一样真实测试，独立 agent，能真实操作页面 / 检查实际结果
- 关键工程原则：**生产-验收分离**。只要评估者足够独立，就能形成「生成 → 检查 → 修复 → 再检查」的有效循环。

- **出处**：Anthropic, "How we built our multi-agent research system", 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system

### 12.3 OpenAI：人类只设计环境 + Progressive Disclosure + 让 agent 看见整个应用

**实践一：人类只设计环境，agent 写所有代码**

- 一个只有几名人类工程师的团队，用 agent 从零构建超百万行代码的生产级应用，100% 代码由 agent 编写。
- 工程师的工作变成三件事：
  1. **拆任务**：把产品目标拆成 agent 能理解的小任务
  2. **给能力**：agent 失败时，问「环境里缺什么能力」而不是「让它更努力一点」
  3. **建反馈链路**：让 agent 真正能看到自己的工作结果

**实践二：Progressive Disclosure（agent.md 当目录页）**

- 早期错误：写了一个巨大的 `agent.md` 把所有规范全塞进去 → agent 更糊涂。
- 改进：`agent.md` 变成目录页 + 索引，详细内容拆到架构 / 设计 / 执行 / 质量 / 安全等子文档，agent 按需钻进去。
- 与 Claude agent skills 同源：上下文优化不是「给得更多」，而是「按需给、分层给、在正确的时机给」。

**实践三：让 agent 看见整个应用（不只让它写代码）**

- 速度上来后，瓶颈不再是「写」而是「验」，人类根本验不过来。
- 解法：让 agent 自己验：
  - 接浏览器（截图、点页面、模拟用户操作）
  - 接日志系统和指标系统（查 log、查监控）
  - 每个任务在独立隔离环境跑（互不影响）
- agent 不再是「写完就说写完了」，而是真的能跑起来看结果、发现 bug、修 bug、再验证。

**实践四：把资深经验写成系统规则**

- 人类 code review 跟不上 agent 提交速度。
- 解法：把资深工程师的经验写成系统规则（模块分层、依赖规则、什么情况必须拦截）。
- 关键：规则不只报错，还把「怎么修」一起反馈给 agent，进入下一轮上下文。
- 这已经不是传统代码规范，而是一套**可持续运行的自动治理系统**。

- **出处**：OpenAI 公开工程访谈与博客，2025（具体 URL 因平台分散；本知识库仅引用经过核实的稳定路径，OpenAI 单条文章 URL 易变动，故以「公开工程访谈 2025」标注）

### 12.4 三家共同的设计哲学

| 设计原则 | LangChain | Anthropic | OpenAI |
|---------|-----------|-----------|--------|
| Harness 决定能否落地 | ✅ 模型不变改 harness | ✅ context reset / 三角分离 | ✅ 人类只设计环境 |
| Progressive Disclosure | — | — | ✅ agent.md 当目录页 |
| 生产-验收分离 | — | ✅ planner/gen/eval | ✅ agent 自己验 + 系统规则 |
| 环境驱动设计 | ✅ harness 是环境 | — | ✅ 缺什么能力 vs 更努力 |
| 反馈链路闭环 | — | ✅ evaluator 真操作 | ✅ 浏览器+日志+指标 |

→ 这就是 `agent架构师` skill 引导用户走 13 步的底层工程哲学。

---

## 13. 术语速查

| 术语 | 含义 |
|------|------|
| **Workflow** | LLM 步骤由代码 / 图预编排的系统 |
| **Agent** | LLM 自主决定行动序列与终止条件的系统 |
| **Harness** | 缰绳 / 马具 / 约束装置；agent 系统中除模型外的整套运行外壳（`agent = model + harness`）|
| **Prompt Engineering** | 让模型听懂任务（角色、few-shot、约束输出）|
| **Context Engineering** | 让模型拿到正确信息（RAG、上下文裁剪、progressive disclosure）|
| **Harness Engineering** | 让模型在真实执行中持续做对（工具 / 编排 / 状态 / 评估 / 约束-恢复 整套外壳）|
| **Context Reset** | 不是 compaction，而是换一个干净的新 agent 接力，把关键状态作为交接物传递 |
| **Context Compaction** | 压缩历史上下文以腾出窗口（仍在原 agent 里）|
| **Progressive Disclosure** | 渐进式披露；agent.md / system prompt 当目录页 + 索引，详细规范按需加载 |
| **Production-Validation Split** | 生产-验收分离；planner / generator / evaluator 三角必须是独立 agent |
| **Environment-Driven Design** | 环境驱动设计；agent 出问题先问「环境里缺什么结构性能力」，而不是「让模型更努力」|
| **ReAct** | Reason + Act 循环范式 |
| **MCP** | Model Context Protocol（Anthropic 主导的工具协议标准）|
| **A2A** | Agent-to-Agent（Google 主导的 agent 互通协议）|
| **HITL** | Human-in-the-Loop（人工审批 / 接管）|
| **Eval** | Evaluation（系统化质量评估）|
| **Trace** | 一次 agent 调用的完整执行轨迹 |
| **Golden Set** | 用于持续评估的高质量样本集 |
| **Prompt Caching** | 复用前缀 prompt 的计算结果以降本提速 |
| **Tool Use / Function Calling** | 模型调用外部函数 / API 的能力 |
| **Computer Use** | 模型通过截图 + 坐标控制 GUI 的能力 |
| **Extended Thinking** | 推理时扩展（让模型在响应前思考更久）|
| **YAGNI** | You Aren't Gonna Need It（避免过度设计）|

---

## 参考一手资料

**Harness Engineering 与 Context Engineering（2025+）**：
- LangChain, "The Rise of Context Engineering", 2025 — https://www.langchain.com/blog/the-rise-of-context-engineering
- Anthropic, "How we built our multi-agent research system", 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system
- OpenAI 公开工程访谈（2025）：人类只设计环境 / agent.md 当目录页 / 让 agent 看见整个应用 / 资深经验写成系统规则（具体 URL 因平台分散，本知识库仅引用经过核实的稳定路径）
- 中文科普：花园老师《最近爆火的 Harness Engineering 到底是个啥》, 2025（"科特秘密花园" 视频，对三阶段演进与 6 层 Harness 的系统讲解）

**Agent 架构基础**：
- Anthropic, "Building Effective Agents", Dec 2024 — https://www.anthropic.com/engineering/building-effective-agents
- Andrew Ng, "Agentic Design Patterns" series, deeplearning.ai, 2024
- Anthropic, Model Context Protocol spec, 2024 — https://modelcontextprotocol.io
- Google, A2A Protocol spec, 2025
- Lilian Weng, "LLM Powered Autonomous Agents", 2023
- Chip Huyen, "Agents", huyenchip.com, 2025
- LangChain blog, agent tracks, 2024–2026
- OpenAI Agents SDK docs, 2025–2026
