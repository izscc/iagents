# iagents

A collection of AI agent design skills for [Claude Code](https://claude.com/claude-code) and other LLM coding tools.

> **i** = iterative, integrated, intelligent — skills that help you design AI agents the right way: simple-first, eval-first, observability-first.

---

## Available Skills

### 🎯 `agent架构师` — AI Agent Architecture Mentor

A heuristic mentor that guides you step-by-step from a vague agent idea to a complete, **vibe-coding-ready** architecture spec.

**What you get**: A markdown file that can be fed directly into Claude Code / Cursor / Trae / Windsurf to vibe-code the actual implementation.

**Built on 2026 SOTA**:
- Anthropic's *Building Effective Agents* (Dec 2024) — workflow vs agent + 6 base patterns
- Andrew Ng's four agentic design patterns (Reflection / Tool Use / Planning / Multi-Agent)
- Model Context Protocol (MCP) and Agent-to-Agent (A2A)
- Modern frameworks: LangGraph, CrewAI, AutoGen, OpenAI Agents SDK, Mastra, Vercel AI SDK
- Eval-First & Observability-First engineering practice

---

## Install

### Option A: Copy (one-shot)

```bash
git clone https://github.com/izscc/iagents.git /tmp/iagents
mkdir -p ~/.claude/skills
cp -R /tmp/iagents/skills/agent架构师 ~/.claude/skills/
```

### Option B: Symlink (auto-update on `git pull`)

```bash
git clone https://github.com/izscc/iagents.git ~/iagents
mkdir -p ~/.claude/skills
ln -s ~/iagents/skills/agent架构师 ~/.claude/skills/agent架构师
```

### Verify

In a new Claude Code session, run `/agent架构师` — you should see the skill activate and ask the **Requirements Triangle** diagnosis questions.

---

## Usage

In Claude Code (or any compatible client supporting `~/.claude/skills/`), invoke:

```
/agent架构师
```

Or just describe your need in natural language — the skill will be auto-detected:

> "I want to build an agent that reads my Stripe weekly report and emails a Chinese summary to my CEO."

The skill will then:

1. **Run a Requirements Triangle Diagnosis** — scenario / constraints / current capability
2. **Walk through 13 steps** — from workflow-vs-agent triage to production checklist
3. **Generate `./[agent-name]-架构方案.md`** in your current working directory

Then you take that markdown and tell your AI coding tool:

> "Read this spec and implement Phase 1's task list step by step. Show me the diff after each task before continuing."

---

## Methodology: 13-Step Iterative Discovery

| Phase | Steps | Output |
|-------|-------|--------|
| **0. Triage** | 1. Workflow vs Agent | Clear architectural decision |
| **1. Mission** | 2. Define purpose · 3. Boundaries | Measurable goals + red lines |
| **2. Pattern** | 4. Base pattern · 5. Decision loop | Architecture choice + ASCII/Mermaid diagram |
| **3. Capability** | 6. Tools + MCP · 7. Memory · 8. Topology | Tool list, memory schema, agent topology |
| **4. Implementation** | 9. Framework + skeleton · 10. Eval-first | Code skeleton + golden set |
| **5. Iteration** | 11. Test & break · 12. Debug + observe · 13. Productionize | Production-ready agent |

### Three Core Principles

1. **Simplicity First** — Use a workflow if you can; only escalate to an agent when truly needed.
2. **Eval First** — Define 5–10 golden cases before writing the first line of code.
3. **Observability First** — Hook in trace (LangSmith / Langfuse / Phoenix) from day one.

---

## File Structure

```
iagents/
├── README.md
├── LICENSE                                   MIT
├── .gitignore
└── skills/
    └── agent架构师/
        ├── SKILL.md                          Main flow + principles + anti-patterns
        ├── reference/
        │   └── architecture-knowledge.md    Full 2026 knowledge base
        └── templates/
            └── agent-architecture-template.md  Vibe-coding-optimized output template
```

---

## Why This Skill?

Most "AI agent how-to" content falls into two camps:

- **Too abstract**: "Build an agent with these design patterns" — no idea where to start.
- **Too framework-specific**: "Here's a LangGraph tutorial" — locks you in before you've decided you need it.

`agent架构师` sits in between: it's a **Socratic mentor** that asks the right diagnostic questions in the right order, references the right 2026 patterns when relevant, and only commits to a specific framework after you've earned that decision.

The output is **vibe-coding-ready**: not a high-level "consider doing X" doc, but a structured spec with project layout, dependency lists, code skeletons, eval cases, and a phased task roadmap that an AI coding tool can execute step by step.

---

## Roadmap

- [ ] More skills (eval design, prompt engineering, MCP server design, ...)
- [ ] English version of `agent架构师`
- [ ] Example architecture specs for common scenarios (customer service, research, SWE assistant)

---

## Contributing

Issues and PRs welcome. To add a new skill:

1. Create `skills/<your-skill-name>/SKILL.md` with proper frontmatter
2. Follow the structure of `agent架构师` (SKILL.md + optional reference/ and templates/)
3. Update this README's "Available Skills" section

---

## License

MIT — see [LICENSE](LICENSE).

## Author

[@izscc](https://github.com/izscc)
