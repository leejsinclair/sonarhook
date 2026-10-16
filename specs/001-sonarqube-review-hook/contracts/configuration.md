# Contract: Configuration

Every setting is an environment variable. Values are read from the process environment first, then from `.env` in the project root (parsed as `KEY=VALUE` lines, never executed).

| Variable | Default | Meaning |
|----------|---------|---------|
| `SONAR_REVIEW_HOST_URL` (alias `SONAR_HOST_URL`) | unset | Explicit server address; skips all detection |
| `SONAR_TOKEN` (fallback `API_TOKEN`) | unset | Access token; if absent the CLI's stored login is used |
| `SONAR_REVIEW_MAX_WAIT` | `240` | Seconds to wait for an analysis before reporting a timeout |
| `SONAR_REVIEW_POLL_INTERVAL` | `3` | Seconds between completion checks |
| `SONAR_REVIEW_PROBE_TIMEOUT` | `2` | Seconds allowed for each server reachability check |
| `SONAR_REVIEW_MAX_ROUNDS` | `3` | Consecutive review rounds before the agent is allowed to stop |
| `SONAR_REVIEW_MAX_MESSAGE_CHARS` | `6000` | Cap on the text returned to the agent |
| `SONAR_REVIEW_OUT_DIR` | `docs/.sonar-review` | Output folder, relative to the project root |
| `SONAR_REVIEW_DISABLE` | unset | Any non-empty value turns all three hooks into no-ops |

A devcontainer supplies the address through `remoteEnv` in `devcontainer.json` or the compose file, for example `"SONAR_REVIEW_HOST_URL": "http://host.docker.internal:9000"`. An example is kept in `examples/devcontainer/`.

**Rules**: the token is never printed, written to a file, or placed on a command line. Invalid numeric values fall back to the default and a note is logged.
