---
name: search
description: Recherche fédérée sur les documents indexés localement (vault piighost) ET sur les sources officielles françaises via OpenLégi (Code civil, jurisprudence, CNIL, JORF, conventions collectives). Retourne une liste classée mêlant les hits LOCAL et LEGAL avec attribution explicite de la source. Auto-annote les références juridiques trouvées dans les documents locaux.
argument-hint: "[--local | --legal | --both] <query>"
---

# /hacienda:search — Recherche fédérée

```
/hacienda:search "responsabilité contractuelle"
/hacienda:search --local "facture acme 2026"
/hacienda:search --legal "article 1240"
```

## Workflow

1. Résoudre le projet actif via `mcp__piighost__resolve_project_for_folder(<folder>)` si Cowork.

2. Selon le scope :
   - `--local` ou par défaut : `mcp__piighost__query(text=<q>, k=10, project=<p>)`
   - `--legal` ou par défaut : `mcp__piighost__search_legal(query=<q>, source="auto", max_results=5)`
   - `--both` (default) : les deux en parallèle.

3. Fusion + classement :
   - Les hits LOCAL en premier (score piighost)
   - Annotation : si le texte d'un hit LOCAL contient une référence juridique extractible, appeler `mcp__piighost__extract_legal_refs(text=<chunk>)` et insérer le résultat de `mcp__piighost__verify_legal_ref(ref=<r>)` directement sous le hit
   - Les hits LEGAL ensuite, par pertinence

4. Afficher avec attribution :
   ```
   [LOCAL]  client_acme/contrat.pdf  p3   "...considérant l'article 1240..."
   ↳ [CODE] Code civil, Art. 1240        "Tout fait quelconque de l'homme..."
   [LOCAL]  client_acme/correspondance.txt p1 "..."
   [LEGAL] Cass. civ. 1re, 15 mars 2023, n°21-12.345
   ```

## Refusals

- Si OpenLégi désactivé et l'utilisateur a passé `--legal` : suggérer `/hacienda:legal:setup`.
- Si aucun projet local n'a été indexé et `--local` : suggérer `/hacienda:index`.
