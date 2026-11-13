# RAPPORT-INITIALISATION-MACHINE

## Statut

- Date : 2026-11-13
- Dépôts concernés : `projet.git` (racine), `Repairdom-backend` (Backend/), `Repairdom-frontend` (Frontend/)
- Branche : `main` sur les trois
- État push : aucun push de code effectué (initialisation uniquement ; pas de modification de code applicatif)
- Rapport : fichier rédigé, non encore committé ni poussé (en attente de validation, RÈGLE 6 AGENTS.md)

## Synthèse

Initialisation complète d'une machine Windows vierge (sans Git, sans Node, sans winget) suivie du guide `INITIALISATION-MACHINE-GUIDE.md` (étapes 1 à 10 exécutées ; étape 11 différée car Node absent). Trois dépôts Git indépendants imbriqués à la racine `C:\Users\Docker\Desktop\Relio\`, authentification OAuth fonctionnelle via Git Credential Manager, droits de lecture/écriture confirmés sur les trois remotes.

## Fichiers

Créés :
- `RAPPORT-INITIALISATION-MACHINE.md` (ce rapport, à la racine)

Modifiés : aucun (pas de modification de code applicatif ni de document existant)

Supprimés : aucun

## Dépôts et HEAD au moment du clone

| Dossier | Remote `origin` | HEAD |
|---|---|---|
| `Relio\` | `https://github.com/digitalholding2026-ux/projet.git` | `432d1ad` docs(rapport): ajout correctifs inscription et auth |
| `Relio\Backend\` | `https://github.com/digitalholding2026-ux/Repairdom-backend.git` | `0541864` feat(security): protéger le suivi par référence avec vérification du numéro de téléphone |
| `Relio\Frontend\` | `https://github.com/digitalholding2026-ux/Repairdom-frontend.git` | `3e89023` feat(security): écran de déverrouillage du suivi par numéro de téléphone |

## Migrations

Aucune.

## Vérifications

- `git --version` : `git version 2.56.0.windows.3` (terminal neuf, PATH machine mis à jour par l'installeur)
- `.git` présent dans `Relio\`, `Relio\Backend\`, `Relio\Frontend\`
- `origin` = fetch et push corrects sur les trois
- Branche `main` + amont `origin/main` sur les trois
- `git status -sb` à la racine : `## main...origin/main` + `?? Backend/` + `?? Frontend/` uniquement (Backend/ et Frontend/ jamais indexés à la racine, RÈGLE 2 respectée)
- `user.name=digitalholding2026-ux`, `user.email=digitalholding2026@gmail.com`, `credential.helper=manager` (globaux)
- `git-credential-manager github list` → `digitalholding2026-ux`
- `cmdkey /list` → `git:https://github.com` (crédentiel dans le coffre Windows, aucun token en clair)
- `push --dry-run origin main` → `Everything up-to-date` sur les trois (preuve lecture + écriture)
- `~/.git-credentials` : inexistant (aucun secret en clair sur le disque)
- Node ≥ 22 : **non installé** (étape 11 du guide différée, hors scope de cette procédure ; `npm.cmd ci` Backend/Frontend à réaliser ultérieurement une fois Node 22+ installé)

Non exécuté (expressément différé par le guide) : `next build`, `npm run dev`, Docker, Postgres local, tests e2e (interdits en local, RÈGLE 3). `oxlint` / `tsc --noEmit` / `vitest run` non lancés (aucun code modifié, dépendances non installées).

## Bugs trouvés

Aucun. Aucun écart par rapport au guide, si ce n'est :
- `winget` absent sur la machine : Git installé via le binaire officiel `Git-2.56.0.3-64-bit.exe` téléchargé depuis github.com/git-for-windows/git (équivalent de l'étape 2 du guide).

## Points d'attention

- Node 22+ absent : l'étape 11 (`npm.cmd ci --prefix Backend` et `--prefix Frontend`) n'a pu être exécutée. À planifier après installation de Node.
- Les `.env` de chaque dépôt applicatif doivent être créés depuis `.env.example` (jamais committés).

## Questions bloquantes

Aucune.
