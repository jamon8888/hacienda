---
name: setup
description: Onboarding wizard for the hacienda plugin. Walks the avocat / notaire / EC / médecin / RH user through a 6-step conversational setup of their RGPD controller profile (cabinet identity, ordinal number, DPO, finalités, durée de conservation). Auto-loads profession-specific defaults from the bundled profiles. Writes ~/.piighost/controller.toml. Triggers on /hacienda:setup, on a fresh install, or when any other RGPD tool detects an empty controller profile.
argument-hint: "[--project <name>]"
---

# /hacienda:setup — Wizard d'onboarding RGPD

```
/hacienda:setup
/hacienda:setup --project <name>
```

This wizard walks the user through 6 steps to fill in the controller profile that drives every downstream RGPD tool (`processing_register`, `dpia_screening`, `render_compliance_doc`, `subject_access`, `forget_subject`).

## Mode detection

- **Bare `/hacienda:setup`** → global mode. Writes `~/.piighost/controller.toml`. Run on first install.
- **`/hacienda:setup --project <name>`** → per-project override mode. Loads global, asks ONLY the override fields, writes `~/.piighost/projects/<name>/controller_overrides.toml`. See the `--project` mode section at the bottom.

## Pre-flight check

Call `mcp__piighost__controller_profile_get(scope="global")`.

- If the result is **non-empty** and the user invoked the bare command, ask whether they want to (a) overwrite, (b) edit one specific section, or (c) cancel. Default to (b).
- If empty (or the user chose overwrite), proceed with the 6-step flow below.

## Step 1 — Profession

Ask: "Quelle est votre profession ?"

Choices: `avocat` / `notaire` / `expert_comptable` / `medecin` / `rh` / `autre` (→ `generic`).

Capture the answer as `<profession>`. Immediately call:

```
mcp__piighost__controller_profile_defaults(profession="<profession>")
```

This returns a dict with `controller.ordinal_label`, `dpo.required_hint`, `defaults.finalites`, `defaults.bases_legales`, `defaults.duree_conservation_apres_fin_mission`, and `suggested_security_measures`. **Hold on to this dict** — it pre-fills steps 3–6.

## Step 2 — Identité du cabinet

Ask three questions in sequence:

1. "Nom du cabinet ou du responsable de traitement ?" → `controller.name`
2. "Adresse postale (numéro + rue + ville + code postal) ?" → `controller.address`
3. "Pays ?" (default `FR`) → `controller.country`

## Step 3 — Numéro d'inscription ordinale

Use the `ordinal_label` from Step 1's defaults dict. Ask: "Quel est votre {{ ordinal_label }} ?" If `ordinal_label` is empty (RH / generic), skip this step.

Capture as `controller.bar_or_order_number`.

## Step 4 — DPO

Surface the `dpo.required_hint` from defaults (e.g. for médecin: "obligatoire — données de santé Art. 9 traitées systématiquement").

Ask: "Avez-vous désigné un DPO ? (oui / non / inconnu)"

- **oui** → ask `dpo.name`, `dpo.email`, `dpo.phone` (the last two optional). Store under the `dpo` table.
- **non** → confirm with the user that the obligation hint doesn't apply to them. If they're unsure, recommend `oui` and pause the wizard.
- **inconnu** → write `dpo = { unknown = true }` and surface a manual_field hint in the resulting profile.

## Step 5 — Finalités

Display the `defaults.finalites` list (pre-filled from the profession). Ask: "Voici les finalités habituelles pour votre profession. Voulez-vous (a) les accepter telles quelles, (b) en éditer/retirer, (c) en ajouter ?"

Capture the final list as `defaults.finalites`.

Same pattern for `defaults.bases_legales` — show the defaults, let the user edit.

## Step 6 — Durée de conservation

Show `defaults.duree_conservation_apres_fin_mission` (pre-filled). Ask: "Voulez-vous garder cette durée par défaut ou la personnaliser ?"

Capture as `defaults.duree_conservation_apres_fin_mission`.

## Write the profile

Build the final profile dict:

```python
profile = {
    "controller": {
        "name": <step 2.1>,
        "profession": <step 1>,
        "address": <step 2.2>,
        "country": <step 2.3>,
        "bar_or_order_number": <step 3, if any>,
    },
    "dpo": <step 4 result>,
    "defaults": {
        "finalites": <step 5 finalites>,
        "bases_legales": <step 5 bases>,
        "duree_conservation_apres_fin_mission": <step 6>,
    },
}
```

Call:

```
mcp__piighost__controller_profile_set(profile=<profile>, scope="global")
```

Then call `mcp__piighost__controller_profile_get(scope="global")` to confirm the round-trip.

## Confirm

Print:

```
✅ Profil cabinet enregistré

Cabinet         : {{ controller.name }}
Profession      : {{ controller.profession }}
N° {{ ordinal_label }} : {{ controller.bar_or_order_number }}
DPO             : {{ dpo.name or "non désigné" }}
Finalités       : {{ defaults.finalites | length }}
Conservation    : {{ defaults.duree_conservation_apres_fin_mission }}

Vous pouvez maintenant utiliser :
  /hacienda:rgpd:registre   — Registre Art. 30
  /hacienda:rgpd:dpia       — Screening DPIA Art. 35
  /hacienda:rgpd:access     — Réponse Art. 15
  /hacienda:rgpd:forget     — Droit à l'oubli Art. 17

Pour ajuster un dossier spécifique :
  /hacienda:setup --project <nom-du-projet>
```

## --project mode

If the user invoked `/hacienda:setup --project <name>`:

1. Call `mcp__piighost__controller_profile_get(scope="project", project=<name>)` and show the merged view (global + any existing override).
2. Ask: "Quels champs voulez-vous surcharger pour ce dossier ?" — list the keys (controller / dpo / defaults).
3. For each chosen key, ask only that section's questions (using the same defaults pre-fill logic if profession changes).
4. Build a partial dict containing ONLY the overridden fields and call:

```
mcp__piighost__controller_profile_set(profile=<partial>, scope="project", project=<name>)
```

The daemon's deep-merge logic (existing in `ControllerProfileService`) preserves global values for unchanged fields.

## Refusals

- If `mcp__piighost__controller_profile_set` returns an error, surface it verbatim — do not retry silently. Likely cause: filesystem permissions on `~/.piighost/`.
- If the user picks `--project <name>` but the project doesn't exist, ask whether to (a) create it via `mcp__piighost__create_project`, or (b) cancel.
- The wizard MUST NOT call any non-controller-profile MCP tool. It writes one file and confirms; downstream skills do the actual compliance work.
