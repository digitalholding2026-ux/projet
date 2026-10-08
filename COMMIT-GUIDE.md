# Guide de commit — RepairDom (2 dépôts distincts)

Ce projet est organisé en **deux dossiers** correspondant à **deux dépôts Git indépendants**.
Ne **jamais** committer le dossier racine comme un seul dépôt, et ne **jamais** mélanger
les changements du backend et du frontend dans le même commit.

## Vue d'ensemble des dépôts

| Dossier    | Dépôt Git (remote `origin`)                                   | Déploiement |
|------------|---------------------------------------------------------------|-------------|
| `backend/` | `https://github.com/digitalholding2026-ux/Repairdom-backend.git` | Railway (API) |
| `frontend/`| `https://github.com/digitalholding2026-ux/Repairdom-frontend.git` | Vercel (statique) |
| racine `/config/projet` | `https://github.com/digitalholding2026-ux/projet.git` | — (dépôt documentaire) |

Chaque dossier est **lui-même un dépôt Git** (il contient son propre `.git`),
y compris la racine.

**La racine est le dépôt documentaire du projet** : on y commite et pousse
uniquement les rapports (`.md`) exigés par RÈGLE 1 de `AGENTS.md`, **jamais de
code**. Voir `CONNEXION-REPOS-GUIDE.md` pour l'état des trois dépôts.

## Règle d'or

> Committer **uniquement** le dossier concerné, vers **son** dépôt.

- Si vous modifiez le backend → commit dans `backend/`, push sur `origin` de **Repairdom-backend**.
- Si vous modifiez le frontend → commit dans `frontend/`, push sur `origin` de **Repairdom-frontend**.
- Si vous modifiez les deux → **deux commits distincts**, chacun dans son propre dossier/dépôt.

## Procédure

Toujours exécuter les commandes **à l'intérieur** du dossier concerné (travail dans
`backend/` ou `frontend/`), jamais `cd` depuis la racine pour ajouter des fichiers de
l'autre dossier.

### Backend

```bash
cd backend
git status                        # vérifier les fichiers modifiés
git diff                          # relire la modification avant de l'indexer
git add <fichiers concernés>      # n'indexer QUE les fichiers backend modifiés
git commit -m "<message descriptif>"
git push origin main
```

### Frontend

```bash
cd frontend
git status
git diff
git add <fichiers concernés>
git commit -m "<message descriptif>"
git push origin main
```

## Recommandations pour les messages de commit

- Message **concis** décrivant l'action, en anglais (style du repo existant).
- Exemples :
  - `backend/` → `Configure CORS to allow Vercel frontend and localhost origins`
  - `frontend/` → `Fix mission status display in dashboard`
- Un commit = un changement logique. Éviter de regrouper des changements sans rapport.

## Fichiers sensibles / à ne jamais committer

`.env`, `node_modules/`, et les fichiers de données locaux sont déjà ignorés
via `.gitignore`. Vérifier avec `git status` qu'aucun fichier sensible (`SUPABASE_ANON_KEY`, etc.)
n'apparaît avant de pousser.

> 🔒 **Rappel — le PAT GitHub.** Il est stocké dans `~/.git-credentials`
> (mode `600`), **hors du dépôt**. Il ne doit jamais apparaître dans :
> - le contenu d'un `.md` (rapports, guides) ;
> - un message de commit ;
> - une URL (`https://user:token@github.com/...`) — elle finit dans `.git/config` et dans les logs ;
> - la sortie d'une commande de diagnostic affichée à l'écran.
>
> En cas de fuite : révoquer le token sur github.com/settings/tokens, en créer
> un nouveau, puis réécrire `~/.git-credentials`.

## Vérification finale

Avant d'envoyer (`push`), confirmer que le push part bien vers le bon dépôt :

```bash
git remote -v        # doit afficher Repairdom-backend.git ou Repairdom-frontend.git selon le dossier
```

En cas de doute sur le dépôt de destination, **demander** avant de pousser.
