# RAPPORT — Fix migration `Service.proAvailable` (Railway)

## Statut
- Date : 2026-11-07
- Dépôt concerné : `Backend/` (`Repairdom-backend`, déploiement Railway) — diagnostic ; **aucune modification de code poussée** dans ce dépôt pour ce correctif
- Dépôt racine : `projet` — ce rapport
- Commit(s) applicatifs : aucun créé par cette intervention (correctif déjà présent)
- État push/déploiement : problème signalé résolu par l’utilisateur après action sur l’environnement Railway/DB ; `/api/services` à confirmer en 200 sans erreur Prisma

## Synthèse
Les requêtes `GET /api/services` renvoyaient 500 en production avec `PrismaClientKnownRequestError` : `The column \`Service.proAvailable\` does not exist in the current database` (appel `prisma.service.findMany()` dans `CatalogV2Service.listServices`). Le schéma Prisma et le code attendaient `Service.proAvailable`, mais la migration correspondante n’était pas appliquée sur la base Railway. La migration et le pipeline de déploiement étaient déjà présents dans le dépôt ; la résolution a consisté à appliquer la migration en attente / réaligner le déploiement Railway.

## Fichiers
### Créés
- `RAPPORT-FIX-PROAVAILABLE-MIGRATION.md` (racine)

### Modifiés
- Aucun

### Supprimés
- Aucun

## Migrations
- `Backend/prisma/migrations/20261106020000_add_service_pro_available/migration.sql` (existante) : `ALTER TABLE "Service" ADD COLUMN IF NOT EXISTS "proAvailable" BOOLEAN NOT NULL DEFAULT true;` (additive, idempotente, `DEFAULT true`).
- Migration précédente liée : `20261106010000_add_catalog_v2_services` (création `Service`/catalogue v2).
- Application prod : à effectuer via `prisma migrate deploy` (pre-deploy Railway ou exécution manuelle contrôlée avec Prisma 7.10.0 et la `DATABASE_URL` publique Railway). **Ne pas** utiliser `prisma migrate reset` en production.

## Vérifications
### Diagnostic (lecture seule, effectué)
- Migration `proAvailable` présente dans `Backend/prisma/migrations/`.
- `Backend/prisma/schema.prisma` : `proAvailable Boolean @default(true)` sur `Service`.
- `Backend/package.json` : `build` = `prisma generate && nest build` ; `db:migrate:deploy` = `prisma migrate deploy` ; `db:seed:catalog` = `node prisma/seed-catalog.ts`.
- `Backend/railway.json` : `preDeployCommand` = `npm run db:migrate:deploy && npm run db:seed:catalog` ; `startCommand` = `npm run start:prod` ; healthcheck `/api/health`.
- `Backend/src/catalog/catalog.service.ts` / controller : usage cohérent de `proAvailable` et tarif selon `clientType`.
- Seed catalogue v2 : `proAvailable` défini (SYSTEME/FRP `true`, SMARTPHONE `false`).

### Exécution locale des tests/lint
- Non exécutée dans cette intervention (aucune modification de code). Node/npm non disponibles dans le PATH de la session au moment de la tentative de commande Prisma locale ; dépendances `Backend/node_modules` absentes localement.

### Preuve de non-régression
- N/A pour le code (aucun changement). Risque résiduel couvert par le caractère idempotent/additif de la migration (`IF NOT EXISTS`, `DEFAULT true`).

### Vérification post-correction (attendue / à confirmer)
- `curl -s https://repairdom-backend-production.up.railway.app/api/services` → HTTP 200, JSON avec les 3 services (SYSTEME, SMARTPHONE, FRP), sans erreur Prisma.
- Logs Railway : plus de `column Service.proAvailable does not exist`.
- Présence/cohérence de `proAvailable` selon la projection API exposée.

## Bugs trouvés
| Bug | Impact | Correctif |
|---|---|---|
| Migration `20261106020000_add_service_pro_available` non appliquée en base prod | `GET /api/services` 500, catalogue indisponible | Appliquer `prisma migrate deploy` sur la base Railway ; s’assurer que Railway exécute `preDeployCommand`/`startCommand` de `railway.json` |

## Points d’attention
- Les logs fournis montraient `npm run start` / `nest start` (dev) plutôt que `start:prod` et sans trace visible du pre-deploy : vérifier dans Railway qu’aucun override n’écrase `railway.json`.
- Toute exécution manuelle de migration doit utiliser **Prisma 7.10.0** (version du projet), pas un `npx prisma` non épinglé.
- Le token/URL de base fournie en conversation est un **secret** : ne jamais le committer, le journaliser ou l’inclure dans un rapport. Le faire tourner si exposé hors canal sécurisé.
- `Backend/` et `Frontend/` apparaissent comme répertoires non suivis dans le dépôt racine ; ne pas les ajouter à l’index racine.

## Questions bloquantes
Aucune.
