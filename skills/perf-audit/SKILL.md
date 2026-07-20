---
name: perf-audit
description: Run a multi-agent performance audit on the current codebase. Optionally pass a model (`sonnet`, `opus`, `haiku`, or `fable`) to pin every finder/verifier agent — omitted, the invoking agent picks one to fit repo size and criticality; optionally scope to a `<path>` or to branch changes only with `--diff`. Launches a background Workflow with one finder per performance category (N+1 queries, slow queries, algorithmic complexity, hot-path allocations, blocking I/O, caching, network, frontend/bundle, resource leaks), adversarially verifies each finding with a skeptic that kills micro-optimizations and cold-path noise, then synthesizes survivors by impact with file:line refs and offers to write a PERF-AUDIT.md report. Use when the user asks for a performance audit, perf review, optimization pass, to find bottlenecks / slow code / N+1 queries, or to "profile using agents" (optionally naming a model).
argument-hint: [sonnet|opus|haiku|fable] [--diff | <path>]
---

# Performance multi-agent audit

Fan out one **finder** per performance category, then have a **skeptic** try to refute each
finding before it counts. Pipeline (no barrier) — a finding verifies the moment its category
finishes. You synthesize survivors by impact.

The skeptic's job is perf-specific: kill **micro-optimizations** (negligible wins) and
**cold-path** findings (code that rarely runs or runs over small `n`). Static analysis only
surfaces *candidates* — a real win must sit on a hot path and be measurable. So the audit
ends pointing at measurement, not declaring victory.

This skill calls the **Workflow** tool — invoking the skill IS the multi-agent opt-in.
Needs a git repo. Audit agents are read-only (no worktree isolation).

## Model selection (optional argument)
An explicit argument (`sonnet` | `opus` | `haiku` | `fable`) pins that model on **every** agent.

No argument → **you decide, deliberately, per situation.** Cost scales with the fan-out
(~10 finders + one verifier per finding), so weigh repo size, how load-bearing the perf work
is, and what the session model costs:
- Broad sweep, large repo, routine gate → `sonnet`.
- Deep pass on a genuinely hot, revenue-critical path → `opus` or `fable`.
- **Never inherit the session model silently.** Omitting `model` on `agent()` inherits it —
  on a fable session that multiplies the most expensive model across the whole fan-out.
  Inherit only when that depth is the point, and say so.

Pass the choice as `args: { model: 'sonnet' | 'opus' | 'haiku' | 'fable' | null }` —
`null`/absent means agents inherit the session model. State the model (or "inherit") and
the rough agent count when you launch.

## Scope (optional argument)
Default audits the whole repo. Narrow it when the repo is large or you only care about recent work:
- `--diff` → audit only files changed on the current branch vs the main branch (PR-gating). Resolve
  the changed-file list during recon and pass it into every finder's SCOPE.
- `<path>` → audit only that directory/file subtree.

State the scope when you launch; recon (step 1) and every finder's `focus` must respect it.

## Procedure

### 1. Recon the codebase (inline, before the workflow)
Map the target so finder prompts are grounded, not generic. Gather:
- Stack: language/framework, server entrypoint, request/render paths, frontend.
- Data layer: DB + ORM/query style, where queries live, loops that touch the DB (N+1 surface),
  pagination/limits, indexing.
- Hot paths: request handlers, render loops, anything that runs per-request or over user-sized data.
- Outbound: server-side `fetch`/HTTP calls, fan-out, retries (network cost).
- Caching: any cache layer, memoization, CDN, HTTP caching headers.
- Frontend/bundle (if web): bundler, code-splitting, re-render hot spots, asset/bundle weight.
- **Existing evidence**: profiler output, benchmarks, slow-query logs, APM traces, a bundle-size
  report — anything that turns reasoning into measurement. Note what exists; the agents prefer it.
- **Performance model**: what is actually hot — request volume, data sizes / `n`, latency or
  throughput budget (p99), what a user notices. Write this down — every agent uses it to calibrate
  impact (don't flag an O(n²) over a fixed 5-element list; don't ignore an N+1 in the main feed).

### 2. Build the category list
Keep the categories that map to real surface in this codebase (drop the rest; total varies by stack):
SQL/ORM **N+1**, **slow queries** (missing index / full scan / `SELECT *` / unbounded result set),
**algorithmic complexity** (accidental quadratic, nested loops over large `n`), **hot-path
allocations / repeated work** (no memoization, rebuilding immutable data per call), **blocking I/O**
(sync I/O or serial `await`s on an async path, event-loop stalls), **concurrency** (lock contention,
missing parallelism, pool exhaustion), **network** (chatty/ waterfall calls, no batching, payload
bloat, no compression), **caching** (missing/ineffective, stampede, wrong TTL), **frontend/bundle**
(unnecessary re-renders, unmemoized props, no code-splitting, large assets), **resource leaks**
(unbounded caches, leaked connections/handles/listeners). Optionally WebSearch framework-specific
perf guidance (e.g. an ORM's N+1 docs) to sharpen a finder's `focus`.

### 3. Author & launch the Workflow
Adapt the template: fill `STACK` from step 1, prune/extend `CATEGORIES` from step 2 (rewrite each
`focus` to point at the real files you found). Launch in **background**, passing the chosen model
(or null to inherit): `Workflow({ script: <filled>, args: { model: 'sonnet' } })`.

```js
export const meta = {
  name: 'perf-audit',
  description: 'Multi-agent performance audit via finder+verifier agents (skeptic kills micro-opt/cold-path noise)',
  phases: [
    { title: 'Find', detail: 'one perf auditor per category' },
    { title: 'Verify', detail: 'skeptic refutes cold-path / micro-opt findings' },
  ],
}

// Model: an explicit arg pins every agent; null/absent -> agents inherit the session model.
const ALLOWED_MODELS = ['sonnet', 'opus', 'haiku', 'fable']
const MODEL = args && typeof args.model === 'string' && ALLOWED_MODELS.includes(args.model.toLowerCase())
  ? args.model.toLowerCase() : null
const MOPT = MODEL ? { model: MODEL } : {}

// FILL from recon: stack facts + explicit performance model + any existing evidence. Every agent reads this.
const STACK = `STACK & PERFORMANCE MODEL
<framework, entrypoint, request/render paths, DB & ORM/query style, caching layer, outbound calls,
frontend/bundle if any, AND existing evidence: profiler output / benchmarks / slow-query log / bundle report>
PERFORMANCE MODEL: <what is actually hot — request volume, data sizes / n, latency or throughput
budget (p99), what users notice; how to calibrate impact — ignore cold paths and small-n>`

const FINDINGS_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    category: { type: 'string' },
    findings: { type: 'array', items: {
      type: 'object', additionalProperties: false,
      properties: {
        title: { type: 'string' },
        impact: { type: 'string', enum: ['critical', 'high', 'medium', 'low'] },
        file: { type: 'string', description: 'real repo-relative path you read' },
        line: { type: 'string' },
        hotPath: { type: 'string', description: 'why this runs often / over large n — the reachability + frequency claim' },
        cost: { type: 'string', description: 'estimated cost: big-O, query count, alloc count, payload/bundle size' },
        evidence: { type: 'string', description: 'profiler/benchmark/query-log/bundle data if available, else "static reasoning"' },
        fix: { type: 'string', description: 'precise: what to change and where' },
      },
      required: ['title', 'impact', 'file', 'hotPath', 'cost', 'fix'],
    } },
  },
  required: ['category', 'findings'],
}

const VERDICT_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    isReal: { type: 'boolean', description: 'true ONLY if on a real hot path AND the win is measurable (not micro-opt)' },
    confidence: { type: 'string', enum: ['high', 'medium', 'low'] },
    adjustedImpact: { type: 'string', enum: ['critical','high','medium','low','micro-opt','false-positive'] },
    measurability: { type: 'string', description: 'how to confirm the win: benchmark, profiler, query log, bundle analyzer' },
    reasoning: { type: 'string', description: 'evidence from the code + performance model actually read' },
  },
  required: ['isReal', 'confidence', 'adjustedImpact', 'reasoning'],
}

// FILL: keep categories that map to real surface; point each focus at real files.
const CATEGORIES = [
  { key: 'N+1',         name: 'N+1 / unbatched queries', focus: `loops issuing per-item queries, missing eager-load/join — <files>` },
  { key: 'SLOW-QUERY',  name: 'Slow queries',            focus: `missing index, full scan, SELECT *, unbounded/unpaginated result sets — <files>` },
  { key: 'ALGO',        name: 'Algorithmic complexity',  focus: `accidental quadratic, nested loops over large n on hot paths — <files>` },
  { key: 'ALLOC',       name: 'Hot-path work & allocs',  focus: `repeated work, no memoization, rebuilding immutable data per call — <files>` },
  { key: 'IO-BLOCK',    name: 'Blocking I/O',            focus: `sync I/O or serial awaits on async paths, event-loop stalls — <files>` },
  { key: 'CONCURRENCY', name: 'Concurrency',             focus: `lock contention, missing parallelism, connection-pool exhaustion — <files>` },
  { key: 'NETWORK',     name: 'Network',                 focus: `chatty/waterfall calls, no batching, payload bloat, no compression — <files>` },
  { key: 'CACHING',     name: 'Caching',                 focus: `missing/ineffective caching, stampede, wrong TTLs — <files>` },
  { key: 'FRONTEND',    name: 'Frontend / bundle',       focus: `unneeded re-renders, unmemoized props, no code-splitting, large assets — <files>` },
  { key: 'LEAK',        name: 'Resource leaks',          focus: `unbounded caches, leaked connections/handles/listeners — <files>` },
]

const finderPrompt = (cat) => `${STACK}

ROLE: Senior performance engineer. Audit the ${cat.name} category.
SCOPE: ${cat.focus}
METHOD:
- Use Read/Grep/Glob to inspect the ACTUAL code. Every finding cites a real file + line you read.
- Each finding must sit on a HOT PATH — state why it runs often or over large n. Estimate cost
  (big-O, query count, alloc count, payload/bundle size). Prefer existing profiler/benchmark/query-log
  evidence over reasoning; set evidence accordingly.
- Real, reachable cost only — NO micro-optimizations, no premature tuning, no style nits. A cold or
  small-n path is not a finding. Clean area => empty findings array.
- Calibrate impact to the PERFORMANCE MODEL above. Dedupe within your category. Precise fixes.`

const verifyPrompt = (cat, f) => `${STACK}

A prior auditor flagged this ${cat.name} finding. ADVERSARIAL job: try to REFUTE it.
Read the cited code (${f.file}${f.line ? ':' + f.line : ''}) + context BEFORE deciding.
Is this a real, measurable win on a genuine hot path under the performance model — or is it a
micro-optimization, a cold/small-n path, premature, or a misread?
- Ground the verdict in code you read (and any real evidence), not the claim.
- Default isReal=false unless you can show it is BOTH hot (runs often / over large n) AND the win
  is measurable. Mark adjustedImpact='micro-opt' for negligible gains, 'false-positive' for misreads.
- Calibrate adjustedImpact to the performance model; correct mis-scoped impact.
- State measurability: how to confirm the win (benchmark, profiler, query log, bundle analyzer).

FINDING
Title: ${f.title}
Claimed impact: ${f.impact}
Location: ${f.file}${f.line ? ':' + f.line : ''}
Hot-path claim: ${f.hotPath}
Estimated cost: ${f.cost}
Evidence: ${f.evidence || '(static reasoning)'}
Proposed fix: ${f.fix}`

phase('Find')
log(`Perf audit: ${CATEGORIES.length} category passes, model=${MODEL || 'inherit'} (finders -> skeptics)`)

const results = await pipeline(
  CATEGORIES,
  (cat) => agent(finderPrompt(cat), { label: `find:${cat.key}`, phase: 'Find', schema: FINDINGS_SCHEMA, ...MOPT }),
  (res, cat) => {
    const findings = res && Array.isArray(res.findings) ? res.findings : []
    if (!findings.length) return { key: cat.key, name: cat.name, findingsCount: 0, verified: [] }
    return parallel(findings.map((f) => () =>
      agent(verifyPrompt(cat, f), { label: `verify:${cat.key}`, phase: 'Verify', schema: VERDICT_SCHEMA, ...MOPT })
        .then((v) => ({ ...f, category: cat.key, categoryName: cat.name, verdict: v }))
    )).then((arr) => ({ key: cat.key, name: cat.name, findingsCount: findings.length, verified: arr.filter(Boolean) }))
  }
)

const clean = results.filter(Boolean)
const all = clean.flatMap((r) => r.verified)
const real = all.filter((f) => f.verdict && f.verdict.isReal &&
  f.verdict.adjustedImpact !== 'false-positive' && f.verdict.adjustedImpact !== 'micro-opt')
log(`Done: ${all.length} raw findings, ${real.length} survived verification (model=${MODEL || 'inherit'})`)
return {
  model: MODEL || 'inherit',
  categories: clean.map((r) => ({ key: r.key, name: r.name, raw: r.findingsCount,
    confirmed: r.verified.filter((f) => f.verdict && f.verdict.isReal).length })),
  real, all,
}
```

### 4. Synthesize when the workflow returns
- **Dedupe across categories first.** The same root cause can surface in several passes (e.g. an
  unbounded query flagged under both SLOW-QUERY and N+1). Merge by `file:line` + root cause, keeping
  the highest adjustedImpact, before ranking.
- Group confirmed survivors by **adjustedImpact** (critical → low), each with `file:line`, the
  hot-path reason, the estimated cost, the fix, and how to measure it (`measurability`).
- Report coverage so nothing reads as silently dropped: per-category raw vs confirmed counts, which
  areas came back clean, and the scope audited (whole repo / path / diff).
- Note the **micro-opts and cold-path findings** the skeptics killed (shows verify worked, and steers
  the user away from premature tuning).
- **Measure, don't assume.** Perf is empirical: recommend confirming the top items with a benchmark /
  profiler / query log before *and after* any fix. Offer to write `PERF-AUDIT.md` (impact-grouped
  findings with `file:line`, cost, fix, measurement plan) so the result persists beyond the chat.
- Offer to fix the top findings as a follow-up (default is a read-only audit).

## Notes
- Model is arg-driven (`MODEL` in the script); no arg → you choose per situation (see
  Model selection). `null` inherits the session model — deliberate choice only.
- Optionally stamp `effort` per role via `agent()` opts (e.g. verifiers `'high'`) when the
  chosen model supports it.
- Scale to the ask: quick check → fewer categories, single verify. "Thorough/comprehensive" → add a
  3-vote adversarial panel per finding and a completeness-critic final pass.
- **Prefer evidence over reasoning.** If a profiler/benchmark/slow-query log/bundle report exists,
  feed it in (step 1) — a measured finding beats a reasoned one, and the skeptic weighs it higher.
- Read-only by default. Perf fixes can change behavior or hurt readability for small gains — that is
  exactly what the skeptic guards against; keep the micro-opt verdict in play.
- Never write literal `Date.now()` / `Math.random()` / `new Date()` in the script text (even inside
  prompt strings) — the Workflow validator rejects it.
