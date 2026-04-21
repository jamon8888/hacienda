---
name: audit
description: Show the per-session redaction audit log — which placeholders were generated, which outbound tools were called, what went to the cloud and what came back. Use when the user wants to verify nothing sensitive leaked in this session, or to prepare evidence for a compliance review.
---

# /audit — Per-session redaction report

```
/audit
```

## Workflow

1. Call `mcp__hacienda__session_audit_read(session_id=<cowork_session_id>)`.
2. Summarise:

```
Session <id>

Queries:     <n>
Anonymize:   <n>   (<total_entities> entities redacted)
Rehydrate:   <n>   (<total_tokens> tokens restored)
Outbound:    <n>   (tool × count)

Redaction summary (label × count):
  PER  <n>
  ORG  <n>
  IBAN <n>
  ...

Last event: <timestamp> — <event>
```

3. Offer: *"Show the full event list?"* — if yes, dump the JSONL events one per line, pretty-printed.

## Safety

This report MUST NOT contain raw PII. The audit log stores placeholders, vault tokens, and counts only. If you see any raw value in a payload field, that is a bug — report it as *"Audit corruption detected, contact the plugin author"* and refuse to continue.

## Compliance note

This log is append-only and lives at `~/.hacienda/sessions/<session_id>.audit.jsonl` on the user's device. It is never transmitted off-device by the plugin. Retention: the user may delete files in `~/.hacienda/sessions/` at any time. No cloud copy exists.
