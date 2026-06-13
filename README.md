# Payment Agent Skills

**Domain-specific engineering skills for payment systems, designed for AI coding agents.**

10 skills covering the two layers that general-purpose agent skills miss:

| Layer | Skills | Source |
|-------|--------|--------|
| **Payment domain** | 4 skills — channel adapter, verification pipeline, log tracing, data flow debugging | Original |
| **General engineering** | 6 skills — planning, code review, incremental impl, simplification, git workflow, spec-driven | Adapted from [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) |

```
  ONBOARD        VERIFY         TRACE          DEBUG
 ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
 │Channel │    │ Verify │    │  Log   │    │  Data  │
 │Adapter │───▶│Pipeline│───▶│ Trace  │───▶│  Flow  │
 └────────┘    └────────┘    └────────┘    └────────┘
```

## Who This Is For

Payment systems that:
- Integrate multiple banks/wallets/third-party channels
- Have order timeout → hold → verification → unfreeze lifecycles
- Use distributed logging (ELK, Loki, etc.)
- Need systematic debugging when stored data doesn't match source data

## Skills

### Payment Domain (4)

| Skill | What It Does |
|-------|-------------|
| `payment-channel-onboard` | 7-step checklist for adding a new payment channel — Handler, Dispatch, Collector, Extractor |
| `unfreeze-verification` | Dual-path (TIMEOUT + HOLD_RETRY) multi-step verification pipeline before releasing funds |
| `es-log-tracing` | Cross-index log tracing via centralized logging — query templates, tag conventions, pitfalls |
| `upi-flow-debug` | 5-step diagnosis when stored data diverges from source — extractor architecture, change detection differences |

### General Engineering (6)

| Skill | Adapted From |
|-------|-------------|
| `planning-and-task-breakdown` | addyosmani/agent-skills |
| `code-review-and-quality` | addyosmani/agent-skills |
| `incremental-implementation` | addyosmani/agent-skills |
| `code-simplification` | addyosmani/agent-skills |
| `git-workflow-and-versioning` | addyosmani/agent-skills |
| `spec-driven-development` | addyosmani/agent-skills |

General skills adapted with GitLab MR support and payment-system examples.

## Usage

Skills are plain Markdown — they work with any agent that accepts instruction files:

- **Hermes Agent**: Copy to `~/.hermes/skills/devops/` or `~/.hermes/skills/software-development/`
- **Claude Code**: `claude --plugin-dir /path/to/payment-agent-skills`
- **Cursor**: Copy to `.cursor/rules/`
- **Any agent**: Include SKILL.md content in your system prompt or AGENTS.md

## Credits

- 6 general engineering skills adapted from [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) by Addy Osmani
- 4 domain-specific payment skills are original — patterns generalized from production payment aggregator systems
