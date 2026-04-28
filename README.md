# Hacienda — le plugin Cowork de Hacienda Ghost

> Plugin Claude Desktop / Claude Code pour les avocats, notaires,
> médecins, experts-comptables et professions réglementées.

Ce plugin ajoute des commandes RGPD, juridiques et de recherche
confidentielle à Claude Desktop. Il fonctionne avec le moteur
[Hacienda Ghost](https://github.com/jamon8888/hacienda-ghost) installé en
local — vos données clients ne sortent jamais de votre poste.

## Ce qu'apporte ce plugin

Le plugin ajoute cinq capacités à Claude Desktop, toutes orchestrées
par des commandes accessibles via `/hacienda:*` :

- **Recherche confidentielle dans vos dossiers clients.** Le moteur
  indexe localement un dossier complet (PDF, Word, Excel, CSV,
  e-mails, notes). Lorsque Claude répond à une question, il cite les
  fichiers d'origine — sans transmettre les noms, adresses, IBAN ou
  numéros de sécurité sociale au cloud.

- **Conformité RGPD complète, en quatre commandes.** Génération du
  registre Art. 30 (registre des activités de traitement) prêt à
  signer ; screening DPIA Art. 35 avec verdict et inputs pour l'outil
  officiel CNIL PIA ; rapport Art. 15 (droit d'accès) avec extraits
  redactés ; cascade Art. 17 (droit à l'oubli) avec aperçu avant
  validation.

- **Vérification de vos citations juridiques** *(optionnelle)*.
  Articles de codes, lois, décrets, jurisprudences contrôlés contre
  les sources Legifrance officielles. Détection des références
  inexistantes, abrogées, ou aux numéros erronés. Nécessite un
  compte gratuit sur https://piste.gouv.fr.

- **Recherche fédérée locale + Legifrance.** Une seule commande
  combine vos documents indexés et les sources juridiques officielles
  (Code civil, jurisprudence judiciaire et administrative, décisions
  CNIL, JORF, conventions collectives). Les résultats sont annotés
  avec leur origine (LOCAL / CODE / JURI / CNIL).

- **Journal d'audit par session.** Chaque échange avec Claude est
  enregistré localement : quelles étiquettes anonymisées ont été
  générées, quels appels sortants ont été émis, ce qui est parti
  vers le cloud et ce qui en est revenu. Vous gardez la preuve que
  rien de sensible n'a été divulgué.

## Workflows types

Comment les commandes s'enchaînent en pratique :

- **Première utilisation d'un cabinet (~10 minutes).**
  `/hacienda:setup` (configure le profil cabinet) → ouvrir un dossier
  client dans Cowork (l'indexation démarre automatiquement) →
  `/hacienda:rgpd:registre` (votre premier registre Art. 30 conforme).

- **Audit RGPD annuel complet.**
  `/hacienda:rgpd:registre` (état des lieux des traitements) →
  `/hacienda:rgpd:dpia` (vérifier si une DPIA complète est requise) →
  `/hacienda:audit` (consulter le journal de toutes les sessions de
  l'année).

- **Répondre à une demande Art. 15 d'un client.**
  `/hacienda:rgpd:access` (rapport listant tous les documents et
  extraits où le sujet apparaît) → rendu Markdown ou PDF prêt à
  envoyer au demandeur.

- **Effacer définitivement un client (Art. 17).**
  `/hacienda:rgpd:forget` en mode aperçu (liste les documents et
  extraits à modifier) → validation par l'utilisateur → exécution
  réelle (le coffre-fort est purgé, les extraits indexés sont
  réécrits, le journal d'audit conserve uniquement les empreintes
  hachées).

- **Vérifier les citations d'une note de plaidoirie.**
  `/hacienda:legal:setup` (la première fois, pour saisir votre
  token PISTE) → `/hacienda:legal:verify` avec votre note collée
  (extraction automatique des références, vérification contre
  Legifrance, rapport).

- **Chercher un fait précis dans le dossier ouvert.**
  `/hacienda:search "responsabilité contractuelle"` (recherche
  fédérée locale + Legifrance) ou `/hacienda:ask "Qui a signé le NDA
  du 12 mars ?"` (question directe, réponse citée).

## Installation

L'installation passe par le moteur Hacienda Ghost. Voir le
[README principal](https://github.com/jamon8888/hacienda-ghost#installation-en-4-%C3%A9tapes)
pour les 4 étapes d'installation.

Résumé : installer `uv`, lancer
`uvx --from piighost piighost install --mode=mcp-only`, puis
`claude plugins add jamon8888/hacienda`, puis redémarrer Claude
Desktop.

## Commandes disponibles

| Commande                  | Quand l'utiliser                                       |
|---------------------------|--------------------------------------------------------|
| `/hacienda:setup`         | Configurer ou modifier le profil cabinet               |
| `/hacienda:ask`           | Poser une question sur le dossier ouvert (réponse citée) |
| `/hacienda:search`        | Recherche fédérée (vos documents + Legifrance)         |
| `/hacienda:rgpd:registre` | Générer le registre Art. 30                            |
| `/hacienda:rgpd:dpia`     | Screening DPIA Art. 35                                 |
| `/hacienda:rgpd:access`   | Rapport Art. 15 (droit d'accès)                        |
| `/hacienda:rgpd:forget`   | Cascade Art. 17 (droit à l'oubli) avec aperçu          |
| `/hacienda:legal:setup`   | Activer la vérification Legifrance                     |
| `/hacienda:legal:verify`  | Vérifier les citations juridiques d'un texte           |
| `/hacienda:audit`         | Journal d'audit de la session                          |
| `/hacienda:status`        | État de l'index du dossier ouvert                      |
| `/hacienda:index`         | Forcer la ré-indexation du dossier                     |

Une douzième règle, `redact-outbound`, n'est pas une commande à
invoquer : c'est une règle automatique que Claude applique à chaque
fois qu'il rédige un message sortant (e-mail, document, message Slack)
pour préserver les étiquettes anonymisées. Elle agit en arrière-plan.

## Sécurité et confidentialité

Hacienda Ghost protège vos données via trois frontières strictes :

- **Reste sur votre poste, chiffré sur disque.** Les noms, IBAN,
  numéros de sécurité sociale, adresses, et l'intégralité de vos
  documents vivent dans `~/.piighost/` en chiffrement AES-256-GCM. La
  clé est dans `~/.piighost/vault.key`. Aucun de ces éléments ne
  quitte votre poste.

- **Sort vers Claude : étiquettes anonymisées uniquement.** Lorsque
  Claude lit le contenu de vos documents pour répondre à une question,
  il ne voit jamais les valeurs originales — seulement des étiquettes
  opaques (`<<nom_personne:abc12345>>`, `<<email:def67890>>`).

- **Sort vers Legifrance : références juridiques uniquement**
  *(intégration optionnelle, désactivée par défaut)*. Si vous activez
  la vérification de citations, seules les références juridiques
  extraites (numéros d'article, numéros de pourvoi, noms de codes)
  sont envoyées à Legifrance. Le contenu de vos dossiers ne sort
  jamais par cette voie.

À tout moment, `/hacienda:audit` affiche le journal complet de la
session : chaque appel sortant, son destinataire, et ce qui a été
envoyé. Vous gardez la preuve auditable que rien de sensible n'a
fuité.

Détails techniques complets :
[README principal — Sécurité et confidentialité](https://github.com/jamon8888/hacienda-ghost#s%C3%A9curit%C3%A9-et-confidentialit%C3%A9).

## Licence

MIT. Voir [LICENSE](LICENSE).
