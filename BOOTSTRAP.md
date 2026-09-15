# AI-Fabric bootstrap plan

How to get from this repository — a playbook with no automation in it — to a fabric that builds
itself. Read [README.md](README.md) first; this document adds no rules, it only sequences the work
the playbook already specifies.

This is deliberately **not** a play artifact and does not live under `/specs/`. Specs and plans
there are permanent output of [D1](README.md#d1-create-the-spec) and
[B1](README.md#b1-create-the-implementation-plan), and [T5](README.md#t5-ai-fabric-evals) replays
them as its eval set. A hand-written plan with no originating issue would poison that corpus. This
file is scaffolding, and it is deleted when the last phase closes.

## The ordering constraint

Every play depends on the `app/ai-fabric` GitHub App, the `fabric-setup` composite action, and the
shared agent-and-PR workflow. Those cannot travel the chain, because they *are* the chain — so
Phase 0 is hand-built by definition, and the goal is to keep it as small as possible and then stop.

Everything after that is ordered by one rule: **validate each play against an app before turning it
on this repo.** The plays assume a real build, a real Playwright suite and a deployable environment.
This repo has none of those, so a play proven only here has not been proven. That is why the sandbox
consumer repo comes before the first play rather than after the last one.

## Rules that hold in every phase

Restated from the README because every phase is a chance to break one:

- Run as the `app/ai-fabric` GitHub App, never `GITHUB_TOKEN` — a `GITHUB_TOKEN` run triggers no
  further workflows, so the chain stalls after the first hop.
- `ai-fabric:go`, applied by an actor with write permission, is the only thing that starts an agent
  run. Verify that permission in every workflow and exit silently otherwise.
- Trigger on the merge of a PR carrying a label, never a path filter, and guard on the issue's
  current labels and Project status before acting. See
  [Triggers and idempotency](README.md#triggers-and-idempotency).
- Guardrails are GitHub Actions. A check that only runs on someone's laptop does not exist.
- The fabric never approves or merges its own work, never widens its own attempt budget, and never
  disables a failing check to pass a gate.

## Phase 0 — Foundations

Hand-built. Nothing here can be dogfooded.

Deliverables

- The `app/ai-fabric` GitHub App with fine-grained repository permissions, installed on this repo
  and on the sandbox repo from Phase 1. Private key in org secrets.
- The full `ai-fabric:*` label set, and a GitHub Project whose statuses are exactly `Backlog`,
  `Plan`, `Design`, `Build`, `Test`, `UAT`, `Deploy` — and no `Blocked` column, per
  [The board](README.md#the-board).
- Branch protection on `main`: minimum approvals, CODEOWNERS approval, author cannot approve. Add
  CODEOWNERS coverage for `.github/` specifically, so a PR that edits the fabric's own workflows
  cannot be self-approved.
- `fabric-setup` composite action, implementing all four steps of
  [The shared composite action](README.md#the-shared-composite-action), including the `if: always()`
  progress-label cleanup.
- The shared `workflow_call` workflow: run the agent with a given prompt, then open a PR.
- `CLAUDE.md` extended with this repo's build, lint and test commands, plus the tooling to back them
  — `actionlint` and `yamllint` at minimum, and a runner for the composite action's scripts.
  [B2](README.md#b2-plan-review-and-approval) gates on `CLAUDE.md` being present and current, so a
  fabric with no test commands fails its own second play.
- `/specs/` skeleton with a generated `/specs/README.md` index.
- A `v1` tag and a promotion workflow that moves it. Callers resolve
  `uses: <org>/ai-fabric/.github/workflows/<play>.yml@v1`, so the tag is load-bearing from the first
  play onward.

Done when the sandbox repo can mint an App token through `fabric-setup`, its write-permission check
rejects a non-collaborator, and a deliberately crashed run leaves no orphaned progress label.

## Phase 1 — The sandbox consumer repo

Create `<org>/ai-fabric-sandbox`: a trivial app, but a genuine one — a real build, a real lint, real
unit tests, a real Playwright suite, and a deployable environment with a UAT approval gate.

Deliverables

- The app, with the `data-testid` convention documented in its own `CLAUDE.md`, because
  [B3](README.md#b3-code-generation) selects on it.
- Local `ci.yml` and `pr-build.yml`. These stay repo-local on purpose — build, package and deploy
  commands are genuinely repo-specific, per [Workflow layout](README.md#workflow-layout).
- The four thin caller workflows — `fabric-issues.yml`, `fabric-pr.yml`, `fabric-pipeline.yml`,
  `fabric-evals.yml` — each job a guard plus a `uses:` into this repo, one job per play so
  `permissions:`, `concurrency:` and check names stay job-level.

Why before any play: the consumer side cannot be exercised from inside this repo at all. Cross-repo
`@v1` resolution, installation on a second repo, and one-required-check-per-play only become real
once something calls in from outside. The sandbox is also the only place T1's deploy and test stages
have anything to run against.

Done when a caller job wrapping a hand-written no-op reusable workflow runs green, and its check
appears as its own required check on a sandbox PR.

## Phase 2 — Express lane, in the sandbox

Implement the shortest closed loop: [P1](README.md#p1-capture-the-intent-of-issues) track
classification, [B3](README.md#b3-code-generation) code generation, and
[B4](README.md#b4-code-review) code review.

Deliverables

- P1 classify workflow, carrying `ai-fabric:classify` while it runs and writing its recommendation
  into the issue, where a maintainer can override it before authorizing.
- B3 on the Express path: triggered by `ai-fabric:go` on an Express issue, with all four auto-fix
  gates bounded at 3 attempts and escalation per
  [Failure handling](README.md#failure-handling-and-escalation).
- B3's static guardrail: Playwright tests parse and compile, and every `data-testid` they reference
  exists in the diff.
- B4's review Action and the [standard PR gates](README.md#standard-pr-gates), including the
  coverage minimum, CodeQL, SAST and dependency scan.

Why Express first: one gate, no spec, no plan, and it still exercises the whole risky surface — the
trust boundary, App-token chaining from one workflow into the next, label-merge triggering, and the
attempt budget. Standard is the same machinery with more prompts.

Done when a sandbox issue goes from `ai-fabric:go` to a merged PR carrying code, unit tests and
Playwright tests — and a second run that exhausts its attempt budget escalates correctly: progress
label removed, `ai-fabric:blocked` added, maintainer assigned, Project status unchanged.

## Phase 3 — Standard lane, in the sandbox

Add [D1](README.md#d1-create-the-spec), [D2](README.md#d2-spec-review-and-approval),
[B1](README.md#b1-create-the-implementation-plan), [B2](README.md#b2-plan-review-and-approval), and
the Standard trigger into B3.

Deliverables

- D1, including front-matter handling, supersession pointers, `/specs/README.md` regeneration, and
  the UI-design Task sub-issue path with `ai-fabric:blocked-on-design` plus the workflow that clears
  the label when the Task closes.
- D1's guardrail that acceptance criteria are testable — a criterion that cannot be asserted is a
  defect at D1, because B3 generates one Playwright test per criterion.
- B1, including reproducing `spec.md` in the PR description inside a collapsed `<details>` block
  with a permalink pinned to the merged commit SHA, without touching `spec.md` to force it into the
  diff.
- The frozen-spec required check, per
  [Spec and plan lifecycle](README.md#spec-and-plan-lifecycle).
- All four rejection paths from
  [Failure handling](README.md#failure-handling-and-escalation), each of which does move Project
  status.

Done when a Standard issue runs issue to spec PR to plan PR to code PR with no workflow edit
mid-flight, and each rejection path returns the issue to the right column — including spec-PR
rejection removing `ai-fabric:go`, so re-entry requires fresh human authorization.

## Phase 4 — Test stage, in the sandbox

Deliverables

- [T1](README.md#t1-ci-pipeline): build, deploy, Playwright, triage, and the UAT environment
  approval, with the concurrency group that sequences test runs.
- The assertion rule enforced as a check rather than as a prompt instruction: a triage diff that
  weakens, relaxes, skips or deletes an assertion traceable to an acceptance criterion fails.
- [T3](README.md#t3-nightly-regression-pipeline) nightly regression, with the suite frozen at run
  time, one open Bug per failing test matched on test id, and repair PRs routed through B4 like any
  other code change.
- [T4](README.md#t4-e2e-coverage-audit) weekly coverage audit — issues only, never diffs, with
  findings landing in `Backlog`.

Done when a genuine sandbox defect and a deliberately broken locator are triaged differently: the
locator gets a repair PR, the defect gets a Bug sub-issue and no assertion change.

## Phase 5 — Crossover: the fabric builds the fabric

Only now turn the chain on this repo. Every remaining change to the fabric arrives as an issue and
travels the plays.

Deliverables

- This repo's own four caller workflows, pinned to `@v1` and never `@main`. A bad merge on `main`
  must not break the fabric that has to build the fix; `v1` advances only by deliberate promotion
  after a green run.
- A resolution of Open decision 1 below, since D1's and B3's Playwright guardrails cannot be
  satisfied by a repository whose deliverable is workflow YAML.
- A standing rule, written into this repo's `CLAUDE.md`: fabric changes are always Standard. P1's
  Express criteria exclude anything touching an authentication or authorization surface, and nearly
  every fabric change touches the `ai-fabric:go` trust boundary.

Done when a fabric change ships through the fabric's own Standard chain, reviewed and merged by a
human.

## Phase 6 — Evals

[T5](README.md#t5-ai-fabric-evals), in this repo and in the sandbox, once Phase 5 has produced
enough merged spec, plan and diff triples to score against.

Deliverables

- The eval workflow, the append-only eval set seeded from Phase 5's merged artifacts, baseline
  storage that advances only on a passing run, and per-run score history so a slow drift is as
  visible as a week-over-week drop.
- A case for every escape to a human from Phases 2 through 5 — those are the regressions the fabric
  has already demonstrated it is capable of.

Done when a deliberately degraded prompt fails the eval stage and opens a Bug naming the case, both
scores, and every configuration commit since the last run.

## Open decisions

Both block the phase that names them. Settle them as a change to the playbook, in the open, rather
than letting an implementation decide them quietly.

1. **Test modality for non-app repos.** D1 requires acceptance criteria expressible as one
   Playwright test each, and B3 statically checks the `data-testid` references. A fabric spec's
   criteria are workflow behaviours — verifiable in the sandbox, but not by Playwright against a
   deployed page.
   Either both plays take a per-repo test-modality input, or fabric specs get an exemption written
   into the plays. Letting the guardrail pass unsatisfied on non-Playwright criteria is the worst of
   the three, because the check then reports green while asserting nothing. Blocks Phase 5.
2. **Release process for `v1`.** Callers resolve `@v1` from the first play onward, so the bootstrap
   needs a release mechanism in Phase 0 — but [Deploy](README.md#5---deploy) is deliberately
   unspecified. Keep it to a minimal tag-promotion workflow, and treat it as the seed of the Deploy
   spec rather than an accidental convention nobody chose. Blocks Phase 0.

## Out of scope

Deploy and Maintain stay unwritten, and this plan does not write them. T2 needs no implementation —
it is human exploratory testing, and the only thing it requires is a UAT environment to test
against, which Phase 1 provides.
