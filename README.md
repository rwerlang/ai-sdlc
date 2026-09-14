# AI-SDLC

This repository is a playbook for AI-native SDLC that uses GitHub issues, projects, actions, and PRs to automate the development process. It is based on Anthropic's [AI-Native SDLC playbook](https://claude.com/blog/the-ai-native-sdlc-playbook).



## Glossary

**AI-Fabric** The agentic tools, scripts, pipelines, etc that run the AI automation.
**SDLC** Software Development LifeCycle



## High-level Process

The process uses the six stages with plays that are the core of the playbook and are grouped into six non-linear stages (Plan, Design, Build, Test, Deploy, Maintain), which together cover the complete lifecycle.

Each repo has a GitHub Project, and one of the views has a Kanban board to show the progress of the current iteration by status:
- **Backlog**: List of ideas/bugs for future implementation
- **Plan**: Intent is captured once as an artifact the next stage can act on
- **Design**: Requirements and design collapse into one session
- **Build**: Generate code and tests
- Test
- Deploy (Release a new version)

### Github issue types
- **Feature**: Describes an intent, idea or the problem to solve.
- **Bug**: Describes an error that must be fixed.
- **Task**: A task is a work item for a human to handle. Example: Create a new UI design using Claude Design.

### 0 - Backlog
#### BG1. Human creates issues in GitHub
Input
- A human creates a new issue with a title and a minimal description of the idea for future implementation.
- No AI involved.
- Issue:
  - Type: Feature | Bug
  - Labels: enhancement | bug
  - Project status: Backlog

Execute
- A human manually moves issues from the backlog to plan (`Status:Plan`) start working on a new set of issues.
- Backlog planning and prioritization happens during this stage.

Guardrails
- AI-Fabric issue classification

Output
- List of issues without details

### 1 - Plan
#### P1. Capture the intent of issues
Input
- An issue from the backlog
- New issue
- Issue:
  - Type: Feature | Bug
  - Labels: enhancement | bug
  - Project status: Plan

Execute
- A human creates a new issue or picks one from the backlog to describe the problem to solve, in their own words.
- Brainstorm with `@claude`, which writes the changes to the issue with a specific proto-spec template.
- One issue may result in multiple sub-issues if needed. Claude creates the new sub-issues when asked.
- When the issue is approved, a human assigns it to the AI-Fabric.

Guardrails
- AI-Fabric issue classification

Output
- Issue as a proto-spec

### 2 - Design
#### D1. Create the spec
Input
- A human assigned an issue to AI-Fabric

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: + ai-fabric:spec
    - Project status: Design
  - Create a new branch
  - Write a new `spec.md` based on the issue
  - Link the spec and the issue
  - When the spec requires a UI, create a new Task issue (`Status:Design`) for a human to work on it
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: + PR approval
  - TODO - Define what assign to AI-Fabric means

Guardrails
- None

Output
- PR containing `spec.md`

#### D2. Spec review and approval
Input
- PR containing `spec.md`

Execute
- A human reviews the spec against the intent (issue)
- Brainstorm with `@claude´ on the PR or human manually edits `spec.md`
- A human can ask another one to review
- A human makes the final call to accept the PR and merge it

Guardrails
- AI-Fabric GitHub Action
  - Review the spec against the intent (issue)
  - Validate `spec.md` has a valid template to guarantee that a human edit kept the required template
- GitHub
  - All comments must be resolved
  - Branch protection rules
  - Minimum approvals
  - CODEOWNERS approval
  - Author cannot approve
  - PR minimum code coverage validation

Output
- PR approved
- `spec.md` approved and merged


### 3 - Build
#### B1. Create the implementation plan
Input
- Issue
- CI trigger on `spec.md`

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: - PR approval, - ai-fabric:spec, + ai-fabric:plan
    - Project status: Build
  - Create a new branch
  - Write a new `plan.md` based on the issue and `spec.md`
  - Change `spec.md` to include it in the PR
  - Link the plan and the issue
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: + PR approval

Guardrails
- None

Output
- PR containing `plan.md` and `spec.md`

#### B2. Plan review and approval
Input
- PR containing `plan.md` and `spec.md`

Execute
- A human reviews the plan
- Brainstorm with `@claude´ on the PR or human manually edits `plan.md`
- A human can ask another one to review
- A human makes the final call to accept the PR and merge it

Guardrails
- AI-Fabric GitHub Action
  - Review the plan against the spec
  - Validate `plan.md` has a valid template to guarantee that a human edit kept the required template
  - Validate that is knows the build, lint, test commands for the next step
- GitHub
  - All comments must be resolved
  - Branch protection rules
  - Minimum approvals
  - CODEOWNERS approval
  - Author cannot approve
  - PR minimum code coverage validation

Output
- PR approved
- `plan.md` and `spec.md` approved and merged

#### B3. Code generation
Input
- Issue
- `spec.md`
- CI trigger on `plan.md`

Execute
- AI-Fabric GitHub Action
  - Issue:
    - Labels: - PR approval, - ai-fabric:plan, + ai-fabric:build
    - Project status: Build
  - Create a new branch
  - Generate code using agents and skills
  - Run lint, auto-fix
  - Run security validations and auto-fix
  - Build the code and auto-fix
  - Generate unit tests
  - Run unit tests and auto-fix
  - Create a new PR -> automatically included in the AI-Fabric project
  - Link the PR to the issue
  - Issue:
    - Labels: - ai-fabric:build, + PR approval, + ai-fabric:code-review
  - TODO - Generate playwright?????

Guardrails
- AI Build pass
- AI Security validations pass
- AI Unit tests pass

Output
- PR containing code changes

#### B4. Code review
Input
- PR containing code changes

Execute
- A human reviews the PR
- Ask `@claude´ to make changes, fixes, improve unit-test coverage
- A human can ask another one to review
- A human makes the final call to accept the PR and merge it

Guardrails
- PR build action
  - Build code
  - Run linter to validate warnings
  - Run unit tests and colect code coverage
- GitHub
  - All comments must be resolved
  - Branch protection rules
  - Minimum approvals
  - CODEOWNERS approval
  - Author cannot approve
  - PR minimum code coverage validation
  - CodeQL scan
  - SAST scan
  - Dependency scan
- AI-Fabric GitHub Action
  - Validate that the code covers the plan and spec
  - Code review

Output
- Code changes
- Unit tests

### 4 - Test
#### T1. CI pipeline
Input:
- Code and unit tests
- CI trigger

Execute:
- CI pipeline action
  - Stage build
    - Issue:
      - Labels: - PR approval, - ai-fabric:code-review, + ai-fabric:ci
    - Build code
    - Run linter to validate warnings
    - Run unit tests and colect code coverage
    - Create package (docker image or build artifact)
  - Stage deploy
    - Use concurrency group to sequence tests
    - Issue:
      - Labels: - ai-fabric:ci, + ai-fabric:test
      - Project status: Test
    - Deploy frontend and backend
  - Stage test
    - Run playwright smoke tests
    - Save test results as pipeline artifacts
    - TODO: Where to generate playwright tests??????
    - AI-Fabric
      - Validate test results and create new bugs for failed tests as sub-issues
      - Validate tests cover all `spec.md` acceptance criteria, create new bugs as sub-issues if needed
    - Issue:
      - Labels: + ai-fabric:tests-passed | ai-fabric:tests-failed
    - Fail the stage when tests fail
  - Stage UAT
    - Wait on environment approval
    - Deploy frontend and backend
    - Issue:
      - Labels: - ai-fabric:test, + ai-fabric:uat

Guardrails:
- Tests pass
- Tests cover acceptance criteria

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
- A human approves the issue when manual tests pass

Guardrails:
- None

Output:
- Issue approved for deploy

TODO - Changelog/release notes
TODO - update documentation and manuals
TODO - generate short videos explaining features
