---
name: knowledge-base
description: Search and answer questions from the user's current Cowork folder using PII-safe hybrid retrieval (BM25 + semantic vectors, cross-encoder reranker). Auto-refreshes the index incrementally before every retrieval, so files added/modified/removed since the last question are picked up without the user invoking /index. Use whenever the user asks about documents, emails, contracts, notes, invoices, or any content in the folder Cowork is currently pointed at. Always cites sources with file paths and excerpts. Placeholders like «PER_001» in retrieved excerpts are intentional — see the redact-outbound skill before including them in any draft sent to external tools (email, Slack, webfetch).
---

# knowledge-base — PII-safe retrieval over the current folder

This skill powers the `/ask` slash command and is implicitly used whenever the user asks about their files. It wraps the `piighost` MCP server and is the only sanctioned way to read folder contents.

## Why this exists

The user is bound by professional secrecy (*secret professionnel*, GDPR Art. 32, equivalent). Raw folder contents cannot be read into the model context unredacted and then paraphrased outbound — every outbound request would leak identifiable client data.

Piighost has already solved this: every retrieval tool returns text where PII has been replaced with opaque placeholders (`«PER_001»`, `«IBAN_003»`, `«ORG_014»`). The vault stores ciphertext; the plugin rehydrates placeholders only for display, never for the model's working context.

**Your job in this skill:** call the right MCP tools, never read raw files via the `Read` tool for content in a Cowork-shared folder, and always cite sources.

## Workflow

### Step 1 — Resolve the active folder

Cowork tells you the active folder path. Call:

```
mcp__piighost__resolve_project_for_folder(folder=<abs_path>)
```

Returns `{"folder": ..., "project": "<slug>-<hash8>"}`. Use `project` in every subsequent call so cross-folder retrieval is impossible.

### Step 2 — Bootstrap (idempotent)

On the first question of a session, call:

```
mcp__piighost__bootstrap_client_folder(folder=<abs_path>)
```

This is cheap on re-run. It ensures the data dir, vault key, and project exist.

### Step 3 — Refresh the index (auto-incremental)

Always run an incremental index before answering, so any file the user
just added/modified/deleted in the folder is picked up without them
having to invoke `/index` first:

```
mcp__piighost__index_path(
  path=<abs_path>,
  project=<project>,
  recursive=true,
  force=false,
)
```

This is **incremental and cheap when nothing changed** (~10ms for a
folder of 14 files where every file's mtime+size matches the index).
Only files that are genuinely new or modified pay the extraction +
embedding cost. The return value tells you what happened:
`{indexed, modified, deleted, unchanged, errors, duration_ms}`.

- If `indexed > 0` or `modified > 0` and `duration_ms > 1500`, briefly
  tell the user *"Indexed N new files first…"* before continuing — this
  avoids surprising them when a question takes longer than usual.
- If `errors` is non-empty, surface the list and continue with retrieval
  on the rest. Don't refuse the question just because one PDF is corrupt.

### Step 4 — Check index status

After the refresh, call:

```
mcp__piighost__folder_status(folder=<abs_path>)
```

The tool returns `{folder, project, state, total_docs, total_chunks, last_indexed_at, errors, errors_truncated, total_errors}`.

- `state == "empty"` and `total_docs == 0`: the folder genuinely has no
  indexable content. Tell the user *"This folder has no indexed
  documents — add files (PDF, DOCX, XLSX, TXT, CSV, …) and ask
  again."* Stop.
- `state == "indexed"`: proceed.

(`state` only emits `empty` or `indexed` — the auto-incremental Step 3
above is what makes "indexing in progress" largely invisible to the
user. Per-file errors are surfaced via the `errors` array.)

### Step 5 — Retrieve

```
mcp__piighost__query(
  text=<user question>,
  k=5,
  project=<project>,
  rerank=true,
  top_n=20,
)
```

The returned excerpts are already redacted. Quote them verbatim.

### Step 6 — Answer with citations

Every claim cites `<filename> p.<page>` (for PDFs) or `<filename>:<line-range>` (for text). If retrieval returns nothing, say so — never fabricate a citation.

### Step 7 — Record the audit event

Once per user turn, append:

```
mcp__piighost__session_audit_append(
  session_id=<project>,
  op="query",
  metadata={"question_hash": <sha256_hex_of_question>, "n_excerpts": <n>, "project": <project>},
)
```

Use the `project` name as `session_id` — that scopes the audit log to one JSONL file per folder (`~/.hacienda/sessions/<project>.audit.jsonl`), which is what the user wants for compliance review. Cowork does not expose a per-conversation ID, and a per-folder log aggregates across conversations without losing any forensic value.

`question_hash` is `hashlib.sha256(question.encode("utf-8")).hexdigest()` (lowercase hex). Keeping the hash — never the raw question — lets `/audit` report query volume without storing identifiable content.

## Outbound content

If the user asks you to draft a reply, email, Slack message, or to call `WebFetch` with folder content:

- Treat every excerpt as **already anonymised** — the placeholders are the correct content to send.
- If the user types a real name in chat (e.g. *"reply to Jean Martin"*), call `mcp__piighost__anonymize_text` on any folder-derived text you include in the draft, then merge.
- See the `redact-outbound` skill for placeholder semantics.

## Refusals

- If the user asks about a folder that is *not* the currently open Cowork folder, refuse and suggest switching folders. Never accept a folder path as a prompt argument.
- If `bootstrap_client_folder` raises an error or `query` keeps returning empty for a folder you know contains files, refuse — something is broken upstream — and tell the user to run `/status`.

## Edge cases

- **Network drive** (`Z:\`, `\\server\share`): the auto-incremental Step 3 still works but each `index_path` call walks the whole folder via SMB, which can take several seconds even when nothing changed. Acceptable for now — if it becomes painful we'll add a debounce.
- **Folder with >5 000 files**: even an unchanged-pass through `index_path` walks every entry; on a cold network drive this can spike to tens of seconds. If you notice a `duration_ms > 5000` in Step 3, mention it to the user before delivering the answer.
- **Just-added file fails to extract** (e.g. encrypted PDF): Step 3's `errors` array surfaces it. Continue answering from the rest; mention the failed file at the end so the user can re-add a clean copy.

## Never do

- Never read files in the Cowork folder with the built-in `Read` tool for content use — only for file type inspection (extension, size). All content must come through `mcp__piighost__query`.
- Never pass a `folder` argument the user typed. Only use the Cowork-declared active folder.
- Never log raw PII in `session_audit_append`. Pass hashes, placeholders, or counts.
