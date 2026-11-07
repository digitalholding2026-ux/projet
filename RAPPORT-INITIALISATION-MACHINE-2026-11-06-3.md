# RAPPORT — Initialisation machine neuve (2026-11-06, exécution 3)

## Statut

- **Date** : 2026-11-06
- **Dépôts concernés** : `projet.git` (racine), `Repairdom-backend`, `Repairdom-frontend`
- **État** : procédure `INITIALISATION-MACHINE-GUIDE.md` exécutée de bout en bout ; checklist §9 **10/10 verte** ; rapport poussé sur `projet.git` (`main`)
- **HEAD au moment du clone** :
  - racine `projet.git` : `29b1a80` — « docs(rapport): initialisation machine neuve — Git 2.56.0.4 et Node 24.22.2 installés, 3 dépôts branchés et authentifiés »
  - `Backend` : `7f53974` — « feat(v2): ne servir que les items tarifiés et exiger les deux prix à l'activation »
  - `Frontend` : `6dadf37` — « feat(v2): retirer le cas « sur devis » du parcours catalogue »

## Synthèse

Machine réinitialisée avec Git for Windows **2.56.0.4** installé silencieusement,
les trois dépôts imbriqués clonés et branchés sur `main...origin/main`, identité
Git globale configurée et authentification GitHub OAuth via Git Credential
Manager fonctionnelle (preuve : `push --dry-run` → `Everything up-to-date` sur
les trois dépôts).

## Fichiers

- **Créés** : `C:\Users\Docker\Desktop\Relio\` (dépôt racine + documents),
  `Backend\` (284 fichiers), `Frontend\` (324 fichiers), chacun avec son `.git`.
- **Créé (hors dépôt)** : `C:\Program Files\Git\` (installation système),
  installateur temporaire `%LOCALAPPDATA%\Temp\Git-2.56.0.4-64-bit.exe`.
- **Modifiés (hors dépôt)** : config Git globale (`user.name`, `user.email`,
  `credential.helper=manager`) ; Gestionnaire d'identifiants Windows
  (`git:https://github.com`, chiffré par l'OS).
- **Aucun** fichier de code modifié ; `npm ci` non exécuté (§11, hors procédure).

## Migrations

Aucune.

## Vérifications

Checklist §9 du guide — résultats réels :

| Contrôle | Résultat |
|---|---|
| `git --version` dans un terminal neuf (PATH reconstruit Machine+User) | `git version 2.56.0.windows.4`, résolu depuis `C:\Program Files\Git\cmd\git.exe` |
| `.git` présent dans les 3 dossiers | Oui (vérifié par `ls -d` + `git log`) |
| `origin` correct (fetch et push) sur les 3 | `projet.git` / `Repairdom-backend.git` / `Repairdom-frontend.git` |
| Branche `main` + amont `origin/main` | `## main...origin/main` sur les trois |
| `git status -sb` racine | `?? Backend/` et `?? Frontend/` uniquement |
| Identité globale | `user.name=digitalholding2026-ux`, `user.email=digitalholding2026@gmail.com`, `credential.helper=manager` |
| `git-credential-manager github list` | `digitalholding2026-ux` |
| `cmdkey /list` | `LegacyGeneric:target=git:https://github.com` présent |
| `push --dry-run origin main` | `Everything up-to-date` sur **les trois** (preuve lecture+écriture) |
| `~/.git-credentials` | Inexistant (aucun secret en clair) |
| Node ≥ 22 | `v26.13.0`, npm `11.20.0` (déjà présents, aucune installation) |

## Bugs trouvés

Aucun bug applicatif (aucun code touché). Obstacles d'environnement rencontrés
et résolus :

1. **`winget` absent** (bash et PowerShell ; App Installer 1.0.30251.0 sans
   `winget.exe`). → Installateur officiel Git 2.56.0.4 téléchargé depuis les
   releases GitHub `git-for-windows/git` et installé en
   `/VERYSILENT /NORESTART /SUPPRESSMSGBOXES /SP-` (exit 0).
2. **Git 2.52.0 résolu dans les terminaux enfants de Qoder** : le PATH
   *processus* de Qoder place son Git embarqué (`~/.qoder/bin/git`) en premier.
   Le PATH **Machine** contient bien `C:\Program Files\Git\cmd` ; un terminal
   réellement neuf (hors Qoder) résout 2.56.0.4 — vérifié en reconstruisant le
   PATH depuis les valeurs registre Machine+User.
3. **`cmdkey /list | grep` muet depuis Git Bash** → vérification faite via
   PowerShell (`Select-String`), entrée bien présente.

## Écarts par rapport au guide (explicitement notés)

| Point | Guide | Exécution réelle | Raison |
|---|---|---|---|
| Installation Git | `winget install Git.Git` (2.56.0.3) | Installateur officiel GitHub, silencieux (2.56.0.4) | `winget` absent de la machine ; même éditeur, même canal de distribution officiel, version plus récente |
| Shell | PowerShell | Git Bash (session Qoder) | Environnement de l'agent ; commandes adaptées (`shopt -s dotglob; mv` pour le déplacement §5, équivalent de `Move-Item -Force` — `.git` et `.gitignore` bien déplacés, vérifié) |
| Node / npm | v24.22.1 / 11.19.1 | v26.13.0 / 11.20.0 | Déjà installés sur cette machine ; exigence ≥ 22 satisfaite |
| Dossiers `Backend`/`Frontend` | À créer (§3) | Déjà présents et vides | Étape 3 réduite à une vérification |
| Clone `projet.git` (§5) | Dépôt décrit comme privé | Clone réussi **sans authentification**, avant la connexion GCM | À ce moment aucun identifiant GitHub n'était en cache : le dépôt était donc lisible anonymement. Signalé comme observation factuelle ; la visibilité réelle du dépôt n'a pas été modifiée ni vérifiée côté GitHub |
| Ordre auth (§8) | Après les clones | Identique ; `github login` a ouvert le navigateur, autorisé par l'utilisateur, exit 0 sans sortie (conforme au guide) | — |

## Points d'attention

- Les terminaux lancés **depuis Qoder** héritent de son Git embarqué 2.52.0 ;
  les commandes de cette procédure ont toutes utilisé le chemin complet
  `C:\Program Files\Git\cmd\git.exe`. Hors Qoder, `git` résout 2.56.0.4.
- `npm ci` (§11) **non exécuté** : volontairement hors procédure.
- Aucune vérification de code (`tsc`, lint, tests) : aucun code modifié.
- Le commit de ce rapport est le seul commit de la tâche, sur la racine
  uniquement (RÈGLE 2 respectée : `Backend/` et `Frontend/` jamais indexés).

## Questions bloquantes

Aucune.