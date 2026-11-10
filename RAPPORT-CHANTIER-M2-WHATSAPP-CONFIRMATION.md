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
  - Backend : `0595181` feat(tracking): add anonymous tracking API and status update endpoint
  - Backend : `01e9334` feat(settings): ajouter le modèle PlatformSetting et API de paramètres globaux
  - Backend : `31f45fa` fix(demandes): remove legacy V1 validations from CreateDemandeDto
  - Frontend : `ae27db8` feat(m2): replace booking fees avec WhatsApp confirmation workflow
  - Frontend : `cce3ae3` feat(ui): utiliser la vraie photo admin comme avatar par défaut dans l'encart technicien
  - Frontend : `87eeea5` feat(wizard): dépôt de demande anonyme pour FRP/Système + suivi par token
  - Frontend : `3ac13b3` feat(tracking): create anonymous tracking page with timeline
  - Frontend : `a8296e2` feat(settings): page paramètres admin + liens WhatsApp configurables
  - Frontend : `c0cad06` fix(wizard): reset state on service change and remove legacy deposit UI
  - Frontend : `c0fbc58` fix(wizard): add missing 'use client' directive
  - Backend : `1600658` feat(tracking): ajouter API de recherche de demandes par numéro de téléphone
  - Backend : `4cfaa95` fix(auth): add rate-limiting to login endpoint
  - Frontend : `7c0894c` feat(tracking): ajouter page de recherche par numéro de téléphone
  - Frontend : `47fe0cf` fix(auth-ui): add guest tracking link and unify password validation rules
- Frontend : `9317517` fix(ui): remplacer le panneau V1 'Techniciens vérifiés' par 'Technicien agréé' sur connexion/inscription
- État push : backend poussé (origin/main à jour), frontend poussé (origin/main à jour)
- Déploiement : en attente (Railway pour le backend, Vercel pour le frontend) — non vérifié factuellement

## Synthèse

Implémentation du workflow de confirmation WhatsApp et suppression de la friction client (chantier M2) :
- FRP et Système : pas de compte requis, dépôt de demande anonyme avec numéro WhatsApp, suivi par token `/suivi/[token]`
- Smartphone : création de compte obligatoire (escrow), paiement sécurisé en ligne
- Corrections de drift production : travelFee, depositAmount, depositPaid, isNoShow
- Suivi public anonyme et workflow de changement de statut admin
- Système de paramètres globaux (Settings) pour configurer le nom, le numéro WhatsApp et l'avatar du technicien depuis le back-office
- Suppression des validations V1 legacy (categoryId, description obligatoire) sur POST /api/demandes
- Correction de la fuite d'état (state leak) lors du changement de service à l'étape 1 du wizard
- Suppression définitive des frais de déplacement (1000 FCFA) pour FRP/Système — paiement 100% à la fin
- **Recherche de demandes par numéro de téléphone (Guest Checkout)** : `POST /api/demandes/track/search` côté backend, page `/suivi` côté frontend avec listage des demandes en cours et historiques
- **Rate-limiting sur `POST /api/auth/login`** : 5 tentatives par minute via `@nestjs/throttler`, réponse 429 en cas de dépassement
- **Lien WhatsApp et suivi anonyme depuis la page de connexion** : texte et lien vers `/suivi` pour les clients sans compte
- **Validation de mot de passe unifiée** : utilisation de `isPasswordStrong()` (8+ caractères, 1 majuscule, 1 minuscule, 1 chiffre) sur les pages d'inscription et de connexion
- **Remplacement du panneau V1 « Techniciens vérifiés »** par « Technicien agréé » sur la page de connexion et d'inscription, texte et image mis à jour pour le modèle Relio V2 (opérateur unique, service de proximité)

## Fichiers

### Créés

**Backend (`Repairdom-backend`) :**
- `prisma/migrations/20261110040000_add_guest_client_support/migration.sql` — migration pour support client invité
- `src/admin/dto/update-admin-demande-status.dto.ts` — DTO statuts V2 (RECEIVED, IN_PROGRESS, COMPLETED, CANCELLED)
- `src/settings/settings.module.ts` — module Settings
- `src/settings/settings.controller.ts` — `GET /api/settings/public`, `GET/PATCH /api/settings/admin`
- `src/settings/settings.service.ts` — logique métier (get/upsert)
- `src/settings/dto/platform-setting.dto.ts` — DTO de lecture
- `src/settings/dto/update-platform-setting.dto.ts` — DTO de mise à jour
- `src/demandes/dto/track-search.dto.ts` — DTO de recherche par téléphone

**Frontend (`Repairdom-frontend`) :**
- `public/admin.jpg` — photo de l'admin (avatar par défaut encart technicien)
- `src/app/suivi/page.tsx` — page de recherche par numéro de téléphone + suivi par référence
- `src/app/suivi/[token]/page.tsx` — page de suivi public anonyme par token UUID
- `src/app/admin/parametres/page.tsx` — page de configuration des paramètres
- `src/lib/api/settings-service.ts` — `getPublicSettings()`, `getAdminSettings()`, `updateAdminSettings()`

### Modifiés

**Backend (`Repairdom-backend`) :**
- `prisma/schema.prisma` — `User.passwordHash` optionnel, `Demande.trackingToken` ajouté, `PlatformSetting` ajouté
- `prisma/migrations/20261110050000_add_platform_setting/` — migration avec seed par défaut
- `src/auth/auth.service.ts` — rejet explicite des comptes sans mot de passe à la connexion
- `src/demandes/demande-helpers.ts` — `trackingToken` dans `DemandeRecord` et `toApiDemande`
- `src/demandes/demandes.controller.ts` — gardes auth retirés de `POST /demandes`, `GET /api/demandes/tracking/:trackingToken` ajouté
- `src/demandes/demandes.service.ts` — `create()` supporte client anonyme (user fantôme + trackingToken), DTO simplifié
- `src/demandes/demandes.module.ts` — `PrismaModule` ajouté aux imports
- `src/demandes/dto/create-demande.dto.ts` — DTO minimal (serviceCode, catalogItemId, address, phone, description)
- `src/tracking/tracking.controller.ts` — `GET /api/tracking/token/:token` ajouté
- `src/tracking/tracking.service.ts` — `trackByToken()` ajouté, `buildPublicTracking` étendu avec adresse/téléphone
- `src/app.module.ts` — `SettingsModule` enregistré

**Frontend (`Repairdom-frontend`) :**
- `src/components/client/demande-wizard.tsx` — étape 3 différenciée, redirection `/suivi/[token]`, état réinitialisé sur changement de service, suppression encart frais de déplacement
- `src/components/client/demande-auth-modal.tsx` — prop `prefillPhone` ajoutée
- `src/lib/api/request-service.ts` — `CreateDemandeInput` minimal, `createDemande()` simplifié
- `src/lib/api/tracking-service.ts` — `trackByToken()` ajouté, `PublicTracking` étendu avec adresse/téléphone
- `src/lib/api/settings-service.ts` — `getPublicSettings()`, `getAdminSettings()`, `updateAdminSettings()`
- `src/app/client/demandes/[id]/page.tsx` — bandeau de succès différencié selon le service
- `src/app/admin/demandes/[id]/page.tsx` — boutons de statut V2, bouton WhatsApp, facture PDF
- `src/app/admin/layout.tsx` — lien "Paramètres" dans la barre latérale admin
- `src/app/suivi/[token]/page.tsx` — page de suivi public avec timeline, bouton WhatsApp, section facture

### Supprimés

Aucun fichier supprimé.

## Migrations

- `20261110010000_add_catalog_item_travel_fee` — ajoute `travelFee` INTEGER sur `CatalogItem` (idempotente).
- `20261110020000_add_demande_deposit_amount` — ajoute `depositAmount` INTEGER sur `Demande` (idempotente).
- `20261110030000_add_demande_deposit_paid` — ajoute `depositPaid` BOOLEAN et `isNoShow` BOOLEAN sur `Demande` (idempotentes).
- `20261110040000_add_guest_client_support` — rend `User.passwordHash` nullable, ajoute `Demande.trackingToken` TEXT UNIQUE (idempotente).
- `20261110050000_add_platform_setting` — crée la table `PlatformSetting` avec une ligne par défaut (`id = 'default'`, `adminName = 'Technicien Relio'`, `adminWhatsApp = '237600000000'`).

## Vérifications

- `tsc --noEmit` (backend, `tsconfig.build.json`) : réussi
- `tsc --noEmit` (frontend) : réussi
- `npm run lint` (backend) : réussi (warnings pré-existants uniquement)
- `npm run lint` (frontend) : réussi (warnings pré-existants uniquement)
- `npx prisma generate` : réussi (Prisma Client 7.10.0 généré)
- `git push origin main` backend : `01e9334..31f45fa main -> main` — OK
- `git push origin main` frontend : `a8296e2..c0cad06 main -> main` — OK
- `git status -sb` à la racine `projet.git` : `?? Backend/` et `?? Frontend/` uniquement (RÈGLE 2 respectée)

## Bugs trouvés et corrigés

1. **Bug backend — `CreateDemandeDto` exige des champs V1 absurdes (categoryId enum plomberie/electricite, description obligatoire 10 caractères).**
   - Symptôme : `POST /api/demandes` refuse les demandes FRP/Système avec erreur de validation.
   - Cause : `BaseDemandeDto` impose `categoryId`, `description` (10 car. min.), `domainId`, etc.
   - Correctif : `CreateDemandeDto` réécrit en DTO minimal (`serviceCode`, `catalogItemId`, `address`, `phone`, `description` optionnel). `demandes.service.ts` simplifié (suppression de `resolveDevice`/`resolveDemandeGeo`).
   - Commit : `31f45fa` sur `Repairdom-backend`.

2. **Bug frontend — Fuite d'état (state leak) à l'étape 1 du wizard.**
   - Symptôme : changer de service (ex: Smartphone → Déblocage) laisse les anciennes données (panne, description) affichées à l'étape 3.
   - Cause : `update()` fusionne les nouveaux états sans réinitialiser les champs du service précédent.
   - Correctif : `update({ service: s, step: 2 })` ne conserve que `service` et `step` ; les autres champs sont réinitialisés via `emptyDraft`.
   - Commit : `c0cad06` sur `Repairdom-frontend`.

3. **Bug frontend/backend — Frais de déplacement (1000 FCFA) toujours affichés pour FRP/Système.**
   - Symptôme : encart "Frais de déplacement à payer : 1 000 FCFA" s'affiche alors que le paiement doit être 100% à la fin.
   - Cause : `depositAmount` persiste dans le draft et est affiché à l'étape 3.
   - Correctif : suppression du champ `depositAmount` du `Draft`, suppression de l'encart d'affichage, suppression du calcul côté service.
   - Commit : `c0cad06` (frontend) et `31f45fa` (backend).

Les 24 échecs de tests vitest backend sont pré-existants (absence de base de données locale PostgreSQL, interdite par RÈGLE 3). Aucun échec lié aux modifications apportées.

## Points d'attention

- Déploiement en attente : Railway (backend) puis Vercel (frontend). À confirmer factuellement après déploiement.
- Node.js v24.22.2 + npm 11.19.1 installés sur la machine (prérequis RÈGLE 4).
- Les fichiers `.env` n'ont pas été créés depuis `.env.example` dans les deux dépôts applicatifs.
- `User.passwordHash` est maintenant nullable : les comptes fantômes créés pour les clients anonymes ne peuvent pas se connecter par mot de passe (comportement voulu).
- Le suivi public par token (`/suivi/[token]`) ne nécessite aucune authentification ; le token UUID est non devinable.

## Questions bloquantes

Aucune.
