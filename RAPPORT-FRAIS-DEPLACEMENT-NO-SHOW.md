# RAPPORT-FRAIS-DEPLACEMENT-NO-SHOW.md

## Statut

- Date : 2026-11-09
- Dépôts concernés : `Repairdom-backend` (Railway), `Repairdom-frontend` (Vercel), racine `projet` (rapport uniquement)
- Commits : backend `ba72a06`, frontend `b4f9a9e`
- Push : backend poussé puis frontend poussé (ordre RÈGLE 2 respecté)
- Déploiement : non vérifié (pas d'attente Railway/Vercel dans cette session)

## Synthèse

Implémentation des "Frais de Déplacement" (booking fee) et de la "Preuve de passage" (no-show) dans le parcours client et admin.

- FRP et Système : le client paie un dépôt (frais de déplacement, ex. 1000 FCFA) à la validation via `pay-escrow` ; le reste est payé après la réparation.
- Smartphone : escrow complet (comportement existant, hold du `finalAmount` total).
- Admin : bouton "Signaler un client absent (No-Show)" qui marque `isNoShow`, libère le FundsHold de dépôt en faveur de Relio, et passe le statut en `CANCELED`.

## Fichiers

### Modifiés — backend (`Repairdom-backend`)

| Chemin | Nature |
|---|---|
| `prisma/schema.prisma` | `Demande.depositAmount` (Int?), `depositPaid` (Boolean), `isNoShow` (Boolean) ; `CatalogItem.travelFee` (Int?) |
| `src/invoice/invoice.controller.ts` | `POST /api/client/demandes/:id/pay-escrow` : dépôt partiel FRP/Système, total Smartphone |
| `src/invoice/invoice.service.ts` | `createEscrowHold` : montant personnalisé, `depositPaid` |
| `src/admin/admin.controller.ts` | `POST /api/admin/demandes/:id/no-show` (était `PATCH`) |
| `src/admin/admin.service.ts` | `noShowDemande` : `isNoSee=true`, libération hold dépôt, `CANCELED` |
| `src/catalog/dto/catalog-item.dto.ts` | `UpdateCatalogItemPricesDto.travelFee` |
| `src/catalog/catalog-admin.service.ts` | `travelFee` dans `ItemRow` et `updatePrices` |

### Modifiés — frontend (`Repairdom-frontend`)

| Chemin | Nature |
|---|---|
| `src/lib/api/catalog-service.ts` | `CatalogItemLite.travelFee` |
| `src/lib/api/admin-service.ts` | `AdminCatalogItem.travelFee`, `updateCatalogItemPrices`, `noShowAdminDemande` (POST), `SupervisedMission` |
| `src/lib/api/request-service.ts` | `DemandeListItem.depositAmount/depositPaid/isNoShow` |
| `src/app/admin/catalogue/page.tsx` | champ "Frais de déplacement" édition item |
| `src/app/client/demandes/[id]/page.tsx` | badge "Réservation payée" + reste à payer |
| `src/app/admin/demandes/[id]/page.tsx` | bouton rouge No-Show + alerte acquisition Relio |
| `src/components/client/demande-wizard.tsx` | `depositAmount` draft, affichage étape 3 |

### Créés — racine (`projet`)

| Chemin | Nature |
|---|---|
| `RAPPORT-FRAIS-DEPLACEMENT-NO-SHOW.md` | ce rapport |

## Migrations

Aucune migration Prisma n'a été générée/exécutée (champs ajoutés au schéma seulement ; `prisma migrate dev` non lancé, base locale absente). À appliquer sur les environnements cibles via `prisma migrate deploy`.

## Vérifications

- `tsc --noEmit -p tsconfig.build.json` (backend) : OK
- `tsc --noEmit` (frontend) : OK
- `oxlint` (backend + frontend) : OK, seuls des warnings pré-existants (imports/vars non utilisés dans des fichiers non modifiés)
- `vitest run` (backend) : 52 fichiers de tests unitaires purs passent ; 8 fichiers échouent car ils nécessitent Postgres local (interdit en local par RÈGLE 3) — échecs pré-existants, non liés à cette tâche
- Non-régression : non prouvée sur `HEAD` propre (absence de base locale) ; les tests purs passent

## Bugs trouvés

Aucun bug introduit. Aucun bug régressif détecté sur les tests purs.

## Points d'attention

- Le solde client et le règlement définitif (reste à payer après réparation) ne sont pas encore connectés au ledger `CLIENT_MISSION_DEBIT` : le dépôt est réservé (FundsHold ACTIVE) mais le débit complet n'intervient qu'à la confirmation via `settleMissionAtConfirmation`, qui traite le hold comme un hold mission classique. Le reste à payer reste à débiter côté client selon la logique métier à préciser.
- L'affichage "reste à payer" côté client est calculé frontend (`finalAmount - depositAmount`) sans écriture ledger correspondante.
- La migration de schéma n'a pas été appliquée (pas de base locale) ; à déployer sur Railway avant que le frontend ne puisse utiliser les nouveaux champs.

## Questions bloquantes

Aucune.
