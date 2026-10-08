# RAPPORT — Reconnexion des 3 dépôts sur nouvelle machine

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-08 |
| Dépôts concernés | `projet` (racine, documentation), `Repairdom-backend`, `Repairdom-frontend` |
| Commits | Un seul commit, sur la racine uniquement. Aucun commit applicatif. |
| Push | Racine : poussé sur `origin/main`. Backend et frontend : **aucun changement**, rien à pousser. |
| Déploiement | Sans objet (aucun code modifié). |

## Synthèse

Les trois dépôts ont été (re)branchés sur leurs remotes respectifs et les
fichiers de documentation ont été rectifiés : ils décrivaient un dépôt racine
`Repairdom.git` qui **n'existe pas**, ainsi que des HEAD périmés.

## Fichiers

Créés :

- `RAPPORT-RECONNEXION-REPOS.md` (le présent rapport)

Modifiés :

- `CONNEXION-REPOS-GUIDE.md` — remotes et HEAD réels, état de l'auth, procédure
  de clone réellement exécutée, checklist finale, points d'attention
- `COMMIT-GUIDE.md` — ajout de la racine comme dépôt documentaire, avertissement
  sur le PAT
- `AGENTS.md` — RÈGLE 1 : interdiction explicite de committer un secret dans un
  rapport

Supprimés : aucun.

Non modifiés : `backend/`, `frontend/` (aucune ligne de code touchée).

## Migrations

Aucune.

## Vérifications

| Contrôle | Résultat |
|---|---|
| `git remote -v` × 3 | ✅ remotes conformes |
| `git status -sb` × 3 | ✅ sur `main`, aligné sur `origin/main` |
| `git push --dry-run origin main` × 3 | ✅ *Everything up-to-date* — auth opérationnelle |
| `git ls-remote` sur `Repairdom.git` | ❌ *Repository not found* (voir Bugs) |
| Scan de motif de token GitHub dans les `.md` | ✅ 0 occurrence |
| `git ls-files \| grep -E '\.env\|\.pem\|credentials'` | ✅ seuls des `.env.example` sont trackés |
| `tsc` / lint / tests | ⚠️ **non exécutés** — voir Points d'attention |

## Bugs trouvés

### 1. Le dépôt racine documenté n'existe pas

**Impact** — `CONNEXION-REPOS-GUIDE.md` désignait `Repairdom.git` comme dépôt
racine, sur 4 sections (tableau, procédure, checklist, points d'attention). Une
 personne suivant le guide sur une machine neuve aurait pushed vers une URL
morte, ou aurait concludes à tort que la racine n'est pas versionnée.

**Correctif** — le dépôt racine est `projet.git`. Les 4 occurrences sont
corrigées, avec la commande de vérification qui aMis en défaut la documentation.

### 2. HEAD de référence périmés

**Impact** — la checklist exigeait `641df65`, `2d9c0aa` et `9c034a4`. Aucun
n'est l'HEAD actuel ; `9c034a4` est introuvable dans l'historique de
`projet.git`. Une vérification faite à la lettre aurait concludes à une
corruption alors que les dépôts sont sains.

**Correctif** — remplacement par les HEAD réels, plus une consigne explicite de
vérifier avec `git log --oneline -1` plutôt que de comparer à une valeur figée.
C'est le point qui a le plus de chances de se redéclencher : tout chantier
livré fait dériver ces valeurs.

### 3. Contradiction entre `AGENTS.md` et `CONNEXION-REPOS-GUIDE.md`

**Impact** — `AGENTS.md` RÈGLE 1 impose de committer et pousser un rapport par
tâche à la racine ; le guide affirmait l'inverse (« on n'y pousse jamais rien »).
Les deux documents ne pouvaient pas être suivis simultanément.

**Correctif** — arbitrage explicité dans les deux fichiers : la racine est un
dépôt **documentaire** (rapports `.md` uniquement, jamais de code), conformément
à RÈGLE 2 d'`AGENTS.md`.

### 4. Alerte « commit non poussé » caduque

**Impact** — le guide annonçait 1 commit local non poussé sur la racine. Faux
alarme sur cette machine : les trois dépôts sont alignés. Conservée telle quelle,
elle aurait fait perdre du temps à une vérification.

**Correctif** — marquée résolue, avec la commande qui l'établit.

## Points d'attention

- **`node_modules` absent** des deux dépôts applicatifs. `npm ci` n'a pas été
  lancé : aucun `tsc`, lint ou test n'a donc pu tourner. Aucune affirmation de
  « 0 régression » ne peut être faite en l'état.
- **Node 20.20.2 installé**, alors que RÈGLE 4 exige Node 22+ pour le runner
  `.ts` natif de `node --test`. `npm run test:unit` côté frontend ne pourra pas
  s'exécuter avant correction.
- **Le PAT GitHub est dans `~/.git-credentials`** (mode `600`), pas dans une
  variable d'environnement : un `export` ne survit pas à une nouvelle session.
  Il est hors du dépôt et n'apparaît dans aucun fichier versionné. S'il fuite, il
  faut le révoquer : un token dans l'historique git n'est pas récupérable.
- Les HEAD inscrits dans les `.md` sont des points de repère datés, pas des
  références. Toute nouveau chantier les fera de nouveau dériver.

## Questions bloquantes

Aucune.