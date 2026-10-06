# RÈGLES PROJET — Relio

> Fichier lu automatiquement par l'agent avant chaque tâche. Toute instruction
> ci-dessous est **obligatoire** et prime sur les conventions habituelles.

---

## RÈGLE 1 — RAPPORT OBLIGATOIRE EN FIN DE CHAQUE TÂCHE

**À la fin de TOUTE tâche (chantier, correction, audit, investigation), même
sans modification de code, je dois :**

1. Écrire un rapport de la tâche dans un fichier `.md` à la **RACINE** de
   `/config/projet` (pas dans `backend/` ni `frontend/`).
2. Le nommer `RAPPORT-CHANTIER-<ID>.md` (ex. `RAPPORT-CHANTIER-D2.5.md`).
   Pour un chantier sans identifiant : `RAPPORT-<SUJET>.md`.
3. Le committer et le pousser sur le dépôt **racine** :
   `https://github.com/digitalholding2026-ux/projet.git` (branche `main`).
4. Le faire **à la fin**, une fois le travail de code terminé et vérifié — pas
   avant, pas en cours de tâche.

**Pourquoi la racine ?** Le dépôt racine `projet.git` est l'espace de travail
qui rassemble les deux dépôts applicatifs ; c'est le seul endroit où un
lecteur trouve l'historique des chantiers sans naviguer dans deux repos.

### Contenu attendu du rapport

Rapport **factuel**, structuré en Markdown. Pas de flatterie, pas de
recommandations non fondées. Sections minimales :

| Section | Contenu |
|---|---|
| Statut | Date, dépôt(s) concerné(s), commit(s), état push/déploiement |
| Synthèse | Ce qui a été fait, en une dictamen |
| Fichiers | Créés / modifiés / supprimés, avec chemins exacts |
| Migrations | Nom + nature |
| Vérifications | `tsc`, lint, tests (X/X verts), et **la preuve de non-régression** |
| Bugs trouvés | Réels, avec impact et correctif — y compris ceux trouvés par mes propres tests |
| Points d'attention | Ce qui n'a pas été vérifié, ce qui reste fragile |
| Questions bloquantes | Ou « Aucune » |

Règles de rédaction :

- **Chiffres réels.** Aucun test non exécuté ne doit être présenté comme vert.
  Si un test n'a pas pu tourner (infrastructure absente), le dire.
- **Les échecs pré-existants se distinguent des régressions.** Comparer avec
  l'état `HEAD` propre (`git stash`) avant d'affirmer « 0 régression ».
- **Les écarts par rapport à la demande sont notés explicitement**, avec la
  raison. Ne jamais faire semblant d'avoir suivi une consigne à la lettre si
  une correction était nécessaire.

---

## RÈGLE 2 — UN DEPÔT PAR DOMAINE, DEUX COMMITS

Ce projet est **deux dépôts Git distincts** :

| Dossier | Remote | Déploiement |
|---|---|---|
| `backend/` | `Repairdom-backend` | Railway |
| `frontend/` | `Repairdom-frontend` | Vercel |
| racine `/` | `projet` (espace de travail + rapports) | — |

- Un commit par dépôt, **jamais** de commit « backend + frontend » confondu.
- Ne **jamais** committer la racine autrement que pour un rapport/document.
- Ne pas ajouter `backend/` ni `frontend/` à l'index racine : ce sont des dépôts
  imbriqués (ils y apparaîtraient comme gitlinks).
- Ordre de push : **backend d'abord**, attendre Railway vert, puis frontend.

---

## RÈGLE 3 — TESTS : CE QUI EST INTERDIT, CE QUI EST OBLIGATOIRE

### Interdit en local

- `next build`, `npm run dev` / `npm start`, Docker, Postgres local.
- Tests e2e nécessitant une base joignable.
- Tout test qui aurait un effet de bord sur la prod.

### Autorisé

- `oxlint`, `tsc --noEmit` (statique).
- `vitest run` et `node --test` sur des tests **unitaires purs** : Prisma
  mocké, services externes mockés, `fetch` mocké.

### Obligatoire avant tout commit

```bash
# backend
npx tsc --noEmit -p tsconfig.build.json && npm run lint && npx vitest run

# frontend
npx tsc --noEmit && npm run lint
npm run test:unit   # nécessite Node 22+ (voir RÈGLE 4)
```

**La non-régression doit être PROUVÉE, pas supposée.** Avant d'affirmer
« 0 régression » : exécuter les mêmes tests sur `HEAD` propre
(`git stash -u`) puis comparer les **noms** des échecs, pas seulement leur
nombre.

### Les doubles de test doivent être FIDÈLES

Un double qui ignore le `where` d'une requête, ou qui n'écrit pas en mémoire,
teste la mécanique du mock et non la logique métier. Les deux pièges ont déjà
produit des tests verts à vide dans ce dépôt. Quand un double doit refléter un
`update`, il le fait.

---

## RÈGLE 4 — ENVIRONNEMENT NODE

- Le projet exige **Node 22+** (runner `.ts` natif de `node --test`).
- Si Node 20 est installé : compiler les fichiers purs avec `tsc` vers un
  répertoire **hors du dépôt**, puis lancer `node --test` sur la sortie.
- ⚠️ **JAMAIS** compiler ni écrire dans un dossier du dépôt (notamment via un
  lien symbolique vers `src/`). Le runner résout les liens et le `rm` traverse
  alors le lien : cela a déjà supprimé un fichier source et injecté des `.js`
  parasites. Vérifier avec `git status` et `find src -name '*.js'` avant commit.

---

## RÈGLE 5 — SÉCURITÉ

- Le token de brouillon (`relio_demande_draft_token`) et les tokens de
  vérification e-mail sont des **secrets**. **Jamais** journalisés, jamais dans
  une URL de tracking, jamais dans un message d'erreur.
- Ne jamais committer de secret. Les fichiers `*.pem`, `.env`, `connexion*.txt`
  sont déjà ignorés à la racine — vérifier qu'ils ne sont pas **déjà trackés**
  (le `.gitignore` n'a aucun effet sur un fichier indexé).
- Après un `git push` rejeté (distant avancé) : `git pull --rebase`, **jamais**
  `--force`.

---

## RÈGLE 6 — PROCESSUS DE VALIDATION

1. Écrire et vérifier (`tsc`, lint, tests).
2. **Présenter le résumé et attendre la validation** — ne pas pousser avant.
3. Pousser selon l'ordre de la RÈGLE 2.
4. Attendre le déploiement (Railway puis Vercel) et le confirmer factuellement.
5. **Écrire et pousser le rapport** (RÈGLE 1).

Si l'utilisateur demande explicitement de pousser sans validation, la demande
prime sur l'étape 2 — mais le rapport (RÈGLE 1) reste obligatoire.