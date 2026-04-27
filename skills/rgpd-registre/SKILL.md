---
name: rgpd-registre
description: Generate the Art. 30 RGPD register (registre des activités de traitement) for the current Cowork folder. Calls processing_register MCP tool, optionally renders to PDF/DOCX with profession-specific templates. Use when the avocat/cabinet needs to produce or update its Art. 30 register for CNIL inspection or annual compliance review.
argument-hint: "[--format=md|pdf|docx]"
---

# /hacienda:rgpd:registre — Registre Art. 30 RGPD

```
/hacienda:rgpd:registre
/hacienda:rgpd:registre --format=pdf
```

## Workflow

### Step 1 — Resolve project + load profile

Call `mcp__piighost__resolve_project_for_folder(folder=<active>)` for the project slug.

Call `mcp__piighost__controller_profile_get(scope="project", project=<project>)` to see if the controller profile is set. If empty, suggest `/hacienda:setup` first — the registre is far less useful without controller identity.

### Step 2 — Generate the register

Call `mcp__piighost__processing_register(project=<project>)`. Returns a `ProcessingRegister` dict with:
- `controller`, `dpo`
- `data_categories` (with `sensitive` Art. 9 flag)
- `documents_summary`
- `manual_fields` (hints for the avocat to fill)
- audit event `registre_generated` written automatically

### Step 3 — Render

If user passed `--format=md` (default), call:
```
mcp__piighost__render_compliance_doc(
  data=<register>, format="md",
  profile=<from controller.profession>,
  output_path="<folder>/rgpd-registre-<date>.md",
)
```
For `pdf` or `docx`, switch the format. PDF requires the `[compliance]` extra installed.

### Step 4 — Show the result

Display the register summary:
```
📋 Registre Art. 30 généré

Cabinet : <controller.name>
Profession : <controller.profession>
Catégories de données : <data_categories.length>
Catégories sensibles (Art. 9) : <sensitive_categories_present.length>
Documents inventoriés : <documents_summary.total_docs>

À compléter manuellement :
- <manual_fields[0].field> : <manual_fields[0].hint>
- ...

Fichier généré : <output_path>
```

### Refusals

- If the controller profile is completely empty (no name, no profession), refuse and direct to `/hacienda:setup` first. A registre without controller identity has no compliance value.
- If `format=pdf` is requested but weasyprint isn't installed, surface the clear ImportError to the user with the install command.
