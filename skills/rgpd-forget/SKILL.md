---
name: rgpd-forget
description: Right-to-be-forgotten (RGPD Art. 17) workflow with tombstone. Given a person's name, identify their tokens, preview the cascade (dry-run), then on user approval purge the vault and rewrite indexed chunks. The audit log retains a 'forgotten' event with hashed token IDs only — defensible per Art. 30. Use when a data subject requests erasure per Art. 17.
argument-hint: "<person name or identifier>"
---

# /hacienda:rgpd:forget — Droit à l'oubli RGPD Art. 17

```
/hacienda:rgpd:forget Marie Dupont
```

## Workflow

### Step 1 — Resolve and cluster

Same as `rgpd-access` Steps 1-2: get the project + cluster the subject. ALWAYS show the user the cluster preview AND require explicit confirmation before any deletion.

### Step 2 — Choose the legal basis

Art. 17 has 6 sub-grounds. Ask the user which applies:
1. **a-finalité_atteinte** — finalité atteinte ou plus nécessaire
2. **b-retrait_consentement** — retrait du consentement (si base = consentement)
3. **c-opposition** — opposition légitime du data subject
4. **d-traitement_illicite** — traitement illicite
5. **e-obligation_legale** — obligation légale d'effacement
6. **f-mineur** — données collectées sur mineur dans contexte de l'offre directe

If the user can't decide, default to **c-opposition** (most common for RGPD requests).

### Step 3 — Dry-run preview

Call:
```
mcp__piighost__forget_subject(
  tokens=<cluster.tokens>,
  project=<project>,
  dry_run=True,
  legal_basis=<chosen>,
)
```

Show the user the preview:
```
⚠️ Le droit à l'oubli va affecter :
  - <chunks_to_rewrite> chunks réécrits
  - <docs_affected.length> documents touchés
  - <tokens_to_purge_hashes.length> tokens purgés du vault
  - durée estimée : <estimated_duration_ms>ms

Cette opération est IRRÉVERSIBLE. Confirmer ? (oui/non)
```

### Step 4 — Apply (only on explicit "oui")

Call the same tool with `dry_run=False`:
```
mcp__piighost__forget_subject(
  tokens=<cluster.tokens>,
  project=<project>,
  dry_run=False,
  legal_basis=<chosen>,
)
```

Display the outcome:
```
✅ Effacement appliqué :
  - <chunks_to_rewrite> chunks rewritten avec <<deleted:HASH>>
  - <docs_affected.length> documents affectés
  - <tokens_to_purge_hashes.length> tokens purgés
  - durée : <actual_duration_ms>ms

Un événement 'forgotten' a été enregistré dans l'audit log
(hashes uniquement, jamais les tokens bruts).
```

### Step 5 — Generate compliance receipt (optional)

Suggest the avocat keep a receipt for the data subject and CNIL:
```markdown
# Récépissé d'effacement (Art. 17 RGPD)

**Date** : <completed_at as ISO>
**Base légale** : Art. 17.1.<legal_basis>
**Cabinet** : <controller name>

Effacement de <tokens_to_purge_hashes.length> identifiants
attachés à votre profil dans nos systèmes :
- <chunks_to_rewrite> chunks réécrits
- <docs_affected.length> documents affectés

Les hashes des tokens purgés sont conservés dans notre journal
d'audit aux fins de preuve d'exécution (Art. 30 RGPD), sans
permettre la reconstitution de vos données.
```

## Refusals

- NEVER apply `dry_run=False` without explicit "oui" from the user. The cascade is irreversible.
- If cluster confidence < 0.7, refuse and ask for manual token list — risk of erasing data of homonyms.
- Cannot forget the controller's own data (the avocat themselves). If the subject query matches the avocat's name, refuse.
