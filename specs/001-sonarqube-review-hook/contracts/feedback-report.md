# Contract: Feedback report

Location: `docs/.sonar-review/feedback/<YYYYMMDD-HHMMSS>.md`, with `-2`, `-3` and so on added if that name exists. Written once per completed review, never overwritten. Git-ignored.

```markdown
# SonarQube review: <YYYY-MM-DD HH:MM:SS UTC>

- Project(s): <key>, <key>
- Branch: <git branch or "n/a">
- Server: <url> (source: config | discovery:<candidate> | default)
- Round: <n> of <max>
- Quality gate: <OK | ERROR | not available>
- Outcome: <clean | issues | skipped | timed-out | failed>

## Skipped or failed projects
- <key>: <reason>          (section omitted when empty)

## Issues
| Tier | Severity | File | Line | Rule | Message | Issue key |
|------|----------|------|------|------|---------|-----------|
| critical | BLOCKER | src/a.sh | 12 | shell:S1234 | ... | AX... |
```

**Rules**: lists every issue found, including those cut from the agent message; contains no credentials; messages are sanitised the same way as in the agent message.
