# Research: SonarQube Review Hook

All Technical Context unknowns are resolved below. Items marked **To verify** are risks to be proven by the first implementation tasks, not blockers for the design.

## R1. How a review is run through the SonarQube CLI

- **Decision**: Call `sonar analyze --file <path> ... -p <projectKey> --format json` once per project with that project's changed files. Use `sonar api get /api/ce/...` to poll for completion where the CLI returns before the server has finished, and `sonar list issues` and `sonar quality-gate status --format json` to read results.
- **Rationale**: `sonar` 1.10 supports `analyze` on specific files, a staged set or a base ref (checked with `sonar analyze --help`). It also has `api`, `list issues` and `quality-gate status`, which cover everything the spec needs without raw HTTP, and it reuses the developer's stored login.
- **Alternatives**: a classic scanner upload followed by polling the compute engine (heavier, analyses the whole project, contradicts the "changed files only" decision); raw `curl` against the web API (duplicates what the CLI does).
- **To verify (task 1 spike)**: whether `sonar analyze` returns findings directly or only after a server task finishes, and the JSON shape it prints. The polling code is written against the compute-engine task endpoints and is skipped when the CLI result is already final.

## R2. Detecting files changed in the current turn

- **Decision**: A PostToolUse hook on `Edit`, `Write`, `MultiEdit` and `NotebookEdit` appends each path to a per-session state file. The Stop hook reads and clears it. A UserPromptSubmit hook clears it and the round counter.
- **Rationale**: Works without git, matches "changed in the current turn" exactly, and costs nothing per edit.
- **Alternatives**: `git status` or `git diff` at Stop (needs git, mixes in the developer's own uncommitted work, so it would review files the agent did not change); file hash snapshots at prompt start (more code, same result).
- **Known limitation**: files changed by shell commands run through the Bash tool are not seen. Accepted for this version; recorded in the quickstart.

## R3. Making the agent act on the findings

- **Decision**: The Stop hook prints `{"decision":"block","reason":"<findings>"}` on stdout to make Claude continue. It prints nothing (exit 0) to let it stop. A short status line for skipped, timed-out or clean reviews goes in `systemMessage` so the developer and agent both see it without forcing a turn.
- **Rationale**: This is the documented Stop hook contract. The consecutive-round limit is counted by the hook itself, so the hook does not depend on `stop_hook_active`.
- **To verify**: that a `systemMessage` reaches the agent's context as intended; if not, deliver non-blocking messages through `additionalContext` on the next prompt.

## R4. Resolving the SonarQube project

- **Decision**: For each changed file, walk up directories to the nearest `sonar-project.properties` and read `sonar.projectKey`. If the file has no key, use the directory name of that file's folder. If no file is found, use the repository root folder name. Group files by key.
- **Rationale**: Matches FR-020 and the clarified fallback. The repository root is the directory holding `.git`, else Claude's working directory.
- **Project creation**: `sonar api post /api/projects/create` with name and key. A 403 or 401 means "no permission", so the project is skipped with a reason.

## R5. Reaching the host from a devcontainer

- **Decision**: Resolution order: (1) `SONAR_REVIEW_HOST_URL` or `SONAR_HOST_URL`, which a devcontainer sets through `remoteEnv` or compose; (2) if in a container, probe in turn `http://host.docker.internal:9000`, the default gateway from `/proc/net/route`, `http://172.17.0.1:9000`, and `http://sonarqube:9000` (compose service name); (3) `http://localhost:9000`. A candidate is accepted when `/api/system/status` returns `"status":"UP"` within 2 seconds.
- **Container detection**: `/.dockerenv` exists, or `REMOTE_CONTAINERS`/`CODESPACES` is set, or `/proc/1/cgroup` mentions docker or containerd.
- **Rationale**: The sample devcontainer shares the host's Docker daemon and puts the dev container on the compose network, so a service name may resolve; Docker Desktop provides `host.docker.internal`; Linux hosts need the gateway. `sonar auth status` on this machine already reports the bridge address `172.17.0.1`.

## R6. Credentials

- **Decision**: Token lookup order: `SONAR_TOKEN`, then `API_TOKEN` (the key used in this project's `.env`), both read from the environment or from `.env` in the project root; otherwise rely on the CLI's stored login. The token is passed to child processes only through the environment and never on a command line. All log output passes through a redactor that removes the token value.
- **Rationale**: Matches FR-012. The `.env` file is parsed with a tiny reader rather than sourced, so its contents cannot run as code.

## R7. Severity mapping and message size

- **Decision**: Accept both severity vocabularies (MQR: BLOCKER, HIGH, MEDIUM, LOW, INFO; standard: BLOCKER, CRITICAL, MAJOR, MINOR, INFO). Map to tiers: critical = BLOCKER; high = HIGH, MEDIUM, CRITICAL, MAJOR; low = LOW, INFO, MINOR. Cap the agent message at 6000 characters by default (`SONAR_REVIEW_MAX_MESSAGE_CHARS`), most severe first.
- **Rationale**: The server may run either mode. The cap is a plan-level default for the spec's "bounded size" and is configurable.

## R8. Tech debt register ownership

- **Decision**: The agent is told to add a register entry, with a reason, for any issue it chooses not to fix. As a backstop the hook appends an entry for any issue that is still reported in the next review (or when the round limit is reached, or the review contained only low issues), with the reason "not fixed after review round N". Entries are keyed by the SonarQube issue key, so repeated reports never create a second entry.
- **Rationale**: FR-019 needs the reason for skipping, which only the agent knows, but the register must be complete even when the agent forgets.

## R9. Concurrency

- **Decision**: One lock directory per project key under `docs/.sonar-review/.state/locks/`, created with `mkdir` (atomic). A lock older than twice the maximum wait is treated as stale and removed. A second review that cannot take the lock is skipped with a message.

## R10. Test strategy

- **Decision**: bats tests drive the real entry points with a fake `sonar` executable and a fake HTTP server placed first on `PATH`. The fakes can return success, delay, 401/403/404, failed tasks and no answer, so that they can fail as the real system can. Real-server runs are manual (see quickstart).
- **Gap**: bats, a test target and a declared test command do not exist yet. The first task must add them; the Verification Plan's command table stays empty until the developer does so.
