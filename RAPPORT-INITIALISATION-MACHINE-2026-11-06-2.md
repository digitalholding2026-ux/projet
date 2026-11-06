# RAPPORT — Initialisation machine neuve (Relio) — exécution complète du guide

## Statut

- **Date** : 2026-11-06
- **Dépôts concernés** : `projet` (racine), `Repairdom-backend` (`Backend/`), `Repairdom-frontend` (`Frontend/`)
- **HEAD au clone** :
  - racine : `446919b` — *docs(rapport): M1.1 — règle de visibilité par tarif, barème et invariant d'activation*
  - `Backend/` : `7f53974` — *feat(v2): ne servir que les items tarifiés et exiger les deux prix à l'activation*
  - `Frontend/` : `6dadf37` — *feat(v2): retirer le cas « sur devis » du parcours catalogue*
- **Push** : aucun code applicatif poussé. Seul ce rapport est ajouté à la racine (`projet.git`, `main`).
- **Déploiement** : aucun.

## Synthèse

`INITIALISATION-MACHINE-GUIDE.md` exécuté de bout en bout (§1 à §9) sur une machine Windows 10 (19045) sans Git machine, sans Node et **sans winget**. Git 2.56.0.windows.4 et Node v24.22.2 ont été installés via les installateurs officiels directs (écart assumé, voir plus bas), les trois dépôts sont clonés et branchés conformément au §0, l'authentification OAuth GCM est fonctionnelle et les trois `push --dry-run` renvoient `Everything up-to-date`. Les 10 cases de la checklist §9 sont vertes. Ce run lève les questions bloquantes 1 et 2 du `RAPPORT-INITIALISATION-MACHINE-2026-11-06.md` (Node et Git machine désormais installés).

## Fichiers

Créés :

- `RAPPORT-INITIALISATION-MACHINE-2026-11-06-2.md` (racine) — ce rapport
- `Backend/` — clone de `Repairdom-backend.git` (dossier préexistant vide)
- `Frontend/` — clone de `Repairdom-frontend.git` (dossier préexistant vide)
- Contenu de la racine déplacé depuis le clone temporaire : `.git`, `.gitignore`, `AGENTS.md`, `COMMIT-GUIDE.md`, `CONNEXION-REPOS-GUIDE.md`, `INITIALISATION-MACHINE-GUIDE.md`, 19 `RAPPORT-*.md`, `SERVEUR-1-SASPAY.md`, `SERVEUR-2-BACKEND.md`

Hors dépôt (machine) :

- `C:\Program Files\Git\` — Git 2.56.0.windows.4 (installateur officiel `Git-2.56.0.4-64-bit.exe`, `/VERYSILENT /NORESTART`)
- `C:\Program Files\nodejs\` — Node v24.22.2 (`node-v24.22.2-x64.msi`, SHA256 vérifié contre `nodejs.org/dist/latest-v24.x/SHASUMS256.txt`, `msiexec /qn`)
- `C:\Users\Docker\Downloads\relio-setup\` — les deux installateurs téléchargés (conservés)

Modifiés : aucun fichier versionné existant. Supprimés : aucun (dossier temporaire `%LOCALAPPDATA%\Temp\projet-clone` retiré après déplacement, conformément au §5).

## Migrations

Aucune. Aucune base contactée (RÈGLE 3).

## Vérifications

| Étape | Résultat |
|---|---|
| §1 `winget --version` | **échec** — absent (App Installer 1.0.30251.0 de 2019, sans `winget.exe`) |
| §1 `node` / `npm.cmd` (départ) | **absents** du PATH bash et PowerShell |
| §1 `git` (départ) | absent du système ; seul le portable Qoder 2.52.0 (`~/.qoder/bin/git`) répondait |
| §2 Installation Git | **OK** — `git version 2.56.0.windows.4` (installateur direct, voir Écarts) |
| §2 PATH machine persistant | `C:\Program Files\Git\cmd` présent dans le PATH machine |
| §3 Arborescence | `Backend\` et `Frontend\` préexistants et vides, casse majuscule conservée |
| §4 Clones Backend / Frontend | OK — `.git` présents, `origin` correct (fetch + push), HEAD présents |
| §5 Racine `projet.git` | clone temporaire puis déplacement de tout le contenu, `.git` et `.gitignore` inclus (`dotglob`) ; pas de `git init` |
| §6 `origin` ×3 | `projet.git` / `Repairdom-backend.git` / `Repairdom-frontend.git` |
| §6 Branche + amont ×3 | `main...origin/main` |
| §6 `git status -sb` racine | `## main...origin/main` + `?? Backend/` + `?? Frontend/` uniquement |
| §6 `git add Backend` / `git add Frontend` | **jamais exécutés** (RÈGLE 2) |
| §7 `git config --global --list` | `user.name=digitalholding2026-ux`, `user.email=digitalholding2026@gmail.com`, `credential.helper=manager` |
| §8.1 GCM présent | `C:\Program Files\Git\ucrt64\bin\git-credential-manager.exe` — oui |
| §8.2 `github login` | exécuté avec Git sur le PATH, exit 0, aucune sortie (autorisation navigateur effectuée par l'utilisateur) |
| §8.3 `github list` | `digitalholding2026-ux` |
| §8.3 Coffre Windows | `LegacyGeneric:target=git:https://github.com` présent |
| §8.3 `push --dry-run origin main` ×3 | `Everything up-to-date` sur les 3 |
| §9 `git` dans un terminal neuf | `2.56.0.windows.4` via `C:\Program Files\Git\cmd\git.exe` (PATH machine+utilisateur rechargé) |
| §9 Node ≥ 22 | `v24.22.2`, `npm.cmd` 11.19.1, `C:\Program Files\nodejs\` au PATH machine |
| §9 `~/.git-credentials` | **inexistant** — aucun secret en clair |
| §11 `npm ci` ×2 | **non exécutés** — hors périmètre du guide (« étapes suivantes ») |
| `tsc`, lint, tests | **non exécutés** — aucun code modifié ; le gate RÈGLE 3 vise les commits applicatifs, pas ce rapport documentaire |

### Preuve de non-régression

Aucun fichier versionné de `Backend/` ni `Frontend/` n'a été touché : clones frais, `git status -sb` sans modification, HEAD locaux identiques à `origin/main` (confirmé par les trois `push --dry-run` → `Everything up-to-date`). À la racine, seuls `?? Backend/` et `?? Frontend/` apparaissent avant ajout de ce rapport.

## Bugs trouvés

1. **Aucun bug applicatif** — périmètre d'initialisation, code non lu ni exécuté.
2. **§2 du guide inexécutable tel quel sur cette machine : `winget` absent.** Le paquet `Microsoft.DesktopAppInstaller` installé est la version 1.0.30251.0 (2019), antérieure à winget, et ne contient pas `winget.exe`. Impact : un lecteur qui applique le guide mot à mot échoue dès §2. Repli appliqué : installateurs officiels directs (voir Écarts).
3. **Confirmation du bug n°3 du rapport précédent (`projet.git` en lecture publique).** Le clone du §5 a réussi **sans aucune authentification** (coffre vide et `github list` vide à ce moment) : la lecture anonyme est donc possible, contrairement au §0 qui déclare le dépôt privé. La conclusion du §8.3 reste valable (`push --dry-run` seule preuve d'écriture), mais sa justification par la confidentialité ne l'est pas.

## Écarts assumés par rapport au guide

| Point | Guide | Appliqué | Raison |
|---|---|---|---|
| Installation Git (§2) | `winget install --id Git.Git` | Installateur officiel `Git-2.56.0.4-64-bit.exe` (github.com/git-for-windows/git, release v2.56.0.windows.4), `/VERYSILENT /NORESTART` | winget absent de la machine |
| Version Git | 2.56.0.3 (vu le 2026-10-27) | 2.56.0.windows.4 | dernière release stable au jour de l'exécution |
| Node (§1 suppose présent) | non installé par le guide | Node v24.22.2 LTS installé (MSI officiel nodejs.org, SHA256 vérifié) | Node absent ; RÈGLE 4 et checklist §9 exigent Node ≥ 22 ; lève la question bloquante 1 du rapport précédent |
| Version Node | v24.22.1 (vu le 2026-10-27) | v24.22.2 | dernier patch de la branche v24 LTS |
| Shell | PowerShell | Git Bash (MSYS) | shell de l'environnement d'exécution ; pièges PowerShell du §10 sans objet, sauf `cmdkey /list` (voir Points d'attention) |

## Points d'attention

- **`git` dans un « terminal neuf »** : vérifié en rechargeant explicitement le PATH machine+utilisateur dans un nouveau processus PowerShell. Les processus déjà lancés avant l'installation (dont le shell de cette session) voient encore l'ancien PATH — le portable Qoder 2.52.0 y reste prioritaire. Comportement attendu, documenté au §2 et §10.4 du guide.
- **`cmdkey /list` depuis Git Bash** : échoue (réécriture MSYS de `/list` en chemin). Utiliser `cmd.exe //c "cmdkey /list"` — confirme le piège déjà signalé dans le rapport précédent, non documenté au §10 du guide.
- **`github list` vide avant login** : contrairement à un poste déjà provisionné, le coffre était vide ; le clone des deux dépôts publics avait réussi anonymement, et seul le §8 a provisionné le credential d'écriture. Le token est stocké chiffré par GCM dans le Gestionnaire d'identifiants Windows, jamais en clair.
- **Visibilité de `projet.git`** : lecture publique confirmée (clone anonyme réussi). Si la confidentialité est voulue, passer le dépôt en privé sur GitHub et corriger §0/§8.3 du guide — décision utilisateur, non effectuée ici.
- **§11 non exécuté** : `npm ci` backend/frontend reste à faire avant toute vérification de code ; les `.env` devront être créés depuis `.env.example`, jamais commités. `next build`, `npm run dev`, Docker et Postgres local restent interdits (RÈGLE 3).
- **Installateurs conservés** dans `C:\Users\Docker\Downloads\relio-setup\` (~98 Mo) — supprimables sans impact.

## Questions bloquantes

Aucune. (Les questions 1 et 2 du rapport du même jour sont levées par ce run ; la question 3 — visibilité de `projet.git` — reste une décision utilisateur, non bloquante.)