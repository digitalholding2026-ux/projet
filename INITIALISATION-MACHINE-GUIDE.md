# Guide d'initialisation complète — Relio sur machine neuve

> **À exécuter de bout en bout, dans l'ordre, avant toute autre chose.**
> Ce document décrit **exactement** la procédure qui a été exécutée et validée
> le **2026-10-27** sur une machine Windows sans Git installé, avec trois
> dossiers vides en attente de leurs dépôts.
>
> Il complète `CONNEXION-REPOS-GUIDE.md` : ce dernier explique la **topologie**
> des dépôts et la reconnexion d'un workspace **déjà présent** ; celui-ci part
> de **zéro** (machine vierge, Git absent, dossiers vides) et va jusqu'à
> l'authentification fonctionnelle.

## 0. Résultat attendu

À la fin, la machine présente cette arborescence, avec **trois dépôts Git
indépendants imbriqués** :

```
Relio\                                  → dépôt `projet.git`     (main) — documentation uniquement
├─ Backend\                             → dépôt `Repairdom-backend`   (main) — NestJS / Prisma
└─ Frontend\                            → dépôt `Repairdom-frontend`  (main) — Next.js / TypeScript
```

| Dossier | Remote `origin` | Visibilité | Branche |
|---|---|---|---|
| `Relio\` | `https://github.com/digitalholding2026-ux/projet.git` | privé | `main` |
| `Relio\Backend\` | `https://github.com/digitalholding2026-ux/Repairdom-backend.git` | public | `main` |
| `Relio\Frontend\` | `https://github.com/digitalholding2026-ux/Repairdom-frontend.git` | public | `main` |

État `git status -sb` attendu **à la racine** à la fin :

```
## main...origin/main
?? Backend/
?? Frontend/
```

`?? Backend/` et `?? Frontend/` sont **normaux et souhaités** (RÈGLE 2 de
`AGENTS.md`). Ces deux dossiers ne doivent **jamais** être ajoutés à l'index de
la racine.

---

## 1. Vérifier l'environnement de départ

```powershell
winget --version                     # gestionnaire de paquets Windows
node --version                       # doit être >= 22 (RÈGLE 4) — vu : v24.22.1
npm.cmd --version                    # vu : 11.19.1
Get-Command git -ErrorAction SilentlyContinue
```

Sur une machine neuve, `Get-Command git` ne renvoie **rien** : Git n'est pas
installé. C'est le cas normal, l'étape 2 le corrige.

> ⚠️ Utiliser **`npm.cmd`**, pas `npm`. Sur Windows, `npm` résout vers
> `npm.ps1`, et l'ExecutionPolicy par défaut bloque les scripts PowerShell :
> « *Impossible de charger le fichier npm.ps1, car l'exécution de scripts est
> désactivée sur ce système* ». `npm.cmd` contourne le problème sans changer
> la stratégie système.

---

## 2. Installer Git

```powershell
winget install --id Git.Git -e --source winget --accept-package-agreements --accept-source-agreements --silent
```

Attendu : `Successfully installed` (vu : **Git 2.56.0.3**).

Vérifier, et **réparer le PATH de la session courante** (obligatoire : le shell
existant a été lancé avant l'installation, il ne voit pas encore `git`) :

```powershell
$git = "C:\Program Files\Git\cmd\git.exe"
& $git --version
# si vous préférez utiliser `git` sans chemin complet dans la session en cours :
$env:PATH = "C:\Program Files\Git\cmd;C:\Program Files\Git\ucrt64\bin;" + $env:PATH
```

L'installateur ajoute `C:\Program Files\Git\cmd` au **PATH machine** : les
terminaux ouverts **après** l'installation disposent donc de `git`
normalement. Seuls les processus déjà lancés doivent être relancés ou patchés
comme ci-dessus.

Contrôle de persistance :

```powershell
[Environment]::GetEnvironmentVariable("PATH","Machine") -split ';' | Where-Object { $_ -like "*Git*" }
# doit afficher : C:\Program Files\Git\cmd
```

---

## 3. Préparer l'arborescence

Créer le dossier de travail et les deux sous-dossiers **vides** :

```powershell
$root = "C:\Users\<utilisateur>\Desktop\Relio"
New-Item -ItemType Directory -Force -Path $root, "$root\Backend", "$root\Frontend"
Get-ChildItem -Force $root     # doit être vide (ou ne contenir que les 2 dossiers vides)
```

> Respecter la casse `Backend` / `Frontend` si elle est déjà imposée par
> l'utilisateur : c'est la casse réellement utilisée sur cette machine
> (le `CONNEXION-REPOS-GUIDE.md` historique écrit `backend`/`frontend` en
> minuscules ; le contenu et le remote sont identiques).

---

## 4. Cloner `Backend` et `Frontend`

Un dossier **vide** est acceptable comme cible de `git clone` : le dépôt est
créé directement à la racine du dossier, sans sous-dossier intermédiaire.

```powershell
$git  = "C:\Program Files\Git\cmd\git.exe"
$root = "C:\Users\<utilisateur>\Desktop\Relio"

& $git clone https://github.com/digitalholding2026-ux/Repairdom-backend.git  "$root\Backend"
& $git clone https://github.com/digitalholding2026-ux/Repairdom-frontend.git "$root\Frontend"
```

**Vérification immédiate** (ne pas se fier au code de sortie, voir §10) :

```powershell
& $git -C "$root\Backend"  remote -v
& $git -C "$root\Frontend" remote -v
Test-Path "$root\Backend\.git"      # True
Test-Path "$root\Frontend\.git"     # True
```

---

## 5. Brancher la racine `projet.git` **sans écraser** les sous-dossiers

`git clone <url> .` **échoue** quand la cible contient déjà des fichiers, même
si le dépôt racine n'a aucun conflit avec eux. La procédure correcte est :
cloner dans un dossier temporaire, puis **déplacer tout le contenu, y compris
les fichiers cachés (`.git`, `.gitignore`)**.

```powershell
$git  = "C:\Program Files\Git\cmd\git.exe"
$root = "C:\Users\<utilisateur>\Desktop\Relio"
$tmp  = "$env:LOCALAPPDATA\Temp\projet-clone"

if (Test-Path $tmp) { Remove-Item -Recurse -Force $tmp }
& $git clone https://github.com/digitalholding2026-ux/projet.git $tmp

# -Force est indispensable : .git et .gitignore sont des éléments cachés
Get-ChildItem -Force $tmp | Move-Item -Destination $root -Force
Remove-Item -Recurse -Force $tmp
```

> **Ne pas** exécuter `git init` à la racine : cela créerait un dépôt sans
> historique, incompatible avec `projet.git`. On déplace le `.git` cloné.

À ce stade la racine contient : `AGENTS.md`, `COMMIT-GUIDE.md`,
`CONNEXION-REPOS-GUIDE.md`, les `RAPPORT-*.md`, plus `Backend\` et
`Frontend\` qui leur sont étrangers.

---

## 6. Vérifier le branchement des trois dépôts

```powershell
$git  = "C:\Program Files\Git\cmd\git.exe"
$root = "C:\Users\<utilisateur>\Desktop\Relio"

foreach ($d in @($root, "$root\Backend", "$root\Frontend")) {
  "== $d =="
  & $git -C $d remote -v
  & $git -C $d status -sb
  & $git -C $d log --oneline -1
}
```

Critères d'acceptation :

- les 3 dossiers ont un `origin` correct (fetch **et** push) ;
- les 3 branches sont `main` avec un amont `origin/main` ;
- la racine affiche `?? Backend/` et `?? Frontend/` ;
- les 3 `git log --oneline -1` renvoient un commit (historique présent, donc
  `.git` correctement déplacé).

### ⛔ Interdit (RÈGLE 2 de `AGENTS.md`)

Ne jamais exécuter `git add Backend` ni `git add Frontend` à la racine : ils y
apparaîtraient comme *gitlinks* (sous-modules implicites) et casseraient le
modèle « un dépôt par domaine ».
Ne jamais committer de code à la racine : elle n'accepte que des documents
(`*.md`).

---

## 7. Configurer l'identité Git (globale)

Une seule fois, valable pour les trois dépôts (aucun `--local` nécessaire) :

```powershell
$git = "C:\Program Files\Git\cmd\git.exe"
& $git config --global user.name  "digitalholding2026-ux"
& $git config --global user.email "digitalholding2026@gmail.com"
& $git config --global credential.helper manager
& $git config --global --list
```

Attendu :

```
user.name=digitalholding2026-ux
user.email=digitalholding2026@gmail.com
credential.helper=manager
```

---

## 8. Authentifier Git (OAuth via Git Credential Manager)

Git pour Windows embarque **Git Credential Manager (GCM)**, qui sait
s'authentifier auprès de GitHub par OAuth navigateur. C'est la méthode
retenue : **aucun token n'est écrit dans un fichier**, il est stocké chiffré
par l'OS dans le Gestionnaire d'identifiants Windows.

### 8.1 Vérifier que GCM est présent

```powershell
Test-Path "C:\Program Files\Git\ucrt64\bin\git-credential-manager.exe"
```

### 8.2 Lancer la connexion

```powershell
$env:PATH = "C:\Program Files\Git\cmd;C:\Program Files\Git\ucrt64\bin;" + $env:PATH
& "C:\Program Files\Git\ucrt64\bin\git-credential-manager.exe" github login
```

> ⚠️ **GCM exige `git.exe` sur le PATH.** Sans lui, l'erreur est :
> `System.Exception: Failed to locate 'git.exe' executable on the path.`
> D'où la ligne `$env:PATH` ci-dessus. Un `git.exe` fraîchement installé n'est
> pas encore visible dans un shell déjà ouvert.

Comportement : une fenêtre de navigateur s'ouvre (ou un code d'appareil est
affiché). **L'utilisateur doit autoriser le compte `digitalholding2026-ux`**
dans le navigateur. La commande ne rend la main qu'après l'autorisation et
n'affiche **aucune sortie** en cas de succès — l'absence de message n'est pas
un échec.

### 8.3 Vérifier l'authentification

```powershell
$gcm = "C:\Program Files\Git\ucrt64\bin\git-credential-manager.exe"
& $gcm github list                       # doit lister : digitalholding2026-ux
cmdkey /list | Select-String "github"    # doit montrer : git:https://github.com
```

Puis la **preuve réelle** — droits de lecture et d'écriture sur les 3 dépôts :

```powershell
$git  = "C:\Program Files\Git\cmd\git.exe"
$root = "C:\Users\<utilisateur>\Desktop\Relio"
foreach ($d in @($root, "$root\Backend", "$root\Frontend")) {
  "== $d =="
  & $git -C $d push --dry-run origin main 2>&1
}
```

Attendu sur les trois : `Everything up-to-date`. C'est la seule preuve
acceptée — `git ls-remote` ne prouve que la lecture, et `projet.git` étant
privé, un simple `ls-remote` ne distinguerait pas un accès anonyme réussi.
`push --dry-run` **écrit** une négociation avec le serveur sans modifier
l'historique.

---

## 9. Checklist finale

- [ ] `git --version` répond (Git 2.56.x) dans un terminal neuf
- [ ] `Relio\`, `Relio\Backend\`, `Relio\Frontend\` ont chacun un `.git`
- [ ] `origin` = `projet.git` / `Repairdom-backend.git` / `Repairdom-frontend.git`
- [ ] Branche `main` + amont `origin/main` sur les trois
- [ ] `git status -sb` à la racine : `?? Backend/` et `?? Frontend/` uniquement
- [ ] `user.name`, `user.email`, `credential.helper=manager` configurés
- [ ] `git-credential-manager github list` → `digitalholding2026-ux`
- [ ] `push --dry-run` → `Everything up-to-date` sur les trois
- [ ] `~/.git-credentials` **inexistant** (aucun secret en clair)
- [ ] Node ≥ 22 (`node --version`)

---

## 10. Pièges PowerShell rencontrés (et leur explication)

1. **`git clone` affiche une erreur alors qu'il réussit.**
   Git écrit sa progression sur *stderr* (`Cloning into '...'`). PowerShell la
   transforme en `NativeCommandError` avec `Exited with code 1`, même quand le
   clone est parfait. **Toujours vérifier `Test-Path "$dossier\.git"` et
   `git log`** au lieu du code de sortie.
2. **`git clone <url> <dossier non vide>` échoue** — quel que soit le contenu.
   Passer par un dossier temporaire et déplacer (§5).
3. **`npm` bloqué par l'ExecutionPolicy** — utiliser `npm.cmd` (§1).
4. **`git` non reconnu juste après l'installation** — le PATH machine n'est lu
   qu'au démarrage d'un nouveau processus ; patcher `$env:PATH` (§2).
5. **`git-credential-manager` sans `git` sur le PATH** — échec
   `Failed to locate 'git.exe'` (§8.2).
6. **`Move-Item $tmp\*` sans `-Force`** oublie `.git` et `.gitignore` : la
   racine se retrouverait sans dépôt (§5).

---

## 11. Étapes suivantes (non exécutées par cette procédure)

```powershell
npm.cmd ci --prefix "C:\Users\<utilisateur>\Desktop\Relio\Backend"
npm.cmd ci --prefix "C:\Users\<utilisateur>\Desktop\Relio\Frontend"
```

À faire avant toute vérification de code. Rappels des règles projet :

- `next build`, `npm run dev`, Docker et Postgres local sont **interdits en
  local** (RÈGLE 3) ;
- autorisé : `oxlint`, `tsc --noEmit`, `vitest run` sur tests unitaires purs ;
- chaque dépôt applicatif s'arrête sur ses `.env` créés depuis `.env.example`
  — **jamais** commités.

---

## 12. Écarts assumés par rapport à `CONNEXION-REPOS-GUIDE.md`

| Point | Guide historique | Procédure appliquée | Raison |
|---|---|---|---|
| Authentification | PAT + `credential.helper = store` (token en clair dans `~/.git-credentials`) | OAuth GCM + `credential.helper = manager` (token dans le coffre Windows) | Même résultat fonctionnel, sans secret en clair sur le disque |
| Casse des dossiers | `backend/`, `frontend/` | `Backend/`, `Frontend/` | Casse imposée par l'arborescence existante de la machine |
| Repro `Repairdom.git` | déclaré inexistant | idem, non utilisé | Le dépôt racine est bien `projet.git` |

Le reste du guide historique (topologie, interdiction d'indexer les
sous-dossiers, ordre de push backend → frontend) reste applicable tel quel.

---

## 13. À faire après cette procédure

Rédiger et pousser sur `projet.git` le rapport de la tâche (RÈGLE 1 de
`AGENTS.md`), en y consignant : version de Git installée, HEAD de chaque
dépôt au moment du clone, résultat des `push --dry-run`, et tout écart
constaté.

---

*Document écrit le 2026-10-27. Toute divergence entre ce guide et l'état réel
de la machine doit être signalée dans un `RAPPORT-*.md` à la racine.*
