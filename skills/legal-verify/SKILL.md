---
name: legal-verify
description: >-
  Verifier les references juridiques francaises (articles de codes, lois,
  decrets, jurisprudences) contre les sources officielles via OpenLegi.
  Detecte les hallucinations (references inexistantes), numeros errones,
  abrogations ignorees, jurisprudences fictives. Trois modes d'entree:
  texte colle, fichier sur disque, document indexe. Genere un rapport JSON
  et Markdown.
argument-hint: "[--doc-id <id> | --project <name> | --file <path>]"
---

# /hacienda:legal:verify — Vérifier les citations juridiques

```
/hacienda:legal:verify              (texte collé dans le prompt)
/hacienda:legal:verify --doc-id abc123
/hacienda:legal:verify --project dossier-acme-2026
/hacienda:legal:verify --file /chemin/vers/document.pdf
```

## Pre-flight

Vérifier que OpenLégi est activé via `mcp__piighost__controller_profile_get(scope="global")`. Si `openlegi.configured = false`, refuser avec "Activez OpenLégi via /hacienda:legal:setup."

## Workflow par mode

### Mode 1 : texte collé

1. Récupérer le texte du prompt utilisateur.
2. `mcp__piighost__extract_legal_refs(text=<texte>)` → liste de LegalReference
3. Pour chaque référence : `mcp__piighost__verify_legal_ref(ref=<r>)` → VerificationResult
4. Agréger en rapport.

### Mode 2 : --doc-id <id>

1. Récupérer le contenu textuel via les chunks indexés.
2. Suite identique à Mode 1.

### Mode 3 : --project <name>

1. Lister tous les documents du projet.
2. Pour chaque doc, mode 2.
3. Agréger les résultats au niveau projet.

### Mode 4 : --file <path>

1. Lire le fichier (avec kreuzberg si binaire).
2. Suite identique à Mode 1.

## Rapport

Structure JSON :
```json
{
  "metadata": {"document": "…", "date": "ISO-8601", "score_global": 0-100},
  "synthese": {"total": N, "exactes": N, "erreurs": N, "hallucinations": N},
  "details": [
    {
      "reference_originale": "article 1240 du Code civil",
      "statut": "VERIFIE_EXACT|HALLUCINATION|…",
      "score": 0-100,
      "type_erreur": null|"REF_INEXISTANTE"|"NUM_ERRONE"|...,
      "url_legifrance": "https://...",
      "correction": null|"texte de correction"
    }
  ]
}
```

(Optionnel) Sauvegarder via `mcp__piighost__render_compliance_doc`.

## Refusals

- Si OpenLégi désactivé : redirection vers `/hacienda:legal:setup`.
- Si aucune référence extraite : afficher "Aucune référence juridique détectée dans l'entrée."
- Pour `--doc-id`/`--project` : si OpenLégi non configuré, refuser proprement.
