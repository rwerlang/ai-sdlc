# AI-SDLC

**This repository is the AI-Fabric.** It holds the reusable workflows, composite actions and scripts
that run the AI automation for every app repo in the org. An app repo carries only thin caller
workflows that delegate here — see [Workflow layout](#workflow-layout). Nothing in this repo is
repo-specific.

The rest of this document is the playbook the fabric implements: an AI-native SDLC that uses GitHub
issues, projects, actions, and PRs to automate the development process, based on Anthropic's
[AI-Native SDLC playbook](https://claude.com/blog/the-ai-native-sdlc-playbook). A workflow here that
contradicts a play is a bug in the workflow.

AI should automate as much as possible. Humans are in the loop to orchestrate, oversee, and have the
final word by approving the work.

## Glossary

- **AI-Fabric**: The agentic tools, scripts, pipelines, etc that run the AI automation.
- **SDLC**: Software Development LifeCycle.
- **Play**: One numbered step of the process, described as Input / Execute / Guardrails / Output.
- **Track**: Express or Standard — how many review gates an issue must pass.
- **Artifact**: A version-controlled file (`spec.md`, `plan.md`, code, tests) that one stage commits and the next stage reads.

## Principles

- **The artifact chain is the audit trail.** Intent, spec, plan, diff and review findings, each
  committed and approved before the next stage starts.
- **Every guardrail is a GitHub Action**, never an agent hook. Nothing depends on what an
  individual runs on their laptop.
- **Every acceptance criterion traces to at least one automated test.** Which runner proves it
  depends on the repo — see [E2E tests are optional](#e2e-tests-are-optional).
- **The AI-Fabric never approves or merges its own work.**

## High-level Process

The playbook is built from plays grouped into six non-linear stages — Plan, Design, Build, Test,
Deploy, Maintain.

> **Scope note:** stages 0 through 4 (Backlog, Plan, Design, Build, Test) are specified below.
> **Deploy and Maintain are named but not yet written.**

An issue travels the pipeline in one of two **tracks**, decided in P1:

- **Standard** — the full chain. Three artifacts, three human review gates: spec, plan, then code.
- **Express** — typo, copy change or small bug. Spec and plan stay in the issue body, and the issue
  goes straight to code generation and a single PR.

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Backlog
    Backlog --> Plan: human pulls into the iteration

    Plan --> Design: Standard track
    Plan --> Build: Express track

    state Design {
        [*] --> D1_spec
        D1_spec: D1 - AI-Fabric writes spec.md
        D1_spec --> D2_review
        D2_review: D2 - human reviews and merges
        D2_review --> [*]
    }

    state Build {
        [*] --> B1_plan
        B1_plan: B1 - AI-Fabric writes plan.md
        B1_plan --> B2_review
        B2_review: B2 - human reviews and merges
        B2_review --> B3_code
        B3_code: B3 - AI-Fabric writes code and tests
        B3_code --> B4_review
        B4_review: B4 - human reviews and merges
        B4_review --> [*]
    }

    state Test {
        [*] --> T1_ci
        T1_ci: T1 - CI, deploy, tests
        T1_ci --> T2_uat
        T2_uat: T2 - human exploratory testing
        T2_uat --> [*]
    }

    Design --> Build: spec.md on main
    Build --> Test: code on main
    Test --> Deploy: UAT approved

    Deploy: Deploy - not yet specified
    Deploy --> [*]

    Design --> Plan: spec PR closed
    Build --> Design: plan PR closed
    Test --> Build: UAT fails
```

### The board

Each repo has a GitHub Project, and one of its views is a Kanban board showing the progress of the
current iteration by status. An issue sitting in a column means:

- **Backlog**: Captured, not yet committed to an iteration. No AI involvement yet.
- **Plan**: Being shaped into a proto-spec with a human. Waiting on human authorization to proceed.
- **Design**: Its spec is being written or is in review.
- **Build**: Its plan or its code is being written or is in review.
- **Test**: Its code is merged and moving through CI, deployment and testing.
- **UAT**: Deployed and awaiting human exploratory testing.
- **Deploy**: Approved for release. *(Not yet specified.)*

There is deliberately **no Blocked column** — an issue that stops keeps its column and is flagged
with `ai-fabric:blocked`. Filter the board on that label to see everything waiting on a human. See
[Failure handling and escalation](#failure-handling-and-escalation).

### Github issue types

Only Github built-in types.

- **Feature**: Describes an intent, idea or the problem to solve.
- **Bug**: Describes an error that must be fixed.
- **Task**: Work item for a human to handle. Example: create a new UI design using Claude Design.

## The Plays

### 0 - Backlog

#### BG1. Human creates issues in GitHub

Input
- An idea or a defect, from anyone. No AI involved.

Execute
- A human creates an issue with a title and a minimal description of the idea for future implementation.
  - Type: Feature | Bug
  - Labels: enhancement | bug
  - Project status: Backlog
- Backlog planning and prioritization happen here.
- A human moves issues from the backlog to plan (`Status:Plan`) to start working on a new set of issues.

Guardrails
- None. Nothing is authorized before P1.

Output
- List of issues without details

### 1 - Plan

#### P1. Capture the intent of issues

Input
- A new issue, or one pulled from the backlog
- Issue:
  - Type: Feature | Bug
  - Labels: enhancement | bug
  - Project status: Plan

Execute
- A human describes the problem to solve, in their own words.
- Brainstorm with `@claude`, which writes the changes to the issue with a specific proto-spec template.
- One issue may result in multiple sub-issues if needed. Claude creates the new sub-issues when asked.
- **Track classification.** The AI-Fabric proposes a track (see
  [High-level Process](#high-level-process)), carrying `ai-fabric:classify` while the Action runs:
  - Express requires *all* of: no schema change, no new dependency, no change to an authentication,
    authorization or other security surface, no public API change, no UI design work, and a single
    component affected. Anything else is Standard.
  - The classification is a recommendation. A maintainer can always force an issue up to Standard;
    only a maintainer can move one down to Express.
- When the issue is approved, a maintainer applies `ai-fabric:go` to hand it to the AI-Fabric.

Guardrails
- AI-Fabric issue classification (as above) — runs as an Action, and its output is visible in the
  issue for the maintainer to override before authorizing.
- **`ai-fabric:go` is the trust boundary of the whole pipeline** — issue bodies and PR comments are
  untrusted input, so nothing the AI-Fabric does happens before this label is applied. The workflow
  verifies the actor who applied it has write permission and exits silently otherwise, so issue text
  from a non-collaborator can never start an agent run.

Output
- Issue as a proto-spec
- Track assigned (Express | Standard)

### 2 - Design

#### D1. Create the spec

Input
- An issue labelled `ai-fabric:go` on the Standard track

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: + ai-fabric:spec
    - Project status: Design
  - Create a new branch from `main`
  - Write a new `/specs/<issue-number>/spec.md` based on the issue, applying the organizational
    skills — security, compliance, brand, UX, accessibility
  - Set the spec front-matter (`issue`, `supersedes`, `status: active`), and if it supersedes an
    existing spec, flip that spec's `status` to `superseded` and record its `superseded-by` in the
    same PR — see [Spec and plan lifecycle](#spec-and-plan-lifecycle)
  - Regenerate `/specs/README.md`, the index of active specs, in the same PR
  - Link the spec and the issue
  - When the spec requires a UI, create a Task sub-issue (`Status:Design`) for a human, apply
    `ai-fabric:blocked-on-design` to the parent, and stop the chain until the Task closes
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: - ai-fabric:spec, + ai-fabric:awaiting-approval

Guardrails
- Validate `spec.md` against the required template at generation time, not only at review.
- Validate the spec states testable acceptance criteria. B3 generates at least one automated test
  per criterion — a Playwright test where the repo runs [E2E](#e2e-tests-are-optional), a unit or
  integration test otherwise — so a criterion that cannot be asserted is a defect here.
- Attempt budget and escalation per [Failure handling](#failure-handling-and-escalation).

Output
- PR containing `spec.md`

#### D2. Spec review and approval

Input
- PR containing `spec.md`

Execute
- A human reviews the spec against the intent (issue)
- Brainstorm with `@claude` on the PR or human manually edits `spec.md`
- A human makes the final call to accept the PR and merge it

Guardrails
- AI-Fabric GitHub Action
  - Review the spec against the intent (issue)
  - Validate a human edit kept the required `spec.md` template
  - Validate front-matter and supersession pointers are consistent
- GitHub — [Standard PR gates](#standard-pr-gates)

Output
- PR approved
- `spec.md` approved and merged

### 3 - Build

#### B1. Create the implementation plan

Input
- Issue
- Merge of a PR labelled `ai-fabric:spec`

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: - ai-fabric:awaiting-approval, + ai-fabric:plan
    - Project status: Build
  - Create a new branch from `main`
  - Write a new `/specs/<issue-number>/plan.md` based on the issue and `spec.md`
  - Write the full content of `spec.md` into the **PR description**, inside a collapsed `<details>`
    block, together with a permalink pinned to the merged commit SHA. Do not modify `spec.md`
    itself to force it into the diff
  - Link the plan and the issue
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: - ai-fabric:plan, + ai-fabric:awaiting-approval

Guardrails
- Validate `plan.md` against the required template at generation time.
- Validate every acceptance criterion in `spec.md` is addressed by at least one step of the plan.
- Attempt budget and escalation per [Failure handling](#failure-handling-and-escalation).

Output
- PR containing `plan.md`, with `spec.md` reproduced in the PR description

#### B2. Plan review and approval

Input
- PR containing `plan.md`, with `spec.md` in the description

Execute
- A human reviews the plan against the spec
- Brainstorm with `@claude` on the PR or human manually edits `plan.md`
- If review shows the spec is wrong, correct `spec.md` in this PR — it is not yet frozen, and
  merging this PR is what freezes it
- A human makes the final call to accept the PR and merge it

Guardrails
- AI-Fabric GitHub Action
  - Review the plan against the spec
  - Validate a human edit kept the required `plan.md` template
  - Validate that it knows the build, lint, and test commands for the next step — in practice, that
    `CLAUDE.md` is present and current
- GitHub — [standard PR gates](#standard-pr-gates)

Output
- PR approved
- `plan.md` approved and merged

#### B3. Code generation

Input
- Issue
- `spec.md`
- Merge of a PR labelled `ai-fabric:plan` (Standard track), or `ai-fabric:go` on an Express issue

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: - ai-fabric:awaiting-approval (Standard) | - ai-fabric:go (Express), + ai-fabric:build
    - Project status: Build
  - Create a new branch from `main`
  - Generate code using agents and skills, guided by `CLAUDE.md` and the organizational skills
  - Run lint, auto-fix
  - Run security validations and auto-fix
  - Build the code and auto-fix
  - Generate unit tests
  - Run unit tests and auto-fix
  - **Cover every acceptance criterion in `spec.md` with at least one automated test.** Where the
    repo runs [E2E](#e2e-tests-are-optional), that is one Playwright test per criterion, selecting
    on the `data-testid` convention from `CLAUDE.md`. A criterion with no UI surface — and every
    criterion in a repo with `e2e: false` — is covered by a unit or integration test instead
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: - ai-fabric:build, + ai-fabric:awaiting-approval

Guardrails
- Build, security validations and unit tests pass
- Every acceptance criterion in `spec.md` maps to at least one test in the diff
- Where E2E is enabled: Playwright tests parse and compile, and every `data-testid` they reference
  exists in the diff — a static check, no environment required
- Every auto-fix loop is bounded — 3 attempts per gate, then escalate per
  [Failure handling](#failure-handling-and-escalation).

Output
- PR containing code changes, unit tests, and — where E2E is enabled — Playwright tests

**Sequencing caveat, E2E repos only:** Playwright tests first *execute* in T1, against the deployed
app. B4 therefore reviews Playwright tests that have never run, and B3's "tests pass" guardrail
covers unit tests only. A broken selector reaches `main` and is only discovered after merge. A repo
with `e2e: false` does not have this gap — every test it generates runs before B4 reviews it.

#### B4. Code review

Input
- PR containing code changes, unit tests, and — where E2E is enabled — Playwright tests

Execute
- A human reviews the PR — the behaviour and the tests that prove it, together
- Ask `@claude` to make changes, fixes, improve unit-test coverage
- A human makes the final call to accept the PR and merge it

Guardrails
- PR build action
  - Build code
  - Run linter to validate warnings
  - Run unit tests and collect code coverage
- GitHub — [standard PR gates](#standard-pr-gates), plus:
  - PR minimum code coverage validation
  - CodeQL scan
  - SAST scan
  - Dependency scan
- AI-Fabric GitHub Action (adds `ai-fabric:code-review` at the start of its run, removes it at the end)
  - Validate that the code covers the plan and spec
  - Validate every acceptance criterion is covered by at least one automated test — a Playwright
    test where [E2E](#e2e-tests-are-optional) is enabled and the criterion has a UI surface, a unit
    or integration test otherwise
  - Code review

Output
- Code changes
- Unit tests
- Playwright tests, where E2E is enabled

### 4 - Test

#### T1. CI pipeline

Input:
- Code, unit tests and — where [E2E](#e2e-tests-are-optional) is enabled — Playwright tests, on
  `main`
- CI trigger

Execute:
- CI pipeline action
  - Stage build
    - Issue:
      - Labels: - ai-fabric:awaiting-approval, + ai-fabric:ci
    - Build code
    - Run linter to validate warnings
    - Run unit tests and collect code coverage
    - Create package (docker image or build artifact)
  - Stage deploy
    - Use concurrency group to sequence tests
    - Issue:
      - Labels: - ai-fabric:ci, + ai-fabric:test
      - Project status: Test
    - Deploy frontend and backend
  - Stage test
    - Run the tests that trace to the acceptance criteria. Where [E2E](#e2e-tests-are-optional) is
      enabled, that is the Playwright smoke suite against the deployed environment. Where
      `e2e: false`, nothing runs against the environment and the stage carries forward the unit
      and integration results from the build stage — same triage, same labels, so the rest of the
      chain is identical
    - Save test results as pipeline artifacts
    - AI-Fabric
      - Triage failures (see the assertion rule below)
      - Validate test results and create new bugs for failed tests as sub-issues
      - Validate tests cover all `spec.md` acceptance criteria, create new bugs as sub-issues if needed
    - Issue:
      - Labels: - ai-fabric:test, + ai-fabric:tests-passed | ai-fabric:tests-failed
    - Fail the stage when tests fail
  - Stage UAT
    - Wait on environment approval
    - Deploy frontend and backend
    - Issue:
      - Labels: + ai-fabric:uat
      - Project status: UAT

Guardrails:
- Tests pass and cover the acceptance criteria
- **The assertion rule.** A failure is either a bad locator or timing assumption, or a genuine
  defect. The AI-Fabric may repair locators, waits and fixtures. It may **not** weaken, relax, skip
  or delete an assertion that traces to an acceptance criterion — that requires a human and a spec
  change. Without this rule the agent resolves real bugs by loosening assertions, and the suite
  silently stops testing anything. The rule is about what an agent may do to a test that proves a
  criterion, so it holds whatever runner executes it — Playwright or otherwise.

Output:
- App running in UAT
- Build artifact
- Test results

#### T2. Manual tests

Input:
- App running in UAT
- Issue

Execute:
- A human manually/exploratory tests the app running in UAT against `spec.md` acceptance criteria
- A human creates new Bug issues as sub-issues for any failures found
- **Anything found here that the automated suite missed is by definition a coverage gap** — the Bug
  sub-issue carries a "add the regression test" requirement, so the suite grows from real escapes
- A human approves the issue when manual tests pass

Guardrails:
- None. This play is the human judgment the automated gates cannot replace.

Output:
- Issue approved for deploy, labelled `ai-fabric:uat-approved`

The issue is **not** closed here. It closes at release, in the not-yet-specified Deploy stage.

#### T3. Nightly regression pipeline

**Runs only where [E2E](#e2e-tests-are-optional) is enabled.** A repo with `e2e: false` has no
suite that needs a deployed environment — its unit and integration tests already run on every
push, in T1 — so there is nothing for a nightly to regress against.

Input:
- Scheduled nightly trigger (cron)
- All existing playwright tests, **unchanged**

Execute:
- CI pipeline action
  - Run all existing playwright tests against the latest deployed environment
  - Save test results as pipeline artifacts
  - AI-Fabric
    - Triage failures (see the assertion rule in T1)
    - Create a Bug sub-issue for each failure, unless an open Bug already names the same test id
    - Open a single PR for any locator, wait or fixture repairs
  - Fail the stage when tests fail

Guardrails:
- **The suite is frozen at run time.** T3 never generates or edits tests before running them. A
  nightly that rewrites its own suite cannot tell a regression from a test that was never going to
  pass, and an agent holding write access to the assertions it is graded on will loosen them to
  reach green. Coverage gaps are T4's job, on a different cadence and with no write access to code.
- The assertion rule from T1 applies here too: locator repairs are allowed, assertion changes are not.
- **Repairs never reach `main` unreviewed.** The repair PR goes through B4 and the
  [standard PR gates](#standard-pr-gates) like any other code change. The nightly job has no direct
  write access to `main`.
- **One open Bug per failing test.** A failure that persists for a week must not file seven issues —
  triage matches on test id against open `ai-fabric:nightly` Bugs and comments on the existing one.

Output:
- Test results
- New Bug issues for failures not already tracked
- Optionally, a PR with locator, wait or fixture repairs

#### T4. Coverage audit

Per-issue coverage is already gated in B4 and T1, and both are scoped to a single issue. This play
catches what neither can see: drift across the suite as a whole.

Input:
- Scheduled weekly trigger (cron)
- All `status: active` specs
- The full suite of tests that trace to acceptance criteria — the Playwright suite where
  [E2E](#e2e-tests-are-optional) is enabled, the unit and integration tests otherwise

Execute:
- AI-Fabric GitHub Action
  - Diff every acceptance criterion of every active spec against the suite, and classify:
    - **Gaps** — a criterion with no test
    - **Silent tests** — a test that exists for a criterion but no longer asserts it, is skipped, or
      was removed by an unrelated refactor
    - **Supersession orphans** — tests written for the criteria of a `status: superseded` spec,
      still asserting behaviour the product deliberately abandoned. Left in place, T3 faithfully
      raises Bugs for correct new behaviour failing an obsolete test
  - Save the full audit as a pipeline artifact
  - Create one sub-issue per finding on the spec's originating issue, skipping findings that already
    have an open sub-issue:
    - Gap or silent test -> Bug, `Status: Backlog`
    - Orphan -> Task, `Status: Backlog`

Guardrails:
- **Issues only, never diffs.** T4 writes no test code. Each finding re-enters the chain as an
  ordinary issue: a human authorizes it with `ai-fabric:go`, B3 writes the test, B4 reviews it, and
  T1 executes it once against a deployed app before it can ever gate a nightly.
- **Findings land in `Backlog`**, so nothing the audit produces starts an agent run on its own and
  the `ai-fabric:go` trust boundary holds.
- Scope is `status: active` specs per [Spec and plan lifecycle](#spec-and-plan-lifecycle) — not
  "recent" specs, which is an arbitrary window, and superseded specs only to find their orphans.
- The audit reports; it does not judge whether a gap is worth closing. A maintainer triaging the
  Backlog makes that call.

Output:
- Coverage audit report as a pipeline artifact
- New Bug and Task sub-issues for gaps, silent tests and orphans

#### T5. AI-Fabric evals

The AI-Fabric is software, and it is the only software in this process with no SDLC of its own.
Without this play, a prompt edit that quietly degrades spec quality is invisible until it reaches a
human reviewer, several plays downstream.

Input:
- Scheduled weekly trigger (cron)
- The current fabric configuration — prompts, skills, `CLAUDE.md`, workflow definitions
- The eval set: real historical issues **from this repo**, with their merged `spec.md`, `plan.md`
  and diffs as the recorded output
- The previous passing run's scores, as the baseline

Execute:
- AI-Fabric evals workflow
  - Replay every eval case against the current configuration
  - Score each generated artifact against the recorded one — template conformance, acceptance
    criteria coverage, and a rubric judged by an agent
  - Compare against the baseline, and save scores and generated artifacts as pipeline artifacts
  - Create a Bug issue in `<org>/ai-fabric` for each case that regressed past its threshold, naming
    the case, the previous and current scores, and every configuration commit since the last run
  - Fail the stage on regression

Guardrails:
- **The eval set is append-only.** A case is added whenever a play escapes to a human, and is never
  deleted or edited to accommodate a regression. Retiring one takes a human and a PR that says why.
- **The fabric never repairs itself in this run.** A regression opens a Bug that travels the normal
  chain like any other defect.
- **The baseline advances only on a passing run**, so a regression cannot quietly become the new
  normal by surviving a week.
- Scores are kept per run, so a slow drift over several weeks is as visible as a week-over-week drop.
- **Each app repo evaluates the shared fabric against its own history.** The configuration under
  test is the same everywhere, but a prompt edit can degrade one domain and not another, and only
  the repo that owns those issues will see it.

Output:
- Eval scores and generated artifacts as pipeline artifacts
- New Bug issues in `<org>/ai-fabric` for regressions

### 5 - Deploy

Not yet specified. Known scope: release, changelog and release notes, documentation and manual
updates, and short videos explaining new features.

### 6 - Maintain

Not yet specified.

---

## Reference

Everything below is detail the plays refer to.

### Standard PR gates

Enforced by GitHub on every AI-Fabric PR:

- All comments resolved
- Branch protection: minimum approvals, CODEOWNERS approval, author cannot approve

Enforced by a required status check on every AI-Fabric PR:

- **Frozen-spec check.** A spec is frozen once the PR labelled `ai-fabric:plan` merges. A PR that
  modifies a frozen `spec.md` fails, unless its only change to that file is the supersession pointer
  (`status`, `superseded-by`). See [Spec and plan lifecycle](#spec-and-plan-lifecycle).

### Spec and plan lifecycle

Specs and plans live at `/specs/<issue-number>/spec.md` and `/specs/<issue-number>/plan.md`. They
are **permanent** — never edited once frozen, superseded instead.

Each spec carries front-matter:

```yaml
issue: 456
supersedes: [123]
superseded-by: null
status: active   # active | superseded
```

**Only `status: active` specs are loaded as agent context.** That is what supersession buys, and the
reason a frozen spec is replaced rather than edited.

### E2E tests are optional

Not every repo has a UI. A backend service, a library or a CLI has acceptance criteria worth proving
and nothing for Playwright to drive, so E2E is a repo capability — each app repo declares it once,
as an input its caller workflows pass down:

```yaml
uses: <org>/ai-fabric/.github/workflows/b3-code.yml@v1
with:
  e2e: true    # default: false
```

It is **declared, not detected.** Sniffing for a `playwright.config.*` makes "this repo has no UI"
indistinguishable from "nobody has written the first test yet", and a config file lost in a refactor
would silently retire a whole gate rather than fail loudly.

The flag chooses the runner, never the gate — making coverage itself conditional would turn "we have
no UI" into "we test nothing".

### Triggers and idempotency

- **Trigger on the merge of a PR carrying a specific label**, never a path filter — a path filter
  lets a human editing a spec, or an unrelated merge from `main`, retrigger a generation play.
- **Guard on state.** Every workflow first reads the issue's current labels and Project status, and
  exits without acting if the issue is not in the state that play consumes.
- **Run as the `app/ai-fabric` GitHub App**, never the default
  `GITHUB_TOKEN` — a `GITHUB_TOKEN` run triggers no further workflows, so the chain would stall
  after the first hop.

### Workflow layout

The AI-Fabric is **centralized in an org-level repo** and consumed by every app repo.

`<org>/ai-fabric` holds all the real logic — one reusable workflow per play (`on: workflow_call`)
plus the shared `fabric-setup` composite action. Nothing in it is repo-specific.

Each app repo carries **four thin caller workflows**, grouped by trigger rather than by play,
because a workflow's `on:` block must be declared in the repo where the event happens:

| Caller workflow | Trigger | Fans out to |
| --- | --- | --- |
| `fabric-issues.yml` | `issues` (opened, edited, labeled, closed) | P1 classify, D1 spec, unblock |
| `fabric-pr.yml` | `pull_request` (opened, synchronize, closed) | D2/B2/B4 reviews, B1 plan, B3 code |
| `fabric-pipeline.yml` | `push` to `main`, `schedule` | T1 CI, T3 nightly regression (E2E repos only), T4 coverage audit |
| `fabric-evals.yml` | `schedule` (weekly), `workflow_dispatch` | T5 AI-Fabric evals |

Each caller job is a guard plus `uses: <org>/ai-fabric/.github/workflows/<play>.yml@v1`, **one job
per play**, so `permissions:`, `concurrency:` and check names stay job-level and each play keeps its
own required check.

**Do not centralize `ci.yml` and `pr-build.yml`.** Build, package and deploy commands are genuinely
repo-specific. They stay local, and may call a shared reusable workflow for the common shape with
repo-level inputs.

#### The shared composite action

The mechanics every play repeats live in `fabric-setup`:

1. Mint a short-lived installation token for the `app/ai-fabric` GitHub App, which holds
   fine-grained repository permissions.
2. Verify the actor who applied the triggering label has write permission; exit silently otherwise.
3. Read the issue's current labels and Project status, and apply the idempotency guard above.
4. Apply the `ai-fabric:<stage>` progress label, with `if: always()` cleanup so a crashed run leaves
   nothing orphaned.

A second shared `workflow_call` workflow — "run the agent with this prompt, then open a PR" — covers
the rest.

### Failure handling and escalation

Every generation play is bounded.

- **Attempt budget:** each auto-fix gate in B3 (lint, security, build, unit tests) gets at most
  3 attempts. The job also carries a wall-clock cap and a token budget.
- **On exhaustion:** remove the progress label, add `ai-fabric:blocked`, assign the maintainer who
  applied `ai-fabric:go`, and comment with the failing gate, the last failing output, and a link to
  the run. **Project status does not change.**
- **The agent never widens its own budget** and never disables a failing check to make a gate pass.

Rejection paths — these *do* move Project status, because the work genuinely returns to an earlier
stage:

- **Spec PR closed unmerged** — status returns to `Plan`, all `ai-fabric:*` labels removed except
  `ai-fabric:go`, which is also removed so re-entry requires a fresh human authorization.
- **Plan PR closed unmerged** — status returns to `Design`. The merged spec stays on `main`; B1 can
  be re-triggered by re-applying `ai-fabric:spec` to a new plan PR.
- **Code PR closed unmerged** — status returns to `Build`, awaiting a new B3 run.
- **UAT fails (T2)** — status returns to `Build`, `ai-fabric:uat` removed, and the Bug sub-issues
  raised by the human become the input to a new B3 run on the parent.

Blocked on human work:

- When D1 creates a UI design Task, it is created as a **sub-issue** of the parent, and the parent
  gets `ai-fabric:blocked-on-design`. The parent keeps status `Design`, where the work actually is.
  B1 does not trigger while that label is present. Closing the Task fires a workflow that clears the
  label and resumes the chain. This is a real dependency, not just a linked issue.

