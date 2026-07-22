# Polish & Scale Plan — CCR Improvement Loop

The final consolidation plan. Everything up to now proved the idea works: the
metrics are defensible ([`README.md`](./README.md), [`decisions.md`](./decisions.md)),
the fetch path is cheaper and burst-safe ([`optimize-api-call-plan.md`](./optimize-api-call-plan.md)
#1/#2/#5 shipped), and the rate-limit landscape is mapped
([`api-rate-limit.md`](./api-rate-limit.md)). This plan takes it from _"works on
one window when babysat"_ to **_"drains a `repos × months` backlog unattended,
under budget, and stays clean enough to hand off."_**

It has one **load-bearing feature** — a **backfill pacer** that spaces work under
GitHub's hourly refill — wrapped in the **polish** (code, tests, docs, dashboard)
that makes the whole package scalable and elegant rather than merely functional.

**Budget assumption.** This plan assumes the workflow repo lives (or will live) in
a **GHEC-owned org**, where the plain `GITHUB_TOKEN` gives **15,000 req/hr/repo**
with zero code or auth change (`api-rate-limit.md §6`). No custom PAT/App token is
introduced — moving the project to the correct repo _is_ the auth story. Confirm
the real number once with `gh api rate_limit` from inside the workflow. At
15,000/hr an uncapped Python month (~4,000+ invocations) fits comfortably, so
per-hour pacing — not per-window capping — is the scaling lever.

Guiding constraints (carried from prior sessions, do not violate):

- **Low maintenance over reliability theater.** Lean on the harness — `cron`
  cadence, the immutable cache, the in-process limiter — not bespoke enforcement
  code. (`design tradeoffs`, `decisions.md` cross-cutting principle.)
- **Proposal-only, deterministic-prep / agent-judgment / deterministic-math**
  stays intact (`decisions.md` D2, D9). The pacer is _orchestration around_ the
  loop; it never touches judgment or metrics.
- **Every change keeps `npm test` (vitest), `npm run typecheck`, `npm run lint`,
  `npm run format:check` green.** Scripts run via `node scripts/foo.ts` (Node ≥
  24, native TS strip). No build step.

---

## The shape of the work

Three tracks, sequenced so the pacer lands on a foundation that can actually
support it. The pacer's one real prerequisite is **cross-run cache persistence**
(so a resumed window re-processes nothing), which comes first as **Track A**; the
pacer itself is **Track B**; the surrounding **polish** is **Track C** and can
proceed in parallel.

| Track | Theme                                                    | Blocking?              |
| ----- | -------------------------------------------------------- | ---------------------- |
| **A** | Scale foundation — cross-run cache persistence           | Prereq for B           |
| **B** | The backfill pacer — spread work under the hourly refill | The centerpiece        |
| **C** | Polish — code, tests, docs, dashboard                    | Parallel, non-blocking |

---

## Track A — Scale foundation (cache persistence)

The pacer's one hard prerequisite, from
[`optimize-api-call-plan.md` §8](./optimize-api-call-plan.md) blocker #2. (Its
other historical blocker — the token ceiling — is dissolved by the GHEC move; see
the budget assumption above.)

### A1 — Persist the cache across runs (make resume real)

**Problem.** `CCR_CACHE` is `${{ github.workspace }}/.ccr-cache`
(`ccr-improvement-loop.md:62`), written **fresh each run** — no `actions/cache`
restore, no artifact download. The immutable `pr-*.json` cache makes each fetch
_idempotent within a run_, but the "resume with zero rework" benefit the pacer
leans on is only **potential** until the cache survives across runs
(`optimize-api-call-plan.md §8`, blocker #2). Until then every run re-fetches its
window from scratch — the pacer still bounds calls/hour, but a retried or resumed
window pays full price again.

**Do (pick the lower-maintenance option — `actions/cache`).**

- Add an `actions/cache` step keyed by the immutable window identity:
  `ccr-cache-${repo}-${window_start}-${window_end}`, restoring `.ccr-cache/pr-*.json`
  before prep and saving after. The key is content-stable (windows are immutable),
  so a resumed run hits the cache and re-fetches nothing.
- **Scope the cache key to the raw PR cache only**, not the judged/emitted
  artifacts — those are cheap to recompute and coupling them invites staleness
  bugs (mirrors the D12 "don't couple test data to live data" instinct).
- Add a `RAW_SCHEMA_VERSION` component to the cache key so a fetch-schema change
  invalidates stale caches automatically instead of silently resuming on an
  incompatible shape.

**Acceptance.** Run the same `(repo, window)` twice; the second run's prep logs
show **zero** `gh api` PR fetches (all cache hits) and finishes in a fraction of
the wall-clock time. A `RAW_SCHEMA_VERSION` bump forces a clean re-fetch.

**Why `actions/cache` over artifact up/download.** Cache is one declarative step
pair with automatic key-based restore; artifacts need explicit
upload/download/run-id plumbing. Low maintenance wins (`design tradeoffs`).

---

## Track B — The backfill pacer (the centerpiece)

A cron-driven pacer that drains an arbitrarily large `repos × windows` backlog by
running one chunk of work per hourly tick — so no single hour exceeds the
15,000/hr ceiling and the total backlog size is **decoupled** from the per-hour
limit. This is the concrete build of `api-rate-limit.md §4` Levers 3 (spread) + 4
(orchestrate) and `optimize-api-call-plan.md §8`.

**The core insight — the cron cadence _is_ the pacer.** No always-on process, no
in-job `sleep` burning runner minutes, no in-run waiting. An hourly `cron` fires,
checks the budget, runs what fits, and exits. GitHub's own scheduler is the clock.
This is the elegant, low-maintenance shape the user asked for.

**Token-free by construction.** The pacer does **not** dispatch a separate
workflow (which would hit GitHub's "the default `GITHUB_TOKEN` can't trigger
another workflow" recursion guard and force a PAT). Instead, the loop is exposed
as a **reusable workflow** (`on: workflow_call`) and the pacer **`uses:`** it as a
job in its own cron-triggered run. A reusable-workflow call is not an event
trigger, so it needs no PAT and no elevated token — the plain `GITHUB_TOKEN`
(15,000/hr on GHEC) does everything. One tick = one run that processes its
admitted window(s) inline via the reusable loop.

### B1 — Backlog model

- Represent the backlog as a committed `pacer/backlog.json`: a flat list of
  `{ repo, window_start, window_end, max_prs, status }` jobs (`status ∈
pending | running | done | skipped`).
- Generate it from a `repos × months` matrix with a tiny helper
  `scripts/gen-backlog.ts` (`--repos a,b --from 2026-01 --to 2026-07 --granularity
month|week`). **Monthly** granularity by default (busy repos → ~180–300 PRs,
  large comparable denominators; `README.md` "Backfilling", D13); **weekly** as an
  opt-in only when you want more trend points at smaller per-point `n`. At
  15,000/hr an uncapped month fits a single tick, so capping is rarely needed.
- Committing the backlog (vs. an artifact/variable) makes progress **auditable in
  git** and trivially resumable — the lowest-maintenance state store.

### B2 — The pacer loop (`ccr-pacer.yml`)

`on: schedule: cron (hourly)` + `workflow_dispatch`. Each tick:

1. **Read budget** — `gh api rate_limit`, take remaining **primary** REST (free,
   read-only). (`rate_limit` sees only the primary pool — the secondary/burst
   limits are still handled in-process by the #5 `Semaphore`, which stays
   essential regardless of tier; the pacer cannot see or pace them.)
2. **Estimate & admit** — using the `~(6 + N) × PRs` cost model
   (`api-rate-limit.md §8`), pick the next 1–N `pending` jobs whose estimated cost
   fits `remaining − safetyMargin`. At 15,000/hr that is ~3 uncapped month-repos
   per tick; admit conservatively and let the next tick take the rest.
3. **Run the window inline** — invoke the reusable loop as a job per admitted
   window (no cross-workflow dispatch, no token):
   ```yaml
   jobs:
     backfill:
       strategy:
         matrix: { window: ${{ fromJSON(needs.plan.outputs.admitted) }} }
       uses: ./.github/workflows/ccr-improvement-loop.yml
       with:
         repo: ${{ matrix.window.repo }}
         window_start: ${{ matrix.window.window_start }}
         window_end: ${{ matrix.window.window_end }}
         max_prs: ${{ matrix.window.max_prs }}
   ```
4. **Mark done** — flip those jobs' `status` in `backlog.json` and commit via
   safe-output, keeping the pacer proposal-clean like the child.
5. **Exit.** Next hour, `cron` drains the next chunk.

**The per-repo `concurrency` group already guards overlap.** `ccr-improvement-loop`
uses `group: ccr-loop-${repo}`, `cancel-in-progress: false` — 1 running + 1
pending per repo. Even if two ticks overlap on the same repo, the group serializes
them, so the pacer needs no locking of its own (`optimize-api-call-plan.md §8`).

### B3 — Self-healing & termination

- **Resume for free** — because A1 persists the cache, a retried or resumed window
  re-processes nothing. A pacer restart just re-reads `backlog.json` and continues
  from `pending`.
- **No-op tolerance** — a window under 50 settled PRs emits no artifact by design
  (`config.json:minPrs`, `README.md`); the pacer marks it `done` (not `failed`) so
  thin windows don't wedge the queue.
- **Termination** — when no `pending` jobs remain, the tick is a no-op. Optionally
  stop the schedule once the backlog is fully `done` (manual, low-maintenance — a
  full backfill is a finite campaign).
- **Failure isolation** — an inline window job failing (e.g. transient 403) leaves
  its job `running`; a reconciliation step at tick start flips genuinely-failed
  windows back to `pending` for one retry, then `skipped` with a logged reason.
  Cap retries at 1 to avoid a poison window looping forever.

### B4 — What the pacer must **not** do (scope guards)

- **No in-job `sleep` to pace** — spacing comes from `cron` cadence, not burned
  runner minutes (`optimize-api-call-plan.md §8`, blocker #3).
- **No parallel searches** — the Search API is 30 req/min; the per-window PR
  listing is one search, never parallelize (`api-rate-limit.md §6`).
- **No judgment / metric logic** — the pacer only admits and runs windows; all
  measurement stays in the reusable loop (D2 separation preserved).

**Acceptance.** A 7-month × 1-repo backlog (`gen-backlog.ts`) drains over a few
hourly ticks on the 15,000/hr GHEC token (roughly ~3 uncapped months/tick) with
**no primary-limit 403** and **no PAT**, each window's run-JSON landing in
`dashboard/data/`, and `backlog.json` ending all-`done`. Re-running the pacer
after completion is a clean no-op.

---

## Track C — Polish (parallel, non-blocking)

Makes the package elegant, cheap, and handoff-ready. None of these block the
pacer (the GHEC budget already covers backfill), but C1 drives per-PR REST cost to
near-zero and the rest turn "it runs" into "it's a product."

### C1 — API-call optimization program (fold in all deferred items)

An agent implements this, so **implementation effort is not a reason to defer
anything** — the earlier "deferred / high-effort / partial-win" labels in
[`optimize-api-call-plan.md`](./optimize-api-call-plan.md) (#3, #4, #6, #7) are
dropped. The **only** gate is information-preservation: every optimization must
produce output **byte-identical** to today's, proven by a golden equivalence test
_written and passing before_ the optimization is wired in. If a change cannot be
shown equivalent, it does not ship — full stop.

The pacer bounds cost _per hour_; this program cuts cost _per PR_ toward **near
zero on the rate-limited path**, so an uncapped month fits a single tick with
headroom. Implement in this order (each step's equivalence test is its
precondition):

**Step 0 — Make the request layer injectable (enabling refactor).** Pass a
`fetchFn`/`gh` client into `fetchPrToCache` (defaulting to the real one) so tests
can substitute a mock recording call count, args, and concurrency
(`optimize-api-call-plan.md`, "Prerequisite for testability"). Behavior-preserving;
unlocks every equivalence test below.

**Step 1 — #6 local-git commit detail (removes the dominant `N` term).** The
per-commit `GET /commits/{sha}` loop is the multiplier. Every field it writes —
`sha`, `committedAt`, `files`, `patches` — is a native git concept
(`optimize-api-call-plan.md §6`), so source them from a local clone at **zero
GitHub API cost** (git-transport budget is a separate, generous pool):

- `committedAt` ← `git log --format=%cI <sha>` (committer date, matching the
  current `c.commit.committer?.date` preference).
- `files` ← `git diff-tree --no-commit-id --name-only -r <sha>`.
- `patches` ← `git show <sha>` / `git diff`.
- Use a **blobless partial clone** (`git clone --filter=blob:none`) of the
  **target** repo (not the workflow repo `actions/checkout` already has) with
  `fetch-depth: 0` (or targeted `git fetch origin <sha>` for exactly the mined
  commits) so even `azure-sdk-for-python` stays cheap — never full-clone up front.
  Add this clone as a prep step in `ccr-improvement-loop.md`, keyed off
  `TARGET_REPO`, and pass its path into `fetch-prs.ts`.
- **Equivalence gate:** a golden test runs `attribute-comments` +
  `build-judge-input` over a fixture with REST-derived vs git-derived commit
  detail and asserts **byte-identical** output (patches, `committedAt` ordering,
  `distinctFiles`, and the `ccrSawCode` gate all unchanged). Ship only when green.

Effect: per-PR API cost `6 + N` → **`6`**. This **supersedes #3** (selective
commit-detail): #6 removes `N` rather than shrinking it, and dissolves #3's
`distinctFiles`-coupling blocker because git yields every commit's files for free,
so `distinctFiles` keeps its exact current contract with no API calls.

**Step 2 — #4 + #7 move the fixed `6` calls onto GraphQL (off the REST ceiling).**
The five fixed REST reads (`pulls/{n}`, reviews, PR comments, issue comments,
commits list) plus thread resolution collapse into GraphQL queries on the
**separate point-metered pool**, and **#7 aliasing** batches multiple PRs into one
HTTP request — cutting in-flight count (direct secondary-limit relief) as well as
primary-REST load:

- Reviews, inline comments (+ thread `isResolved`), issue comments, commit shas,
  and linked issues all come back in one aliased query per batch of PRs.
- **Pagination correctness is mandatory, not optional:** these are paginated
  connections (threads@100, linked issues/labels@20). The GraphQL path must
  **cursor-loop to exhaustion**, matching the fully-paginated REST result exactly
  — no silent truncation. This is precisely what the equivalence test pins.
- **Equivalence gate:** a golden test compares the assembled per-PR record from
  the GraphQL/batched path against the current REST path over a fixture that
  **includes a PR exceeding one page** of threads/comments; assert byte-identical
  normalized output. Ship only when green.
- Batch width is tuned and still admitted through `ghRequestLimiter` (#5), so an
  over-wide batch can't exhaust the GraphQL point pool or trip its CPU-cost guard.
- Commit **patch text** stays out of GraphQL (it isn't cleanly exposed) — but
  Step 1 already moved patches to git, so nothing regresses here.

Effect: the fixed `6` leaves the primary-REST budget for the GraphQL pool, in
fewer HTTP requests. Composed with Step 1, per-PR **primary-REST** consumption
drops toward **near zero** — the qualitative shift beyond the shipped #1/#2/#5.

**Net after C1:** per-PR primary-REST cost ≈ **0** (patches on git, fixed reads on
the GraphQL pool), every byte identical to today, each step locked by a
fixture-driven golden test (`tests/`, vitest, no live `gh`). The GHEC hourly
budget then effectively bounds only GraphQL points + git bandwidth, not the REST
ceiling that historically failed runs.

### C2 — Preflight budget guard in the child (defense in depth)

Add an **optional** `gh api rate_limit` preflight at the top of `prep-run.ts`
(behind a `--check-budget` flag, default on in CI): if remaining primary REST <
the estimated window cost, **abort early with a clear message and non-zero exit**
rather than dying mid-fetch with a partial cache. This complements — does not
replace — the pacer (which spaces at admission time) and the #5 limiter (which
guards secondary limits). Coarse primary-budget admission control, explicitly left
as a hook in `optimize-api-call-plan.md §5`.

### C3 — Dashboard scale & polish

The static, zero-backend, manifest-driven dashboard (D11) already pools slices
(handoff §C). Scale-oriented polish:

- **Auto-manifest for backfill volume.** A one-liner
  `scripts/gen-manifest.ts` that regenerates `dashboard/data/manifest.json` from
  the `run-*.json` files present, so a 50-window backfill doesn't require
  hand-editing the manifest (the browser can't glob — D11). Keep it a build-free
  local script, not a runtime dependency.
- **PR-size bucket dimension (S/M/L/XL)** — the one slice the user asked about
  that isn't yet possible (`handoff.md` candidate next steps). `additions` /
  `deletions` are already captured per PR; add a `sizeBucket` slice dimension
  through schema + `compute-metrics.ts` + a dashboard chart. Requires a history
  re-migration (like the `ccrRecallRate` migration in handoff §B) — do it as one
  deterministic recompute over existing raw fields, **no agent re-run**.
- **Repo comparison affordance** — the filename/manifest already carry
  `owner_repo`; ensure trend charts can filter/facet by repo so a multi-repo
  backfill reads cleanly (never compare repos on raw absolute burden — trend
  within a repo only, `README.md` reading rules).

### C4 — Code, test, and doc hygiene

- **Cost-model regression test.** Lock the post-C1 invariant with a mock `gh`
  recording call count/args: **zero** `commits/{sha}` REST calls (patches now come
  from git) and the fixed reads issued as batched GraphQL, not per-PR REST. A
  future change that reintroduces a per-commit or per-PR REST call fails CI. This
  is the guard that keeps the pacer's cost estimate honest.
- **Pacer dry-run mode** (`--dry-run`) — print the admission plan (jobs + estimated
  cost/tick) without running anything, for safe review before a big campaign.
- **Docs convergence.** Fold the shipped pieces of this plan into the existing
  docs rather than letting four rate-limit docs drift: `api-rate-limit.md §7`
  recommended path → "pacer shipped"; `optimize-api-call-plan.md §8` status → ✅;
  a short **"Backfilling at scale"** section in `README.md` pointing at the pacer.
  Add a `decisions.md` **D15 — backfill pacer** entry (decision / why / rejected
  alternatives [always-on runner, in-job sleep, cross-workflow dispatch requiring a
  PAT, per-run cap only] / consequences) and a **D16 — API cost reduced to
  near-zero REST (git patches + GraphQL fixed reads), gated on byte-equivalence**
  entry, both matching the existing D-series format.
- **`handoff.md` update** — new session entry summarizing pacer + Track A/C
  outcomes (and the reusable-workflow / GHEC-budget assumptions), git state, and
  validation table, matching the existing format.

---

## Sequencing & milestones

```
A1 (cross-run cache) ─▶ B1 (backlog) ─▶ B2/B3 (pacer loop) ─▶ B4 guards ─▶ ✅ unattended backfill
                                                           │
C1 (Step 0 inject ─▶ #6 git ─▶ #4/#7 GraphQL, each equivalence-gated)
C2 (preflight) ─ C3 (dashboard) ─ C4 (hygiene)             ── parallel, merge continuously
```

1. **Milestone 1 — Foundation.** A1 lands (cross-run cache), plus expose the loop
   as `workflow_call`; recompile lock. _Proof:_ one uncapped month completes
   without a 403 on the 15,000/hr token; a repeated window is all cache hits.
2. **Milestone 2 — Pacer MVP.** B1 + B2 drain a 3-window backlog under budget.
   _Proof:_ 3 run-JSONs appear; `backlog.json` all-`done`; no 403; no PAT used.
3. **Milestone 3 — Robust pacer.** B3 + B4 (resume, no-op tolerance, failure
   isolation, scope guards). _Proof:_ a poison/thin window doesn't wedge the queue;
   a mid-backfill restart resumes cleanly.
4. **Milestone 4 — Near-zero REST + scale polish.** C1 (Step 0 → #6 → #4/#7), C3,
   C4 merged; docs converged. _Proof:_ each C1 step's golden test asserts
   byte-identical output; the cost-model regression test asserts **zero** per-PR
   REST invocations; a 50-window backfill needs no manual manifest edits.

---

## Definition of done

- A `repos × months` backfill drains **unattended** under `cron`, never exceeding
  the hourly ceiling, with **zero rework** on resume.
- Per-PR **primary-REST cost is ~0** (commit patches sourced from local git, fixed
  reads batched onto the GraphQL pool), with **every emitted byte identical** to
  the pre-optimization output — proven by a golden equivalence test per C1 step.
- The child loop is unchanged in what it _measures_ — proposal-only,
  deterministic-prep / agent-judgment / deterministic-math intact (D2, D9).
- Dashboard ingests a large backfill without hand-editing, facets by repo, and
  reads honestly (`n/a` over false precision).
- All gates green: `npm test`, `npm run typecheck`, `npm run lint`,
  `npm run format:check`; lock recompiled and committed; docs converged; `D15`,
  `D16`, and a `handoff.md` entry recorded.

## Explicit non-goals

- **No information loss for API savings.** Every call-reduction ships only after a
  byte-equivalence golden test passes; a change that can't be proven equivalent is
  rejected regardless of how much budget it would save.
- **No always-on orchestrator / external scheduler.** The `cron` cadence is the
  pacer (`design tradeoffs` — low maintenance).
- **No custom PAT / GitHub App token.** The GHEC `GITHUB_TOKEN` (15,000/hr) plus a
  reusable-workflow (`workflow_call`) invocation cover both budget and triggering —
  no elevated credential is introduced.
- **No new reliability/enforcement layer.** Lean on `concurrency`, the immutable
  cache, and the `Semaphore`; add code only where the harness genuinely can't
  cover the gap.
- **No metric or judgment changes** driven by scale — the pacer is orchestration
  only. Metric evolution (e.g. reframing the headline) stays a separate, deliberate
  decision (`handoff.md`).
- **No importing/wrapping of prior TS/Python implementations** — this package
  stands on its own (`project scope`).
