# Optimize GitHub API Call Plan — `ccr-improvement-loop`

Goal: reduce GitHub API consumption (and secondary-rate-limit risk) in the PR
fetch path so uncapped windows stay under the shared GitHub App installation
budget (~5,000 req/hr).

## Cost model (today)

Per PR, `fetchPrToCache` (`scripts/fetch-prs.ts`) issues:

- **5 REST** (fixed): `pulls/{n}`, `.../reviews`, `.../comments`,
  `issues/{n}/comments`, `.../commits`
- **1 GraphQL** (fixed): thread resolution + linked issues
- **2 REST per commit** (the multiplier): `commits/{sha}` (files/patches) and
  `commits/{sha}/pulls` (commit→PR mapping)

Total ≈ **6 + 2N** *endpoint invocations* for a PR with *N* commits. The
per-commit loop dominates on large PRs.

> ⚠️ These are **lower bounds, not request counts.** Every async call uses
> `gh api --paginate`, so reviews/comments/commits (and commits touching >300
> files) can each span multiple HTTP requests, and `ghApiJsonAsync` retries add
> more. Do **not** treat `6 + 2N` as a budget guarantee — validate with a
> `GET /rate_limit` reading before/after a representative run.

---

## 1. Delete the `commits/{sha}/pulls` call + `commitPrs` field — IMPLEMENTED ✅

`commitPrs` is **written but never read** by any consumer (verified: appears
only in `fetch-prs.ts`, the optional `types.ts` field, and test fixtures).

**Deletion is safe because the field is unread — full stop.** (Do not justify
it via squash-merge: squash controls base-branch *integration*, not whether a
contributor merged/rebased `main` *into* a PR branch, and the workflow can be
dispatched against any repo per the README. Any merge-sync timeline pollution
already exists today and is unaffected by removing an unused field.)

The one *hypothetical* use would be filtering base-branch/merge commits out of
the attribution timeline (`attribute-comments.ts`, `build-judge-input.ts`). If a
merge-commit repo is ever targeted and attribution accuracy matters, the fix is
to **wire `commitPrs` in** (keep a commit only if its introducing PR == this
PR), not to have left it unused. **Decision: delete now; document the topology
gap.**

Effect: per-PR cost `6 + 2N` → **`6 + N`** (halves the dominant term).

Changes:
- `scripts/fetch-prs.ts`: remove the `commitPrs` accumulator, the
  `commits/{sha}/pulls` fetch (the `try/catch` block), and `commitPrs` from the
  written payload. Drop the "commit→PR mapping" line from the header comment.
- Add a one-line note that commit→PR filtering is the correct fix if a
  merge-commit repo is ever targeted.
- `scripts/types.ts`: remove the optional `commitPrs` field.
- Tests/fixtures: remove `commitPrs` from `tests/build-judge-input.test.ts`,
  `tests/attribute-comments.test.ts`, and `tests/fixtures/pr-sample.json`.

Risk: none — field is unread. Optionally add a regression test for a PR branch
containing a merge-from-base commit to document the unsupported topology. Bump
`RAW_SCHEMA_VERSION` if cache compatibility matters; otherwise stale caches
simply omit an unused field.

---

## 2. Parallelize the fan-out — IMPLEMENTED ✅

Naive `Promise.all` is **not safe here**: `runWithConcurrency` caps PR *workers*
(default 6), not requests. Fanning out all fixed calls + every commit means a
65-commit PR × 6 workers ≈ ~390 concurrent `gh` processes → **worse**
secondary-rate-limit risk, the opposite of the goal. Also `fetchThreadResolution`
was synchronous (`ghJsonSync`) and couldn't join `Promise.all`.

**Done (depends on #5's limiter):**
- `fetchThreadResolution` converted to async via `ghApiGraphqlAsync` so the
  GraphQL call is bounded by the same limiter.
- In `fetchPrToCache`, the 5 independent per-PR reads + thread resolution now run
  under one `Promise.all`; the per-commit `commits/{sha}` detail fetches run under
  a second `Promise.all` (`commitsRaw.map`), which preserves commit order.
- All requests flow through the shared `ghRequestLimiter`, so total in-flight is
  capped regardless of `workers × per-worker fan-out`.

Result: lower wall-clock per PR at a *controlled* request rate; identical payload
(same calls, order-preserving assembly). Verified by `Semaphore` ceiling test +
typecheck + full suite (97 tests green).

---

## 3. Selective commit-detail fetching — DEFERRED (blocked at fetch layer) ⛔

Intent: fetch `commits/{sha}` files/patches only for paths tied to surviving
candidate comments, removing much of the `N` term.

**Investigation result — not safely doable at the fetch layer today.** Two
blockers found:

1. **`distinctFiles` couples to *every* commit's file list.**
   `prep-summary.ts:78-82` builds `distinctFiles` by iterating `commit.files`
   across **all** PRs (not just commented ones). Skipping commit-detail for
   no-comment PRs would silently shrink that emitted stat — a behavior
   regression, not a free optimization.
2. **Per-commit *timing* attribution needs per-commit files.**
   `attribute-comments.ts:118-127` and `build-judge-input.ts:43-55` pick the
   latest commit that touched a path *before* the comment. `GET /pulls/{n}/files`
   (one aggregate call) gives the union of changed files but **loses per-commit
   `committedAt`**, so it cannot replace the per-commit fetch for attribution.

The one clean sub-win — skip the whole `commits/{sha}` loop when a PR has **zero
inline comments** (the only consumer of commit files/patches) — is still blocked
by (1): those PRs' files would vanish from `distinctFiles`.

**Correct fix (out of scope here):** move commit-detail fetching into a later
stage that runs *after* comment filtering, and source `distinctFiles` from a
PR-level `pulls/{n}/files` count (1 call/PR) decoupled from the attribution
timeline. That is a cross-stage refactor touching `fetch-prs` → `filter` →
`prep-summary`; deferred to keep this change low-risk. #1 + #2 + #5 already
address the acute rate-limit pressure.

## 4. Collapse REST fan-out into GraphQL (larger change) — DEFERRED 🕐

A GraphQL query per PR *can* return reviews, inline comments (+ thread
`isResolved`), issue comments, and commits together. **Caveats confirmed during
#2/#5 work:**
- These are paginated connections — the existing thread query already truncates
  threads@100 and linked issues/labels@20 — so a "single query" claim is false
  for large PRs; it needs cursor loops + field-equivalence tests vs. the
  fully-paginated REST path.
- Per-commit **file patches** (`build-judge-input`) are not cleanly available in
  GraphQL, so the dominant `N` term stays on REST regardless.

Given #1 (`6+2N`→`6+N`) + #2 (bounded parallelism) + #5 (limiter/backoff) already
relieve the acute pressure, and the migration is high-effort for a partial win,
this is deferred unless real runs still hit the ceiling. The new
`ghApiGraphqlAsync` helper (added in #2) is the building block if revisited.

---

## 5. Centralized limiter + honor rate-limit headers — IMPLEMENTED ✅

`ghApiJsonAsync` previously only backed off *after* a 403/secondary error, with
fixed exponential backoff that ignored GitHub's `Retry-After` header, and nothing
bounded cross-call concurrency.

**Done (in `scripts/utils.ts`):**
- `Semaphore` class — a counting semaphore bounding concurrent async ops.
- `ghRequestLimiter` — a process-wide `Semaphore` (default 8, override via
  `CCR_GH_MAX_CONCURRENCY`), kept well under GitHub's ~100-concurrent guidance.
- `ghApiJsonAsync` **and** the new async `ghApiGraphqlAsync` both admit every
  request through `ghRequestLimiter` (shared REST+GraphQL ceiling) via a common
  `spawnGhJson` runner.
- `parseRetryAfterMs` + `withGhRetry` — retry now prefers the server-advertised
  `Retry-After` / "retry after N seconds" delay, falling back to exponential.

Note: `GET /rate_limit` preflight was intentionally **not** added — it covers
only the *primary* budget and cannot prevent secondary (burst) limits, which the
limiter now handles directly. It remains a possible coarse primary-budget
admission control if ever needed.

Verified: `Semaphore` peak-concurrency + release-on-rejection tests and
`parseRetryAfterMs` tests in `tests/utils.test.ts` (7 tests); typecheck + lint +
full suite (97 tests) green.

---

## Sequencing — STATUS

1. **#1 dead-code deletion** — ✅ DONE (`6+2N` → `6+N`).
2. **#5 global request limiter + header-aware backoff** — ✅ DONE.
3. **#2 bounded parallel fan-out** (on top of #5) — ✅ DONE.
4. **#3 selective commit-detail fetching** — ⛔ DEFERRED: blocked by
   `distinctFiles` coupling + per-commit timing needs; requires a cross-stage
   refactor.
5. **#4 GraphQL migration** — 🕐 DEFERRED: partial win, high effort; revisit only
   if real runs still hit the ceiling.

Net shipped: fewer requests per PR (#1) **and** those requests now issued
concurrently under a hard, header-aware secondary-rate-limit ceiling (#2 + #5).
Remaining follow-ups (#3, #4) are documented with concrete blockers.

---

## Test Plan

Tooling: `vitest` (`npm test` = `vitest run`; `npm run test:coverage`). Tests
are pure-function unit tests over helpers exported from `scripts/*.ts` with
fixtures under `tests/fixtures/` — no live `gh` calls (see
`tests/fetch-prs.test.ts`). Every change below must keep `npm test`,
`npm run typecheck`, and `npm run lint` green.

### Prerequisite for testability
The fetch layer currently calls `gh` via module-level `ghApiJsonAsync` /
`ghJsonSync`, which unit tests cannot observe. Items #2/#3/#5 require making the
request function **injectable** (pass a `fetchFn`/`gh` client into
`fetchPrToCache`, defaulting to the real one) so tests can substitute a mock
that records call order, arguments, and concurrency. Do this refactor first; it
is behavior-preserving.

### Item #1 — delete `commitPrs` + `commits/{sha}/pulls`
- **Regression (no dead field):** given a mocked PR with commits, assert the
  written payload has **no `commitPrs` key** and that the mock recorded **zero**
  `commits/{sha}/pulls` requests.
- **Call-count guard:** for a PR with *N* commits, assert exactly *N*
  `commits/{sha}` requests and **no** `.../pulls` requests (locks in `6+N`).
- **Fixture cleanup:** remove `commitPrs` from `tests/fixtures/pr-sample.json`,
  `tests/build-judge-input.test.ts`, `tests/attribute-comments.test.ts`; those
  suites must still pass unchanged (proves consumers never read it).
- **Type guard:** `npm run typecheck` fails if any code still references
  `commitPrs` after the `types.ts` field is removed — that *is* the test.
- **Schema:** if `RAW_SCHEMA_VERSION` is bumped, extend `tests/run-schema.test.ts`
  to pin the new value.

### Item #3 — selective commit-detail fetching
- **Fetch-only-what-matters:** mock a PR with 3 commits touching files A/B/C but
  candidate comments only on A; assert `commits/{sha}` details are fetched only
  for commits touching A, and skipped commits incur **zero** detail requests.
- **No-candidate PR:** a PR with no surviving comments fetches **zero**
  `commits/{sha}` details.
- **Accuracy invariant (golden):** run `attribute-comments` + `build-judge-input`
  over a fixture **before and after** the reorder; assert byte-identical output
  (patches + `ccrSawCode` gate unchanged). This is the safety net proving the
  optimization is behavior-preserving.

### Item #5 — global request limiter + header-aware backoff
- **Concurrency ceiling:** drive many requests through the limiter with a mock
  that records max simultaneous in-flight; assert it never exceeds `K`.
- **`Retry-After` honored:** mock a `403` secondary-limit response carrying
  `Retry-After: 2`; assert the limiter waits ~2s (fake timers) rather than the
  fixed exponential value.
- **Primary-budget admission:** mock `GET /rate_limit` with low `remaining`;
  assert the run pauses/defers until reset instead of hammering into a `403`.
- **Backoff regex regression:** keep the existing `utils.ts` retriable-error
  matching covered (rate limit / secondary / abuse / 5xx / timeout).

### Item #2 — parallelize the fan-out
- **Cap respected:** with the injected limiter and a mock recording peak
  concurrency, fan out the fixed + per-commit calls and assert peak in-flight
  ≤ `K` (guards against the `workers × per-worker fan-out` explosion).
- **Equivalence:** assert the cached payload is identical whether calls run
  sequentially or parallelized (ordering of merged arrays must be
  deterministic — sort if needed).
- **No limiter → no parallelism:** a guard/test ensuring parallel fan-out is not
  enabled unless the limiter is present (fail-safe wiring).

### Cross-cutting / manual validation
- **Full suite + coverage:** `npm test` and `npm run test:coverage` after each
  item; no coverage regression on `fetch-prs.ts`.
- **Live smoke (manual, capped):** one real `--max-prs 5` run against a target
  repo with `GET /rate_limit` sampled **before and after**; record actual
  request consumption to validate the `6+N` (and post-#3) model against reality —
  the plan's cost numbers are lower bounds until measured this way.
