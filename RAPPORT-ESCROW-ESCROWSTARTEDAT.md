# RAPPORT-ESCROW-ESCROWSTARTEDAT.md

## Statut

- Date : 2026-11-09
- Dépôt concerné : `Repairdom-backend` (Railway), remote `origin` = `https://github.com/digitalholding2026-ux/Repairdom-backend.git`
- Commits : `02b9fb1` (ajout colonne escrowStartedAt) + `fc7dce4` (récupération tables financières), backend `main`
- État push : poussés sur `origin/main`
- État déploiement : en attente du prochain déploiement Railway (le `preDeployCommand` `prisma migrate deploy` appliquera les migrations)

## Synthèse

Correction d'une régression en production : le backend redémarrait en boucle sur Railway à cause d'erreurs Prisma au démarrage du `EscrowCronService`.

Deux problèmes successifs ont été identifiés et corrigés :

1. **Colonne `Demande.escrowStartedAt` manquante** (`P2022`) : la colonne était déclarée dans `schema.prisma` mais aucune migration ne la créait en base. Ajout de la migration manquante `20261109010000_add_demande_escrow_started_at`.

2. **Tables financières manquantes** (`P2021`, `table public.FundsHold does not exist`) : après application de la première migration, le service a poursuivi son exécution et échoué sur l'absence de la table `FundsHold` (et vraisemblablement `FinancialTransaction`, `TopupIntent`, `WithdrawalRequest`). La base de production est en drift : certaines migrations financières sont enregistrées comme appliquées dans `_prisma_migrations` mais leurs tables n'existent pas réellement, donc `prisma migrate deploy` les saute. Ajout d'une migration de récupération `20261109020000_recover_financial_tables` qui recrée tous les objets financiers avec des gardes d'idempotence (`IF NOT EXISTS`, `EXCEPTION WHEN duplicate_object`, `undefined_table`).

## Fichiers

- Créé : `Backend/prisma/migrations/20261109010000_add_demande_escrow_started_at/migration.sql`
- Créé : `Backend/prisma/migrations/20261109020000_recover_financial_tables/migration.sql`

## Migrations

- `20261109010000_add_demande_escrow_started_at` : `ALTER TABLE "Demande" ADD COLUMN IF NOT EXISTS "escrowStartedAt" TIMESTAMP(3);` — additive, nullable, idempotent.
- `20261109020000_recover_financial_tables` : recrée les énumérations (`FinancialTransactionType`, `FinancialTransactionDirection`, `FinancialTransactionStatus`, `FinancialTransactionMode`, `SasPayOperationStatus`, `FundsHoldStatus`), les tables (`FinancialTransaction`, `FundsHold`, `TopupIntent`, `WithdrawalRequest`, `RelioWithdrawal`), les index et les clés étrangères, toutes avec gardes d'idempotence.

## Vérifications

- `tsc --noEmit`, `oxlint`, `vitest run` : **non exécutés localement** — `node_modules` absent (Règle 3 : `npm install`/`npm ci` non encore lancés localement, `next build`/`npm run dev` interdits).
- Preuve fonctionnelle : les deux pushes vers `origin/main` ont réussi (`c0dd45e..02b9fb1` puis `02b9fb1..fc7dce4`). Les migrations seront appliquées par Railway via `preDeployCommand` au prochain redéploiement.
- Preuve de non-régression : **non établie** (pas d'exécution de tests locaux possible sans `npm ci`).

## Bugs trouvés

- Colonne `Demande.escrowStartedAt` manquante en production (impact : crash P2022, redémarrage en boucle). Correctif : commit `02b9fb1`.
- Tables `FinancialTransaction`, `FundsHold`, `TopupIntent`, `WithdrawalRequest` manquantes en production (impact : crash P2021 après correction de la colonne). Correctif : commit `fc7dce4`.
- Racine de cause : drift entre `_prisma_migrations` et l'état réel de la base de production.

## Points d'attention

- Les migrations ne sont pas encore appliquées en production : elles le seront au prochain déploiement Railway (déclenché par le push ou un redéploiement manuel).
- Aucune vérification statique locale (`tsc`, `oxlint`) n'a pu être effectuée (pas de `node_modules`).
- `npm.cmd ci` n'a pas été lancé dans `Backend/` ni `Frontend/`.

## Questions bloquantes

Aucune.
