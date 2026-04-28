# Hacienda — le plugin Cowork de Hacienda Ghost

> Plugin Claude Desktop / Claude Code pour les avocats, notaires,
> médecins, experts-comptables et professions réglementées.

Ce plugin ajoute des commandes RGPD, juridiques et de recherche
confidentielle à Claude Desktop. Il fonctionne avec le moteur
[Hacienda Ghost](https://github.com/jamon8888/hacienda-ghost) installé en
local — vos données clients ne sortent jamais de votre poste.

## Ce qu'apporte ce plugin

Le plugin expose plusieurs familles de commandes accessibles via la
barre de commandes Claude :

- **`/hacienda:rgpd:*`** — registre Art. 30, screening DPIA Art. 35,
  réponse Art. 15, droit à l'oubli Art. 17.
- **`/hacienda:legal:*`** — vérification de citations juridiques
  (Legifrance), recherche dans le code et la jurisprudence.
- **`/hacienda:search`** — recherche fédérée combinant vos documents
  locaux et les sources officielles.
- **`/hacienda:setup`** — wizard de configuration du cabinet en 6
  questions.

## Installation

L'installation passe par le moteur Hacienda Ghost. Voir le
[README principal](https://github.com/jamon8888/hacienda-ghost#installation-en-4-%C3%A9tapes)
pour les 4 étapes d'installation.

Résumé : installer `uv`, lancer
`uvx --from piighost piighost install --mode=mcp-only`, puis
`claude plugins add jamon8888/hacienda`, puis redémarrer Claude
Desktop.

## Commandes disponibles

| Commande                  | Quand l'utiliser                              |
|---------------------------|-----------------------------------------------|
| `/hacienda:setup`         | Configurer ou modifier le profil cabinet     |
| `/hacienda:rgpd:registre` | Générer le registre Art. 30                  |
| `/hacienda:rgpd:dpia`     | Screening DPIA Art. 35                       |
| `/hacienda:rgpd:access`   | Réponse à une demande Art. 15                |
| `/hacienda:rgpd:forget`   | Droit à l'oubli Art. 17                      |
| `/hacienda:legal:setup`   | Activer la vérification Legifrance           |
| `/hacienda:legal:verify`  | Vérifier les citations juridiques d'un texte |
| `/hacienda:search`        | Recherche fédérée (vos docs + Legifrance)    |
| `/hacienda:audit`         | Journal d'audit de la session                |
| `/hacienda:status`        | État de l'index du dossier ouvert            |
| `/hacienda:index`         | Forcer la ré-indexation du dossier           |

## Sécurité et confidentialité

Toutes les données restent sur votre poste, chiffrées sur disque.
Les requêtes vers Claude contiennent uniquement des étiquettes
anonymisées. Détails complets :
[README principal — Sécurité et confidentialité](https://github.com/jamon8888/hacienda-ghost#s%C3%A9curit%C3%A9-et-confidentialit%C3%A9).

## Licence

MIT. Voir [LICENSE](LICENSE).
