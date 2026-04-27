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
2. Call `mcp__piighost__bootstrap_client_folder(folder=<active>)` → `project`. This is idempotent on re-run and already resolves the project hash, so no separate `resolve_project_for_folder` call is needed.
3. Call `mcp__piighost__index_path(path=<active>, recursive=true, force=<from step 1>, project=<project>)`.
4. Poll `mcp__piighost__folder_status(folder=<active>)` every few seconds. Stream progress to the user: *"Indexing: 134 / 247 files …"*. The tool returns `{folder, project, state, total_docs, total_chunks, last_indexed_at, errors, errors_truncated, total_errors}`.
5. When `state == "indexed"`, report: *"Indexed 247 files, 1 823 chunks. Ready."* (file-level error reporting is not yet exposed by the v0 status resource — if `index_path` returned a non-empty `errors` list in step 3, surface that instead.)
6. If `index_path` reported errors in step 3, show the list and suggest re-running `/index full` on the affected files.

## Errors

- If the folder does not exist, tell the user and stop.
- If indexing fails mid-way, surface the error from the `index_path` return value; do not retry automatically — the user may need to fix a corrupt file first.
