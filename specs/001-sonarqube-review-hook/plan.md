# Implementation Plan: SonarQube Review Hook

**Branch**: `001-sonarqube-review-hook` | **Date**: 2026-10-16 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-sonarqube-review-hook/spec.md`

## Summary

A set of three Claude Code hooks, written in Bash, that close the loop between the agent's edits and SonarQube:

1. **Record** (PostToolUse on file-editing tools) notes which files the agent touched this turn.
2. **Reset** (UserPromptSubmit) starts a new task: clears the touched-file list and the review-round counter.
3. **Review** (Stop) groups the touched files by SonarQube project, runs the SonarQube CLI on them, waits for the analysis, collects the findings, saves a timestamped Markdown report, updates the tech debt register, and either lets the agent stop or returns a `block` decision whose reason carries the findings so the agent fixes them.

The server address is resolved from configuration first, then (inside a container) by probing the host, then localhost. Every failure path ends in a short message and a normal stop.

See [research.md](research.md) for decisions, [data-model.md](data-model.md), [contracts/](contracts/) and [quickstart.md](quickstart.md).

## Technical Context

**Language/Version**: Bash 5 (POSIX tools plus `jq` and `curl`)

**Primary Dependencies**: SonarQube CLI (`sonar` 1.10+), `jq`, `curl`; Claude Code hook protocol (PostToolUse, UserPromptSubmit, Stop)

**Storage**: Plain files only: per-session state under `docs/.sonar-review/.state/`, reports under `docs/.sonar-review/feedback/`, register at `docs/.sonar-review/tech-debt.md`

**Testing**: bats (Bash Automated Testing System) with a fake `sonar` CLI and fake SonarQube HTTP endpoint as test doubles. Not yet installed or declared in the project; see the gap noted in the Verification Plan.

**Target Platform**: Linux and macOS shells, on the host and inside a devcontainer (Debian/Ubuntu-based images)

**Project Type**: Single project: a small hook toolkit with an installer

**Performance Goals**: Feedback within 2 minutes for a small project (SC-002); the hook's own overhead beyond SonarQube analysis time is under 2 seconds

**Constraints**: Never block the agent indefinitely (default maximum wait 240 s, configurable); no secrets in any output; dependencies limited to what a standard devcontainer has plus `jq`

**Scale/Scope**: One developer session at a time; typically 1–20 changed files per turn; monorepos with several `sonar-project.properties` files

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

`.specify/memory/constitution.md` is still the unfilled template, so it defines no principles and imposes no gates. **Result: no violations.** Recommendation: run `/speckit-constitution` before `/speckit-implement` so principles such as test-first can be enforced. The Integrated TDD process rules already apply independently.

**Post-design re-check**: unchanged, no violations.

## Project Structure

### Documentation (this feature)

```text
specs/001-sonarqube-review-hook/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   ├── hook-protocol.md
│   ├── configuration.md
│   ├── feedback-report.md
│   └── tech-debt-register.md
└── tasks.md             # Phase 2 output (/speckit-tasks, not created here)
```

### Source Code (repository root)

```text
bin/
├── sonar-review-record      # PostToolUse hook entry point
├── sonar-review-reset       # UserPromptSubmit hook entry point
└── sonar-review-stop        # Stop hook entry point
lib/
├── config.sh                # defaults, env and .env loading, credential lookup
├── server.sh                # environment detection, address resolution, status probe
├── project.sh               # sonar-project.properties lookup, grouping, fallback key
├── state.sh                 # touched files, round counter, per-project lock
├── analyze.sh               # project creation, CLI invocation, polling, result fetch
├── severity.sh              # SonarQube severity to critical/high/low mapping
├── report.sh                # feedback file and agent message rendering
├── debt.sh                  # tech debt register read, dedupe, append
└── log.sh                   # redacting logger
install.sh                   # merges hook registration into .claude/settings.json, updates .gitignore
examples/
└── devcontainer/            # sample devcontainer.json and compose override
tests/
├── helpers/                 # fake sonar CLI, fake SonarQube HTTP server, fixtures
├── unit/
├── integration/
└── fixtures/
Makefile                     # `make test` entry point (to be created in the first task)
```

**Structure Decision**: Single project. Thin executables in `bin/` read the hook JSON and call small, individually testable functions in `lib/`. Runtime output lands in the target project under `docs/.sonar-review/`, not in this repository's source tree.

## Complexity Tracking

No constitution violations to justify.

## Verification Plan

### Verification commands

| ID | Command | Scope | Source |
|----|---------|-------|--------|

### Verification obligations

| ID | Requirements | Scenario | Expected outcome | Level | Origin |
|----|--------------|----------|------------------|-------|--------|
| VO-001 | FR-001, FR-002, US1-AS4 | A turn in which the agent edits no project file, and a turn in which it edits three files across several tool calls | No analysis and no output after the first turn; exactly one review, run once at the end of the turn, after the second | integration | planned |
| VO-002 | FR-003, FR-005, US1-AS1, US1-AS3 | The agent changes one file containing a known issue and the server reports it | The analysis is requested for that file only, and the agent receives a message listing the issue with its severity, file, line and description | integration | planned |
| VO-003 | FR-003 | The server also holds issues for a file the agent did not change in this turn | Those issues appear in neither the agent message nor the saved report | integration | planned |
| VO-004 | US1-AS2 | The analysis reports no issues and the quality gate passes | The agent is told the review passed in a clear one-line message and is not made to continue | integration | planned |
| VO-005 | FR-004 | The analysis is still running at the first status check and completes after a few checks | The hook checks repeatedly, retrieves results only once complete, and the results are those of that analysis | integration | planned |
| VO-006 | FR-004, US4-AS3, SC-004 | The analysis never completes within the configured maximum wait | The hook stops waiting at the maximum wait, tells the agent the review timed out, and the agent is allowed to stop | integration | planned |
| VO-007 | FR-006, US2-AS1, US2-AS3 | A review completes with two issues and a failing quality gate | A Markdown file exists in `docs/.sonar-review/feedback/` whose name holds the date and time, showing when it ran, project, branch, quality gate result and each issue | integration | planned |
| VO-008 | US2-AS2, SC-005 | Two reviews complete within the same minute, and then 50 consecutive reviews run | Each review leaves exactly one new file; no earlier file is overwritten or lost | integration | planned |
| VO-009 | FR-007, FR-017 | A review finds a critical issue | The Stop hook output makes the agent continue, and the continuation text contains the findings | integration | planned |
| VO-010 | FR-017 | A review finds only Low and Info issues | The agent is allowed to stop; the issues are in the saved report and in the tech debt register | integration | planned |
| VO-011 | FR-017 | Reviews keep finding a critical issue for more rounds than the configured limit (default 3) | After the last allowed round the agent is allowed to stop, and the unresolved critical issue is in the tech debt register | integration | planned |
| VO-012 | FR-017 | A review is clean after two rounds with issues; separately, the developer sends a new prompt after two rounds | In both cases the next review starts a fresh count with the full limit available | integration | planned |
| VO-013 | FR-018 | Issues rated Blocker, High, Medium, Low and Info are reported | Blocker is presented as critical (must fix), High and Medium as high (should fix unless it risks functionality), Low and Info as low (may fix) | unit | planned |
| VO-014 | FR-019 | Some issues remain unfixed after a round; the same issues are reported in a later review | The register holds one entry per issue with severity, file, line, description, reason and first-recorded date, and no issue appears twice | integration | planned |
| VO-015 | FR-008 | A review returns far more issues than fit the configured message cap | The message to the agent is within the cap, lists the most severe issues first and names the saved file; the saved file holds every issue | integration | planned |
| VO-016 | FR-015 | An issue message contains text written as an instruction to the agent | The text is delivered inside a delimited block labelled as untrusted review data, with control characters removed | unit | planned |
| VO-017 | FR-009, US3-AS2 | The hook runs outside any container with no address configured | It uses the local address and performs no container-specific probing | integration | planned |
| VO-018 | FR-010, FR-011, US3-AS1 | The hook runs inside a container, nothing is configured, and only one of the candidate host addresses answers the server status check | It selects the answering address, and the report states which address and which source (discovery) were used | integration | planned |
| VO-019 | FR-011, US3-AS3 | An explicit address is configured, both outside and inside a container | The configured address is used in both cases, with no detection attempted | integration | planned |
| VO-020 | FR-011, EC-01 | EC-01 (no server address can be found inside a container): inside a container, configuration is absent and no candidate answers | The review is skipped, the agent is told so with how to set the address, and the agent is allowed to stop | integration | planned |
| VO-021 | FR-013, US4-AS1, SC-004 | The server is down, the connection is refused, packets are silently dropped, or a proxy answers with a non-SonarQube response | In every case the hook gives up within the bounded probe time, tells the agent the review was skipped because the server could not be reached, and the agent is allowed to stop | integration | planned |
| VO-022 | FR-012, US4-AS2, SC-006 | No token is configured, and separately the server rejects the token | The agent is told the review was skipped for authentication with a fix hint; the token value appears in no output, saved file or log | integration | planned |
| VO-023 | US4-AS4 | The server reports the analysis task as failed with a reason | The agent is told the review failed, with the reason, and is allowed to stop | integration | planned |
| VO-024 | FR-020 | Changed files lie under two different `sonar-project.properties` files and a third lies under none | Each file is analysed under the project of its closest properties file; files without one share a project named after the repository root folder; results are combined into one report | integration | planned |
| VO-025 | FR-021 | A resolved project does not exist: once with credentials that may create projects, once without | With permission the project is created and analysed; without it that project is skipped with the reason in the feedback while other projects still complete | integration | planned |
| VO-026 | FR-014 | A second review starts while one is running for the same project | The second makes no concurrent analysis, tells the agent it was skipped because a review is in progress, and the first completes normally | integration | planned |
| VO-027 | FR-022 | The hook runs in a git project for the first time and again later | Reports go to `docs/.sonar-review/feedback/` and the register to `docs/.sonar-review/tech-debt.md`; ignore rules cover the feedback and state folders but not the register; repeating the run adds no duplicate rules | integration | planned |
| VO-028 | FR-013, EC-02 | EC-02 (the saved-feedback location is not writable): the output folder cannot be written | The agent still receives the findings together with a note that they could not be saved, and the session continues | integration | planned |
| VO-029 | FR-002, DC-01 | DC-01 (files under `docs/.sonar-review/` are never reviewed): the agent's only edit in a turn is to the tech debt register or another file in the output folder | No review starts | integration | planned |

### Verification exemptions

| Requirement | Reason | Alternative verification |
|-------------|--------|--------------------------|
| FR-016 | A real SonarQube server and a real devcontainer are needed, which the automated suite cannot start | The manual procedure in `quickstart.md` is run in both the local environment and a devcontainer, with the known-issue scenario of SC-007; results are noted in the pull request |
| SC-002 | The 2-minute target depends on the real server and project size | Timed run of the quickstart scenario on a small project in both environments |
