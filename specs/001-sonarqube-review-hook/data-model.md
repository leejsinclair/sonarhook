# Data Model: SonarQube Review Hook

All data is held in plain files. Paths are relative to the target project root. `OUT` means `docs/.sonar-review`.

## Review Run (in memory, summarised in the report header)

| Field | Notes |
|-------|-------|
| `started_at` | ISO-8601 UTC timestamp; the report file name uses it as `YYYYMMDD-HHMMSS` |
| `session_id` | From the hook input |
| `environment` | `local` or `container` |
| `server_url`, `server_source` | `config`, `discovery:<candidate>`, or `default` |
| `round` | Consecutive review round, 1-based |
| `projects[]` | One entry per resolved project: key, files, `outcome` |
| `outcome` | `clean`, `issues`, `skipped`, `timed-out`, `failed` |

State transitions: `started → (skipped | timed-out | failed | clean | issues)`. `issues` leads to `block` when any critical or high issue exists, otherwise to `allow-stop`.

## Session State (`OUT/.state/<session_id>.json`, git-ignored)

| Field | Rule |
|-------|------|
| `touched[]` | Absolute paths; added by the record hook; de-duplicated; cleared by reset and after each review |
| `round` | Integer; incremented by each review that finds issues; set to 0 by a clean review or by the reset hook |

## Project Resolution

| Field | Rule |
|-------|------|
| `key` | `sonar.projectKey` from the closest `sonar-project.properties`, else the folder name of that properties file, else the repository root folder name |
| `files[]` | Changed files that resolve to this key, excluding anything under `OUT/` |
| `lock` | `OUT/.state/locks/<key>.lock/` directory, holds the owner pid and start time |

## Issue

| Field | Notes |
|-------|-------|
| `key` | SonarQube issue key; the identity used for de-duplication |
| `severity_raw` | As reported by the server |
| `tier` | `critical`, `high` or `low`, from the mapping in research R7 |
| `rule`, `file`, `line`, `message` | From the server; `message` is sanitised before delivery |

## Feedback Report (`OUT/feedback/<YYYYMMDD-HHMMSS>[-n].md`, git-ignored)

One file per completed review. If the name already exists, a numeric suffix is added, so a file is never overwritten. Layout in [contracts/feedback-report.md](contracts/feedback-report.md).

## Tech Debt Register (`OUT/tech-debt.md`, committed)

One table row per unfixed issue, keyed by issue key. Layout and rules in [contracts/tech-debt-register.md](contracts/tech-debt-register.md).

## Server Connection

| Field | Notes |
|-------|-------|
| `url` | Resolved base URL |
| `source` | How `url` was chosen |
| `token` | Never persisted or logged; held only in the process environment |
