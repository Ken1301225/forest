# Forest

Multi-agent deliberation system for OpenCode. Coordinates a configurable panel of AI models through structured multi-round discussion to produce optimal plans with minimal code changes.

## How It Works

1. **Configure** — Forest asks which subagents to use (up to 6 different models) and how many discussion rounds
2. **Phase 0** — Forest explores your project and builds context
3. **Phase A** — All subagents independently analyze the problem and produce plans
4. **Phase B** — Each subagent cross-validates the others' plans, identifying gaps and strengths
5. **Consensus** — Forest checks for convergence; continues to more rounds if needed
6. **Synthesis** — Forest produces a unified final plan combining the best insights

## Architecture

| Component | Role | Model |
|-----------|------|-------|
| `forest` (primary) | Deliberation coordinator | deepseek-v4-pro |
| `forest-dsv4` | Analyst | deepseek-v4-pro |
| `forest-kimi` | Analyst | kimi-k2.6 |
| `forest-mimo` | Analyst | mimo-v2.5-pro |
| `forest-qwen` | Analyst | qwen3.6-plus |
| `forest-minimax` | Analyst | minimax-m2.7 |
| `forest-glm` | Analyst | glm-5.1 |

> **Note**: `forest-dsv4` uses the same model as the primary coordinator. It still provides independent reasoning in a fresh context, but does not add model diversity.

## Key Design Principles

- **Isolated context** — every subagent dispatch is a fresh session
- **Parallel dispatch** — independent phases run simultaneously
- **Streaming Phase B** — cross-validation starts as soon as 2 plans are ready, no need to wait for all
- **Evidence-based** — plans grounded in actual project code, not assumptions
- **Read-only** — Forest produces plans, not code changes

## Quick Install

In OpenCode:

```
Fetch and follow instructions from https://raw.githubusercontent.com/<your-repo>/main/INSTALL.md
```

Or manually — see [INSTALL.md](INSTALL.md).

## Verification

Start a new OpenCode session, switch to the **Forest** agent (Tab), and ask:

> "Analyze this project for potential bugs"

Forest should load the orchestration skill, ask you to configure subagents and rounds, then begin deliberation.

## License

MIT
