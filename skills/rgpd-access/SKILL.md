---
name: rgpd-access
description: Right-of-access (RGPD Art. 15) workflow. Given a person's name (or email/phone), find every PII token referring to them across the project, build an Art. 15 report listing all documents, excerpts, categories, purposes and legal bases. Use when a data subject (client, salarié, contact) requests their personal data per Art. 15 RGPD.
argument-hint: "<person name or identifier>"
---

# /hacienda:rgpd:access — Demande d'accès RGPD Art. 15

```
/hacienda:rgpd:access Marie Dupont
```

## Workflow

### Step 1 — Resolve project

Call `mcp__piighost__resolve_project_for_folder(folder=<active>)` to get the project slug.

### Step 2 — Cluster the subject

Call `mcp__piighost__cluster_subjects(query="<arg>", project=<project>)`. Returns `{"clusters": [...]}` where each cluster is a candidate group of tokens (nom_personne + email + phone + IBAN + …) that probably refer to the same real person.

If `clusters` is empty: tell the user *"Aucune trace de '<query>' dans ce dossier."* Stop.

If multiple clusters: present them to the user in a numbered list:
```
1. Marie Dupont (confidence 0.95) — 5 docs, 12 tokens — first seen 2024-03-15
2. Marie Dupont autre (confidence 0.74) — 1 doc, 2 tokens — first seen 2025-08-01
```
Ask which cluster (or "all") to apply.

If single cluster with confidence ≥ 0.85: proceed automatically with confirmation.

### Step 3 — Subject access

Call `mcp__piighost__subject_access(tokens=<cluster.tokens>, project=<project>)`. Returns a `SubjectAccessReport` dict with:
- `subject_tokens`, `subject_preview`
- `categories_found` (label → count)
- `documents` (list of `SubjectDocumentRef`: doc_id, file_name, doc_type, doc_date, occurrences)
- `processing_purpose`, `legal_basis`, `retention_period`
- `excerpts` (redacted, with cluster tokens replaced by `<<SUBJECT>>`)

### Step 4 — Render the response

Format the report as a Markdown document for the avocat to send to the data subject:

```markdown
# Réponse à votre demande d'accès (Art. 15 RGPD)

**Sujet** : <subject_preview joined>
**Date du rapport** : <generated_at as ISO>
**Cabinet** : <controller name from profile>

## Catégories de données traitées
- nom_personne : <count>
- email : <count>
- ...

## Documents concernés (<n_docs>)
| Document | Type | Date | Occurrences |
|---|---|---|---|
| ... | contrat | 2024-04-15 | 3 |

## Finalités du traitement
<processing_purpose>

## Base légale
<legal_basis>

## Durée de conservation
<retention_period>

## Extraits redactés (<total_excerpts>)
> [excerpt 1, with <<SUBJECT>> placeholders preserved]
> ...
```

### Step 5 — Save to file

Suggest saving the rendered Markdown to `<folder>/rgpd-access-<date>-<subject>.md` so the avocat has a record. Use the standard `Write` tool with a sanitised filename.

## Refusals & edge cases

- If the cluster confidence is < 0.5, warn the user and ask for confirmation before proceeding — false positives could include data of OTHER people.
- The placeholder `<<SUBJECT>>` in excerpts is the cluster's tokens. Other PII placeholders (`<<PER_001>>`, `<<EMAIL_002>>`) belong to OTHER people and stay in the output as-is — never rehydrate them.
- Audit event `subject_access` is recorded automatically by the server. To verify, run `/hacienda:audit`.
