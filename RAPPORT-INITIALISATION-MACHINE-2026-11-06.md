# RAPPORT — Initialisation machine (Relio)

## Statut

- **Date** : 2026-11-06
- **Dépôts concernés** : `projet` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **HEAD au clone** :
  - racine : `ca8ab3a` — *docs: report chantier M0.3 residiuel backend + nettoyage frontend v2*
  - `Backend/` : `229d613` — *chore(v2): drop dead geo-eligibility and geo-matching helpers*
  - `Frontend/` : `65a891b` — *refactor(v2): remove technician pages, services and tests*
- **Push** : aucun code applicatif poussé. Seul ce rapport est ajouté à la racine.
- **Déploiement** : aucun.

## Synthèse

`INITIALISATION-MACHINE-GUIDE.md` exécuté de §3 à §8 sur une instance Windows 10 (19045) où les trois dépôts n'étaient pas présents. §1–§2 (installation de Git par `winget`) et §11 (`npm ci`) **n'ont pas pu être exécutés tels que écrits** : `winget` est absent et aucun Node n'est installé sur le PATH. Les trois dépôts sont clonés, branchés et authentifiés ; les trois `push --dry-run` sont verts ; aucun secret n'est écrit en clair sur le disque.

État final conforme au §0, à l'exception de la visibilité de `projet.git` (voir Bugs trouvés §3 et Points d'attention).

## Fichiers

Créés :

- `RAPPORT-INITIALISATION-MACHINE-2026-11-06.md` (racine) — ce rapport
- `Backend/` — clone de `Repairdom-backend.git`
- `Frontend/` — clone de `Repairdom-frontend.git`
- Contenu de la racine : `.git`, `.gitignore`, `AGENTS.md`, `COMMIT-GUIDE.md`, `CONNEXION-REPOS-GUIDE.md`, `INITIALISATION-MACHINE-GUIDE.md`, 18 `RAPPORT-*.md`, `SERVEUR-1-SASPAY.md`, `SERVEUR-2-BACKEND.md`

Modifiés : aucun fichier versionné existant.

Supprimés / renommés :

- `backend/` (vide) → `Backend/` ; `frontend/` (vide) → `Frontend/`

Le renommage de `backend` en `Backend` a dû passer par un nom intermédiaire (`Backend-TMP`) : Windows ignore la casse dans `mv` sur un même chemin, un `mv backend Backend` laisse le dossier en minuscules.

## Migrations

Aucune migration appliquée ni exécutée. Aucune base n'a été contactée (RÈGLE 3).

## Vérifications

| Étape | Résultat |
|---|---|
| `winget --version` | **échec** — commande introuvable (PowerShell 5.1 et Git Bash) |
| Installation Git (§2) | **non exécutée** — voir Points d'attention |
| `git --version` | `2.52.0.windows.1`, via `C:\Users\Docker\.qoder\bin\git\mingw64\bin\git.exe` |
| `C:\Program Files\Git\cmd\git.exe` | absent — le guide suppose une installation machine |
| `node --version` | **échec** — `node` introuvable dans le PATH bash et PowerShell |
| `npm.cmd --version` | non testé, `npm` Absent puisque Node absent |
| `.git` présent (3 dépôts) | oui |
| `origin` correct (fetch + push) | oui sur les 3 (`projet.git` / `Repairdom-backend.git` / `Repairdom-frontend.git`) |
| Branche `main` + amont `origin/main` | oui sur les 3 |
| `git log --oneline -1` | un commit dans chacun des 3 dépôts |
| `git status -sb` racine | `## main...origin/main` + `?? Backend/` + `?? Frontend/` uniquement |
| `git status -sb` Backend / Frontend | aucun fichier modifié, aucun non suivi |
| `git config --global --list` | `user.name=digitalholding2026-ux`, `user.email=digitalholding2026@gmail.com`, `credential.helper=manager` |
| `git-credential-manager github list` | `digitalholding2026-ux` |
| `git credential fill` (host=github.com) | renvoie un username et un password (valeurs non affichées) |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com`, `User: digitalholding2026-ux` |
| `push --dry-run origin main` ×3 | `Everything up-to-date` sur les 3 |
| `~/.git-credentials` | inexistant |
| `git add Backend` / `git add Frontend` | **jamais exécutés** (RÈGLE 2) |
| `npm ci` backend / frontend (§11) | **non exécutés** — Node absent |
| `tsc`, lint, tests | **non exécutés** — Node absent, donc gate RÈGLE 3 non franchissable |

### Preuve de non-régression

Aucun fichier versionné de `Backend/` ni `Frontend/` n'a été touché : `git status --porcelain` y est vide, et la racine n'affiche que `?? Backend/` et `?? Frontend/`. L'opération est une initialisation à partir du distant ; le HEAD local de chaque dépôt est identique à `origin/main` (confirmé par les trois `push --dry-run` → `Everything up-to-date`).

Aucun test n'a été exécuté : l'environnement d'exécution (§11) n'est pas en place sur cette machine. Rien n'est donc présenté comme vert.

## Bugs trouvés

1. **Aucun bug applicatif recherché ni rencontré** dans cette tâche — le périmètre était l'initialisation, et le code applicatif n'a pas été lu ni exécuté.
2. **§1 et §2 du guide inexécutables sur cette machine.** Le guide suppose `winget` présent et un Git installable dans `C:\Program Files\Git`. Ici `winget` n'existe pas et Node non plus. Le repli utilisé est le Git portable déjà fourni par Qoder (`C:\Users\Docker\.qoder\bin\git\mingw64\bin\git.exe`), qui embarque bien `git-credential-manager.exe` — le §8 fonctionne donc sans modification, à condition de remplacer tous les chemins `C:\Program Files\Git\...` du guide par le dossier `mingw64\bin` de cette installation. Impact : un lecteur qui applique le guide mot à mot échoue dès §1 sur une machine identique à celle-ci.
3. **§0 et §8.3 sur la visibilité de `projet.git` : l'affirmation est fausse.** Le guide écrit que `projet.git` est privé et que, pour cette raison, `ls-remote` ne prouverait rien. Test exécuté ici avec le helper de credentials désactivé (`git -c credential.helper= -c credential.helper= ls-remote`) : le dépôt répond `ca8ab3a…` en accès anonyme, exit 0. Le dépôt est donc **en lecture publique**. La conclusion du §8.3 reste valable — `push --dry-run` est la seule preuve d'écriture — mais le raisonnement qui la justifie (« un accès anonyme ne distinguerait pas ») ne l'est plus.

## Points d'attention

- **Git 2.52.0 (portable Qoder) au lieu de Git 2.56.0 machine.** Conséquences réelles : (a) `git` n'est sur le PATH d'aucun terminal utilisateur — il faut le chemin absolu ou l'ajout au PATH machine ; (b) le contrôle de persistance §2 (`[Environment]::GetEnvironmentVariable("PATH","Machine")` contenant `C:\Program Files\Git\cmd`) échoue puisqu'aucun de ces dossiers n'existe ; (c) `git-bash.exe` et GCM existent mais sous `mingw64\bin`, pas `ucrt64\bin` comme écrit aux §8.1 et §8.2. Une installation machine de Git reste à faire si l'on veut que `git` réponde dans un terminal neuf (§9, case 1). Non faite ici : elle exige un téléchargement et une installation système, hors du repli validé.
- **Node absent (RÈGLE 4 non satisfaite).** Ni `node`, ni `npm`, ni `npm.cmd` ne sont joignables. Les `.nvmrc` des deux dépôts pinnaient Node 22 ; rien n'installe Node sur cette machine. Tout le §11 et toutes les vérifications statiques obligatives avant commit sont donc bloquées. **Conséquence directe : aucun commit backend ou frontend n'est possible en l'état**, et l'initialisation ne doit pas être confondue avec un environnement prêt à travailler.
- **Casse choisie : `Backend/` / `Frontend/`** (majuscules), sur demande explicite, conformément au §0 et au tableau du guide. Le `RAPPORT-REINITIALISATION-MACHINE-2026-10-31.md` Conservait la casse minuscule ; les deux casses coexistent donc dans l'historique des rapports. La racine affiche `?? Backend/` et `?? Frontend/`.
- **Le §5 appliqué malgré une racine paraissant vide.** La demande initiale était un `git clone` direct dans `Relio\`. Cette commande est refusée par Git dès que la cible contient un dossier, même vide — et `Backend/`/`Frontend/` y étaient déjà. La procédure §5 (clone dans `%LOCALAPPDATA%\Temp\projet-clone` puis déplacement incluant `.git` et `.gitignore`) a donc été exécutée. Écart assumé par rapport à la demande, imposé par le comportement de Git.
- **`gcm github login` : exécuté, exit 0, aucune sortie.** Contrairement au rapport du 2026-10-31, qui notait un credential déjà provisionné sans login, cette machine était partie de zéro : `github list` renvoyait vide **avant** la commande, le clone des deux dépôts publics ayant réussi sans invite. Après `login`, `github list` renvoie `digitalholding2026-ux` et la cible `git:https://github.com` apparaît dans le coffre. Le token est stocké par GCM dans le Gestionnaire d'identifiants Windows, jamais en clair.
- **`cmdkey /list` échoue depuis Git Bash** (« The command line parameters are incorrect ») : Git Bash réécrit `/list` en chemin Windows. Fonctionne via `cmd.exe //c "cmdkey /list"`. Piège non documenté au §10 du guide.
- **Vérifications §9 non satisfaites** à ce jour : case 1 (`git` dans un terminal neuf) et la case Node ≥ 22. Les 8 autres cases sont vertes.
- **Non vérifié** : exécution des applications, `prisma generate`, connexion à une base, déploiement Railway/Vercel — hors périmètre ou interdits (RÈGLE 3), et de toute façon impossibles sans Node.

## Questions bloquantes

1. Faut-il installer Node (et quelle version — le `.nvmrc` demande 22) pour débloquer §11 et le gate de la RÈGLE 3, ou l'environnement d'exécution est-il volontairement maintenu hors de cette machine ?
2. Faut-il installer Git en version machine (2.56.x) pour que `git` réponde dans un terminal neuf et que les chemins `C:\Program Files\Git\...` du guide redeviennent exacts ?
3. `projet.git` est en lecture publique alors que le guide le déclare privé : est-ce un état voulu ? Si non, le passer en privé, et corriger §0 et §8.3 dans le guide.
4. Pousser ce rapport sur `projet.git` (`main`) maintenant (RÈGLE 6) ?
