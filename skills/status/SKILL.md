---
name: status
description: Show the index state of the folder currently open in Cowork — how many files are indexed, when the last update happened, any errors encountered. Use when the user wants a health check before asking questions, or when troubleshooting unexpected retrieval results.
---

# /status — Folder health

```
/status
```

## Workflow

1. Call `mcp__piighost__resolve_project_for_folder(folder=<active>)` → `folder, project`.
2. Read `piighost://folders/{b64_path}/status`.
3. Render:

```
Folder:       <absolute path>
Project:      <project hash>
State:        <indexed|empty>
Indexed docs: <total_docs>
Chunks:       <total_chunks>
Last update:  <last_indexed_at ISO 8601> (or "never" when null)
Errors:       <total_errors>
  - <errors[0].file_name> — <errors[0].category> (<relative time from indexed_at>)
  - <errors[1].file_name> — <errors[1].category> (<relative time>)
  ... up to 5 lines, then "(and <len(errors) - 5> more)" if applicable
  ↳ Showing 50 most recent of <total_errors>. Run /index to refresh.
```

Render rules for the Errors block:
- Show the first 5 entries from `errors` as bullet points.
- After 5 entries, if `len(errors) > 5`, append `... (and <len(errors) - 5> more)`.
- If `errors_truncated` is true, append the `↳ Showing 50 most recent of <total_errors>. Run /index to refresh.` footer line.
- If `total_errors == 0`, omit the bullet list entirely (just show `Errors: 0`).
- Compute the relative time (`3 days ago`) from `indexed_at` (unix epoch seconds) in the skill — the server returns the raw integer.
- The `category` value is one of: `password_protected`, `corrupt`, `unsupported_format`, `timeout`, `other`.

4. If `state == "empty"` and `total_errors == 0`, suggest `/index` to the user.
5. If `state == "empty"` and `total_errors > 0`, surface the errors and suggest `/index full` to retry the failed files (after the user has fixed whatever caused them).
6. If `last_indexed_at` is older than 10 minutes on a network drive, suggest `/index incremental`.
