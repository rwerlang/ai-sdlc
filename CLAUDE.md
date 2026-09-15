# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

**This repo is the AI-Fabric.** It is the org-level repository that holds the real automation —
one reusable GitHub Actions workflow per play (`on: workflow_call`), the shared `fabric-setup`
composite action, and the supporting scripts and prompts. Every app repo in the org consumes this;
an app repo carries only four thin caller workflows that guard on state and delegate here with
`uses: <org>/ai-fabric/.github/workflows/<play>.yml@v1`.

Nothing in this repo may be repo-specific. If a change needs to know an app's build, package or
deploy commands, it belongs in the app repo instead — see *Workflow layout* in the README, which is
explicit that `ci.yml` and `pr-build.yml` stay local.

[README.md](README.md) is the specification the fabric implements: the playbook of plays, guardrails
and invariants, based on Anthropic's
[AI-Native SDLC playbook](https://claude.com/blog/the-ai-native-sdlc-playbook). Read it before
changing anything here — a workflow that contradicts a play is a bug in the workflow.

**Current state:** the playbook is written; the workflows and actions it describes are not yet
committed. `README.md` and `.gitignore` are the only tracked files, so there is no build, lint or
test tooling yet. Adding those is the work ahead, not a gap to paper over.

## Target layout

Per the README's *Workflow layout* section:

- `.github/workflows/<play>.yml` — one reusable workflow per play (`P1`, `D1`, `B1`, `B3`, `T1`,
  `T3`, `T4`, `T5`, …), each `on: workflow_call`.
- `fabric-setup` composite action — the mechanics every play repeats: mint a short-lived
  installation token for the `app/ai-fabric` GitHub App, verify the actor who applied the triggering
  label has write permission (exit silently otherwise), read the issue's current labels and Project
  status for the idempotency guard, and apply the `ai-fabric:<stage>` progress label with
  `if: always()` cleanup.
- A second shared `workflow_call` workflow — "run the agent with this prompt, then open a PR" —
  covers the rest of the repetition.
- Callers get **one job per play**, so `permissions:`, `concurrency:` and check names stay job-level
  and each play keeps its own required check.

## How the playbook is structured

Work moves through plays grouped into six stages (Plan, Design, Build, Test, Deploy, Maintain);
Deploy and Maintain are deliberately unwritten. An issue runs on one of two tracks — **Standard**
(spec → plan → code, three human gates) or **Express** (straight to code, one gate).

Every play is written as **Input / Execute / Guardrails / Output**, and new or edited plays keep that
shape. Play IDs (`BG1`, `P1`, `D1`, `D2`, `B1`–`B4`, `T1`–`T5`) are referenced from prose throughout,
and a workflow file is expected to name its play — renaming or renumbering one means updating every
reference, in the document and in the fabric.

Everything after `## Reference` is detail the plays link into (PR gates, spec and plan lifecycle,
triggers and idempotency, workflow layout, failure handling). Plays link there instead of restating
a rule; keep it that way. The `stateDiagram-v2` block in *High-level Process* encodes the same state
machine the plays describe, so any change to how work moves between stages — rejection paths
included — must land in both.

## Invariants the fabric is built on

These are load-bearing, and most of them are a security boundary rather than a style choice. A change
that contradicts one is a change to the playbook's design — raise it rather than quietly making the
code consistent.

- **`ai-fabric:go` is the trust boundary of the whole pipeline.** Issue bodies and PR comments are
  untrusted input, so nothing runs before a maintainer with write permission applies that label, and
  every workflow verifies that permission before acting. Anything automated that creates work lands
  in `Backlog` so it cannot start an agent run on its own.
- **Run as the `app/ai-fabric` GitHub App, never the default `GITHUB_TOKEN`** — a `GITHUB_TOKEN` run
  triggers no further workflows, so the chain stalls after the first hop.
- **Trigger on the merge of a PR carrying a specific label, never a path filter**, and guard on
  state: read the issue's current labels and Project status first, exit without acting if the issue
  is not in the state the play consumes.
- **The AI-Fabric never approves or merges its own work**, never widens its own attempt budget, and
  never disables a failing check to pass a gate. Auto-fix gates get 3 attempts, then escalate:
  remove the progress label, add `ai-fabric:blocked`, assign the maintainer, comment with the failing
  gate and run link — and leave Project status unchanged.
- **Guardrails are GitHub Actions, never agent hooks** — nothing depends on what an individual runs
  on their laptop.
- **E2E is a repo capability, not a fixed part of the chain.** Not every repo has a UI, so the
  `e2e` caller input (default `false`) turns Playwright generation, the T1 smoke stage and the T3
  nightly on or off. Make the runner conditional, never the coverage gate.
- **The assertion rule (T1, T3).** An agent may repair Playwright locators, waits and fixtures; it
  may never weaken, relax, skip or delete an assertion that traces to an acceptance criterion.
- **T3 never edits the suite it runs**, and **T4 writes issues, never diffs** — an agent with write
  access to the assertions it is graded on will loosen them.
- **Specs and plans are permanent.** `/specs/<issue-number>/spec.md` freezes when the plan PR merges;
  it is superseded via front-matter (`status`, `supersedes`, `superseded-by`), never edited. Only
  `status: active` specs load as agent context.
- **There is no Blocked column.** A stalled issue keeps its column and carries `ai-fabric:blocked`.
  Progress labels move freely; Project status changes only when work genuinely returns to an earlier
  stage.

## Conventions

- Labels are `ai-fabric:<something>`; Project statuses are capitalized (`Backlog`, `Plan`, `Design`,
  `Build`, `Test`, `UAT`, `Deploy`).
- README prose wraps at roughly 100 columns, and sections cross-link with GitHub anchors
  (`[Failure handling](#failure-handling-and-escalation)`) — renaming a heading breaks them silently,
  so grep for the old anchor after any rename.
- Rationale belongs with the rule it justifies — most guardrails state *why* in the same bullet.
  Preserve that; a guardrail stripped of its reasoning invites someone to remove it.
- Commit messages are short, lowercase, imperative (`add evals`, `refactor identity section`).
