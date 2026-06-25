# ps-claude-skills

A small collection of [Claude Code](https://claude.com/claude-code) skills by Peter Stanchev. Most package a repeatable, agentic workflow as a single `SKILL.md` that Claude can invoke on demand; two encode a reference standard.

| Skill | Invoke | What it does |
|-------|--------|--------------|
| **grill-me** | `/grill-me` | Interviews you relentlessly about a plan or design — one question at a time — until every decision branch is resolved. |
| **owasp-audit** | `/owasp-audit [opus\|sonnet]` | Multi-agent OWASP Top 10 security audit: one finder per category, an adversarial skeptic that tries to refute each finding, then a severity-ranked synthesis. |
| **perf-audit** | `/perf-audit [opus\|sonnet]` | Multi-agent performance audit: one finder per perf category (N+1, slow queries, complexity, allocs, blocking I/O, caching, network, frontend, leaks), a skeptic that kills micro-opt and cold-path noise, then an impact-ranked synthesis. |
| **test-author-skeptic** | `/test-author-skeptic [sonnet\|opus] [--auto] [--recheck-all]` | Mutation-verified test authoring: an author writes/augments tests, an independent skeptic mutates the source to prove the tests actually catch bugs, and a strengthen loop closes the gaps. |
| **ml-eval** | `/ml-eval` | Pick and compute the right ML/LLM/RAG evaluation metric — regression, classification, text-generation, LLM-judge — and validate automatic scores against human judgment. |
| **fastapi-skeleton** | `/fastapi-skeleton` | Scaffold or standardize a FastAPI service on the official "Bigger Applications" layout — routers, dependencies, config, health — with a GPU/ML model-serving addendum. |

## What's the common thread?

Four of the skills lean on the **author + critical skeptic** pattern: one agent produces work, a second *independent* agent adversarially tries to tear it down, and only what survives is kept. `owasp-audit`, `perf-audit`, and `test-author-skeptic` fan this out across many agents via the Claude Code **Workflow** tool; `grill-me` turns the skeptic on *you* to harden a plan before any code is written.

`ml-eval` and `fastapi-skeleton` are a different kind — **reference skills** that encode a recognized standard (ML evaluation metrics; the official FastAPI project layout) for Claude to apply on demand.

## Install

### Option A — Plugin marketplace (recommended, one-command updates)

```
/plugin marketplace add PeterStanchev/ps-claude-skills
/plugin install ps-skills@ps-claude-skills
```

Restart the session (or reload) and the four skills appear in the skill list.

### Option B — Plain copy (no plugin machinery)

Copy the skill folders into your user skills directory:

```bash
# macOS / Linux
cp -r skills/* ~/.claude/skills/

# Windows (PowerShell)
Copy-Item skills\* "$env:USERPROFILE\.claude\skills\" -Recurse
```

Then restart the session. Skills are just `SKILL.md` files — no build step.

## Requirements & notes

- **Claude Code** with the Skill system. `owasp-audit`, `perf-audit`, and `test-author-skeptic` also use the **Workflow** tool (multi-agent orchestration) and run in the background.
- These multi-agent skills need a **git repository** to run against.
- **`owasp-audit`** is read-only (it audits, it doesn't change code). Pass `opus` (default) or `sonnet` to pick the model for every agent. Optionally scope to a `<path>` or `--diff` (branch changes only); on completion it offers to write `SECURITY-AUDIT.md`.
- **`perf-audit`** is read-only. Pass `opus` (default) or `sonnet`; optionally scope to a `<path>` or `--diff`. Its skeptic discards micro-optimizations and cold-path findings, and the synthesis points at a benchmark/profiler to confirm each win — perf is empirical. Offers to write `PERF-AUDIT.md`.
- **`test-author-skeptic`** writes files and runs your test suite. It defaults to **sonnet** for cost; pass `opus` for deeper reasoning. It is **token-heavy** — mutation testing reruns the suite once per mutant, so cost scales with (branches × targets × rounds). Keep target lists tight and prefer a real mutation tool (`mutmut`/`cosmic-ray`/`Stryker`) at scale. It is resumable: same-session via the Workflow run id, cross-session via a `.test-author-skeptic/verified.json` ledger.

## License

[MIT](LICENSE) © 2026 Peter Stanchev
