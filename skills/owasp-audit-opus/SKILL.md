---
name: owasp-audit
description: Run a multi-agent OWASP Top 10 security audit on the current codebase. Pass `opus` (default) or `sonnet` to choose the model for every finder/verifier agent. Launches a background Workflow with one finder per OWASP category (latest edition + retained legacy categories), adversarially verifies each finding with a skeptic agent, then synthesizes survivors by severity with file:line refs. Use when the user asks for a security audit, OWASP review, vulnerability sweep, pen-test-style code review, or to "audit using agents" (optionally naming opus or sonnet).
argument-hint: [opus|sonnet]
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
  { key: 'A01:2025', name: 'Broken Access Control', focus: `...` },
  // A02 Security Misconfiguration, A03 Software Supply Chain Failures,
  // A04 Cryptographic Failures, A05 Injection, A06 Insecure Design,
  // A07 Authentication Failures, A08 Software or Data Integrity Failures,
  // A09 Security Logging and Alerting Failures, A10 Mishandling of Exceptional Conditions,
  // + dedicated SSRF pass, + dedicated dependency-CVE pass (when relevant).
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
- Group confirmed survivors by **adjustedSeverity** (critical → info), each with
  `file:line`, the exploit path, and the fix.
- Report coverage so nothing reads as silently dropped: per-category raw vs confirmed
  counts, and which areas came back clean.
- Note high-confidence **false positives** the skeptics killed (shows verify worked).
- Offer to fix the top findings as a follow-up (default is a read-only audit).

## Notes
- Model is arg-driven (`MODEL` in the script). Default opus; `sonnet` for cheaper sweeps.
- Scale to the ask: quick check → fewer finders, single verify. "Thorough/comprehensive"
  → add a 3-vote adversarial panel per finding and a completeness-critic final pass.
- Never write literal `Date.now()` / `Math.random()` / `new Date()` in the script text
  (even inside prompt strings) — the Workflow validator rejects it.