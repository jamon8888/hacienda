---
name: index
description: Force (re)index the folder currently open in Cowork. Use when files have changed on a network share and the status chip shows stale data, when a new folder was just opened, or when the user explicitly wants a full rescan. Indexing is async — this command starts the job and streams progress from the status resource.
argument-hint: "[full | incremental]"
---

# /index — (Re)index the current folder

```
/index              # incremental (default)
/index full         # rescan everything, re-embed every document
/index incremental  # only changed files (default)
```

## Workflow

1. Parse `$1`: `"full"` → `force=true`, anything else → `force=false`.
2. Call `mcp__hacienda__resolve_project_for_folder(folder=<active>)` → `project`.
3. Call `mcp__hacienda__bootstrap_client_folder(folder=<active>)` (idempotent).
4. Call `mcp__hacienda__index_path(path=<active>, recursive=true, force=<from step 1>, project=<project>)`.
5. Poll `hacienda://folders/{b64_path}/status` every few seconds. Stream progress to the user: *"Indexing: 134 / 247 files …"*.
6. When `state == "ready"`, report: *"Indexed 247 files, 1 823 chunks, 0 errors. Ready."*
7. If `errors` is non-empty, show the list and suggest re-running `/index full` on the affected files.

## Errors

- If the folder does not exist, tell the user and stop.
- If indexing fails mid-way, surface the error from the status resource; do not retry automatically — the user may need to fix a corrupt file first.
