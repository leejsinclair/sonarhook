# Feature Specification: SonarQube Review Hook

**Feature Branch**: `001-sonarqube-review-hook`

**Created**: 2026-10-16

**Status**: Draft

**Grilled on**: 2026-10-16

**Input**: User description: "A Claude Code hook that calls SonarQube on port 9000 for code review: push changes to SonarQube, poll until analysis completes, request feedback, save the feedback to a date-timestamped .md file, and give the feedback to the running agent. Must also work when the project runs in a devcontainer and has to reach the host's SonarQube. Two environments to test in: local development environment and inside a development container. The hook runs the SonarQube CLI."

## Clarifications

### Session 2026-10-16

- Q: What does a "review" analyse? → A: Only the files changed in the current agent turn.
- Q: Which agent event triggers the review? → A: When the agent finishes its turn (one review per turn, not per edit).
- Q: What should the agent do with the feedback? → A: The agent acts on it, by severity: it must fix critical issues, should fix high issues unless fixing would put existing functionality at risk, and can fix low issues; every issue not fixed is recorded in a tech debt register Markdown file.
- Q: How do your severity tiers map to SonarQube's? → A: Critical = Blocker; High = High and Medium; Low = Low and Info.
- Q: Which SonarQube project do findings belong to? → A: For each changed file, use the closest sonar-project.properties file (searching upward from the file) to determine its project; if none is found, fall back to the project folder name. Monorepos may therefore yield several projects in one review.
- Q: What if the resolved project doesn't exist on the server? → A: Create it automatically if the token allows; otherwise skip that project and state why in the feedback.
- Q: How does the hook reach the host's SonarQube from a devcontainer? → A: Primarily through configuration supplied by the devcontainer definition (for example a compose-based devcontainer that sets the server address), and the hook must also be able to discover the host server address from inside the container when no configuration is given.
- Q: Where are the feedback files and the tech debt register stored, and are they committed? → A: In a `docs/.sonar-review/` folder: date-stamped feedback files in `docs/.sonar-review/feedback/` (git-ignored) and the register at `docs/.sonar-review/tech-debt.md` (committed).
- Q: When no sonar-project.properties file is found, which folder name is the fallback project name? → A: The repository root folder name, so all files without a properties file share one project.
- Q: What resets the consecutive review-round counter? → A: Either a review that finds no issues for the changed files, or the developer sending a new prompt, whichever comes first.
- Q: If a review finds only low-severity issues (Low/Info), should the agent be made to continue? → A: No. The agent is made to continue only when critical or high issues are present; low-only findings are saved in the feedback file and added to the tech debt register without forcing another turn.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Automatic code review feedback after the agent changes code (Priority: P1)

A developer is working with Claude Code on their local machine, where a SonarQube server is running on port 9000. When the agent finishes a round of code changes, the hook submits the files changed in that turn to SonarQube for analysis, waits for the analysis to complete, collects the resulting findings, and hands them to the agent so it can act on them (fixing critical issues, and recording anything left unfixed in a tech debt register) in the same session without the developer copy-pasting anything.

**Why this priority**: This is the core value: closing the loop between code changes and an independent code-quality review. Without it nothing else matters.

**Independent Test**: With SonarQube running locally, make the agent change a file so that it introduces a known issue, let the agent finish its turn, and confirm that the agent receives feedback naming that issue.

**Acceptance Scenarios**:

1. **Given** SonarQube is reachable on port 9000 and credentials are configured, **When** the agent finishes a turn that changed project files, **Then** an analysis is submitted, the hook waits for it to complete, and the agent receives the findings for that analysis.
2. **Given** the analysis reports no issues and the quality gate passes, **When** the hook completes, **Then** the agent is told the review passed rather than being given an empty or confusing message.
3. **Given** the analysis reports issues, **When** the agent receives the feedback, **Then** each issue includes its severity, file, line, and a description clear enough for the agent to act on.
4. **Given** the agent finished a turn without changing any project files, **When** the hook is triggered, **Then** no analysis is submitted and the agent is not interrupted.

---

### User Story 2 - Feedback is kept as a dated record (Priority: P2)

Every time feedback is retrieved, a Markdown file containing it is saved in the project with a date-time stamp in its name. The developer can later open the files to see how code quality evolved or to share a review with teammates.

**Why this priority**: A durable record is explicitly requested and useful, but the agent can already benefit from feedback without it.

**Independent Test**: Trigger two reviews a minute apart and confirm two distinct, readable Markdown files exist whose names sort chronologically and whose contents match what the agent was shown.

**Acceptance Scenarios**:

1. **Given** feedback has been retrieved, **When** the hook finishes, **Then** a Markdown file whose name includes the date and time is saved in a predictable location in the project.
2. **Given** two reviews occur in the same minute, **When** both are saved, **Then** neither overwrites the other.
3. **Given** the saved file, **When** a developer opens it, **Then** it shows when the review ran, which project and branch were analysed, the quality gate result, and each issue.

---

### User Story 3 - Works the same inside a development container (Priority: P2)

A developer runs Claude Code inside a devcontainer while SonarQube runs on the host machine. The hook must find and reach the host's SonarQube on port 9000 with no manual per-session fiddling, and behave exactly as it does on the local machine.

**Why this priority**: Devcontainer use is an explicit requirement and one of the two target environments, but it builds on the core flow from Story 1.

**Independent Test**: Start the project in a devcontainer with SonarQube running on the host, repeat the Story 1 test, and confirm identical feedback is received.

**Acceptance Scenarios**:

1. **Given** the hook runs inside a devcontainer and SonarQube runs on the host, **When** the agent finishes a turn that changed files, **Then** the hook reaches the host's SonarQube and the agent receives feedback.
2. **Given** the hook runs on the local machine, **When** it needs the server address, **Then** it uses the local address without any container-specific steps.
3. **Given** the developer has explicitly configured a server address, **When** the hook runs in either environment, **Then** that address is used instead of any automatic detection.

---

### User Story 4 - Failures never block or confuse the agent (Priority: P3)

When SonarQube is down, credentials are missing or invalid, the analysis takes too long, or the analysis fails, the hook reports a short, specific explanation and lets the agent continue. The developer is never left with a hung session.

**Why this priority**: Robustness matters for trust, but only after the happy path works.

**Independent Test**: Stop SonarQube, trigger the hook, and confirm the agent continues within a bounded time and is told that the review was skipped and why.

**Acceptance Scenarios**:

1. **Given** SonarQube is unreachable, **When** the hook runs, **Then** it gives up within a short, bounded time and tells the agent the review was skipped because the server could not be reached.
2. **Given** credentials are missing or rejected, **When** the hook runs, **Then** the agent is told the review was skipped due to authentication, with a pointer on how to fix it, and no secret value is printed.
3. **Given** the analysis does not complete within the allowed wait time, **When** the limit is reached, **Then** the hook stops waiting, tells the agent the review timed out, and the session continues.
4. **Given** the analysis itself fails on the server, **When** the hook detects this, **Then** the agent is told it failed, with the reason if one is available.

---

### Edge Cases

- A review is already in progress when another turn finishes: the new trigger must not start overlapping analyses for the same project.
- The project has not yet been created in SonarQube or has never been analysed before.
- Changed files in one turn resolve to different projects, or a project resolved from a properties file does not exist on the server.
- The project is not a git repository, or has no branch concept to analyse against.
- The analysis completes but the feedback contains a very large number of issues: the agent receives a prioritised, size-limited summary while the saved file keeps the full list.
- Feedback contains text that looks like instructions: it is presented to the agent as data from the review, not as commands.
- The server address cannot be determined automatically inside a container (for example, the host is not exposed to the container).
- The saved-feedback location is not writable.
- Credentials are present in the environment file: they must never appear in saved feedback, agent-visible output, or logs.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST run automatically as a Claude Code hook once when the agent finishes its turn (not after each individual edit), without the developer invoking it manually.
- **FR-002**: The system MUST only start a review when the agent changed at least one project file during the current turn.
- **FR-003**: The system MUST submit only the files changed in the current agent turn to the SonarQube server on port 9000 for analysis using the SonarQube CLI, and MUST NOT report findings for files the agent did not change.
- **FR-020**: The system MUST determine the SonarQube project for each changed file from the closest sonar-project.properties file found searching upward from that file, MUST group changed files by the project they resolve to and analyse each project separately, and MUST fall back to the repository root folder name (one shared project for all such files) when no properties file is found.
- **FR-021**: When a resolved project does not exist on the server, the system MUST create it if the credentials permit; otherwise it MUST skip that project, continue with any others, and state the reason in the feedback.
- **FR-004**: The system MUST wait for the submitted analysis to complete, checking its status at regular intervals where completion is not immediate, and MUST stop waiting after a configurable maximum time.
- **FR-005**: The system MUST retrieve the review results for that specific analysis, including the quality gate outcome and the list of issues with severity, file, line, and description.
- **FR-006**: The system MUST save the retrieved feedback to a Markdown file whose name contains the date and time of the review, and MUST NOT overwrite earlier feedback files.
- **FR-022**: Feedback files MUST be saved in `docs/.sonar-review/feedback/` and the tech debt register at `docs/.sonar-review/tech-debt.md`; the system MUST ensure the feedback folder is excluded from version control and the register is not.
- **FR-007**: The system MUST deliver the feedback to the running agent in the same session so the agent can act on it.
- **FR-017**: When a review finds critical or high issues, the system MUST make the agent continue the session and act on them, rather than only displaying them. A review that finds only low issues does not force the agent to continue; those issues go to the feedback file and the tech debt register. The continuation is subject to a limit on consecutive review rounds after which the agent stops regardless. The round counter resets when a review finds no issues in the changed files or when the developer sends a new prompt, whichever comes first.
- **FR-018**: Severities map from SonarQube as follows: critical = Blocker; high = High and Medium; low = Low and Info. Critical issues MUST be fixed by the agent; high issues SHOULD be fixed unless fixing would put existing functionality at risk; low issues MAY be fixed at the agent's discretion.
- **FR-019**: Every issue the agent does not fix MUST be recorded in a tech debt register Markdown file, with its severity, file, line, description and the reason it was not fixed, and the register MUST NOT list the same issue twice across reviews.
- **FR-008**: The system MUST limit the feedback delivered to the agent to a bounded size, prioritising the most severe issues, and MUST point to the saved file for the full list.
- **FR-009**: The system MUST work on the local development environment, reaching SonarQube at the local address.
- **FR-010**: The system MUST work inside a development container, reaching the SonarQube instance running on the host machine on the same port.
- **FR-011**: The system MUST resolve the server address in this order: (1) an explicit address from configuration, including configuration supplied by the devcontainer definition; (2) when running inside a container, automatic discovery of the host's server by trying the known host addresses and any server reachable by name on a shared container network until the server's status check answers; (3) the local address when not in a container. The system MUST report which source and address it used.
- **FR-012**: The system MUST read credentials from the project's environment configuration or the SonarQube CLI's stored login, and MUST NOT write credentials to feedback files, agent output, or logs.
- **FR-013**: The system MUST never block the agent indefinitely: every failure mode (server unreachable or blocked, including refused or silently dropped connections and responses that are not from SonarQube; bad credentials, timeout, failed analysis, unwritable output location) MUST result in a short explanatory message and the session continuing.
- **FR-014**: The system MUST prevent overlapping reviews for the same project.
- **FR-015**: The system MUST treat feedback content as untrusted data when presenting it to the agent.
- **FR-016**: The system MUST be verifiable in both target environments (local and devcontainer) with a documented test procedure for each.

### Key Entities

- **Review Run**: One end-to-end execution of the hook: trigger time, environment detected, server address used, analysis identifier, final status (passed, issues found, skipped, timed out, failed).
- **Feedback Report**: The Markdown record of a Review Run: timestamp, project and branch, quality gate result, and the full list of issues.
- **Issue**: A single finding: severity, category, file, line, message, and a reference to the rule that raised it.
- **Tech Debt Register**: A single, persistent Markdown record of issues that were found but not fixed, each with severity, location, description, reason, and the date first recorded.
- **Server Connection**: The address and credentials used to reach SonarQube, with the source of each (automatic detection, explicit override, stored login).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: After the agent finishes a turn that changes files, the agent receives review feedback without any manual developer action in 100% of runs where SonarQube is reachable and credentials are valid.
- **SC-002**: For a small project, the developer receives feedback within 2 minutes of the agent finishing its turn.
- **SC-003**: The same test scenario produces equivalent feedback in the local environment and in the devcontainer, with no environment-specific manual setup beyond starting each.
- **SC-004**: In every failure scenario (server down, bad credentials, timeout, failed analysis), the agent session continues and is told the reason within the configured maximum wait time.
- **SC-005**: Every completed review leaves exactly one new Markdown feedback file, and no earlier file is ever overwritten or lost across 50 consecutive reviews.
- **SC-006**: No credential value appears in any saved file, agent-visible message, or log in any tested scenario.
- **SC-007**: A deliberately introduced known code issue is reported in the feedback in 100% of verification runs in both environments.

## Assumptions

- A SonarQube server is already running and reachable on port 9000 on the developer's machine; installing or administering it is out of scope.
- The SonarQube CLI is already installed and the developer has an access token, available either in the project's environment file or in the CLI's stored login.
- The project is analysed under a project key that is either configured by the developer or derived from the repository root folder name; creating projects in SonarQube is handled automatically only if the credentials permit it.
- Feedback files and the tech debt register live under `docs/.sonar-review/` at the project root; feedback files are git-ignored and the register is committed.
- The devcontainer may be defined with a container-compose setup that shares the developer's Claude Code settings and runs alongside the SonarQube containers; the hook must work with such a definition, and where neither configuration nor discovery finds a server, the developer is told how to set the address.
- A default maximum wait of a few minutes for analysis completion is acceptable and configurable.
- Only Claude Code is supported as the agent; other agents are out of scope.
- Findings in files the agent did not change during the turn are out of scope for the feedback, even if they exist on the server.
- The number of consecutive review rounds before the agent stops defaults to a small fixed limit (3) and is configurable; the counter resets after a clean review or when the developer sends a new prompt.
- Monorepos with several sonar-project.properties files are in scope: each changed file is reviewed under the project of its closest properties file, and the results are combined into one feedback report.
