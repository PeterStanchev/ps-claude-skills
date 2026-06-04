---
name: test-author-skeptic
description: Multi-agent test authoring with adversarial mutation verification. Phase 0 (inline) detects the language + test stack, sets one up if missing (recommends/asks), reads the verified-test ledger, and proposes targets. Then a background Workflow runs two roles per target — an author writes/augments tests and runs them green, then an INDEPENDENT skeptic mutation-tests them (a real tool like mutmut/cosmic-ray/Stryker if present, else LLM hand-mutation) to prove each test actually catches bugs; a strengthen loop closes surviving mutants until clean, no-progress, or a round cap. Fixes weak tests it wrote, flags pre-existing ones (offers to fix on confirm). Caches verified (test,source) pairs by content hash to skip unchanged work. Pass `sonnet` (default) or `opus`; `--auto` runs hands-off; `--recheck-all` ignores the ledger. Use when asked to write tests with agents, harden or strengthen a test suite, set up a test framework, find coverage gaps via mutation testing, or "test using agents".
argument-hint: [sonnet|opus] [--auto] [--recheck-all]
---

# Test author + skeptic (multi-agent, mutation-verified)

Per target, two roles. An **author** writes (or augments) tests and runs them green. An
**independent skeptic** then **mutation-tests** them — break the source, rerun: a test that
stays green against broken code is worthless, so surviving mutants are real gaps. A
**strengthen** loop (author role again) closes those gaps; the skeptic re-judges. The judge
is never the writer — the separation that makes this better than one agent grading itself.

Calls the **Workflow** tool — invoking the skill IS the multi-agent opt-in. Needs a git repo.

## Why these mechanics (decisions baked in)
- **Main working tree, not worktrees.** A fresh git worktree lacks `node_modules`/`.venv`
  (gitignored, not in HEAD), so tests can't run there without a slow per-worktree install.
  The main tree is where deps live *if they live anywhere local* — but never assume they do
  (see run-context detection below; deps may be partial, or only in a container). Safety comes
  from a **copy-and-restore checkpoint**: any agent that mutates a source file copies it first
  (`<src>.bak`) and restores it after — parallel-safe (never `git stash`, which is global),
  and survives WIP in the file.
- **Hybrid skeptic engine.** If a real mutation tool is configured the skeptic runs it scoped to
  the source file — `mutmut`/`cosmic-ray` (pytest), `Stryker` (JS/TS), `PIT` (Java/JVM),
  `cargo-mutants` (Rust), `gremlins` (Go), `Stryker.NET` (C#), `mutant` (Ruby). Otherwise it
  hand-mutates: flip `</<=`, negate a condition, `return null/0`, off-by-one, drop a side effect.
- **Serialize per source file, parallel across files.** Two agents mutating the same file
  collide; targets are grouped by source file, sequential within a group, groups run parallel.
- **Verified-test ledger.** `.test-author-skeptic/verified.json` records, per verified test, a
  sha256 of the test file AND its source file(s) + score + date. A test is skipped next run
  only if BOTH hashes still match — any source change auto-rechecks. `--recheck-all` ignores it.
  Read/written **inline** (never inside parallel agents) to avoid concurrent-write races.

## Invocation arguments
- `sonnet` (default) / `opus` — model for every agent. Sonnet is the default for cost; pass `opus`
  only for gnarly logic where the skeptic's reasoning needs the extra depth.
- `--auto` — hands-off: auto-discover targets, auto-pick + set up the recommended stack for
  the language, audit existing tests but **flag-only**, **never auto-edit pre-existing tests**
  (hard trust boundary, even in `--auto`), respect the ledger.
- `--recheck-all` — ignore the ledger; re-verify everything.

## Procedure

### Phase 0 — Detect, set up, scope (INLINE, before the Workflow)
Workflow agents run in the background and can't prompt the user, so all interaction happens here.
1. **Explore + detect**: identify language(s) and whether a test stack exists (config, test dirs,
   a runner in the manifest). Detect any configured mutation tool for the language (`mutmut`/`cosmic-ray` py,
   `Stryker` JS/TS, `PIT` JVM, `cargo-mutants` Rust, `gremlins` Go, `Stryker.NET` C#, `mutant` Ruby).
   **Hard-exclude `node_modules/`, `.venv/`, `dist/`, `build/`, generated, and vendored dirs**
   from all discovery globs — a dependency's own tests (e.g. `node_modules/zod/**/*.test.ts`)
   are not your targets and will flood results.
2. **Establish run context + a GREEN BASELINE** (the make-or-break step — do not skip):
   - **Detect where tests actually run.** Don't assume the main tree has deps. Check, in order:
     a local env (`.venv`, `poetry`, `node_modules`), a global interpreter (deps may be *partial*
     — e.g. `pytest` present but `psycopg`/`pytest-asyncio` absent), or **container-only**
     (`docker-compose.yml` / `Dockerfile` → tests run via `docker compose exec <svc> ...`).
   - **Resolve import-time / infra coupling.** A `conftest.py` (or test bootstrap) that imports the
     DB/app layer can fail collection before any test runs — e.g. an engine built at import needs a
     driver you don't have. Fix with env overrides (a sqlite `DATABASE_URL`, a dummy `SECRET_KEY`),
     required env vars, or by running inside the container. Record the exact env + command.
   - **Actually run it green.** Run the existing suite (or the smallest relevant subset) and confirm
     green. Pin the exact, reproducible command (with env) — this is the baseline every agent uses.
   - **No green baseline ⇒ stop.** Mutation testing is meaningless against a suite that can't run.
     If you can't reach green (missing infra, broken suite), report what's blocking and ask the user
     rather than launching agents that will flail. In `--auto`, attempt the obvious fixes (sqlite
     URL, container exec); if still red, abort with a clear reason.
3. **Bootstrap if missing**: no test stack → recommend one for the language; unless `--auto`,
   confirm via AskUserQuestion. Then set it up (add deps, config, a smoke test) and prove it runs.
   In `--auto`, pick the recommended default and set it up without asking.
4. **Existing tests**: if a suite exists, ask whether to also audit it or leave it be (in `--auto`,
   audit but flag-only). Audited-existing targets are marked so the skeptic judges them too.
5. **Targets**: propose a prioritized list (untested/under-tested modules, core logic first;
   exclude trivial/generated/config), each mapped to a distinct source file + test file + mode
   (`create` | `augment` | `audit-existing`). Prefer pure, low-infra modules first — they mutation-
   test cleanly; DB/IO-heavy modules need fixtures and are slower to verify. Unless `--auto`, let the
   user edit/confirm. Verify target source files have no uncommitted changes (the copy-restore
   checkpoint assumes a clean source); warn and skip any that are dirty.
6. **Per-target run command.** Give each target the narrowest command that exercises its tests
   (e.g. `pytest path::test -k ...`), not the whole suite — so a pre-existing failing or unrunnable
   test elsewhere (often env-dependent) never pollutes a mutation verdict.
7. **Resume + ledger filter**: first fold any `.test-author-skeptic/partial/*.json` left by an
   interrupted run into `verified.json` (entries with `remaining == 0` become verified). Then read
   the ledger and drop targets whose (test,source) hashes are unchanged, unless `--recheck-all`.
   Report what was skipped and what was resumed (never silently).

### Phase 1 — Author & launch the Workflow (BACKGROUND)
Fill `args` from Phase 0 and launch in the background:
`Workflow({ script: <below>, args: { model, maxRounds, stackDesc, runContext, commands, targets } })`
where `stackDesc`/`runContext` are strings (from Phase 0 step 2), `commands = { mutationTool }`, and each
`targets[i] = { key, source, test, mode, cmd }` — `cmd` is the pinned per-target green command (Phase 0 step 6).
Keep the printed `runId` + `scriptPath` — they enable same-session resume (see *Resuming an interrupted run*).

> **Reliability — embed data, don't trust the `args` channel.** The `args` object can reach the script
> empty (symptom: a 0-target / 0-agent run that returns in milliseconds). Prefer to EMBED `targets`,
> `stackDesc`, `runContext`, and `commands` as literals directly in the script body (replace the
> `args && …` reads at the top with the actual values) and launch via `scriptPath`. This is also what
> makes the Write/Edit-then-rerun iteration loop work. The fallback `<fill>` placeholders below are only
> for the rare case the channel works.

```js
export const meta = {
  name: 'test-author-skeptic',
  description: 'Author tests, an independent skeptic mutation-tests them, strengthen loop closes gaps until clean/capped',
  phases: [
    { title: 'Author',     detail: 'write/augment tests per target, run green' },
    { title: 'Sabotage',   detail: 'independent skeptic mutation-tests (real tool or hand-mutation)' },
    { title: 'Strengthen', detail: 'close surviving mutants; skeptic re-judges; loop' },
  ],
}

const MODEL = args && typeof args.model === 'string' && args.model.toLowerCase() === 'opus' ? 'opus' : 'sonnet'
const MAX_ROUNDS = args && Number.isInteger(args.maxRounds) ? args.maxRounds : 3
const TARGETS = (args && Array.isArray(args.targets)) ? args.targets : []
const CMD = (args && args.commands) || {}

const STACK = `STACK & TEST MODEL
Language/framework: ${(args && args.stackDesc) || '<fill>'}
RUN CONTEXT: ${(args && args.runContext) || '<fill: local venv / global interpreter / docker compose exec — plus any required env vars>'}
Mutation tool (if configured): ${CMD.mutationTool || '(none — hand-mutate)'}
Your task gives a PER-TARGET test command. Run EXACTLY that (it is the pinned green baseline from
Phase 0); do not broaden it to the whole suite — unrelated env-dependent failures would pollute your verdict.

OUTPUT CONTRACT (non-negotiable): you MUST finish by calling the StructuredOutput tool with your result.
If you are running low on budget, STOP working immediately and emit what you have so far. Ending a turn
without StructuredOutput loses the entire target — and for a skeptic/reverify mid-mutation it can leave a
source file un-restored. Budget your run so the final emit always happens.

SAFETY (main working tree, shared with other agents):
- Only ever touch the ONE source/test file named in your task.
- Before mutating a source file: copy it to <src>.bak. After: restore from <src>.bak, delete the bak,
  and confirm the baseline is green again. NEVER use git stash (it is global and races other agents).`

const MUTANT = {
  type: 'object', additionalProperties: false,
  properties: {
    location: { type: 'string', description: 'source file:line mutated' },
    mutation: { type: 'string', description: 'what changed, e.g. < -> <=' },
    gap: { type: 'string', description: 'behavior the suite fails to assert' },
    suspectedEquivalent: { type: 'boolean', description: 'true if no test could ever kill it (semantically identical mutation)' },
  },
  required: ['location', 'mutation', 'gap'],
}

const AUTHOR_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    target: { type: 'string' },
    testFile: { type: 'string', description: 'repo-relative path written' },
    mode: { type: 'string', enum: ['created', 'augmented'] },
    behaviorsCovered: { type: 'array', items: { type: 'string' } },
    behaviorsSkipped: { type: 'array', items: { type: 'string' } },
    passing: { type: 'boolean', description: 'true ONLY if you ran it and it went green' },
  },
  required: ['target', 'testFile', 'mode', 'behaviorsCovered', 'passing'],
}

const WEAK = {
  type: 'object', additionalProperties: false,
  properties: {
    test: { type: 'string', description: 'test name + file' },
    flaw: { type: 'string', enum: ['tautology', 'mock-only', 'asserts-nothing', 'flaky', 'over-fit'] },
    fix: { type: 'string' },
    preExisting: { type: 'boolean', description: 'present in the committed file at HEAD (check git show HEAD:<file>)' },
  },
  required: ['test', 'flaw', 'fix', 'preExisting'],
}

const SKEPTIC_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    engine: { type: 'string', enum: ['tool', 'hand'] },
    mutationsTried: { type: 'integer' },
    survived: { type: 'array', items: MUTANT },
    weakTests: { type: 'array', items: WEAK },
    mutationScore: { type: 'number', description: 'killed / tried, 0..1' },
  },
  required: ['engine', 'mutationsTried', 'survived', 'weakTests', 'mutationScore'],
}

const STRENGTHEN_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    behaviorsAdded: { type: 'array', items: { type: 'string' } },
    weakTestsFixed: { type: 'array', items: { type: 'string' } },
    passing: { type: 'boolean', description: 'suite green after restoring all source' },
  },
  required: ['behaviorsAdded', 'weakTestsFixed', 'passing'],
}

const REVERIFY_SCHEMA = {
  type: 'object', additionalProperties: false,
  properties: {
    stillSurvived: { type: 'array', items: MUTANT, description: 'of the given mutants, those the new tests STILL fail to kill' },
  },
  required: ['stillSurvived'],
}

const authorPrompt = (t) => `${STACK}

ROLE: Test author for ${t.key} (source: ${t.source}, tests: ${t.test}, mode: ${t.mode}).
METHOD:
- Read the ACTUAL source. Enumerate every branch, boundary, error path, edge case.
- mode=create: write a new test file at ${t.test}. mode=augment: ADD cases to the existing file —
  NEVER delete, rewrite, or weaken existing TEST BODIES (editing the import line to pull in more
  symbols is fine; existing test functions stay byte-for-byte).
- Assert exception MESSAGES, not just the exception type — a bare \`raises(Err)\` with no message
  match lets message-text and exception-ordering mutations survive (observed in the smoke run).
- Assert observable behavior + outputs, not implementation detail. Mock only true external
  boundaries; never write a test that only asserts the mock.
- RUN your per-target command: ${t.cmd}. Report passing=true ONLY if green. List behaviors covered + skipped.`

const skepticPrompt = (t, a) => `${STACK}

A prior author wrote/augmented tests at ${a.testFile} for ${t.key} (source: ${t.source}).
ADVERSARIAL job: prove these tests would FAIL to catch real bugs. Stay independent — judge the
code as written, not the author's claims.
METHOD (mutation testing):
- If a mutation tool is configured, run it scoped to ${t.source} and parse surviving mutants (engine=tool).
- Else hand-mutate (engine=hand): pick AT MOST ~8-12 of the HIGHEST-VALUE mutations (boundaries/thresholds,
  flip </<=, negate a condition, return null/0, off-by-one, swapped class/string, drop a side effect) — do
  NOT exhaustively mutate every line. When each test run is slow (browser/jsdom ~2-3s startup, compiled
  Rust/Go/JVM, or DB/integration suites), stay at the LOW end of that range: N slow mutant×rerun cycles plus
  verbose reasoning is exactly what blows the agent budget and aborts the run before StructuredOutput. For
  each: apply ONE mutation, rerun ${t.cmd}, REVERT (copy-restore per SAFETY). Test fails => killed; suite
  still green => SURVIVED.
- Flag weak tests (tautology / mock-only / asserts-nothing / flaky / over-fit); set preExisting by
  checking git show HEAD:${a.testFile}.
- Mark a survivor suspectedEquivalent=true only if no test could ever kill it (semantically identical).
- Ground every survivor in source file:line + the unasserted behavior. Report mutationScore = killed/tried.
- LAST step (resume checkpoint): write the file .test-author-skeptic/partial/${t.key}.json containing
  {"key":"${t.key}","source":"${t.source}","test":"${a.testFile}","remaining":<survivor count>,"done":<true if 0 survivors else false>}.

Author covered: ${a.behaviorsCovered.join('; ')}`

const strengthenPrompt = (t, testFile, survivors, ownWeak) => `${STACK}

ROLE: Close gaps in ${testFile} for ${t.key} (source: ${t.source}). You authored these tests — fair to edit them.
1. For each surviving mutant below, add a test that FAILS when that mutation is present.
2. Fix each weak test listed (these are ones you wrote — never touch pre-existing tests).
3. Restore any source you mutated while checking (copy-restore), run ${t.cmd}, set passing accordingly.
4. Do not weaken existing assertions.
SURVIVING MUTANTS:
${survivors.map((m) => `- ${m.location}: ${m.mutation} -> ${m.gap}`).join('\n')}
WEAK TESTS YOU WROTE TO FIX:
${(ownWeak && ownWeak.length) ? ownWeak.map((w) => `- ${w.test} [${w.flaw}]: ${w.fix}`).join('\n') : '(none)'}`

const reverifyPrompt = (t, testFile, survivors) => `${STACK}

INDEPENDENT re-judge for ${t.key}. The author added tests to ${testFile} claiming to kill the mutants below.
For EACH, re-apply that exact mutation to ${t.source} (copy-restore per SAFETY), rerun ${t.cmd}, and
record in stillSurvived any mutant whose tests still pass. Mark suspectedEquivalent=true if it cannot
be killed by any test. Restore source and confirm green before returning.
- LAST step (resume checkpoint): overwrite .test-author-skeptic/partial/${t.key}.json with
  {"key":"${t.key}","source":"${t.source}","test":"${testFile}","remaining":<stillSurvived count>,"done":true}.
MUTANTS TO RE-CHECK:
${survivors.map((m) => `- ${m.location}: ${m.mutation} -> ${m.gap}`).join('\n')}`

// Group by source file: serialize within a group (same file), parallel across groups.
const groupsMap = {}
for (const t of TARGETS) { (groupsMap[t.source] = groupsMap[t.source] || []).push(t) }
const GROUPS = Object.keys(groupsMap).map((src) => ({ source: src, targets: groupsMap[src] }))

phase('Author')
log(`Test build: ${TARGETS.length} targets in ${GROUPS.length} file-groups, model=${MODEL}, cap=${MAX_ROUNDS} rounds`)

async function processTarget(t) {
  const a = await agent(authorPrompt(t), { label: `author:${t.key}`, phase: 'Author', model: MODEL, schema: AUTHOR_SCHEMA })
  if (!a || !a.passing) return { key: t.key, source: t.source, authored: a, skeptic: null, rounds: [], remaining: [], weakTests: [], note: 'author failed/red' }

  const s = await agent(skepticPrompt(t, a), { label: `skeptic:${t.key}`, phase: 'Sabotage', model: MODEL, schema: SKEPTIC_SCHEMA })
  const weakTests = (s && Array.isArray(s.weakTests)) ? s.weakTests : []
  const ownWeak = weakTests.filter((w) => !w.preExisting)
  let survivors = (s && Array.isArray(s.survived)) ? s.survived : []

  const rounds = []
  let round = 0
  let lastRemaining = survivors.length
  while (survivors.length && round < MAX_ROUNDS) {
    round++
    const str = await agent(strengthenPrompt(t, a.testFile, survivors, ownWeak), { label: `strengthen:${t.key} r${round}`, phase: 'Strengthen', model: MODEL, schema: STRENGTHEN_SCHEMA })
    if (!str || !str.passing) { rounds.push({ round, remaining: survivors.length, note: 'strengthen left tree red — stopped' }); break }
    const rv = await agent(reverifyPrompt(t, a.testFile, survivors), { label: `reverify:${t.key} r${round}`, phase: 'Strengthen', model: MODEL, schema: REVERIFY_SCHEMA })
    const still = (rv && Array.isArray(rv.stillSurvived)) ? rv.stillSurvived : survivors
    rounds.push({ round, killed: survivors.length - still.length, remaining: still.length })
    survivors = still
    if (still.length >= lastRemaining) break   // no progress (likely equivalent/infeasible) -> stop
    lastRemaining = still.length
  }
  return { key: t.key, source: t.source, authored: a, skeptic: s, rounds, remaining: survivors, weakTests }
}

const grouped = await parallel(GROUPS.map((g) => async () => {
  const out = []
  for (const t of g.targets) { out.push(await processTarget(t)) }   // sequential within a source file
  return out
}))

const results = grouped.filter(Boolean).flat()
const scored = results.filter((r) => r.skeptic)
const avg = scored.length ? scored.reduce((sum, r) => sum + (r.skeptic.mutationScore || 0), 0) / scored.length : 0
const remainingMutants = results.reduce((n, r) => n + (r.remaining ? r.remaining.length : 0), 0)
const preExistingWeak = results.flatMap((r) => (r.weakTests || []).filter((w) => w.preExisting))
log(`Done: avg mutation score ${avg.toFixed(2)}, remaining mutants ${remainingMutants}, pre-existing weak tests ${preExistingWeak.length}`)
return { model: MODEL, avgScore: avg, remainingMutants, results, preExistingWeak }
```

### Phase 2 — Synthesize, update ledger, offer (INLINE, on return)
- **Report per target**: tests created/augmented, behaviors covered, mutation score, gaps closed by
  the strengthen loop, and any **remaining mutants** (file:line + unasserted behavior) — mark
  suspected-equivalent ones as "can't be killed", not "we failed". Note targets left red (`passing=false`).
- **Update the ledger**: for each target that ended with zero remaining (or only suspected-equivalent)
  mutants, write its (test,source) sha256 + score + date into `.test-author-skeptic/verified.json`.
  Inline only. **Always write it, even when `.test-author-skeptic/` is gitignored** — it still powers
  same-machine skip-on-rerun; do NOT skip this step just because it won't be committed. Then say whether
  it's committed or gitignored, and offer to flip that. (Also record manually-verified targets — e.g. a
  skeptic that aborted but whose tests you teeth-checked by hand — with `method: "manual-teeth-check"`.)
  Then delete `.test-author-skeptic/partial/` — its entries are now folded into the ledger.
- **Weak tests**: ours were fixed in the loop. List **pre-existing** weak tests with suggested fixes,
  then — unless the user already declined existing-test edits — ask via AskUserQuestion whether to fix
  them too, and only edit on confirmation. In `--auto`, flag only; never edit pre-existing tests.

## Resuming an interrupted run
Long runs die mid-way (usage cutoff, kill, crash). Two recovery layers:

- **Same session — Workflow native resume.** The launch prints a `runId` and a `scriptPath`. Re-invoke
  `Workflow({ scriptPath, resumeFromRunId: <runId> })`: every completed `agent()` call with an unchanged
  (prompt, opts) returns its cached result instantly, so only the in-flight target onward re-runs. Cheapest
  path — try it first. (Stop the dead run with TaskStop if it's still listed.)

- **New session — the ledger is the durable checkpoint.** Cross-session the Workflow journal may be gone,
  so progress must already be on disk. Each target writes its own checkpoint the instant it finishes:
  `.test-author-skeptic/partial/<key>.json` — a **unique file per target**, so there is no concurrent
  writer even across parallel groups (no append, no race). Phase 0 step 7 folds these into the ledger
  before filtering, so re-invoking the skill skips everything already done and re-runs only the rest. A
  target caught half-done is simply redone — `augment` mode + "don't duplicate existing tests" keep that
  safe and bounded to the one in-flight target. `partial/` is transient — gitignore it; only `verified.json`
  is durable/shared.

## Notes
- Scale to the ask: quick → smaller target list, hand-mutation; "thorough" → configure a real mutation
  tool, raise `maxRounds`, widen targets.
- **Cost profile (measured).** One ~50-line pure module, sonnet, hand-mutation, 1 round ≈ 4 agents /
  ~140k tokens / ~9.5 min — the skeptic runs N mutant×rerun cycles, so cost scales with (branches ×
  targets × rounds). At scale, prefer a real mutation tool (one batched run beats N LLM reruns), keep
  the target list tight, and default sonnet over opus.
- The copy-restore checkpoint + serialize-per-file are what make main-tree parallelism safe — keep both.
- **Slow test runners blow the skeptic budget.** Browser/jsdom (~2-3s/start), compiled (Rust/Go/JVM), and
  DB/integration suites make every mutant×rerun cycle expensive. Cap the skeptic's mutation count (see its
  prompt) so the run finishes and emits StructuredOutput; an aborted skeptic loses the target *and* can
  leave a source mutated. The author→green phase is robust; it's the skeptic loop that needs the cap.
- **Recovery after a dead skeptic.** If a run aborts mid-mutation you may find sources left mutated with
  orphan `<src>.bak` files. Before relaunching: restore them — `git checkout -- <sources>` (clean sources)
  or `mv <src>.bak <src>` — delete the bak, confirm the baseline is green. The authored tests usually
  survive (already written + run green); re-verify them (rerun the skeptic, or teeth-check by hand) rather
  than rewriting. Then ledger them with the method used.
- Never write literal `Date.now()` / `Math.random()` / `new Date()` in the script text (even inside
  prompt strings) — the Workflow validator rejects it. Stamp the ledger date inline after the run.
```
