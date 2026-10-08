# Guide de reconnexion des dossiers à leurs dépôts (nouvelle machine)

> À lire **en premier**, AVANT de toucher au code. Le but : rebrancher chaque
> dossier du projet à son dépôt Git respectif sur la nouvelle machine.

## 1. Vue d'ensemble — 3 dépôts, 3 dossiers

Le projet est découpé en **3 dépôts Git indépendants** (pattern « repos imbriqués »
dans un même espace de travail). Chaque dossier porte **son propre `.git`**.

| Dossier          | Remote `origin`                                              | Visibilité | Branche | HEAD au 2026-10-08 |
|------------------|--------------------------------------------------------------|------------|---------|---------------------|
| `backend/`       | `https://github.com/digitalholding2026-ux/Repairdom-backend.git` | public  | `main`  | `da71861` — `feat(rewards): replace mission-count programme with a margin-based LTV scheme` |
| `frontend/`      | `https://github.com/digitalholding2026-ux/Repairdom-frontend.git` | public  | `main`  | `50002d9` — `feat(rewards): rebuild the loyalty page around margin-based credits, badges and nature rewards` |
| racine `/config/projet` | `https://github.com/digitalholding2026-ux/projet.git`    | —          | `main`  | `a09a46a` — `docs: record the prior audit of SasPay fee transparency (Option A)` |

> ⚠️ **Correction du 2026-10-08.** La version précédente de ce guide annonçait des
> HEAD `641df65` / `2d9c0aa` / `9c034a4` et un dépôt racine `Repairdom.git`.
> Ces informations sont **obsolètes** : les repos ont avancé et
> `Repairdom.git` **n'existe pas** (`git ls-remote` → *Repository not found*,
> y compris avec le token). Le dépôt racine est `projet.git`.
> Les HEAD ci-dessus sont des points de repère, pas des références figées :
> vérifier avec `git -C <dossier> log --oneline -1` plutôt que de comparer à
> une valeur en dur.

- `backend/` et `frontend/` sont publics : clonage et fetch sans identifiants.
- La racine est un dépôt Git à part entière. **Contradiction avec `AGENTS.md`**, qui
  impose (RÈGLE 1) d'y committer et pousser un rapport par tâche, alors que le
  présent guide disait historiquement de n'y pousser jamais rien.
  **Règle retenue depuis le 2026-10-08 :** la racine est un dépôt documentaire —
  on y push **uniquement** des rapports/`.md`, jamais de code (cf. RÈGLE 2 de
  `AGENTS.md`). `backend/` et `frontend/` restent en untracked à la racine et ne
  doivent jamais être ajoutés à son index.

## 2. Identifiants requis

- Compte GitHub : **`digitalholding2026-ux`**
- Email Git : **`digitalholding2026@gmail.com`**
- Auth : **HTTPS** avec `credential.helper = store`.

### État au 2026-10-08 (machine courante)

La configuration globale est en place et le PAT est enregistré dans
`~/.git-credentials` (mode `600`). `git push --dry-run` répond
*Everything up-to-date* sur les trois dépôts : l'auth est opérationnelle.

```bash
git config --global user.name  "digitalholding2026-ux"    # ✅ fait
git config --global user.email "digitalholding2026@gmail.com"  # ✅ fait
git config --global credential.helper store                # ✅ fait
```

> 🔒 **Le PAT lui-même n'est écrit dans aucun fichier de ce dépôt** et n'apparaît
> dans aucun `.md`. Il ne vit que dans `~/.git-credentials`, hors du workspace,
> avec les permissions `600`. Ne jamais le recopier dans un rapport, un message
> de commit ou une issue GitHub — un PAT dans l'historique git est irrevocable
> sans rotation. S'il est compromis : le révoquer sur github.com/settings/tokens
> et en générer un nouveau.

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

Commandes **réellement exécutées** sur cette machine le 2026-10-08 :

```bash
cd /config/projet

# 1) la racine (dépôt projet.git) — cloné en premier, dans un dossier vide
git clone https://github.com/digitalholding2026-ux/projet.git /tmp/clone-projet
# puis : mv /tmp/clone-projet/* /config/projet/

# 2) backend/ et frontend/ : publics, clonables sans détails
git clone https://github.com/digitalholding2026-ux/Repairdom-backend.git backend
git clone https://github.com/digitalholding2026-ux/Repairdom-frontend.git frontend
```

> ⚠️ **`git clone <url> .` échoue** si le répertoire cible contient déjà des
> fichiers, même vides. Cloner dans un dossier temporaire puis déplacer le
> contenu (`shopt -s dotglob && mv /tmp/clone/* /config/projet/`) — c'est ce qui
> a été fait pour la racine, qui contenait deux dossiers vides.

### Cas B' — dépendances

`npm ci` **n'a pas été lancé** sur cette machine : ni `backend/node_modules` ni
`frontend/node_modules` n'existent. À faire avant toute vérification :

```bash
npm ci --prefix backend
npm ci --prefix frontend
```

⚠️ **Node 20.20.2 est installé, or `AGENTS.md` RÈGLE 4 exige Node 22+** pour le
runner `.ts` natif de `node --test` (donc pour `npm run test:unit` côté
frontend). Installer Node 22 avant de considérer la suite de tests comme
exécutable.

### Cas C — reparer un `.git` cassé (origin inconnu / mauvais remote)

```bash
cd /chemin/vers/projet/backend
git remote set-url origin https://github.com/digitalholding2026-ux/Repairdom-backend.git
git fetch origin
git reset --hard origin/main        # ⚠️ jette le travail local non commité éventuel

# idem pour frontend avec Repairdom-frontend.git
```

## 4. Vérification finale obligatoire (avant tout développement)

État vérifié le **2026-10-08** :

- [x] `backend/`  → `origin` = `Repairdom-backend.git`, `main`, HEAD `da71861`
- [x] `frontend/` → `origin` = `Repairdom-frontend.git`, `main`, HEAD `50002d9`
- [x] racine      → `origin` = `projet.git`, `main`, HEAD `a09a46a`
- [x] Auth : `git push --dry-run` OK sur les trois dépôts
- [x] Aucun secret tracké (`git ls-files` ne remonte que des `.env.example`)
- [ ] `node_modules/` **absents** des deux dépôts → `npm ci` à lancer
- [ ] Node 22+ à installer (actuellement 20.20.2)
- [x] `sdtest/` : inexistant sur cette machine, rien à brancher

```bash
# contrôles
cd backend  && git remote -v && git status -sb
cd frontend && git remote -v && git status -sb
cd ..        && git remote -v && git status -sb
```

`git status -sb` à la racine doit afficher `?? backend/` et `?? frontend/` :
c'est **normal et souhaité** (RÈGLE 2). Ne pas les `git add`.

En cas de doute sur le dépôt de destination d'un push → **demander avant de pousser**.

## 5. Points d'attention

1. ~~**Commit non poussé** sur le dépôt racine~~ — **résolu le 2026-10-08** :
   le clone est propre sur les trois dépôts (`git push --dry-run` →
   *Everything up-to-date*). Le commit `9c034a4` mentionné dans l'ancienne
   version de ce guide n'existe pas dans l'historique de `projet.git` ; cette
   alerte est caduque.
2. **Règles de commit** : deux dépôts applicatifs distincts, un commit par
   dépôt, jamais mélanger backend et frontend. Voir `COMMIT-GUIDE.md` et
   RÈGLE 2 de `AGENTS.md`.
3. **Auth** : le PAT du compte `digitalholding2026-ux` est enregistré dans
   `~/.git-credentials` (mode `600`, hors du dépôt). Il n'est référencé dans
   aucun fichier versionné.
4. **`Repairdom.git` n'existe pas.** Toute mention de ce dépôt dans ce guide
   est à considérer comme une erreur historique.