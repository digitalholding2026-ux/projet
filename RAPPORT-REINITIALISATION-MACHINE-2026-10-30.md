# RAPPORT — Ré-initialisation machine (Relio)

## Statut

- **Date** : 2026-10-30
- **Dépôts concernés** : `projet` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **HEAD au clone** :
  - racine : `c2e7e5b` — *docs(audit): rapport d'audit complet du backend*
  - `backend/` : `9851cfd` — *chore(tech): stab tests, lockfile et assemblage AppModule*
  - `frontend/` : `4874951` — *refactor(mission): convergence du rendu du devis et de l'historique*
- **Push** : aucun code applicatif poussé. Seul ce rapport est ajouté localement à la racine ; le push reste à valider (RÈGLE 6).
- **Déploiement** : aucun.

## Synthèse

`INITIALISATION-MACHINE-GUIDE.md` réexécuté de bout en bout sur une instance Windows vierge (Git absent, dossiers `backend/` et `frontend/` vides). À la différence de l'exécution du `2026-10-27`, `winget` s'est révélé inutilisable : Git a été installé via l'installateur officiel en mode silencieux. Les trois dépôts sont clonés et branchés, l'identité Git et GCM sont opérationnels, les `push --dry-run` sont verts, les dépendances sont installées et les contrôles statiques exécutés.

Le rapport de l'exécution précédente (`RAPPORT-INITIALISATION-MACHINE.md`, commit `6ef24dc`) est conservé intact : ce document est un **nouveau** rapport, pas un remplacement. Toutes les mesures ci-dessous ont été **re-mesurées sur cette machine**, elles ne sont pas recopiées.

## Fichiers

Créés (non versionnés) :

- `RAPPORT-REINITIALISATION-MACHINE-2026-10-30.md` (racine) — ce rapport
- `backend/.env` et `frontend/.env` — copies de `.env.example` (ignorés par Git)
- `backend/src/generated/prisma/` — client Prisma 7.10.0 (ignoré : `src/generated/`)
- `backend/node_modules/`, `frontend/node_modules/`

Modifiés : aucun fichier versionné. Supprimés : aucun.

## Migrations

Aucune.

## Vérifications

| Étape | Résultat |
|---|---|
| `winget --version` | **échec** — commande introuvable (voir Points d'attention) |
| Git installé | `git version 2.56.0.windows.4`, exit 0 |
| `node --version` | `v26.13.0` (≥ 22, RÈGLE 4) |
| `npm.cmd --version` | `11.20.0` |
| `.git` présent (3 dépôts) | oui |
| `origin` correct (fetch + push) | oui sur les 3 |
| Branche `main` + amont `origin/main` | oui sur les 3 |
| `git log --oneline -1` | un commit dans chacun des 3 dépôts |
| `git status -sb` racine | `?? backend/` + `?? frontend/` uniquement |
| `git config --global --list` | `user.name`, `user.email`, `credential.helper=manager` |
| `git-credential-manager github list` | `digitalholding2026-ux` |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com` |
| `push --dry-run origin main` ×3 | `Everything up-to-date` sur les 3 |
| `~/.git-credentials` | inexistant |
| Secrets déjà indexés à la racine | aucun (`Relio.pem`, `.env` non suivis) |
| `npm ci` backend | exit 0 — 19 vulnérabilités signalées (7 modérées, 11 hautes, 1 critique) |
| `npm ci` frontend | exit 0 — 195 paquets, 12 vulnérabilités signalées |
| `npx prisma generate` | client 7.10.0 généré dans `src/generated/prisma`, exit 0 |
| backend `tsc --noEmit -p tsconfig.build.json` | exit 0 |
| backend `npm run lint` (oxlint `src/ test/`) | exit 0, aucune sortie |
| backend `npx vitest run` | **1019 passés, 55 ignorés, 5 fichiers en échec** (81 fichiers) |
| frontend `tsc --noEmit` | exit 0 |
| frontend `npm run lint` | 0 erreur, 5 avertissements `react-hooks/exhaustive-deps` |
| frontend `npm run test:unit` | **613 / 615 passés, 2 échecs** (8 suites) |
| Fichiers `.js` parasites dans les sources | 0 (hors `generated/`) |

### Preuve de non-régression

Aucun fichier versionné n'a été modifié : `git status -sb` est propre sur `backend/`, `frontend/` et la racine (hors `backend/`/`frontend/` non suivis). Les échecs ci-dessous sont donc **pré-existants** par construction, et les compteurs obtenus sont identiques à ceux du rapport `6ef24dc` (1019/55/5 et 613/2), ce qui confirme la reproductibilité sur cette machine.

- **Backend, 5 fichiers** : `catalog-public-access`, `verification-reminder.module`, `demande-draft.module`, `referrals.module`, `rewards.module`. Cause observée dans la sortie : `ConfigModule` valide les variables d'environnement et rejette les valeurs vides de `.env.example` (`JWT_SECRET`, `FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`). Aucun test n'a été lancé contre une base (RÈGLE 3).
- **Frontend, 2 tests** : `src/lib/demande-draft-routing.test.ts` — « aucun lien ne pointe plus vers /client/demande » et « les routes /client/demandes sont INTACTES ». Cause réelle dans la trace : `scandir 'C:\C:\Users\...\frontend\src\'` → `ENOENT`. Le chemin est doublé parce que `new URL('../', import.meta.url).pathname` renvoie `/C:/Users/...` sous Windows. Bug spécifique à Windows, indépendant de l'initialisation.

## Bugs trouvés

1. **`frontend/src/lib/demande-draft-routing.test.ts` — chemin Windows doublé.** Impact : 2 tests sur 615 échouent sur toute machine Windows. Correctif proposé (déjà formulé dans le rapport précédent) : `import { fileURLToPath } from 'node:url'; const SRC = fileURLToPath(new URL('../', import.meta.url));`. **Non appliqué** : hors périmètre et toute modification de code exige validation (RÈGLE 6).
2. **`npm ci` n'exécute pas les `postinstall` de Prisma** (npm 11.20, `allowScripts`) : `@prisma/engines` et `prisma` sont listés « install scripts not yet covered ». Impact : `tsc` échouerait sur `PrismaService` sans client généré. Contournement appliqué : `npx prisma generate` exécuté explicitement. À refaire après chaque `npm ci` frais.
3. **`.env` issu de `.env.example` inutilisable tel quel** : les 5 specs backend échouent tant que les valeurs sont vides. Aucune valeur réelle n'a été inventée ; ce sont des secrets à fournir par le propriétaire du projet.

## Points d'attention

- **`winget` inutilisable (écart majeur, §2 du guide)** : la commande est absente du PATH ; le paquet `Microsoft.DesktopAppInstaller` enregistré est la version `1.0.30251.0`, qui ne contient **pas** `winget.exe`, et le binaire de la version `1.29.380.0` présent dans `C:\Program Files\WindowsApps` n'est pas exécutable (`Accès refusé`). Repli utilisé : `Git-2.56.0.4-64-bit.exe` téléchargé depuis `github.com/git-for-windows/git` (tag `v2.56.0.windows.4`), SHA256 `4961A4A8CC778D8B6E00A5819E5FE4D078B754555AFF7AEDC2D77A5D74C2DE80`, installation silencieuse avec le composant `credentialmanager` (exit 0). Le guide reste donc à corriger sur ce point.
- **Version Git** : `2.56.0.windows.4` alors que le guide attend `2.56.0.3` — même série 2.56.0, seule la révision du correctif diffère.
- **Authentification déjà provisionnée** : `git-credential-manager github list` renvoyait déjà `digitalholding2026-ux` **avant** tout `login`, et le clone du dépôt privé `projet.git` a réussi sans invite. Aucune commande `gcm github login` n'a donc été exécutée sur cette instance ; le coffre Windows de l'image contenait déjà le credential. Le token n'est pas en clair sur le disque (`~/.git-credentials` absent).
- **Casse des dossiers** : `backend/` et `frontend/` existaient en minuscules et vides ; conservés tels quels conformément au §3 du guide. Le `git status` de la racine affiche donc `?? backend/` et `?? frontend/`, et non la variante majuscule du §0.
- **PATH** : `C:\Program Files\Git\cmd` est bien inscrit au PATH machine, mais le shell déjà lancé ne le voit pas (piège §10.4 confirmé). `git` et GCM ont été appelés par chemin absolu / via `$env:PATH` patché.
- **Node** : `v26.13.0`, alors que le guide note `v24.22.1`. Compatible RÈGLE 4.
- **Prisma** : la génération écrit dans le dépôt (`backend/src/generated/prisma`) mais le dossier est ignoré ; `git status` reste propre et aucun `.js` parasite n'a été trouvé (vérifié, RÈGLE 4).
- Les 5 avertissements de lint frontend sont pré-existants, sans erreur.
- Les 19 et 12 vulnérabilités `npm audit` sont pré-existantes ; aucun `npm audit fix` n'a été lancé (hors périmètre, risque de changement de lockfile).
- Non vérifié : exécution de l'application, connexion à une base, déploiement Railway/Vercel — interdits ou hors périmètre (RÈGLE 3).

## Questions bloquantes

Aucune. Deux décisions restent à la main du propriétaire, inchangées depuis le rapport précédent :

1. Faut-il corriger le bug de chemin Windows de `demande-draft-routing.test.ts` (2 tests) sur `Repairdom-frontend` ?
2. Faut-il renseigner `backend/.env` / `frontend/.env` avec les vraies valeurs (base, e-mail, JWT) pour rendre les 5 specs backend vertes ?
3. Faut-il pousser ce rapport sur `projet.git` (`main`) maintenant ?
