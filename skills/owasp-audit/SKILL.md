---
name: owasp-audit
description: Run a multi-agent OWASP Top 10 security audit on the current codebase. Pass `opus` (default) or `sonnet` to choose the model for every finder/verifier agent; optionally scope to a `<path>` or to branch changes only with `--diff`. Launches a background Workflow with one finder per OWASP category (latest edition + retained legacy categories), adversarially verifies each finding with a skeptic agent, then synthesizes survivors by severity with file:line refs and offers to write a SECURITY-AUDIT.md report. Use when the user asks for a security audit, OWASP review, vulnerability sweep, pen-test-style code review, or to "audit using agents" (optionally naming opus or sonnet).
argument-hint: [opus|sonnet] [--diff | <path>]
---

# OWASP multi-agent audit

Fan out one **finder** per OWASP category, then have a **skeptic** try to refute each
finding before it counts. Pipeline (no barrier) — a finding verifies the moment its
category finishes. You synthesize survivors by severity.

This skill calls the **Workflow** tool — invoking the skill IS the multi-agent opt-in.
Needs a git repo. Audit agents are read-only (no worktree isolation).

## Model selection (the invocation argument)
The argument picks the model for **every** agent:
- `sonnet` → all finders + skeptics run Sonnet (cheaper/faster, broad sweeps).
- `opus`, empty, or anything else → Opus (default, deepest reasoning).

Pass the choice into the Workflow as `args: { model: 'opus' | 'sonnet' }`; the script
reads it into `MODEL` and stamps it on every `agent()`. State which model you picked
when you launch.

## Scope (optional argument)
Default audits the whole repo. Narrow it when the repo is large or you only care about recent work:
- `--diff` → audit only files changed on the current branch vs the main branch (PR-gating). Resolve
  the changed-file list during recon and pass it into every finder's SCOPE.
- `<path>` → audit only that directory/file subtree.

State the scope when you launch; recon (step 1) and every finder's `focus` must respect it.

## Procedure

### 1. Recon the codebase (inline, before the workflow)
Map the target so finder prompts are grounded, not generic. Gather:
- Stack: language/framework, server entrypoint, route layout, frontend.
- Auth: where requests are authenticated, what's trusted, any bypasses.
- Data: DB + query style (parameterized?), storage/buckets, migrations.
- Outbound: every server-side `fetch`/HTTP call (SSRF surface).
- Deps: the manifest (`package.json`/`requirements.txt`/`go.mod`...) + lockfile presence.
- **Threat model**: who the real users are, what gates them, blast radius. Write this
  down — every agent uses it to calibrate severity (don't over-rate a cross-user leak
  in a single-user app; never under-rate secret/key exposure, auth bypass, RCE, SQLi,
  stored XSS, or SSRF-to-internal).

### 2. Refresh the OWASP list from the web (don't trust memory)
WebSearch the **current** OWASP Top 10 edition and confirm the exact category list +
which legacy categories were merged/renamed. As of the 2025 edition: SSRF folded into
A01 Broken Access Control; Vulnerable & Outdated Components expanded into A03 Software
Supply Chain Failures; new A03 (Supply Chain) and A10 (Mishandling of Exceptional
Conditions). Re-verify each run — editions change.

### 3. Build the category list
Latest-edition categories **plus** any retained legacy category that maps to real
surface in this codebase, kept as its own dedicated pass (so total can exceed 10).
e.g. a dedicated SSRF pass when the app makes user-influenced outbound calls; a
dedicated dependency-CVE pass separate from the broad supply-chain pass.

### 4. Author & launch the Workflow
Adapt the template: fill `STACK` from step 1, `CATEGORIES` from steps 2–3 (rewrite each
`focus` to point at the real files you found). Launch in **background**, passing the
chosen model: `Workflow({ script: <filled>, args: { model: 'opus' } })`.

```js
export const meta = {
  name: 'owasp-audit',
  description: 'OWASP Top 10 (latest) multi-agent security audit via finder+verifier agents',
  phases: [
    { title: 'Find', detail: 'one auditor per OWASP category' },
    { title: 'Verify', detail: 'skeptic adversarially refutes each finding' },
  ],
}

// Model from the invocation arg: 'sonnet' -> sonnet, else opus (default).
const MODEL = args && typeof args.model === 'string' && args.model.toLowerCase() === 'sonnet' ? 'sonnet' : 'opus'

// FILL from recon: stack facts + explicit threat model. Every agent reads this.
const STACK = `STACK & THREAT MODEL
<framework, entrypoint, routes, auth + trusted inputs + bypasses, DB & query style,
storage, outbound fetch sites, deps>
THREAT MODEL: <who the real users are, what gates them, how to calibrate severity>`

const FINDINGS_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    category: { type: 'string' },
    findings: { type: 'array', items: {
      type: 'object', additionalProperties: false,
      properties: {
        title: { type: 'string' },
        severity: { type: 'string', enum: ['critical', 'high', 'medium', 'low', 'info'] },
        file: { type: 'string', description: 'real repo-relative path you read' },
        line: { type: 'string' },
        description: { type: 'string' },
        exploit: { type: 'string', description: 'concrete reachable exploit path' },
        fix: { type: 'string', description: 'precise: what to change and where' },
      },
      required: ['title', 'severity', 'file', 'description', 'exploit', 'fix'],
    } },
  },
  required: ['category', 'findings'],
}

const VERDICT_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    isReal: { type: 'boolean' },
    confidence: { type: 'string', enum: ['high', 'medium', 'low'] },
    adjustedSeverity: { type: 'string', enum: ['critical','high','medium','low','info','false-positive'] },
    reasoning: { type: 'string', description: 'evidence from the code actually read' },
  },
  required: ['isReal', 'confidence', 'adjustedSeverity', 'reasoning'],
}

// FILL: latest-edition categories + retained legacy passes. Point focus at real files.
const CATEGORIES = [
  // Latest-edition passes (re-verify names/order against the web check in step 2).
  // Rewrite each `focus` to point at the REAL files found in recon; drop any that don't apply.
  { key: 'A01:2025', name: 'Broken Access Control', focus: `authz on every route/handler, IDOR, missing ownership checks, SSRF now lives here — <files>` },
  { key: 'A02:2025', name: 'Security Misconfiguration', focus: `CORS, security headers, debug flags, default creds, verbose errors — <files>` },
  { key: 'A03:2025', name: 'Software Supply Chain Failures', focus: `manifest + lockfile, unpinned/abandoned deps, postinstall scripts — <files>` },
  { key: 'A04:2025', name: 'Cryptographic Failures', focus: `secret handling, password hashing, TLS, token signing, randomness — <files>` },
  { key: 'A05:2025', name: 'Injection', focus: `SQL/NoSQL, command, template, XSS sinks vs the query/render style — <files>` },
  { key: 'A06:2025', name: 'Insecure Design', focus: `missing rate limits, weak workflows, trust-boundary gaps — <files>` },
  { key: 'A07:2025', name: 'Authentication Failures', focus: `login, session, password reset, MFA, JWT validation — <files>` },
  { key: 'A08:2025', name: 'Software or Data Integrity Failures', focus: `insecure deserialization, unsigned updates, CI/CD trust — <files>` },
  { key: 'A09:2025', name: 'Security Logging & Alerting Failures', focus: `auth-event logging, secrets leaking into logs — <files>` },
  { key: 'A10:2025', name: 'Mishandling of Exceptional Conditions', focus: `error paths leaking detail, fail-open defaults, swallowed exceptions — <files>` },
  // Retained dedicated passes — keep any that map to real surface here (total may exceed 10):
  { key: 'SECRETS', name: 'Hardcoded Secrets & Credentials', focus: `keys/tokens/passwords in source, config, committed history — <files>` },
  { key: 'SSRF', name: 'Server-Side Request Forgery', focus: `user-influenced outbound fetch/HTTP calls — <files>` },
  { key: 'DEP-CVE', name: 'Known-Vulnerable Dependencies', focus: `lockfile versions vs known CVEs; is the vulnerable path actually reached? — <files>` },
]

const finderPrompt = (cat) => `${STACK}

ROLE: Senior application security auditor. Audit OWASP ${cat.key} - ${cat.name}.
SCOPE: ${cat.focus}
METHOD:
- Use Read/Grep/Glob to inspect the ACTUAL code. Every finding cites a real file + line you read.
- Concrete, grounded issues only — no best-practice filler. Clean area => empty findings array.
- Calibrate severity to the threat model above. Dedupe within your category. Precise fixes.`

const verifyPrompt = (cat, f) => `${STACK}

A prior auditor flagged this ${cat.key} - ${cat.name} finding. ADVERSARIAL job: try to REFUTE it.
Read the cited code (${f.file}${f.line ? ':' + f.line : ''}) + context BEFORE deciding.
Real reachable vuln in THIS codebase under the threat model, or false positive
(framework/platform already mitigates, unreachable, misread, out of threat model)?
- Ground the verdict in code you read, not the claim.
- Default isReal=false if you can't trace a concrete reachable exploit path.
- Calibrate adjustedSeverity to the threat model; correct mis-scoped severity.

FINDING
Title: ${f.title}
Claimed severity: ${f.severity}
Location: ${f.file}${f.line ? ':' + f.line : ''}
Description: ${f.description}
Claimed exploit: ${f.exploit || '(none)'}
Proposed fix: ${f.fix}`

phase('Find')
log(`OWASP audit: ${CATEGORIES.length} category passes, model=${MODEL} (finders -> skeptics)`)

const results = await pipeline(
  CATEGORIES,
  (cat) => agent(finderPrompt(cat), { label: `find:${cat.key}`, phase: 'Find', model: MODEL, schema: FINDINGS_SCHEMA }),
  (res, cat) => {
    const findings = res && Array.isArray(res.findings) ? res.findings : []
    if (!findings.length) return { key: cat.key, name: cat.name, findingsCount: 0, verified: [] }
    return parallel(findings.map((f) => () =>
      agent(verifyPrompt(cat, f), { label: `verify:${cat.key}`, phase: 'Verify', model: MODEL, schema: VERDICT_SCHEMA })
        .then((v) => ({ ...f, category: cat.key, categoryName: cat.name, verdict: v }))
    )).then((arr) => ({ key: cat.key, name: cat.name, findingsCount: findings.length, verified: arr.filter(Boolean) }))
  }
)

const clean = results.filter(Boolean)
const all = clean.flatMap((r) => r.verified)
const real = all.filter((f) => f.verdict && f.verdict.isReal && f.verdict.adjustedSeverity !== 'false-positive')
log(`Done: ${all.length} raw findings, ${real.length} survived verification (model=${MODEL})`)
return {
  model: MODEL,
  categories: clean.map((r) => ({ key: r.key, name: r.name, raw: r.findingsCount,
    confirmed: r.verified.filter((f) => f.verdict && f.verdict.isReal).length })),
  real, all,
}
```

### 5. Synthesize when the workflow returns
- **Dedupe across categories first.** The same root cause can surface in several passes (e.g.
  one injection flagged under both A05 and A06). Merge by `file:line` + root cause, keeping the
  highest adjustedSeverity, before ranking.
- Group confirmed survivors by **adjustedSeverity** (critical → info), each with
  `file:line`, the exploit path, and the fix.
- Report coverage so nothing reads as silently dropped: per-category raw vs confirmed
  counts, which areas came back clean, and the scope audited (whole repo / path / diff).
- Note high-confidence **false positives** the skeptics killed (shows verify worked).
- **Offer to write `SECURITY-AUDIT.md`** (severity-grouped findings with `file:line`, exploit,
  and fix) so the result persists beyond the chat — default audit stays read-only.
- Offer to fix the top findings as a follow-up.

## Notes
- Model is arg-driven (`MODEL` in the script). Default opus; `sonnet` for cheaper sweeps.
- Scale to the ask: quick check → fewer finders, single verify. "Thorough/comprehensive"
  → add a 3-vote adversarial panel per finding and a completeness-critic final pass.
- Never write literal `Date.now()` / `Math.random()` / `new Date()` in the script text
  (even inside prompt strings) — the Workflow validator rejects it.