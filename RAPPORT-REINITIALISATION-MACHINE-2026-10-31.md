# RAPPORT — Ré-initialisation machine (Relio)

## Statut

- **Date** : 2026-10-31
- **Dépôts concernés** : `projet` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **HEAD au clone** :
  - racine : `0a26403` — *Delete RAPPORT-CHANTIER-CONTRASTE-INSCRIPTION-TECHNICIEN.md*
  - `backend/` : `366dc46` — *chore(v2): drop technician, dispatch, kyc, zone-coverage and referral schema*
  - `frontend/` : `4874951` — *refactor(mission): convergence du rendu du devis et de l'historique*
- **Push** : aucun code applicatif poussé. Seul ce rapport est ajouté localement à la racine ; le push attend la validation (RÈGLE 6).
- **Déploiement** : aucun.

## Synthèse

`INITIALISATION-MACHINE-GUIDE.md` réexécuté de bout en bout sur une instance Windows vierge : Git absent, `winget` absent, dossiers `backend/` et `frontend/` vides, racine ne contenant que le guide. Git a été installé via l'installateur officiel en mode silencieux, les trois dépôts ont été clonés et branchés, l'identité Git et GCM sont opérationnels, les trois `push --dry-run` sont verts, les dépendances sont installées et les contrôles statiques exécutés.

Le HEAD backend a avancé depuis le rapport du `2026-10-30` (`9851cfd` → `366dc46`, commit « v2 ») et **casse la compilation TypeScript du dépôt** : la migration supprime des modèles Prisma encore utilisés par les services. C'est un état pré-existant du dépôt, pas une conséquence de l'initialisation.

## Fichiers

Créés (non versionnés) :

- `RAPPORT-REINITIALISATION-MACHINE-2026-10-31.md` (racine) — ce rapport
- `backend/.env` et `frontend/.env` — copies de `.env.example` (ignorés par Git)
- `backend/src/generated/prisma/` — client Prisma 7.10.0 (ignoré : `src/generated/`)
- `backend/node_modules/`, `frontend/node_modules/`

Modifiés : aucun fichier versionné. Supprimés : aucun.

## Migrations

Aucune migration appliquée. Le dépôt backend contient au HEAD une migration SQL (`prisma/migrations/…/migration.sql`, +93 lignes) issue du commit `366dc46` ; elle n'a pas été exécutée (aucune base joignable, RÈGLE 3).

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
| `git credential fill` (host=github.com) | renvoie `digitalholding2026-ux` |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com` |
| `push --dry-run origin main` ×3 | `Everything up-to-date` sur les 3 |
| `~/.git-credentials` | inexistant |
| Secrets déjà indexés à la racine | aucun (seul le rapport est ajouté) |
| `npm ci` backend | exit 0 — 474 paquets, 19 vulnérabilités (7 modérées, 11 hautes, 1 critique) |
| `npm ci` frontend | exit 0 — 195 paquets, 12 vulnérabilités (7 modérées, 4 hautes, 1 critique) |
| `npx prisma generate` | client 7.10.0 généré dans `src/generated/prisma`, exit 0 |
| backend `tsc --noEmit -p tsconfig.build.json` | **échec — 67 erreurs dans 8 fichiers** |
| backend `npm run lint` (oxlint `src/ test/`) | exit 0, aucune sortie |
| backend `npx vitest run` | **759 passés, 55 ignorés, 5 fichiers en échec** (63 fichiers) |
| frontend `tsc --noEmit` | exit 0 |
| frontend `npm run lint` | 0 erreur, 5 avertissements `react-hooks/exhaustive-deps` |
| frontend `npm run test:unit` | **613 / 615 passés, 2 échecs** (8 suites) |
| Fichiers `.js` parasites dans les sources | 0 (hors `generated/`) |

### Preuve de non-régression

Aucun fichier versionné n'a été modifié : `git status --porcelain` est vide sur `backend/` et `frontend/`, et la racine n'affiche que `?? backend/` et `?? frontend/` (dépôts imbriqués, RÈGLE 2). Les échecs ci-dessous sont donc **pré-existants** au HEAD cloné.

- **Backend `tsc` — 67 erreurs, 8 fichiers** : `src/admin/admin.service.ts` (33), `src/referrals/referrals.service.ts` (12), `src/demandes/demandes.service.ts` (9), `src/collaboration/collaboration.service.ts` (6), `src/financial/financial.service.ts` (2), `src/admin/catalog.service.ts` (2), `src/auth/auth.service.ts` (2), `src/auth/verification-reminder.scheduler.ts` (1). Le commit `366dc46` a retiré du `schema.prisma` les modèles/colonnes `technician`, `dispatch`, `kyc`, `zone-coverage` et `referral` (262 lignes supprimées) alors que le code applicatif les référence encore (`technicianId`, `PrismaService.referral`, `role === 'TECHNICIAN'`, etc.). Le client Prisma généré est conforme au schéma du HEAD : l'incohérence est dans le dépôt, pas dans l'environnement.
- **Backend tests — 5 fichiers** : `catalog-public-access`, `verification-reminder.module`, `demande-draft.module`, `referrals.module`, `rewards.module`. Cause observée dans la sortie : `ConfigModule` valide les variables d'environnement et rejette les valeurs vides de `.env.example` (`JWT_SECRET`, `FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`). Aucun test n'a été lancé contre une base (RÈGLE 3). Ce sont les 5 mêmes fichiers qu'au rapport du `2026-10-30`.
- **Frontend, 2 tests** : `src/lib/demande-draft-routing.test.ts` — « aucun lien ne pointe plus vers /client/demande » et « les routes /client/demandes sont INTACTES ». Cause réelle dans la trace : `scandir 'C:\C:\Users\Docker\Desktop\Relio\frontend\src\'` → `ENOENT`. Chemin doublé par `new URL('../', import.meta.url).pathname` sous Windows. Bug spécifique à Windows, identique au rapport précédent.
- **Volumétrie des tests backend** : 759/55 sur 63 fichiers, contre 1019/55 sur 81 fichiers au `2026-10-30`. La différence vient du HEAD (`c63fb77` a supprimé les modules technician/dispatch/kyc/zone-coverage et leurs tests), pas de l'initialisation.

## Bugs trouvés

1. **Backend — `tsc` cassé au HEAD `366dc46` (67 erreurs, 8 fichiers).** Impact : le gate obligatoire de la RÈGLE 3 (`tsc --noEmit`) est rouge avant toute modification ; aucun commit backend ne peut être validé en l'état. Correctif attendu : terminer la migration v2 (supprimer ou réécrire les services qui référencent les modèles retirés) ou revenir en arrière sur `366dc46`. **Non appliqué** : hors périmètre de l'initialisation et toute modification de code exige validation (RÈGLE 6).
2. **`frontend/src/lib/demande-draft-routing.test.ts` — chemin Windows doublé.** Impact : 2 tests sur 615 échouent sur toute machine Windows. Correctif proposé : `import { fileURLToPath } from 'node:url'; const SRC = fileURLToPath(new URL('../', import.meta.url));`. **Non appliqué** (RÈGLE 6).
3. **`npm ci` n'exécute pas les `postinstall` de Prisma** (npm 11.20, `allowScripts`) : `@prisma/engines` et `prisma` sont listés « install scripts not yet covered ». Impact : `tsc` échouerait sur `PrismaService` sans client généré. Contournement appliqué : `npx prisma generate` exécuté explicitement. À refaire après chaque `npm ci` frais.
4. **`.env` issu de `.env.example` inutilisable tel quel** : les 5 specs backend échouent tant que les valeurs sont vides. Aucune valeur réelle n'a été inventée ; ce sont des secrets à fournir par le propriétaire du projet.

## Points d'attention

- **`winget` inutilisable (écart majeur, §2 du guide)** : commande absente du PATH, aucun `winget.exe` dans `%LOCALAPPDATA%\Microsoft\WindowsApps`. Repli utilisé : `Git-2.56.0.4-64-bit.exe` téléchargé depuis `github.com/git-for-windows/git` (tag `v2.56.0.windows.4`), installation silencieuse (exit 0). Le guide reste à corriger sur ce point.
- **Version Git** : `2.56.0.windows.4` au lieu de `2.56.0.3` attendu par le guide — même série 2.56.0.
- **`git-lfs.exe` absent** de `C:\Program Files\Git\cmd` après installation silencieuse. Non requis par les trois dépôts (aucun fichier LFS suivi) ; à installer séparément si un besoin apparaît.
- **Authentification déjà provisionnée** : `git-credential-manager github list` renvoie `digitalholding2026-ux` **sans qu'aucun `gcm github login` n'ait été exécuté**, et le clone du dépôt privé `projet.git` a réussi sans invite. Le coffre Windows contenait donc déjà le credential. Fait non expliqué : le premier `cmdkey /list` (avant installation de Git) ne montrait aucun cible GitHub, et un premier appel à `gcm github list` est revenu vide ; la cible `git:https://github.com` n'est apparue qu'après l'installation et la configuration. Aucun login n'ayant été lancé, le credential préexistait ; l'instant exact de sa visibilité n'est pas établi. Le token n'est pas en clair sur le disque (`~/.git-credentials` absent).
- **Casse des dossiers** : `backend/` et `frontend/` existaient en minuscules et vides ; conservés tels quels conformément au §3 du guide. Le `git status` de la racine affiche donc `?? backend/` et `?? frontend/`, et non la variante majuscule du §0.
- **PATH** : `C:\Program Files\Git\cmd` est bien inscrit au PATH machine, mais le shell déjà lancé ne le voit pas (piège §10.4 confirmé). `git` et GCM ont été appelés par chemin absolu / via `$env:PATH` patché.
- **Node** : `v26.13.0`, alors que le guide note `v24.22.1`. Compatible RÈGLE 4. PowerShell 5.1.19041.2673, Windows 10 19045.
- **Prisma** : la génération écrit dans le dépôt (`backend/src/generated/prisma`) mais le dossier est ignoré ; `git status` reste propre et aucun `.js` parasite n'a été trouvé (vérifié, RÈGLE 4).
- Les 5 avertissements de lint frontend sont pré-existants, sans erreur.
- Les 19 et 12 vulnérabilités `npm audit` sont pré-existantes ; aucun `npm audit fix` n'a été lancé (hors périmètre, risque de changement de lockfile).
- Non vérifié : exécution de l'application, connexion à une base, déploiement Railway/Vercel — interdits ou hors périmètre (RÈGLE 3).

## Questions bloquantes

Aucune pour l'initialisation elle-même. Décisions à la main du propriétaire :

1. Pousser ce rapport sur `projet.git` (`main`) maintenant (RÈGLE 6) ?
2. Le backend est incompilable au HEAD `366dc46` : faut-il traiter la migration v2 (services encore branchés sur les modèles retirés) ou revenir à `9851cfd` ?
3. Faut-il corriger le bug de chemin Windows de `demande-draft-routing.test.ts` (2 tests) sur `Repairdom-frontend` ?
4. Faut-il renseigner `backend/.env` / `frontend/.env` avec les vraies valeurs (base, e-mail, JWT) pour rendre les 5 specs backend vertes ?
