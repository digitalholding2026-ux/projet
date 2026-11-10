# RAPPORT-CHANTIER-M2-WHATSAPP-CONFIRMATION

## Statut

- Date : 2026-11-10
- Dépôts concernés : `Repairdom-backend` (main), `Repairdom-frontend` (main), `projet` (main, rapport uniquement)
- Commits :
  - Backend : `6c2309d` feat(m2): replace booking fees avec WhatsApp confirmation workflow
  - Backend : `deefbaa` fix(api): ajouter colonne travelFee manquante sur CatalogItem (drift production)
  - Frontend : `ae27db8` feat(m2): replace booking fees avec WhatsApp confirmation workflow
- État push : backend poussé (origin/main à jour), frontend poussé (origin/main à jour)
- Déploiement : en attente (Railway pour le backend, Vercel pour le frontend) — non vérifié factuellement

## Synthèse

Implémentation du workflow de confirmation WhatsApp (chantier M2) : abandon des frais de réservation en ligne pour FRP et Système, remplacement par une confirmation de rendez-vous via WhatsApp avec escrow conservé uniquement pour le parcours Smartphone.

## Fichiers

### Créés

Aucun fichier créé.

### Modifiés

**Backend (`Repairdom-backend`) :**
- `src/admin/admin.controller.ts` — `noShowDemande` passe de `POST` à `PATCH /api/admin/demandes/:id/no-show`
- `src/admin/admin.service.ts` — `noShowDemande` met désormais `confirmationStatus = CANCELLED_BY_ADMIN`, `isNoShow = true`, `status = CANCELED` ; libère le FundsHold de dépôt

**Frontend (`Repairdom-frontend`) :**
- `src/lib/api/request-service.ts` — ajout du champ `confirmationStatus` au type `CreateDemandeResult`
- `src/lib/api/admin-service.ts` — `noShowAdminDemande` passe en `PATCH`, réponse inclut `confirmationStatus`
- `src/app/client/demandes/[id]/page.tsx` — affichage conditionnel du statut de confirmation : "En attente de confirmation WhatsApp" (PENDING), "Rendez-vous confirmé" avec date/heure prévue (CONFIRMED), "Confirmation annulée" (CANCELLED_BY_ADMIN / NO_SHOW)
- `src/app/demande/succes/page.tsx` — message mis à jour : "Votre demande est enregistrée. Un technicien vous contactera sur WhatsApp… Aucun paiement en ligne n'est requis pour le moment"

### Supprimés

Aucun fichier supprimé.

## Migrations

- `20261110010000_add_catalog_item_travel_fee` — ajoute la colonne `travelFee` INTEGER sur `CatalogItem` (idempotente, `ADD COLUMN IF NOT EXISTS`). Corrige le drift production : le schéma Prisma déclare ce champ mais la migration `20261106010000_add_catalog_v2_services` avait créé la table sans.
- L'enum `ConfirmationStatus` et le champ `confirmationStatus` existaient déjà dans le schéma Prisma et la migration `20261109030000_add_demande_confirmation_status`. L'endpoint `PATCH /api/admin/demandes/:id/confirm` existait déjà avec la vérification Escrow (FundsHold ACTIVE) pour le SMARTPHONE.

## Vérifications

- `tsc --noEmit` (backend, `tsconfig.build.json`) : réussi
- `tsc --noEmit` (frontend) : réussi
- `npm run lint` (backend) : réussi (warnings pré-existants uniquement, aucun introduit)
- `npm run lint` (frontend) : réussi (warnings pré-existants uniquement, aucun introduit)
- `npx vitest run` (backend) : 24 échecs, tous pré-existants (absence de base de données locale PostgreSQL, interdite en local par RÈGLE 3 ; mocks désuets sur Collaboration/Reviews/Disputes). Aucun échec lié aux modifications apportées.
- `git push --dry-run` sur les 3 dépôts avant modification : `Everything up-to-date`
- `git push origin main` backend : `ba72a06..6c2309d main -> main` — OK
- `git push origin main` backend (fix travelFee) : `6c2309d..deefbaa main -> main` — OK
- `git push origin main` frontend : `b4f9a9e..ae27db8 main -> main` — OK
- `git status -sb` à la racine `projet.git` : `?? Backend/` et `?? Frontend/` uniquement (RÈGLE 2 respectée, pas d'indexation des sous-dossiers)

## Bugs trouvés

1. **Bug de production (Railway) — `CatalogItem.travelFee` manquant.**
   - Symptôme : `prisma.catalogItem.upsert()` dans `seed-catalog-data.ts` échoue avec `P2022: ColumnNotFound` sur `CatalogItem.travelFee`.
   - Cause : le schéma Prisma déclare `CatalogItem.travelFee Int?` mais la migration `20261106010000_add_catalog_v2_services` a créé la table sans cette colonne. La base de production est en drift.
   - Correctif : nouvelle migration `20261110010000_add_catalog_item_travel_fee` ajoutant `ALTER TABLE "CatalogItem" ADD COLUMN IF NOT EXISTS "travelFee" INTEGER` (idempotente).
   - Commit : `deefbaa` sur `Repairdom-backend`, poussé sur `origin/main`.

Les 24 échecs de tests vitest backend sont pré-existants (absence de base de données locale PostgreSQL, interdite par RÈGLE 3 ; mocks désuets sur Collaboration/Reviews/Disputes). Aucun échec lié aux modifications apportées.

## Points d'attention

- Déploiement en attente : Railway (backend) puis Vercel (frontend). À confirmer factuellement après déploiement.
- Node.js v24.22.2 + npm 11.19.1 installés sur la machine (prérequis RÈGLE 4).
- Les fichiers `.env` n'ont pas été créés depuis `.env.example` dans les deux dépôts applicatifs (étape 11 du guide d'initialisation, non couverte par ce chantier).

## Questions bloquantes

Aucune.
