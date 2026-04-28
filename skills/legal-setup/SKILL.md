---
name: legal-setup
description: Configure or rotate the OpenLégi (Legifrance) integration. Walks the user through obtaining a PISTE token from https://piste.gouv.fr and writes it securely to ~/.piighost/credentials.toml. Use to enable, disable, or rotate the integration outside of the main /hacienda:setup wizard.
---

# /hacienda:legal:setup — Configurer OpenLégi

```
/hacienda:legal:setup
/hacienda:legal:setup --disable
```

## Workflow

### Mode: enable / rotate

1. Vérifier l'état actuel via `mcp__piighost__controller_profile_get(scope="global")`. Si `openlegi.configured = true`, demander à l'utilisateur s'il veut faire une rotation du token ou désactiver.

2. Expliquer la procédure :
   - Aller sur https://piste.gouv.fr
   - Créer un compte et demander l'accès à l'application Legifrance
   - Récupérer la clé d'API (token PISTE)

3. Demander : "Collez votre token PISTE ici (il sera écrit dans ~/.piighost/credentials.toml avec des permissions strictes — chmod 600 sur Linux/Mac) :"

4. Capturer la réponse, appeler `mcp__piighost__legal_credentials_set(token=<saisi>)`.

5. Confirmer avec un test ping :
   ```
   mcp__piighost__search_legal(query="test", source="code", max_results=1)
   ```

   Si le résultat est `[]` ou contient une erreur d'auth, indiquer "Token invalide. Réessayez."

6. Afficher : "✅ OpenLégi activé. Vous pouvez maintenant utiliser /hacienda:legal:verify et /hacienda:search."

### Mode: --disable

1. Appeler `mcp__piighost__legal_credentials_set(token="")` (efface le token).
2. Indiquer à l'utilisateur de mettre `[openlegi] enabled = false` dans son `controller.toml` s'il veut désactiver complètement.

## Refusals

- Si le token est vide → ne rien écrire, prévenir l'utilisateur.
- Ne JAMAIS afficher le token dans la conversation après l'avoir écrit.
