# RAPPORT-CHANTIER-M2-WHATSAPP-CONFIRMATION

## Statut

- Date : 2026-11-10
- Dépôts concernés : `Repairdom-backend` (main), `Repairdom-frontend` (main), `projet` (main, rapport uniquement)
- Commits :
  - Backend : `6c2309d` feat(m2): replace booking fees avec WhatsApp confirmation workflow
  - Backend : `deefbaa` fix(api): ajouter colonne travelFee manquante sur CatalogItem (drift production)
  - Backend : `dcd2e95` fix(api): ajouter colonne depositAmount manquante sur Demande (drift production)
  - Backend : `a1ffa30` fix(api): ajouter colonnes depositPaid et isNoShow manquantes sur Demande (drift production)
  - Backend : `1380c13` feat(api): autoriser dépôt de demande anonyme (client invité FRP/Système)
  - Frontend : `ae27db8` feat(m2): replace booking fees avec WhatsApp confirmation workflow
  - Frontend : `cce3ae3` feat(ui): utiliser la vraie photo admin comme avatar par défaut dans l'encart technicien
  - Frontend : `87eeea5` feat(wizard): dépôt de demande anonyme pour FRP/Système + suivi par token
- État push : backend poussé (origin/main à jour), frontend poussé (origin/main à jour)
- Déploiement : en attente (Railway pour le backend, Vercel pour le frontend) — non vérifié factuellement

## Synthèse

Implémentation du workflow de confirmation WhatsApp et suppression de la friction client (chantier M2) :
- FRP et Système : pas de compte requis, dépôt de demande anonyme avec numéro WhatsApp, suivi par token
- Smartphone : création de compte obligatoire (escrow), paiement sécurisé en ligne
- Corrections de drift production : travelFee, depositAmount, depositPaid, isNoShow

## Fichiers

### Créés

**Backend (`Repairdom-backend`) :**
- `prisma/migrations/20261110040000_add_guest_client_support/migration.sql` — migration pour support client invité

**Frontend (`Repairdom-frontend`) :**
- `public/admin.jpg` — photo de l'admin (avatar par défaut encart technicien)
- `src/app/suivi/[token]/page.tsx` — page de suivi public anonyme par token UUID

### Modifiés

**Backend (`Repairdom-backend`) :**
- `prisma/schema.prisma` — `User.passwordHash` optionnel, `Demande.trackingToken` ajouté
- `src/auth/auth.service.ts` — rejet explicite des comptes sans mot de passe à la connexion
- `src/demandes/demande-helpers.ts` — `trackingToken` dans `DemandeRecord` et `toApiDemande`
- `src/demandes/demandes.controller.ts` — gardes auth retirés de `POST /demandes`, ajoutés sur toutes les autres routes
- `src/demandes/demandes.service.ts` — `create()` supporte client anonyme (user fantôme + trackingToken)
- `src/tracking/tracking.controller.ts` — `GET /api/tracking/token/:token` ajouté
- `src/tracking/tracking.service.ts` — `trackByToken()` ajouté

**Frontend (`Repairdom-frontend`) :**
- `src/components/client/demande-wizard.tsx` — étape 3 différenciée : FRP/Système sans compte, SMARTPHONE avec compte
- `src/components/client/demande-auth-modal.tsx` — prop `prefillPhone` ajoutée
- `src/lib/api/request-service.ts` — `trackingToken` dans `CreateDemandeResult`
- `src/lib/api/tracking-service.ts` — `trackByToken()` ajouté
- `src/app/client/demandes/[id]/page.tsx` — bandeau de succès différencié selon le service

### Supprimés

Aucun fichier supprimé.

## Migrations

- `20261110010000_add_catalog_item_travel_fee` — ajoute `travelFee` INTEGER sur `CatalogItem` (idempotente).
- `20261110020000_add_demande_deposit_amount` — ajoute `depositAmount` INTEGER sur `Demande` (idempotente).
- `20261110030000_add_demande_deposit_paid` — ajoute `depositPaid` BOOLEAN et `isNoShow` BOOLEAN sur `Demande` (idempotentes).
- `20261110040000_add_guest_client_support` — rend `User.passwordHash` nullable, ajoute `Demande.trackingToken` TEXT UNIQUE (idempotente).

## Vérifications

- `tsc --noEmit` (backend, `tsconfig.build.json`) : réussi
- `tsc --noEmit` (frontend) : réussi
- `npm run lint` (backend) : réussi (warnings pré-existants uniquement)
- `npm run lint` (frontend) : réussi (warnings pré-existants uniquement)
- `npx prisma generate` : réussi (Prisma Client 7.10.0 généré)
- `git push origin main` backend : `a1ffa30..1380c13 main -> main` — OK
- `git push origin main` frontend : `cce3ae3..87eeea5 main -> main` — OK
- `git status -sb` à la racine `projet.git` : `?? Backend/` et `?? Frontend/` uniquement (RÈGLE 2 respectée)

## Bugs trouvés

1. **Bug de production (Railway) — `CatalogItem.travelFee` manquant.**
   - Symptôme : `prisma.catalogItem.upsert()` échoue avec `P2022: ColumnNotFound`.
   - Cause : migration manquante pour la colonne `travelFee`.
   - Correctif : `20261110010000_add_catalog_item_travel_fee` (commit `deefbaa`).

2. **Bug de production (Railway) — `Demande.depositAmount` manquant.**
   - Symptôme : `EscrowCronService` échoue avec `P2022: ColumnNotFound` sur `Demande.depositAmount`.
   - Cause : migration manquante pour la colonne `depositAmount`.
   - Correctif : `20261110020000_add_demande_deposit_amount` (commit `dcd2e95`).

3. **Bug de production (Railway) — `Demande.depositPaid` et `Demande.isNoShow` manquants.**
   - Symptôme : `EscrowCronService` échoue avec `P2022: ColumnNotFound` sur `Demande.depositPaid`.
   - Cause : migrations manquantes pour ces colonnes.
   - Correctif : `20261110030000_add_demande_deposit_paid` (commit `a1ffa30`).

Les 24 échecs de tests vitest backend sont pré-existants (absence de base de données locale PostgreSQL, interdite par RÈGLE 3). Aucun échec lié aux modifications apportées.

## Points d'attention

- Déploiement en attente : Railway (backend) puis Vercel (frontend). À confirmer factuellement après déploiement.
- Node.js v24.22.2 + npm 11.19.1 installés sur la machine (prérequis RÈGLE 4).
- Les fichiers `.env` n'ont pas été créés depuis `.env.example` dans les deux dépôts applicatifs.
- `User.passwordHash` est maintenant nullable : les comptes fantômes créés pour les clients anonymes ne peuvent pas se connecter par mot de passe (comportement voulu).

## Questions bloquantes

Aucune.
