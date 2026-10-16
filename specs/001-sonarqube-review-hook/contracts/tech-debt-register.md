# Contract: Tech debt register

Location: `docs/.sonar-review/tech-debt.md`. Committed to version control. Created on first use.

```markdown
# Tech debt register

Issues found by the SonarQube review that were not fixed. One row per issue key.

| Issue key | Severity | File | Line | Description | Reason not fixed | First recorded |
|-----------|----------|------|------|-------------|------------------|----------------|
| AX... | LOW | src/a.sh | 12 | ... | Style only, no functional impact | 2026-10-16 |
```

**Rules**
1. One row per SonarQube issue key; a key already present is never added again, from any review.
2. The agent adds rows for issues it deliberately skips and gives the reason. The hook adds a row for any issue still reported in the next review, for any issue left when the round limit is reached, and for every low issue found in a review that does not force a continuation, with the reason `Not fixed after review round <n>` or `Low severity, recorded without action`.
3. Existing rows are never edited or removed by the hook.
4. The register is excluded from review (design constraint DC-01), so editing it never starts a review.
