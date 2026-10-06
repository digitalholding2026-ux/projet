# Guide de reconnexion des dossiers à leurs dépôts (nouvelle machine)

> À lire **en premier**, AVANT de toucher au code. Le but : rebrancher chaque
> dossier du projet à son dépôt Git respectif sur la nouvelle machine.

## 1. Vue d'ensemble — 3 dépôts, 3 dossiers

Le projet est découpé en **3 dépôts Git indépendants** (pattern « repos imbriqués »
dans un même espace de travail). Chaque dossier porte **son propre `.git`**.

| Dossier          | Remote `origin`                                              | Visibilité | Branche | HEAD (commit) attendu |
|------------------|--------------------------------------------------------------|------------|---------|-----------------------|
| `backend/`       | `https://github.com/digitalholding2026-ux/Repairdom-backend.git` | **public** | `main`  | `641df65` — `feat: cancel and reschedule mission routes` |
| `frontend/`      | `https://github.com/digitalholding2026-ux/Repairdom-frontend.git` | **public** | `main`  | `2d9c0aa` — `feat: dépôt de panne avec validation prix + technicien auto-assigné` |
| `projet/` (racine) | `https://github.com/digitalholding2026-ux/Repairdom.git`      | **privé** | `main`  | `9c034a4` — `feat: section Vue d'ensemble avec cartes statistiques (dashboard technicien)` |
| `sdtest/`        | Aucun — simple dossier de tests, **non tracké**               | —          | —       | — |

- `backend/` et `frontend/` sont **publics** : clonage et fetch sans identifiants.
- `projet/` (racine, dépôt `Repairdom`) est **privé** : nécessite un token GitHub.
- La racine ne doit servir QUE d'espace de travail : on n'y pousse **jamais rien**.
  Elle contient `COMMIT-GUIDE.md`, `.gitignore`, ce guide et des fichiers obsolètes
  (anciennes versions de `backend/*` / `frontend/*`) qui n'ont pas été retirés du repo.

## 2. Identifiants requis

- Compte GitHub : **`digitalholding2026-ux`**
- Email Git : **`digitalholding2026@gmail.com`**
- Méthode d'auth actuelle : **HTTPS** avec `credential.helper = store`
  (un PAT GitHub était stocké dans `~/.git-credentials` sur l'ancienne machine).
- Pour les push (et pour cloner `Repairdom` privé), il faut un **token d'accès GitHub**
  (PAT classic, scope `repo`) du compte `digitalholding2026-ux`.
  → Le demander à l'ancien responsable s'il n'est pas fourni.

Config globale à reproduire si absente :

```bash
git config --global user.name  "digitalholding2026-ux"
git config --global user.email "digitalholding2026@gmail.com"
git config --global credential.helper store   # stocke le PAT après la 1ère saisie
```

## 3. Procédure de reconnexion (à exécuter AVANT de coder)

### Cas A — les dossiers arrivent avec leur `.git` (copie intégrale)

Rien à recréer. Il suffit de **vérifier** chaque branchement :

```bash
cd /chemin/vers/projet/backend
git remote -v && git status -sb          # origin = Repairdom-backend.git, branche main

cd /chemin/vers/projet/frontend
git remote -v && git status -sb          # origin = Repairdom-frontend.git, branche main

cd /chemin/vers/projet
git remote -v && git status -sb          # origin = Repairdom.git (privé), branche main
```

Si un `origin` manque (dossier sans `.git`) → appliquer le **Cas B**.

### Cas B — dossiers vides / sans `.git` → reconstruction propre

```bash
mkdir -p /chemin/vers/projet && cd /chemin/vers/projet

# 1) backend/ et frontend/ : publics, clonables sans détails
git clone https://github.com/digitalholding2026-ux/Repairdom-backend.git backend
git clone https://github.com/digitalholding2026-ux/Repairdom-frontend.git frontend

# 2) recopier à la racine : COMMIT-GUIDE.md, .gitignore, ce guide
#    (le repo racine Repairdom n'est pas re-cloné ici : il contient d'anciens
#     fichiers backend/ + frontend/ qui écraseraient les dossiers réels)

# 3) la racine reste un simple espace de travail (pas de .git à la racine).
```

> ⚠️ **Piège des repos imbriqués** : ne jamais cloner/merge le dépôt racine
> `Repairdom` par-dessus `backend/` ou `frontend/`, et ne jamais `git push` depuis
> la racine `projet/`. Chaque dossier pousse vers SON remote uniquement.

### Cas C — reparer un `.git` cassé (origin inconnu / mauvais remote)

```bash
cd /chemin/vers/projet/backend
git remote set-url origin https://github.com/digitalholding2026-ux/Repairdom-backend.git
git fetch origin
git reset --hard origin/main        # ⚠️ jette le travail local non commité éventuel

# idem pour frontend avec Repairdom-frontend.git
```

## 4. Vérification finale obligatoire (avant tout développement)

- [ ] `backend/`  → `origin` = `Repairdom-backend.git`, branche `main`, HEAD proche de `641df65`
- [ ] `frontend/` → `origin` = `Repairdom-frontend.git`, branche `main`, HEAD proche de `2d9c0aa`
- [ ] `projet/` (racine) → `origin` = `Repairdom.git` (privé), branche `main`, HEAD `9c034a4`
- [ ] Aucun fichier `.env` / `node_modules/` n'apparaît dans `git status`
- [ ] `sdtest/` : ne rien brancher (aucun dépôt)

```bash
# exemples de contrôle
cd backend  && git remote -v && git status -sb
cd frontend && git remote -v && git status -sb
```

En cas de doute sur le dépôt de destination d'un push → **demander avant de pousser**.

## 5. Points d'attention transmis par l'ancienne machine

1. **Commit non poussé** — le dépôt racine `Repairdom` (HEAD `9c034a4`,
   `feat: section Vue d'ensemble...`) avait **1 commit local non encore push**.
   Vérifier avant de quitter l'ancienne machine (`git -C projet push origin main`)
   ou sur la nouvelle (`git status -sb` / `git log origin/main..HEAD`).
2. **Règles de commit** : deux dépôts distincts, un commit par dépôt, jamais mélanger
   backend et frontend. Voir `COMMIT-GUIDE.md`.
3. **Auth** : repos publics = aucun mot de passe ; seul `Repairdom` (privé) et les
   push demandent le PAT du compte `digitalholding2026-ux`.