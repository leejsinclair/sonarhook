# Quickstart: validating the SonarQube review hook

These procedures prove the feature end-to-end in both target environments (FR-016). The automated suite covers the logic with fake servers; this guide covers the real thing.

## Prerequisites

- SonarQube running on the host, port 9000: `curl -s http://localhost:9000/api/system/status` returns `"status":"UP"`
- `sonar` CLI installed and logged in (`sonar auth status`), or `SONAR_TOKEN` in `.env`
- `jq` and `curl` installed
- This repository's hooks installed into a throwaway test project: `./install.sh /path/to/test-project`
- A test project containing a file with a known issue, for example a shell script with an unquoted variable in `rm -rf $dir/*`

## Scenario A: local environment

1. In the test project, start Claude Code and ask it to create or edit the file with the known issue.
2. When the agent finishes, expect a continuation from the hook containing the issue with its severity, file and line (SC-007), within about 2 minutes on a small project (SC-002).
3. Expect `docs/.sonar-review/feedback/<timestamp>.md` to exist and match what the agent was shown.
4. Let the agent fix it; expect the next round to report clean and the round counter to reset.
5. Ask the agent to leave a low issue unfixed; expect a row in `docs/.sonar-review/tech-debt.md`.

## Scenario B: devcontainer

1. Open the test project in a devcontainer (see `examples/devcontainer/`) with SonarQube still running on the host.
2. Repeat scenario A steps 1–3 without setting any address; expect the report to state `source: discovery:<candidate>`.
3. Set `SONAR_REVIEW_HOST_URL` in `remoteEnv`, rebuild, and expect `source: config`.
4. Expect the same issue and the same quality gate result as scenario A (SC-003).

## Scenario C: failure behaviour

| Do this | Expect |
|---------|--------|
| Stop SonarQube | Within the probe time the agent is told the review was skipped because the server could not be reached; the session continues |
| Set `SONAR_TOKEN=bad` | Skipped for authentication; the token string appears nowhere in output or files |
| Set `SONAR_REVIEW_MAX_WAIT=1` | Timed-out message, session continues |

## Known limitation

Files changed through the Bash tool (for example `sed -i`) are not detected as agent edits.

## Automated suite

Once bats and a `make test` target exist (first implementation task), run `make test`. The declared command is then recorded in the plan's Verification Plan.
