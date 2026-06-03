# ps-claude-skills

A small collection of [Claude Code](https://claude.com/claude-code) skills by Peter Stanchev. Each one packages a repeatable, agentic workflow as a single `SKILL.md` that Claude can invoke on demand.

| Skill | Invoke | What it does |
|-------|--------|--------------|
| **grill-me** | `/grill-me` | Interviews you relentlessly about a plan or design — one question at a time — until every decision branch is resolved. |
| **owasp-audit** | `/owasp-audit [opus\|sonnet]` | Multi-agent OWASP Top 10 security audit: one finder per category, an adversarial skeptic that tries to refute each finding, then a severity-ranked synthesis. |
| **test-author-skeptic** | `/test-author-skeptic [sonnet\|opus] [--auto] [--recheck-all]` | Mutation-verified test authoring: an author writes/augments tests, an independent skeptic mutates the source to prove the tests actually catch bugs, and a strengthen loop closes the gaps. |

> The folder `skills/owasp-audit-opus/` defines the skill whose invocation name is `owasp-audit`.

## What's the common thread?

All three lean on the **author + critical skeptic** pattern: one agent produces work, a second *independent* agent adversarially tries to tear it down, and only what survives is kept. `owasp-audit` and `test-author-skeptic` fan this out across many agents via the Claude Code **Workflow** tool; `grill-me` turns the skeptic on *you* to harden a plan before any code is written.

## Install

### Option A — Plugin marketplace (recommended, one-command updates)

```
/plugin marketplace add PeterStanchev/ps-claude-skills
/plugin install ps-skills@ps-claude-skills
```

Restart the session (or reload) and the three skills appear in the skill list.

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

- **Claude Code** with the Skill system. `owasp-audit` and `test-author-skeptic` also use the **Workflow** tool (multi-agent orchestration) and run in the background.
- Both multi-agent skills need a **git repository** to run against.
- **`owasp-audit`** is read-only (it audits, it doesn't change code). Pass `opus` (default) or `sonnet` to pick the model for every agent.
- **`test-author-skeptic`** writes files and runs your test suite. It defaults to **sonnet** for cost; pass `opus` for deeper reasoning. It is **token-heavy** — mutation testing reruns the suite once per mutant, so cost scales with (branches × targets × rounds). Keep target lists tight and prefer a real mutation tool (`mutmut`/`cosmic-ray`/`Stryker`) at scale. It is resumable: same-session via the Workflow run id, cross-session via a `.test-author-skeptic/verified.json` ledger.

## License

[MIT](LICENSE) © 2026 Peter Stanchev
