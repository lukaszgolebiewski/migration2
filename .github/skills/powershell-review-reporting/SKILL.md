# Review Reporting Skill

## Purpose
Generate consistent, structured reports for modernization and security review outputs. Ensures all agent outputs follow the same format for comparability and tracking.

If the caller is `powershell-modernizer`, generate a **Modernization Report** and use prefix `MOD-NNN`. If the caller is `powershell-security-reviewer`, generate a **Security Assessment** and use prefix `SEC-NNN`. If the caller is `powershell-debugger`, generate a **Debug Analysis** and use prefix `DBG-NNN`. Do not invent a report type.

If the provided report type is not one of Modernization Report, Security Assessment, or Debug Analysis, do not generate a report; return a brief error asking the caller to select one of the supported types.

## When to Use
- At the end of a modernization pass (Modernization Report)
- At the end of a security review (Security Assessment)
- At the end of a debug analysis (Debug Analysis)

## Report Template

### Header
`{Report Type}` is one of: `Modernization Report`, `Security Assessment`, `Debug Analysis`.
```markdown
# {Report Type}
**Scripts:** {file path(s) reviewed, comma-separated}
**Date:** {YYYY-MM-DD}
**Agent:** {agent name}
**Scope:** {1 sentence describing the files reviewed and the main change or finding area}
```
If no file path is provided, write `Unknown path` in the Scripts field. If the path list is invalid, do not fabricate paths.

### Summary
```markdown
## Summary
| Severity | Count |
|----------|-------|
| 🔴 Critical | N |
| 🟠 High | N |
| 🟡 Medium | N |
| 🔵 Low | N |
| **Total** | **N** |
```

### Findings
```markdown
## Findings

### {ID}: {One-line title}

Assign finding IDs in ascending order using the selected prefix (`MOD-001`, `SEC-001`, or `DBG-001`); do not reuse an ID within one report.

- **Severity:** 🔴/🟠/🟡/🔵
- **Category:** naming | parameters | error-handling | logging | security | style
- **Location:** `{file}:{line}`
- **Finding:** {one-sentence description}
- **Evidence:**
  ```powershell
  # Before (≤ 3 lines)
  ```
  If the evidence exceeds three lines, truncate to the first three lines and note `(truncated)`.
- **Recommendation:**
  ```powershell
  # After (≤ 3 lines)
  ```
  If the recommendation exceeds three lines, truncate to the first three lines and note `(truncated)`.
- **Standard:** {exact instruction source file name} — {exact section heading used for this finding}. If the instruction source is unavailable, write `Not provided`.
```

### Handoff Recommendations
```markdown
## Recommended Next Steps
- [ ] {Action item with agent reference if handoff needed}
```

## Report Types

| Type | Used By | ID Prefix |
|---|---|---|
| Modernization Report | powershell-modernizer | MOD-NNN |
| Security Assessment | powershell-security-reviewer | SEC-NNN |
| Debug Analysis | powershell-debugger | DBG-NNN |

## Inputs
- Findings list from the calling agent
- Report type (must be one of: Modernization Report, Security Assessment, Debug Analysis)
- Complete list of reviewed file paths as a list

If the findings list is empty or contains no actionable findings, produce the header and summary table with zero counts and omit the Findings section.

## Outputs
- Complete Markdown report following the template above
