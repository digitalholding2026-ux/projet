# RAPPORT — FIX URGENT + OUTILLAGE : régression 4-A sur `/technicien/demandes/[id]`

> Correction du crash « Rendered more hooks than during the previous render »
> et ajout du filet qui l would've laissé passer. Déployé Vercel puis Railway.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Frontend | `Repairdom-frontend` — commit **`0e1c284`** `fix(hooks): remove conditional useMemo crashing technician mission pages, enforce rules-of-hooks` — poussé `7dc1818..0e1c284` |
| Backend | `Repairdom-backend` — commit **`0984c2e`** `fix(realtime): mask unauthorized mission streams as 404 to match the API` — poussé `09cc75f..0984c2e` |
| Vercel | ✅ `/` → `200`, `/technicien/demandes` → `200`. **Réserve** : le nouveau bundle n'est pas certifiable depuis un endpoint public (route pré-rendue, `etag` inchangé) — le fix est dans le JS client. Preuve fonctionnelle attendue du commanditaire. |
| Railway | ✅ **vert** : `status=ok`, `database=up`, uptime 322 s (processus redémarré après le push) |
| Rapport obsolète | `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` — bandeau **OBSOLÈTE** ajouté |
| Outillage ajouté | 3 devDependencies : `eslint@9.39.5`, `eslint-plugin-react-hooks@7.1.1`, `@typescript-eslint/parser@8.71.1` (+137 paquets) |

## Synthèse

Le `useMemo` ajouté par le chantier 4-A ligne 447 de la page détail mission
technicien était appelé **après** trois returns anticipés (`if (loading)` l.326,
`if (error && !demande)` l.340, `if (!demande) return null` l.388). Le nombre de
hooks variait entre deux rendus — 14 sur le skeleton, 15 avec les données — donc
React levait « Rendered more hooks than during the previous render » et
**toute** page détail mission technicien tombait sur `src/app/error.tsx`,
indépendamment de l'état assigné ou disponible de la mission.

Le correctif supprime le `useMemo` (mémoïsation inutile d'un calcul trivial sur
un nombre). Trois filets sont posés en complément : un test structurel de
non-régression, un lint `rules-of-hooks` bloquant, et l'affichage du digest dans
`error.tsx`. Le SSE masquant désormais en 404 comme le reste de l'API, la fuite
d'information résiduelle est fermée.

## Fichiers

### Frontend — 1 créé, 5 modifiés

| Type | Chemin | Nature |
|---|---|---|
| Créé | `eslint.config.mjs` | flat config ESLint, **2 règles react-hooks uniquement** + stub local `@next/next` |
| Modifié | `src/app/technicien/demandes/[id]/page.tsx` | `useMemo` supprimé, appel direct, import nettoyé, commentaire d'interdiction |
| Modifié | `src/app/error.tsx` | lien replié « Détails techniques » (digest ; message + stack hors production) |
| Modifié | `src/lib/technician-quote.test.ts` | +3 tests de non-régression |
| Modifié | `package.json` | `lint` enchaîne `lint:hooks` ; nouveau script `lint:hooks` |
| Modifié | `package-lock.json` | +137 paquets |

### Backend — 2 modifiés

| Type | Chemin | Nature |
|---|---|---|
| Modifié | `src/realtime/realtime.controller.ts` | `ForbiddenException` → `NotFoundException` ; import retiré ; commentaire de masquage |
| Modifié | `src/realtime/realtime.spec.ts` | +3 tests ; import `ForbiddenException` retiré |

### Dépôt racine

| Type | Chemin |
|---|---|
| Créé | `RAPPORT-FIX-404-HOOKS-TECHNICIEN.md` |
| Modifié | `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` — bandeau OBSOLÈTE |

## Migrations

**Aucune.** Ni schéma Prisma, ni enum, ni variable d'environnement.

## Vérifications

| Commande | Backend | Frontend |
|---|---|---|
| `tsc --noEmit` | **OK** (`-p tsconfig.build.json`) | **OK** |
| `npm run lint` | **OK** (oxlint, 0 warning) | **OK, exit 0** (oxlint + ESLint, ~15 s) |
| Tests | **990 tests, 3 échecs pré-existants** | **392 tests, 9 échecs d'artefacts miroir** |
| Non-régression | **prouvée** | **prouvée** |

### Backend — détail
- `HEAD` `09cc75f` : `987 tests, 3 failed`
- Après `0984c2e` : `990 tests, 3 failed` (**+3**)
- `diff` des **noms** d'échecs : **identiques**. Les 3 échecs sont pré-existants
  (`src/demandes/demande-multimedia.spec.ts`, `docs/UX-BACKLOG.md:38-42`).
- `src/realtime/realtime.spec.ts` : **13/13** (dont 3 nouveaux).

### Frontend — détail et réserve méthodologique
`npm run test:unit` **n'a pas pu être lancé tel quel** : Node local = **20.20.2**,
le runner `.ts` natif exige **Node 22+** (limitation pré-existante, `AGENTS.md`
RÈGLE 4). Vérification par compilation `tsc` vers un **miroir hors dépôt**
(`/tmp/opencode/fe-mirror`), puis `node --test`.

- Baseline `7dc1818` : **9 échecs**
- Après correctif : **9 échecs, noms identiques** (+3 tests)
- Ces 9 échecs sont des **artefacts du miroir** : il ne contient que `src/`, pas
  `public/sw.js`, `docs/`, `logo/`, `animatio json/`, `package.json` ni `.git`.
  Identiques sur `HEAD` propre, donc sans rapport avec les modifications.

### Non-vacuité des nouveaux tests — vérifiée
Les 3 tests de non-régression **échouent sur la version 4-A** et passent après
correctif :

| Test | Version 4-A | Après correctif |
|---|---|---|
| `aucun hook après un return anticipé` | ❌ « hook l.418 APRÈS return l.326 » | ✅ |
| `le bloc dérivé ne contient aucun hook` | ❌ « useMemo réintroduit » | ✅ |
| `aucun hook après return (autres fichiers 4-A)` | ✅ (déjà propres) | ✅ |
| → total | 20/22 | **22/22** |

Le test multi-fichiers assert `verifies >= 3` : il ne peut pas passer à vide sur
une liste de fichiers devenue irrelevante.

### Non-vacuité de l'outillage — vérifiée
`npx eslint "src/app/technicien/demandes/[id]/page.tsx"` sur la version 4-A :

```
447:24  error  React Hook "useMemo" is called conditionally. React Hooks must be
                called in the exact same order in every component render. Did you
                accidentally call a React Hook after an early return?
                react-hooks/rules-of-hooks
✖ 1 problem (1 error, 0 warnings)     EXIT=1
```

Le lint aurait donc **bloqué** le commit `7dc1818`.

### Périmètre (A.2)
Scan des 11 fichiers modifiés par 4-A + ESLint sur **tout** `src/` :
**0 violation `rules-of-hooks`**. Les 5 warnings `exhaustive-deps` restants sont
**préexistants** et portent sur des fichiers non modifiés
(`admin/catalog/**`, `technicien/profil/page.tsx`).

> **Non exécuté volontairement** (interdit par la mission) : `next build`,
> `npm run dev`, Docker, Postgres local, tests e2e.

## Bugs trouvés

| # | Constat | Emplacement | Impact | Suite |
|---|---|---|---|---|
| **1** | **`useMemo` après returns anticipés** — régression 4-A (`7dc1818`). | `src/app/technicien/demandes/[id]/page.tsx:447` | **Bloquant.** `/technicien/demandes/[id]` inutilisable pour **toute** mission technicien. | **Corrigé** (`0e1c284`) + 3 filets |
| **2** | **`tsc` est aveugle** sur l'ordre des hooks, et **`oxlint` n'a aucune règle react-hooks** (vérifié : `oxlint --rules \| grep -i hook` → aucun résultat). Aucune règle n'aurait pu attraper ce bug. | `frontend/oxlint.json` | Récidive garantie | **Corrigé** : ESLint + `rules-of-hooks: error`, branché sur `lint` |
| **3** | **Mes propres tests du chantier 4-A passaient à vide.** Les 19 assertions de `technician-quote.test.ts` sont des regex vérifiant la **présence** des appels (`previewTechnicianQuote(...)`), **jamais leur position**. Elles ne pouvaient structurellement pas voir ce bug. | `src/lib/technician-quote.test.ts` | Faux sentiment de sécurité sur la régression | **Corrigé** : +3 tests structurels |
| **4** | **Fuite d'information SSE** : `403` là où l'API masque en 404, révélant l'existence d'une mission à un utilisateur non concerné. | `backend/src/realtime/realtime.controller.ts:72` | Mineure | **Corrigé** (`0984c2e`) |
| **5** | **`error.tsx` muet** : exception journalisée mais invisible, ni message ni digest. Coût : 2 cycles d'audit perdus. | `frontend/src/app/error.tsx:15-17, 23` | Diagnostic très lent en production | **Corrigé** : digest en `<details>`-équivalent, replié |
| **6** | **Analyse erronée de ma part** : le rapport précédent concluait « aucun décompilateur, aucune exception imputable à 4-A ». J'avais vérifié l'import et la déclaration du `useMemo`, **pas sa position** par rapport aux `return`. | `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` | Conclusion fausse | **Corrigé** : bandeau OBSOLÈTE |
| 7 | `eslint@9.39.5` est signalé **déprécié** par npm. | `frontend/package.json` | Aucun impact fonctionnel ; à surveiller | Signalé |

## Points d'attention

1. **Outillage lourd — c'est le choix le plus coûteux de la mission.** oxlint 1.82
   n'ayant aucune règle react-hooks, l'Option 2 était **impossible** : j'ai pris
   l'Option 1 avec **3 devDependencies** et **+137 paquets**. `npm run lint` passe
   de ~1 s à **~15 s**. Rollback : retirer les 3 lignes de `package.json` et le
   `&& npm run lint:hooks`.

2. **`npm run lint` est désormais bloquant** pour tout commit frontend : une
   violation `rules-of-hooks` fait échouer le lint. C'est l'objectif, mais tout
   futur chantier frontend en dépend.

3. **Le test structurel est ciblé, pas général.** Il valide les `if (` indentés de
   2 espaces dans des fichiers connus. J'ai **tenté puis écarté** un parseur
   général : il produisait un faux positif sur `technicien/page.tsx` (un `return`
   dans un callback de `useEffect`). Plutôt que de livrer un heuristique peu
   fiable, la couverture générale est confiée à ESLint. **À VÉRIFIER** : ce test
   ne couvre pas un composant dont les returns conditionnels seraient indentés
   différemment.

4. **Stub `@next/next` dans la config ESLint.** Deux fichiers hors périmètre
   (`technicien/kyc`, `admin/kyc/[technicianId]`) contiennent
   `eslint-disable-next-line @next/next/no-img-element`. Sans stub, ESLint échouait
   dessus (*« Definition for rule not found »*). Le stub rend la directive
   résoluble, règle à `off`. **Ce n'est pas une règle fonctionnelle** — c'est un
   rattrapage pour ne pas toucher à des fichiers hors périmètre.

5. **Le bundle Vercel n'est pas certifiable en lecture seule.** La route
   `/technicien/demandes` est pré-rendue (`x-nextjs-prerender: 1`,
   `x-vercel-cache: HIT`) et son `etag` est resté identique après le push : le
   correctif vit dans le JS client, invisible depuis un endpoint public.
   **La preuve fonctionnelle doit venir du Test 1 du commanditaire.**

6. **Les 5 warnings `exhaustive-deps`** (4 dans `admin/catalog/**`, 1 dans
   `technicien/profil/page.tsx`) sont **préexistants** et **non corrigés** :
   hors périmètre. `exhaustive-deps` est en `warn`, donc ils ne bloquent pas.

7. **`npm run test:unit` non exécuté en conditions réelles** (Node 20 local,
   Node 22 requis). Sur Vercel / poste de dev, les 35 fichiers tourneront
   réellement, dont les 3 nouveaux tests.

8. **Le SSE passe de 403 à 404.** Aucun impact UI : le frontend absorbait déjà
   ces 403 (`sse-client.ts` basculait en `polling`). Le Test 3 du commanditaire
   doit maintenant montrer **404**.

## Questions bloquantes

**Aucune.**

---

## Leçon à retenir — règle permanente à ajouter aux prompts frontend

> **Critère d'acceptation obligatoire** : « Aucun hook React (`useState`,
> `useEffect`, `useMemo`, `useCallback`, `useRef`) n'est appelé **après** un
> `return` conditionnel dans les fichiers modifiés. »

Cette classe de bug est **invisible à `tsc`** (il ne voit pas l'ordre des
hooks), **invisible aux tests statiques par regex** (qui vérifient la présence,
pas la position) et **invisible à oxlint**. Seul un lint react-hooks l'attrape
systématiquement — c'est maintenant en place et branché sur `npm run lint`.

Corollaire pour l'agent : **relire ses propres tests et demander « ce test
peut-il passer à vide ? »**. C'est exactement ce qui s'est produit au 4-A.

## Scénario de test production

**Test 1 — Mission disponible**
1. Se connecter comme technicien KYC validé.
2. `/technicien/demandes` → cliquer une mission.
3. **Attendu** : page affichée, **plus d'écran d'erreur**.
4. Console : **aucune** erreur `Rendered more hooks`.

**Test 2 — Mission assignée**
1. Dashboard `/technicien` → « Mes interventions » → ouvrir une mission.
2. **Attendu** : page intégrale (diagnostics, devis, chat, formulaire de devis).

**Test 3 — Masquage SSE**
1. Ouvrir une mission **non assignée** → DevTools Network.
2. **Attendu** : `GET /api/realtime/missions/:id` → **404** (plus 403).

**Test 4 — Digest**
1. Provoquer une erreur quelconque.
2. **Attendu** : écran d'erreur + lien replié **« Détails techniques »** avec le
   `digest` (en production, ni message ni stack).

**Test 5 — Le lint attrape la faute**
1. Réintroduire un `useMemo` sous un `return` conditionnel dans la page mission.
2. **Attendu** : `npm run lint` échoue avec `rules-of-hooks`, et
   `npm run test:unit` échoue sur les 3 tests de non-régression.