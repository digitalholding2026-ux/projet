# Rapport — Initialisation machine neuve (Relio)

## Statut

- Date : 2026-11-07
- Dépôt concerné : `projet.git` (racine) — documentation uniquement
- Branche : `main`
- Push : effectué (voir ci-dessous)
- Déploiement : non applicable (dépôt documentation)

## Synthèse

Les trois dépôts Git (`projet.git`, `Repairdom-backend`, `Repairdom-frontend`)
ont été clonés et branchés sur une machine Windows vierge. Git a été installé
via l'installateur officiel (winget absent). L'authentification a été réalisée
par OAuth via Git Credential Manager (token stocké dans le coffre Windows, aucun
secret en clair). Les `push --dry-run` réussissent sur les trois dépôts.

## Fichiers

- Créés à la racine : `RAPPORT-INITIALISATION-MACHINE.md` (ce fichier).
- Aucune modification dans `Backend/` ni `Frontend/` (dépôts indépendants).

## Migrations

Aucune.

## Vérifications

- Git : `2.47.1.windows.1` (installé via `Git-2.47.1-64-bit.exe`).
- `Backend/.git` : présent — HEAD `dce7e9c chore(cleanup): remove v1 technician modules and reputation controller`.
- `Frontend/.git` : présent — HEAD `41f07ef fix(auth): remove role field from signup payload (rejected by validation pipe)`.
- Racine `.git` : présent — HEAD `3eb74c9 docs(audit): rapport audit intégral v2 cleanup`.
- `origin` (fetch + push) correct sur les trois dépôts.
- Branche `main` avec amont `origin/main` sur les trois.
- `git status -sb` racine : `?? Backend/` et `?? Frontend/` uniquement (attendu).
- `git-credential-manager github list` : `digitalholding2026-ux`.
- `cmdkey /list` : `git:https://github.com` présent.
- `push --dry-run origin main` : `Everything up-to-date` sur les trois dépôts.
- `~/.git-credentials` : inexistant (aucun secret en clair).

## Écarts par rapport au guide

| Point | Guide | Appliqué | Raison |
|---|---|---|---|
| Installation Git | `winget install Git.Git` | Installateur `.exe` téléchargé | `winget` absent sur la machine |
| Version Git | 2.56.0.3 | 2.47.1.windows.1 | version disponible téléchargée |
| Chemin GCM | `C:\Program Files\Git\ucrt64\bin\` | `C:\Program Files\Git\mingw64\bin\` | layout de la version installée |
| Node | ≥ 22 attendu | non installé | étape 11 (npm ci) non exécutée par cette procédure |

## Points d'attention

- Node.js n'est pas installé : `npm ci` (étape 11 du guide) n'a pas été
  exécuté. À faire avant toute vérification de code.
- Les variables `.env` des deux dépôts applicatifs restent à créer depuis
  `.env.example` ; jamais commités.
- Les dossiers `Backend/` et `Frontend/` restent non indexés à la racine
  (RÈGLE 2).

## Questions bloquantes

Aucune.
