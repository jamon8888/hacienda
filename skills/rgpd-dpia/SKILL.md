---
name: rgpd-dpia
description: Run a DPIA-lite screening (Art. 35 RGPD) for the current Cowork folder. Detects triggers (Art. 35.3 + CNIL guidance), emits verdict (required / recommended / not required), and prepares pre-filled inputs for the official CNIL PIA software. Use when the avocat needs to assess whether a full DPIA is required for the current dossier.
---

# /hacienda:rgpd:dpia — Screening DPIA-lite Art. 35

```
/hacienda:rgpd:dpia
```

## Workflow

### Step 1 — Resolve project

`mcp__piighost__resolve_project_for_folder(folder=<active>)` → project slug.

### Step 2 — Screen

Call `mcp__piighost__dpia_screening(project=<project>)`. Returns a `DPIAScreening` dict:
- `verdict` — "dpia_required" / "dpia_recommended" / "dpia_not_required"
- `triggers` — list of Art. 35.3 + CNIL criteria detected
- `cnil_pia_inputs` — JSON ready to import into the CNIL PIA software
- `cnil_pia_url` — link to download the CNIL tool

### Step 3 — Display verdict

```
🛡️ Screening DPIA Art. 35

Verdict : {{verdict}}
{{verdict_explanation}}

Triggers détectés ({{triggers.length}}) :
- {{trigger.code}} ({{trigger.severity}}) : {{trigger.name}}
  Évidence : {{trigger.matched_evidence}}
- ...
```

### Step 4 — Render to MD (optional)

If the user wants a paper trail:
```
mcp__piighost__render_compliance_doc(
  data=<dpia>, format="md",
  profile=<from controller.profession>,
)
```
The daemon writes to `~/.piighost/exports/<project>-dpia_screening-<ts>.md` by default.
For security, `output_path` (if specified) must resolve under `~/.piighost/`.

### Step 5 — Enrichissement CNIL (optionnel)

Si OpenLégi est activé (`controller_profile_get` → `openlegi.configured = true`),
chercher les décisions CNIL pertinentes par rapport au verdict :

```
hits = mcp__piighost__search_legal(
    query=<verdict_explanation + premier trigger.name>,
    source="cnil",
    max_results=3,
)
```

Afficher chaque hit en complément du verdict :
```
📋 Décisions CNIL pertinentes
- {{hit.title}} — {{hit.url}}
```

Ne pas faire échouer la skill si l'appel échoue ou retourne `[]` ;
c'est un enrichissement, pas une exigence.

### Step 6 — Outil CNIL PIA (si DPIA requise)

If verdict is `dpia_required` or `dpia_recommended`, surface:
```
⚠️ Une DPIA complète est {{required ? "OBLIGATOIRE" : "recommandée"}}.

Pour la produire :
1. Télécharger l'outil CNIL : {{cnil_pia_url}}
2. Importer les inputs ci-dessus (cnil_pia_inputs)
3. Compléter les sections que piighost ne peut pas inférer (consultation des
   parties prenantes, mesures complémentaires)

piighost ne génère PAS de DPIA complète — c'est l'outil CNIL qui fait foi.
```

### Refusals

- DPIA screening is informational. Never refuse — even on an empty project, the screening adds value (e.g. flags `cnil_5` IA/NER trigger).
