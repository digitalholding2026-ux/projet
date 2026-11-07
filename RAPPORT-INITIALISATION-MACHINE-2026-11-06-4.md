# RAPPORT — Initialisation machine neuve (2026-11-06, exécution 4)

## Statut

- **Date** : 2026-11-06
- **Dépôts concernés** : `projet.git` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **État** : procédure `INITIALISATION-MACHINE-GUIDE.md` exécutée de bout en bout ; checklist §9 **10/10 verte** ; rapport committé et poussé sur `projet.git` (`main`)
- **HEAD au moment du clone** :
  - racine `projet.git` : `8d3683b` — « docs(rapport): initialisation machine 2026-11-06 (3) — Git 2.56.0.4 via installateur officiel (winget absent), 3 dépôts branchés, OAuth GCM vérifié »
  - `Backend` : `00821a8` — « fix(deploy): builder Railpack pour que la CLI Nest survive à l'installation »
  - `Frontend` : `6dadf37` — « feat(v2): retirer le cas « sur devis » du parcours catalogue »

## Synthèse

Les trois dépôts imbriqués ont été clonés et branchés sur `main...origin/main`
dans des dossiers `Backend\` et `Frontend\` initialement vides. Git for Windows
**2.56.0.4** était déjà installé (étape 2 réduite à une vérification) ; Node.js,
absent de la machine, a été installé en **v24.22.2 LTS** via le MSI officiel
(puisque `winget` est absent). L'identité Git globale a été créée (§7) et
l'authentification GitHub OAuth via Git Credential Manager était **déjà
fonctionnelle** (compte présent dans le coffre Windows) — preuve :
`push --dry-run` → `Everything up-to-date` sur les trois dépôts.

## Fichiers

- **Créés** : `C:\Users\Docker\Desktop\Relio\` (dépôt racine + documents),
  `Backend\` (287 fichiers trackés), `Frontend\` (324 fichiers trackés), chacun
  avec son `.git`.
- **Créés (hors dépôt)** : `C:\Program Files\nodejs\` (installation système
  Node v24.22.2), `C:\Users\Docker\.gitconfig` (identité globale), installateur
  temporaire `%LOCALAPPDATA%\Temp\node-v24.22.2-x64.msi` (33 333 248 octets,
  SHA-256 vérifié avant exécution).
- **Modifiés (hors dépôt)** : PATH machine (entrée `C:\Program Files\nodejs\`
  ajoutée par le MSI) ; aucune modification du Gestionnaire d'identifiants
  Windows (entrée `git:https://github.com` préexistante, réutilisée telle quelle).
- **Supprimé** : dossier temporaire `%LOCALAPPDATA%\Temp\projet-clone` après
  déplacement de son contenu (§5).
- **Aucun** fichier de code modifié ; `npm ci` non exécuté (§11, hors procédure).

## Migrations

Aucune.

## Vérifications

Checklist §9 du guide — résultats réels :

| Contrôle | Résultat |
|---|---|
| `git --version` dans un terminal neuf (PATH reconstruit Machine+User) | `git version 2.56.0.windows.4`, résolu depuis `C:\Program Files\Git\cmd\git.exe` |
| `.git` présent dans les 3 dossiers | Oui (vérifié par `test -d` + `git log --oneline -1`) |
| `origin` correct (fetch et push) sur les 3 | `projet.git` / `Repairdom-backend.git` / `Repairdom-frontend.git` |
| Branche `main` + amont `origin/main` | `## main...origin/main` sur les trois |
| `git status -sb` racine | `?? Backend/` et `?? Frontend/` uniquement |
| Identité globale | `user.name=digitalholding2026-ux`, `user.email=digitalholding2026@gmail.com`, `credential.helper=manager` |
| `git-credential-manager github list` | `digitalholding2026-ux` |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com` présent |
| `push --dry-run origin main` | `Everything up-to-date` sur **les trois** (preuve lecture+écriture) |
| `~/.git-credentials` | Inexistant (aucun secret en clair) |
| Node ≥ 22 | `v24.22.2` (LTS Krypton), npm `11.19.1` — installés pendant cette exécution ; résolus depuis `C:\Program Files\nodejs\` dans un terminal neuf |

Contrôles complémentaires (RÈGLE 5) :

- `git ls-files` racine : aucun fichier `*.pem`, `.env` ou `connexion*.txt` tracké.
- MSI Node : SHA-256 `9231bf64d19bef391dd5ee8af65411d3799c3c44a842d2320849cc1da60b6b6d`
  conforme à `SHASUMS256.txt` officiel (`sha256sum -c` → OK) ; `msiexec /qn /norestart` exit 0.

## Bugs trouvés

Aucun bug applicatif (aucun code touché). Obstacles d'environnement rencontrés
et résolus :

1. **`winget` absent** (introuvable dans le PATH et dans
   `%LOCALAPPDATA%\Microsoft\WindowsApps`). → Comme à l'exécution 3 :
   installateur officiel téléchargé depuis `nodejs.org/dist`, empreinte
   vérifiée, installation silencieuse.
2. **Node.js absent** alors que l'exécution 3 du même jour le voyait en
   v26.13.0 : ni `node.exe` dans les emplacements usuels, ni entrée Node dans
   le PATH registre Machine+User, ni gestionnaire nvm/fnm/Volta. La machine a
   donc été partiellement réinitialisée entre les deux exécutions (dossiers
   Relio vidés, Node retiré ; Git et le coffre GCM ont survécu). → Installation
   de Node v24.22.2 LTS (choix : LTS plutôt que la v26 vue précédemment ;
   l'exigence RÈGLE 4 est ≥ 22).
3. **Fichier `.gitconfig` global inexistant** (`fatal: unable to read config
   file`). → Créé par les trois commandes §7, vérifié par `git config --global --list`.

## Écarts par rapport au guide (explicitement notés)

| Point | Guide | Exécution réelle | Raison |
|---|---|---|---|
| Installation Git (§2) | `winget install Git.Git` (2.56.0.3) | Non exécutée : Git **2.56.0.4 déjà installé** et présent au PATH machine | Étape réduite à une vérification ; `winget` de toute façon absent |
| Node / npm (§1) | Présents, ≥ 22 (vu v24.22.1 / 11.19.1) | **Absents** → installés v24.22.2 LTS / 11.19.1 via MSI officiel nodejs.org, SHA-256 vérifié | Le guide suppose Node présent mais ne décrit pas son installation ; exigence checklist §9 « Node ≥ 22 » satisfaite en fin de procédure |
| Shell | PowerShell | Git Bash (session Qoder) | Environnement de l'agent ; §5 adapté : `shopt -s dotglob; mv` (équivalent de `Move-Item -Force`), `.git` et `.gitignore` bien déplacés (vérifié), dossier temporaire supprimé par `rmdir` |
| Connexion GCM (§8.2) | `git-credential-manager github login` avec autorisation navigateur | **Non exécutée** : `github list` renvoyait déjà `digitalholding2026-ux` et `cmdkey /list` montrait `git:https://github.com` | Identifiant OAuth déjà dans le coffre Windows (hérité de l'exécution 3) ; la preuve §8.3 (`push --dry-run` ×3) est verte, l'objectif fonctionnel est atteint sans reconnexion |
| Clone `projet.git` (§5) | Dépôt décrit comme privé | Clone réussi sans invite interactive | Les identifiants GCM préexistants ont été utilisés ; aucune modification de visibilité côté GitHub |
| Dossiers `Backend`/`Frontend` (§3) | À créer | Déjà présents et vides | Étape 3 réduite à une vérification ; casse `Backend`/`Frontend` respectée |
| Version Git | 2.56.0.3 | 2.56.0.windows.4 | Version déjà installée (installateur officiel de l'exécution 3), plus récente que celle du guide |

## Points d'attention

- `npm ci` (§11) **non exécuté** : volontairement hors procédure. Les `.env`
  n'ont pas été créés (hors procédure également).
- Aucune vérification de code (`tsc`, lint, tests) : aucun code modifié.
- Le commit de ce rapport est le seul commit de la tâche, sur la racine
  uniquement (RÈGLE 2 respectée : `Backend/` et `Frontend/` jamais indexés,
  visibles comme `??` dans `git status -sb`).
- L'installateur MSI temporaire est laissé dans `%LOCALAPPDATA%\Temp\`
  (nettoyage OS) ; aucun secret sur disque en clair (`~/.git-credentials`
  absent, token dans le coffre Windows).
- Divergence mineure : le guide (§13) demande de consigner la version de Git
  « installée » — aucune installation Git n'a eu lieu ici, la version
  **présente** (2.56.0.4) est consignée à la place.

## Questions bloquantes

Aucune.