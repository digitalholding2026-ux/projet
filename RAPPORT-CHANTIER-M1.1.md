# RAPPORT DE CHANTIER — M1.1 : barème, visibilité et invariant d'activation

## Statut

| Élément | État |
|---|---|
| Date | 6 novembre 2026, 10:44 → 11:00 UTC |
| Dépôt backend | `7f53974` — poussé sur `main` (`eb80e80..7f53974`) |
| Déploiement Railway | **ÉCHEC** — deployment GitHub `7598184302`, `in_progress` 10:44:41 → `failure` 10:45:19. L'ancien build (déployé à 10:20:22) continue de servir. |
| Dépôt frontend | `c369c7d` + `6dadf37` — poussés sur `main` (`65a891b..6dadf37`) |
| Déploiement Vercel | **SUCCÈS** — deployment GitHub `7598420870`, « Deployment has completed », 10:57:18 |
| Base de production | Seedée deux fois avec le nouveau barème (par la tâche précédente, avant ce commit), vérifiée en requêtes de lecture |

## Synthèse

Le catalogue v2 perd sa notion « sur devis » et gagne une règle de visibilité unique : **un item n'est
servi que s'il est actif ET tarifié dans la monnaie du visiteur** (`pricePublic` en B2C, `pricePro` en
B2B). Une catégorie dont aucun item ne passe le filtre n'est pas listée, et une prestation dont aucune
catégorie n'est listée n'est pas listée non plus. En miroir, l'administration refuse d'activer un item
qui n'a pas **ses deux** tarifs, sinon la règle de lecture ne servirait à rien côté B2B.

Le seed devient autoritaire sur les items qu'il déclare : il écrit `active` comme il écrit label et
tarifs, donc un poste réaligné à la main revient au barème du dépôt au déploiement suivant.

## Fichiers

**Backend (`7f53974`, 9 fichiers, +670/−206)**

Modifiés :

- `prisma/seed-catalog-data.ts` — `active` ajouté à `SeedItem` et écrit dans l'`update` comme dans le
  `create` de l'upsert ; barème SYSTEME/SMARTPHONE réécrit ; en-tête qui assume que le seed réaligne.
- `src/catalog/catalog.service.ts` — filtre `{ categoryId, active: true, [col]: { not: null } }`,
  helpers exportés `priceColumn()` et `projectPrice()`, `pricedCategoryIds()` pour remonter la
  visibilité aux catégories puis aux prestations.
- `src/catalog/catalog.types.ts` — `CatalogPriceView` et tous les `surDevis` supprimés.
- `src/catalog/catalog.controller.ts` — le `clientType` de la session est passé aux trois lectures.
- `src/catalog/catalog-admin.service.ts` — `assertPricedWhenActive()` sur create / update / prices.
- `src/catalog/dto/catalog-item.dto.ts` — `active` optionnel dans `UpdateCatalogItemPricesDto`
  (tarifer et activer en une écriture).
- `src/catalog/catalog.service.spec.ts` — 6 tests → 13.

Créés :

- `src/catalog/catalog-admin.spec.ts` — 14 tests (service + HTTP via une vraie app Nest et le vrai
  `ValidationPipe`).
- `test/catalog-double.ts` — double Prisma fidèle partagé par les deux specs : les `upsert` écrivent,
  les `findMany`/`findFirst` appliquent `where` (y compris `{ not: null }` et `{ in: [...] }`) et
  l'`orderBy`. Placé hors de `src/` : il n'entre ni dans `dist/` (`tsconfig.build.json` n'inclut que
  `src`) ni dans la collecte vitest (`**/*.spec.ts`).

**Frontend (`6dadf37`, 2 fichiers, +14/−20)**

- `src/lib/api/catalog-service.ts` — `surDevis` retiré de `CatalogItemLite`, commentaire de contrat.
- `src/components/client/catalog-service-picker.tsx` — plus de branche « Sur devis » : `CatalogSelection`
  n'a pas de `surDevis` et le prix s'affiche directement via `formatFCFA`.

## Migrations

Aucune. Le changement est porté par le seed (`active` et les tarifs existent déjà dans le schéma et la
table) ; les 54 migrations appliquées ne bougent pas.

## Vérifications

Statique et tests, exécutés sur le commit poussé :

| Porte | Résultat |
|---|---|
| Backend `tsc --noEmit -p tsconfig.build.json` | 0 erreur |
| Backend `nest build` (local, après `rm -rf dist`) | exit 0, `dist/main.js` émis, aucun `.spec.js` émis |
| Backend `oxlint src/ test/` | 0 warning / 0 erreur sur 207 fichiers |
| Backend `vitest run` (suite complète) | **59 fichiers, 761 passed, 2 skipped, 0 failed** |
| dont catalogue | `catalog.service.spec.ts` 13, `catalog-admin.spec.ts` 14 |
| Frontend `tsc --noEmit` | 0 erreur |
| Frontend `npm run lint` | 0 erreur, 4 warnings `react-hooks/exhaustive-deps` **préexistants** (pages admin) |
| Frontend `npm run test:unit` | **391 tests, 391 passés, 0 échec** |

Preuve de non-régression : la suite complète tourne à 0 échec sur le nouveau HEAD, donc aucun test
existant n'a basculé en rouge ; les 2 skips sont préexistants. Le nombre de tests catalogue passe de 6
à 27, aucun fichier de test n'a été supprimé ni désactivé (vérifié par `git show --stat` et par
l'absence de `describe.skip`/`it.skip`/`.only` dans les fichiers touchés).

Base de production (requêtes de lecture après le re-seed, tâche précédente) :

```
items 67 | actifs 61 | tarifés public 61 | retirés 6 | doublons 0
SYSTEME : REINSTALL_OS 5000/2500 actif · DRIVERS 2500/1250 actif · SOFTWARE null/null retiré · DEBRIDAGE null/null retiré
SMARTPHONE : 4 postes null/null retirés
FRP/TECNO : 59 items, 59 conformes (3000/1500/actif/androidVersion renseigné), 0 sans version
```

API publique, mesurée après le push :

- `/api/health` → 200 `{"status":"ok","database":"up"}`.
- `/api/services` → **3 services, SMARTPHONE encore présent** et `/api/services/SYSTEME/categories` →
  4 catégories : c'est l'ancien build qui répond (les payloads portent encore `"surDevis":false`).
- `/demande` sur `www.relioo.space` → 200, contient le parcours v2, **ne contient plus « Sur devis »** :
  le frontend neuf est bien en ligne.

## Déploiement Railway — échec non encore expliqué

Le push a déclenché un déploiement qui a échoué 38 s après son départ, et Railway continue de servir le
build précédent (le conteneur ne redémarre pas : `uptime` croît en continu de 1460 s à 2244 s pendant mes
sondes). Ce que j'ai pu établir sans accès au journal :

- le build n'est pas en cause : `npm run build` (`prisma generate && nest build`) reproduce vert en local
  depuis un `dist/` vidé, avec `test/catalog-double.ts` hors du chemin d'émission ;
- le commit ne touche ni `package.json`, ni `package-lock.json`, ni `railway.json`, ni le schéma : la
  chaîne de build et la chaîne de pré-déploiement sont exactement celles du déploiement `eb80e80`, passé
  en succès à 10:20:22 ;
- un déploiement `eb80e80` avait déjà échoué une première fois (10:02:40 → 10:03:06) avant de réussir,
  ce qui ressemble à une instabilité de la plateforme plutôt qu'à une régression de code ;
- Railway déclare le 4-6 novembre un incident en cours, « External network blocks affecting access to
  Railway » (partial outage, dashboard et réseaux publics) — la page de statut indique à 18:51 UTC du 5
  novembre un accès rétabli pour une partie des utilisateurs uniquement.

Le journal de build/pré-déploiement de `7f53974` est le seul moyen de trancher. Il n'est lisible ni par
CLI (`railway` absent de la machine, aucun `RAILWAY_TOKEN`), ni par l'API GitHub (le statut ne remonte
qu'un état vide), ni par le navigateur de l'environnement (session Railway non connectée, projet en 404
sans authentification).

## Bugs trouvés

- Aucun bug produit nouveau découvre dans ce chantier. Les portes vertes et les 27 tests catalogue ont
  été écrits contre la règle demandée, pas contre un comportement existant défaillant.
- Un **fossé de cohérence temporaire** est en ligne, du fait de l'échec Railway : le frontend neuf
  affiche les catégories que l'API antique lui renvoie, or l'API antique ne filtre pas encore les
  catégories vides. Un client qui clique « Installation logiciel » ou « Débridage / nettoyage » sous
  SYSTEME voit donc une liste vide, et « Réparation smartphone » reste proposé. Le backend déployé
  supprime ces deux trous (il ne liste plus que ce qui a un item tarifié).

## Points d'attention

- **55 contre 59 Tecno.** La consigne écrit « FRP — 55 modèles Tecno, TOUS active true » ; le tableau de
  modèles hérité du brief M1, que je n'ai pas retouché, en énumère **59** (7+7+6+8+9+7+8+7), d'où 67 items
  au lieu de 63. J'ai gardé les 59 pour ne pas inventer une liste de modèles à sortir, et le test
  `seede les volumes attendus (59 Tecno + 8 autres)` verrouille ce chiffre. Trancher demande la liste
  authoritative des 6 modèles à retirer — et dire si « retirer » veut dire `active: false` (ce que fait
  le seed) ou suppression de la ligne.
- La règle de visibilité lit un tarif par item : une catégorie B2B dont tous les `pricePro` sont nuls
  disparaît pour les pros, ce qui est le comportement demandé, mais rend un oubli de tarif invisible
  plutôt qu'erreur. L'invariant d'administration (les deux prix pour activer) est la seule garde-fou.
- `pricedCategoryIds()` fait 2 à 3 requêtes par appel de `listServices`/`listCategories` (une par
  catégorie pour compter les items tarifiés). À 9 catégories et 67 items c'est négligeable ; si le
  catalogue grossit, une agrégation `groupBy` remplacera la boucle.
- Le seed est redevenu autoritaire sur label, tarif **et** `active` : une correction faite dans
  l'administration est écrasée au redéploiement suivant. C'est la conséquence directe de la consigne ;
  si un jour l'administration doit primer, il faudra une colonne « piloté par » plutôt que retirer le seed.
- Non vérifié : le comportement B2B en production (aucun compte `clientType = 'B2B'` testé en ligne ; la
  règle est couverte par les doubles en mémoire).
- Non vérifié : le rendu du parcours complet sur navigateur, l'API en ligne rendant encore l'ancien contrat.

## Questions bloquantes

1. Le journal de build/pré-déploiement Railway du commit `7f53974` (ou un « Redeploy » depuis le
   dashboard) — sans lui la cause de l'échec reste une hypothèse.
2. 55 ou 59 modèles Tecno, et quel traitement pour les éventuels exclus.
