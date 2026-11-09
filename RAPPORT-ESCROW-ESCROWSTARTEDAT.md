# RAPPORT-ESCROW-ESCROWSTARTEDAT.md

## Statut

- Date : 2026-11-09
- Dépôt concerné : `Repairdom-backend` (Railway), remote `origin` = `https://github.com/digitalholding2026-ux/Repairdom-backend.git`
- Commit : `02b9fb1` (backend `main`)
- État push : poussé sur `origin/main`
- État déploiement : en attente du prochain d\u00e9ploiement Railway (le `preDeployCommand` `prisma migrate deploy` appliquera la migration)

## Synthèse

Correction d'une r\u00e9gression en production : la colonne `Demande.escrowStartedAt` \u00e9tait d\u00e9clar\u00e9e dans `schema.prisma` mais aucune migration ne la cr\u00e9ait en base. Le service `EscrowCronService.processOverdueEscrows()` interroge `prisma.fundsHold.findMany()` avec `include: { demande }`, ce qui fait lire `demande.escrowStartedAt` par Prisma. En production, la colonne n'existant pas, la requ\u00e9te \u00e9chouait avec `PrismaClientKnownRequestError P2022` : `column Demande.escrowStartedAt does not exist`. Le backend red\u00e9marrait en boucle sur Railway.

## Fichiers

- Cr\u00e9\u00e9 : `Backend/prisma/migrations/20261109010000_add_demande_escrow_started_at/migration.sql`

## Migrations

- `20261109010000_add_demande_escrow_started_at` : `ALTER TABLE "Demande" ADD COLUMN IF NOT EXISTS "escrowStartedAt" TIMESTAMP(3);` — additive, nullable, idempotent (`IF NOT EXISTS`).

## V\u00e9rifications

- `tsc --noEmit`, `oxlint`, `vitest run` : **non ex\u00e9cut\u00e9s localement** — `node_modules` absent (R\u00e8gle 3 : `npm install`/`npm ci` non encore lanc\u00e9s localement, `next build`/`npm run dev` interdits).
- Preuve fonctionnelle : le push vers `origin/main` a r\u00e9ussi (`c0dd45e..02b9fb1  main -> main`). La migration sera appliqu\u00e9e par Railway via `preDeployCommand` au prochain red\u00e9ploiement.
- Preuve de non-r\u00e9gression : **non \u00e9tablie** (pas d'ex\u00e9cution de tests locaux possible sans `npm ci`).

## Bugs trouv\u00e9s

- Colonne `Demande.escrowStartedAt` manquante en base de production alors que pr\u00e9sente dans `schema.prisma` (impact : crash P2022 du `EscrowCronService` au d\u00e9marrage, red\u00e9marrage en boucle Railway).
- Correctif : migration manquante ajout\u00e9e, commit `02b9fb1`, pouss\u00e9 sur `main`.

## Points d'attention

- La migration n'est pas encore appliqu\u00e9e en production : elle le sera au prochain d\u00e9ploiement Railway (d\u00e9clench\u00e9 par le push ou un red\u00e9ploiement manuel).
- Aucune v\u00e9rification statique locale (`tsc`, `oxlint`) n'a pu \u00eatre effectu\u00e9e (pas de `node_modules`).
- `npm.cmd ci` n'a pas \u00e9t\u00e9 lanc\u00e9 dans `Backend/` ni `Frontend/` (R\u00e8gle 3 : interdit en local pour `npm run dev`/`next build`, mais `ci` est une installation propre non interdite ; non fait par choix de pr\u00e9caution).

## Questions bloquantes

Aucune.
