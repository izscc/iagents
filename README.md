# iagents

适用于 [Claude Code](https://claude.com/claude-code) 等 LLM 编程工具的 **AI Agent 设计 skill 集合**。

> **i** = iterative（迭代）/ integrated（集成）/ intelligent（智能）——帮你按正确方式设计 AI agent：simple-first、eval-first、observability-first。

---

## 已有 Skills

### 🎯 `agent架构师` —— AI Agent 架构启发式导师

通过 13 步「迭代式发现」引导你从模糊的 agent 想法走到完整、**vibe-coding 就绪** 的架构方案。

**最终交付**：一份 markdown 文件，可直接喂给 Claude Code / Cursor / Trae / Windsurf 完成实现。

**基于 2026 SOTA**：
- **Harness Engineering（2025 一线共识）** —— `agent = model + harness`，13 步流程对应 6 层 Harness（上下文 / 工具 / 编排 / 记忆 / 评估 / 约束-恢复）
- Anthropic *Building Effective Agents* (Dec 2024) —— workflow vs agent + 6 大基础模式
- Andrew Ng 四大 agentic 设计能力（Reflection / Tool Use / Planning / Multi-Agent）
- Model Context Protocol (MCP) / Agent-to-Agent (A2A)
- 主流框架：LangGraph、CrewAI、AutoGen、OpenAI Agents SDK、Mastra、Vercel AI SDK
- Eval-First & Observability-First 工程实践

---

## 安装

### 方式 A：复制（一次性）

```bash
git clone https://github.com/izscc/iagents.git /tmp/iagents
mkdir -p ~/.claude/skills
cp -R /tmp/iagents/skills/agent架构师 ~/.claude/skills/
```

### 方式 B：软链接（`git pull` 即同步更新）

```bash
git clone https://github.com/izscc/iagents.git ~/iagents
mkdir -p ~/.claude/skills
ln -s ~/iagents/skills/agent架构师 ~/.claude/skills/agent架构师
```

### 验证

新开一个 Claude Code 会话，输入 `/agent架构师` —— 应该能看到 skill 被激活，并向你提出「需求三角诊断」的问题。

---

## 使用

在 Claude Code（或任何兼容 `~/.claude/skills/` 的客户端）里调用：

```
/agent架构师
```

或者直接用自然语言描述需求 —— skill 会被自动识别：

> "我想做一个 agent，每周一早上自动拉取我的 Stripe 报告并写一段中文摘要邮件给老板。"

skill 会按以下流程跑：

1. **需求三角诊断** —— 场景 / 约束 / 当前能力
2. **13 步引导** —— 从 workflow vs agent 预诊一路走到生产化清单
3. **生成 `./[agent名称]-架构方案.md`** 到你当前的工作目录

然后把这份 markdown 拖给你的 AI coding 工具，告诉它：

> "读这份 spec 并按 Phase 1 的任务清单逐步实现，每完成一个任务给我看 diff 后再继续。"

---

## 方法论：13 步迭代式发现

| 阶段 | 步骤 | 输出 |
|------|------|------|
| **0. 预诊** | 1. Workflow vs Agent | 明确架构选择 |
| **1. 使命** | 2. 定义使命 · 3. 边界与红线 | 可衡量目标 + 红线 |
| **2. 模式** | 4. 基础模式 · 5. 决策循环 | 模式选择 + ASCII/Mermaid 图 |
| **3. 能力** | 6. 工具 + MCP · 7. 记忆 · 8. 拓扑 | 工具集 / 记忆 schema / agent 拓扑 |
| **4. 实现** | 9. 框架选型 + 骨架 · 10. Eval 先行 | 代码骨架 + golden set |
| **5. 迭代** | 11. 测试与破坏 · 12. 调试 + 可观测 · 13. 生产化 | 生产就绪的 agent |

### 三大核心原则

1. **Simplicity First（简单优先）** —— 能用 workflow 不用 agent；只在必要时才升级。
2. **Eval First（Eval 先行）** —— 写第一行代码前先约定 5–10 个 golden 用例。
3. **Harness First（Harness 先行）** —— 决定 agent 能否落地的是除模型外的整套系统；改 prompt 之前先问 harness 6 层是否完整。可观测性（trace / log / 指标）是 Harness 第 5 层的实现，第一天就接入（LangSmith / Langfuse / Phoenix）。

---

## Harness Engineering 视角

> 2025 业界的统摄共识：`agent = model + harness`。本 skill 把这个视角嵌入 13 步流程。

### 三阶段演进（包含关系，非替代）

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

- **Prompt Engineering**：让模型听懂任务（角色、few-shot、约束输出）
- **Context Engineering**：让模型拿到正确信息（RAG、长文压缩、progressive disclosure）
- **Harness Engineering**：让模型在真实执行中持续做对（工具 / 编排 / 状态 / 评估 / 约束-恢复 整套外壳）

### 13 步与 Harness 6 层的映射

| Harness 层 | 解决的问题 | 13 步映射 |
|----|---|----|
| **L1 上下文** | 角色 / 信息边界 / progressive disclosure | 步骤 2-3, 7 |
| **L2 工具系统** | 给什么 / 何时调 / 结果回写 | 步骤 6 |
| **L3 执行编排** | 步骤怎么串、跑偏怎么办 | 步骤 1, 4-5, 8 |
| **L4 记忆与状态** | working / episodic / semantic / procedural | 步骤 7 |
| **L5 评估与观测** | 独立验收 + 持续 trace（生产-验收分离） | 步骤 10, 12 |
| **L6 约束-校验-恢复** | 红线 / 校验 / retry / context reset | 步骤 3, 11, 13 |

参考：
- LangChain, "The Rise of Context Engineering", 2025 — https://www.langchain.com/blog/the-rise-of-context-engineering
- Anthropic, "How we built our multi-agent research system", 2025 — https://www.anthropic.com/engineering/built-multi-agent-research-system

---

## 文件结构

```
iagents/
├── README.md
├── LICENSE                                          MIT
├── .gitignore
└── skills/
    └── agent架构师/
        ├── SKILL.md                                 主流程 + 三大原则 + 反模式
        ├── reference/
        │   └── architecture-knowledge.md           完整 2026 知识库
        └── templates/
            └── agent-architecture-template.md      vibe-coding 优化的输出模板
```

---

## 为什么需要这个 skill？

市面上「AI agent 怎么做」的内容大多落入两个极端：

- **太抽象**：「用这些设计模式构建 agent」—— 完全不知道从哪开始。
- **太框架化**：「这是一个 LangGraph 教程」—— 还没决定要不要用，就被绑死。

`agent架构师` 卡在中间：它是一个 **苏格拉底式导师**，按正确顺序问正确的诊断问题，必要时引用 2026 主流模式，**只在你已经赚到这个决策权时才推荐具体框架**。

更重要的是，它把 **Harness Engineering**（2025 业界统摄共识：`agent = model + harness`）作为底层视角，13 步流程显式对应 6 层 Harness。生成的方案不是「选什么框架」的高层决策，而是「6 层 harness 各靠什么落地」的可执行清单 —— 包括对一线公司踩过的最新坑（上下文焦虑 / 自评失真 /「更努力」幻觉 / agent.md 大杂烩）的反模式纠偏。

输出是 **vibe-coding 就绪** 的 —— 不是「考虑做 X」的高层文档，而是带项目结构、依赖清单、代码骨架、eval 用例、Harness 6 层映射表、环境驱动设计自检和分阶段任务路线图的结构化 spec —— AI coding 工具可以一步步执行。

---

## Roadmap

- [ ] 更多 skills（eval 设计、prompt 工程、MCP server 设计……）
- [ ] `harness-debugger` skill —— 针对线上 agent 不稳定时的 6 层 Harness 体检（按层定位是缺哪一层导致跑偏）
- [ ] `agent架构师` 的英文版
- [ ] 常见场景的示例架构 spec（客服、研究、SWE 助手）

---

## 贡献

欢迎 issues 与 PR。新增 skill 的步骤：

1. 创建 `skills/<your-skill-name>/SKILL.md`，带正确的 frontmatter
2. 参考 `agent架构师` 的结构（SKILL.md + 可选的 `reference/` 与 `templates/`）
3. 在本 README 的「已有 Skills」章节加上你的 skill

---

## License

MIT —— 详见 [LICENSE](LICENSE)。

## 作者

[@izscc](https://github.com/izscc)
