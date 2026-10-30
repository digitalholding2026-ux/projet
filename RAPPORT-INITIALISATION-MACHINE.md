# RAPPORT — Initialisation machine complète (Relio)

## Statut

- **Date** : 2026-10-30
- **Dépôts concernés** : `projet` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **Commits de départ (HEAD au clone)** :
  - racine : `0e3056b` — *docs: add full machine initialization guide (git install, 3-repo clone and wiring, GCM auth)*
  - `backend/` : `9851cfd` — *chore(tech): stab tests, lockfile et assemblage AppModule*
  - `frontend/` : `4874951` — *refactor(mission): convergence du rendu du devis et de l'historique*
- **Push** : les trois dépôts sont à `main...origin/main`, sans commit local en retard. Seul ce rapport a été ajouté et poussé sur `projet.git`.
- **Déploiement** : aucun (aucun code applicatif modifié).

## Synthèse

Procédure `INITIALISATION-MACHINE-GUIDE.md` exécutée de bout en bout sur une machine Windows 10 Pro 64 bits où **`winget` était absent** (écart majeur, voir plus bas) : Git installé, les trois dépôts clonés et branchés, identité Git et authentification GCM configurées, `push --dry-run` validé sur les trois, dépendances installées et contrôles statiques exécutés.

## Fichiers

Créés :

- `RAPPORT-INITIALISATION-MACHINE.md` (racine) — ce rapport
- `backend/.env` et `frontend/.env` — copies de `.env.example` (ignorés par Git, jamais commités)
- `backend/src/generated/prisma/` — client Prisma généré (ignoré : `src/generated/`)

Modifiés : aucun fichier versionné. Supprimés : aucun.

## Migrations

Aucune.

## Vérifications

| Étape | Résultat |
|---|---|
| `git --version` | `git version 2.56.0.windows.4` |
| `node --version` | `v26.13.0` (≥ 22) |
| `npm.cmd --version` | `11.20.0` |
| `.git` présent (3 dépôts) | oui |
| `origin` correct (fetch + push) | oui sur les 3 |
| Branche `main` + amont `origin/main` | oui sur les 3 |
| `git status -sb` racine | `?? backend/` + `?? frontend/` uniquement |
| `git config --global --list` | `user.name`, `user.email`, `credential.helper=manager` |
| `git-credential-manager github list` | `digitalholding2026-ux` |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com` |
| `push --dry-run origin main` ×3 | `Everything up-to-date`, exit 0 |
| `~/.git-credentials` | inexistant |
| `npm ci` backend | exit 0, 344 paquets dans `node_modules` |
| `npm ci` frontend | exit 0, 195 paquets ajoutés |
| `npx prisma generate` | client 7.10.0 généré |
| backend `tsc --noEmit -p tsconfig.build.json` | exit 0 |
| backend `npm run lint` (oxlint) | exit 0 |
| backend `npx vitest run` | 1019 passés, 55 ignorés, **5 fichiers en échec** (env vide) |
| frontend `tsc --noEmit` | exit 0 |
| frontend `npm run lint` | 0 erreur, 5 avertissements `react-hooks/exhaustive-deps` |
| frontend `npm run test:unit` | **613 / 615 passés, 2 échecs** (bug de chemin Windows) |

### Preuve de non-régression

Aucun code n'a été modifié : les échecs ci-dessous ne peuvent pas être des régressions introduites par cette tâche. Ils sont reproduits sur les `HEAD` clonés, sans aucune modification locale (`git status` propre sur `backend/` et `frontend/`).

- **Backend, 5 fichiers** : `catalog-public-access`, `verification-reminder.module`, `demande-draft.module`, `referrals.module`, `rewards.module`. Cause vérifiée : `ConfigModule` valide `.env`, or `.env.example` contient des valeurs **vides** et `@IsNotEmpty()` rejette la chaîne vide. Preuve : `rewards.module.spec.ts` passe **12/12** en fournissant `FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`, `JWT_SECRET` factices en variables d'environnement. Aucun test exécuté contre une base (conforme RÈGLE 3).
- **Frontend, 2 tests** : `demande-draft-routing.test.ts`, tests « aucun lien ne pointe plus vers /client/demande » et « routes /client/demandes intactes ». Cause : ligne 21, `const SRC = new URL('../', import.meta.url).pathname` renvoie `/C:/Users/...` sous Windows ; concaténé, il produit `C:\C:\Users\...` → `ENOENT`. Bug **spécifique à Windows**, indépendant de cette initialisation.

## Bugs trouvés

1. **Chemin Windows dans `frontend/src/lib/demande-draft-routing.test.ts:21`** — impact : 2 tests sur 615 échouent sur toute machine Windows. Correctif : `import { fileURLToPath } from 'node:url'; const SRC = fileURLToPath(new URL('../', import.meta.url));`. Non appliqué : hors périmètre de l'initialisation, et une modification de code exige validation (RÈGLE 6).
2. **`npm ci` n'exécute pas les scripts `postinstall` de Prisma** (npm 11.20, politique `allowScripts`) — impact : `tsc` échoue sur `PrismaService` tant que le client n'est pas généré. Correctif appliqué : `npx prisma generate` exécuté explicitement. À refaire après chaque `npm ci` frais.
3. **`.env` issu de `.env.example` inutilisable tel quel** — 5 specs backend chargent `ConfigModule` et échouent sur les valeurs vides. Il faut renseigner au minimum `DATABASE_URL`, `JWT_SECRET`, `FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`. Non renseigné : ce sont des secrets, à fournir par l'utilisateur.

## Points d'attention

- **`winget` absent** : l'étape 2 du guide est inapplicable. Git a été installé via l'installateur officiel `Git-2.56.0.4-64-bit.exe` (`v2.56.0.windows.4`, SHA256 `4961A4A8CC778D8B6E00A5819E5FE4D078B754555AFF7AEDC2D77A5D74C2DE80`), en mode silencieux. GCM est bien présent (`C:\Program Files\Git\ucrt64\bin\git-credential-manager.exe`).
- **Casse des dossiers** : `backend/` et `frontend/` en minuscules, casse déjà présente sur la machine ; le guide demande de la respecter. Le `git status` racine affiche donc `?? backend/` et `?? frontend/` (et non `?? Backend/`).
- **PATH** : `C:\Program Files\Git\cmd` est bien inscrit dans le PATH machine, mais les processus déjà lancés (dont ce shell) ne le voient pas — piège §10.4 confirmé. `git` et GCM exigent donc un `$env:PATH` patché dans la session courante ; un terminal neuf est OK.
- **Node v26.13.0** au lieu de la version notée dans le guide (v24.22.1) : compatible RÈGLE 4 (≥ 22).
- **`prisma generate` écrit dans le dépôt** (`backend/src/generated/prisma`), mais le dossier est déjà ignoré par `backend/.gitignore` : `git status` reste propre. Aucun fichier `.js` parasite.
- Les 5 avertissements de lint frontend sont pré-existants et sans erreur.
- Non vérifié : exécution de l'application (interdite en local, RÈGLE 3), connexion réelle à une base, déploiement Railway/Vercel.

## Questions bloquantes

Aucune. Deux décisions restent à votre main, hors périmètre d'initialisation :

1. Faut-il corriger le bug de chemin Windows dans `demande-draft-routing.test.ts` (2 tests) et le pousser sur `Repairdom-frontend` ?
2. Faut-il renseigner `backend/.env` / `frontend/.env` avec les vraies valeurs (base, e-mail, JWT) pour rendre les 5 specs backend vertes ?
