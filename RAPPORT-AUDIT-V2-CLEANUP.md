# RAPPORT-AUDIT-V2-CLEANUP — Audit intégral Relio v2 (Marketplace v1 → Opérateur unique)

**Dépôt :** `C:\Users\Docker\Desktop\Relio` (dossiers `Backend/` et `Frontend/`)  
**Nature :** Audit en lecture seule — aucun fichier modifié, aucune migration créée, aucun commit de code  
**Date :** 07/11/2026  
**Contexte :** Pivot v1 (marketplace avec techniciens/dispatch) → v2 (Relio opère lui-même les réparations, catalogue v2 avec prix publics/pros)

---

## 1. Cartographie Globale & État des Lieux

### 1.1 Arborescence Backend (`Backend/src/`)

```
Backend/src/
├── app.module.ts, main.ts
├── config/env.validation.ts
├── common/filters/http-exception.filter.ts
├── prisma/prisma.module.ts
├── auth/ (actif) — auth.controller.ts, auth.service.ts, auth.types.ts, guards, dto, email.service.ts
├── demandes/ (actif, cœur v2) — demandes.controller.ts, demandes.service.ts, demande-draft.controller.ts, demande-draft.service.ts, demande-media.service.ts, demandes-lifecycle.ts, dto/
├── collaboration/ (actif) — chat, diagnostics, devis
├── catalog/ (actif, v1+v2) — catalog.controller.ts (GET /services), catalog.service.ts (CatalogV2Service), catalog-admin.controller.ts (ADMIN CRUD v1)
├── admin/ (actif, super-pouvoirs) — admin.controller.ts, admin.service.ts, catalog.service.ts
├── reviews/ (actif) — reviews.controller.ts, reputation.controller.ts (GET /technicians/:id/reputation → RÉFÉRENCE TECHNICIEN MORT)
├── disputes/ (actif)
├── financial/ (actif, ledger) — financial.service.ts, finance.controller.ts, financial-admin.controller.ts, finance-admin.controller.ts
├── saspay/ (actif, paiement) — saspay-webhook.controller.ts (POST /webhooks/saspay, PUBLIQUE)
├── rewards/ (actif, fidélité)
├── notifications/ (actif)
├── tracking/ (actif, public)
├── push/ (actif, VAPID)
├── realtime/ (actif, SSE)
├── health/ (actif)
├── backoffice-agent/ (actif, ADMIN)
├── technician/ (MORT v1) — supabase-storage.service.ts, nationalities.ts, avatar-file.ts (AUCUN module NestJS)
└── generated/ (Prisma Client généré)
```

### 1.2 Arborescence Frontend (`Frontend/src/`)

```
Frontend/src/
├── app/ (Next.js App Router)
│   ├── layout.tsx, page.tsx (landing)
│   ├── demande/page.tsx (wizard public /demande)
│   ├── suivi/page.tsx
│   ├── client/ (layout RoleGuard CLIENT) — demande/page.tsx, demandes/, profil/, solde/, recompenses/, parrainage/, notifications/, technicien/[id]/page.tsx
│   └── admin/ (layout RoleGuard ADMIN) — missions/, litiges/, kyc/, catalog/, utilisateurs/, finances/, communication/, assistant/
├── components/
│   ├── client/demande-wizard.tsx (1425 lignes)
│   ├── client/catalog-service-picker.tsx
│   ├── admin/* (squelettes KYC, finances, missions)
│   ├── mission/* (dispatch-sonar-widget.tsx, mission-stepper.tsx → RÉFÉRENCES DISPATCH MORTES)
│   ├── technician/public/* (squelettes technicien v1)
│   ├── ui/ (design system)
│   └── auth/role-guard.tsx
└── lib/api/ — auth-service.ts, request-service.ts, catalog-service.ts, admin-service.ts, finance-service.ts, technician-service.ts, referrals-service.ts, ...
```

### 1.3 Base de données vs Schéma Prisma — Pourquoi l'erreur `Service.proAvailable` est arrivée

**Migration manquante / non appliquée en prod, pas de dérive de schéma.**

- `Backend/prisma/migrations/20261106020000_add_service_pro_available/migration.sql` existe dans le dépôt avec :
  ```sql
  ALTER TABLE "Service" ADD COLUMN IF NOT EXISTS "proAvailable" BOOLEAN NOT NULL DEFAULT true;
  ```
- `Backend/prisma/schema.prisma` contient bien `proAvailable Boolean @default(true)` sur le modèle `Service`.
- `Backend/package.json` : `build` = `prisma generate && nest build` ; `db:migrate:deploy` = `prisma migrate deploy` ; `db:seed:catalog` = `node prisma/seed-catalog.ts`.
- `Backend/railway.json` : `preDeployCommand` = `npm run db:migrate:deploy && npm run db:seed:catalog` ; `startCommand` = `npm run start:prod`.

**Cause racine :** La migration `20261106020000_add_service_pro_available` existe dans le code mais n'a pas été appliquée sur la base Railway en production (ou le déploiement a utilisé un ancien build/cache où le client Prisma attend la colonne mais la table `Service` ne l'a pas). Le client Prisma généré attend `proAvailable`, la requête `prisma.service.findMany({ where: { proAvailable: true } })` dans `CatalogV2Service.listServices` (`Backend/src/catalog/catalog.service.ts:25`) échoue avec `The column \`Service.proAvailable\` does not exist in the current database.`

### 1.4 Modules NestJS actifs vs morts

| Module | Statut | Commentaire |
|--------|--------|-------------|
| AppModule, AuthModule, DemandesModule, CollaborationModule, CatalogModule, AdminModule, ReviewsModule, DisputesModule, FinancialModule, SasPayModule, RewardsModule, NotificationsModule, TrackingModule, PushModule, RealtimeModule, HealthModule, BackofficeAgentModule | ACTIF | Utilisés par le code courant |
| TechnicianModule, DispatchModule | **MORTS** | Supprimés par migration `20261031040000_remove_technician_and_dispatch` ; dossier `Backend/src/technician/` ne contient qu'`supabase-storage.service.ts` (utilisé par médias/diagnostics) + `nationalities.ts` + `avatar-file.ts`, aucun module NestJS |

---

## 2. Nettoyage : Code Mort et Scories du V1 (MARKETPLACE)

### 2.1 Backend — contrôleurs, services, DTO, routes, modèles Prisma à supprimer

#### Fichiers backend à supprimer (v1 mort)

| Chemin | Raison |
|--------|--------|
| `Backend/src/technician/supabase-storage.service.ts` + `.spec.ts` | Stockage Supabase pour l'ancien domaine technicien ; remplace par Supabase direct ou stockage générique si encore utilisé |
| `Backend/src/technician/nationalities.ts` | Nomenclature ISO 3166-1 KYC technicien — plus de techniciens |
| `Backend/src/technician/avatar-file.ts` | Validation avatar technicien — plus de techniciens |
| `Backend/src/reviews/reputation.controller.ts` | `GET /technicians/:id/reputation` et `GET /clients/:id/reputation` — référence `technicianId` mort |
| `Backend/src/auth/auth.types.ts` (ligne 1) | `UserRole = 'CLIENT' | 'TECHNICIAN' | 'ADMIN'` — `TECHNICIAN` absent du schéma Prisma (enums retirés par `20261031040000`) |
| `Backend/src/auth/roles.guard.ts` + `roles.decorator.ts` | Réfèrent encore `'TECHNICIAN'` comme rôle valide — cohérence à casser |

#### Références vestigiales v1 dans le code actif (à nettoyer, pas supprimer)

- `Backend/src/catalog/catalog.service.ts` : `CatalogV2Service` utilise `prisma.service.findMany` avec `proAvailable` — **actif et correct** après migration appliquée.
- `Backend/src/catalog/catalog-admin.controller.ts` + `catalog-admin.service.ts` : CRUD sur `ServiceDomain`, `DeviceBrand`, `DeviceModel`, `Problem`, `CatalogDiagnostic`, `CatalogIntervention`, `Pricing` — catalogue **v1 historique conservé** (cohérence des demandes existantes). À ne PAS supprimer, mais à marquer "v1 legacy".
- `Backend/src/quotes.spec.ts`, `diagnostic-libre.spec.ts` : tests référençant `technicianId` sur `Quote`/`Diagnostic` — `technicianId` colonne existe encore en DB mais la table `TechnicianProfile` n'existe plus → risque d'incohérence ORM.

### 2.2 Frontend — composants, hooks, pages, types, services API à supprimer

#### Pages/routes frontend v1 mortes

| Chemin | Raison |
|--------|--------|
| `Frontend/src/app/client/technicien/[id]/page.tsx` | Profil technicien côté client — plus de techniciens |
| `Frontend/src/components/technician/public/public-technician-hero.tsx` + `public-technician-skeleton.tsx` | Vitrine technicien v1 |
| `Frontend/src/components/technician/public/*` | Tout le dossier `technician/public/` |
| `Frontend/src/components/mission/dispatch-sonar-widget.tsx` | Widget de dispatch v1 |
| `Frontend/src/components/mission/mission-stepper.tsx` | Stepper mission avec références dispatch |

#### Composants admin v1 à supprimer (backoffice à refondre)

| Chemin | Raison |
|--------|--------|
| `Frontend/src/app/admin/kyc/page.tsx` + `kyc/[technicianId]/page.tsx` | Gestion KYC technicien — plus de techniciens |
| `Frontend/src/components/admin/kyc/kyc-detail-skeleton.tsx` + `kyc-list-skeleton.tsx` | Squelettes KYC mort |
| `Frontend/src/app/admin/communication/page.tsx` | Message direct admin → technicien — destinataire mort |
| `Frontend/src/components/admin/finances/relio-funds-section.tsx` | Fonds Relio — à garder mais renommer (wallet opérateur) |

#### Services API frontend v1 morts

| Chemin | Raison |
|--------|--------|
| `Frontend/src/lib/api/technician-service.ts` | Appels vers routes technicien supprimées |
| `Frontend/src/lib/api/referrals-service.ts` | Parrainage v1 — remplacé par `clientType` B2B |
| `Frontend/src/lib/api/referrals-rules.ts` | Règles parrainage v1 |

#### Estimation volume code mort

| Domaine | Fichiers | Lignes estimées |
|---------|----------|------------------|
| Backend v1 mort | ~8 fichiers | ~600-800 lignes |
| Frontend v1 mort (technicien, dispatch, KYC, com, parrainage) | ~12 fichiers | ~1 200-1 800 lignes |
| Références vestigiales à nettoyer (non supprimables) | ~6 fichiers | ~400-600 lignes |
| **Total estimé** | **~26 fichiers** | **~2 200-3 200 lignes** |

---

## 3. Chasse aux Bugs (Frontend & Backend)

### 3.1 Wizard Client (`Frontend/src/components/client/demande-wizard.tsx`)

**Bug identifié :** Le bouton "Continuer" reste inactif si la catégorie d'appareil v1 n'est pas remplie, même si le modèle v2 est sélectionné.

- **Lignes ~430-445** : `canContinue` — à l'étape 0 (`step === 0`), la condition est `selectedItemId !== ''` uniquement. Si l'utilisateur sélectionne un service (via `CatalogServicePicker`) mais pas d'item, le bouton reste bloqué.
- **États obsolètes (useState) :**
  - `coords`, `geoLoading`, `geoError` (lignes ~110-115) — GPS V1 opt-in, non requis par le modèle v2, ajoute de la complexité inutile.
  - `voiceKey` (ligne ~117) — remonte l'enregistreur après validation, état de contournement.
  - `draftToken`, `draftSyncedRef` (lignes ~123-128) — brouillon anonyme, logique lourde pour un wizard qui devrait être linéaire.

**Impact :** L'utilisateur ne peut pas passer à l'étape suivante s'il sélectionne un service sans item, bloquant le parcours de demande.

### 3.2 Frontend — appels API vers routes supprimées / incohérences

| Fichier | Ligne | Problème |
|---------|-------|----------|
| `Frontend/src/lib/api/catalog-service.ts` | `listCatalogDomains`, `listCatalogBrands`, etc. | Appelle `/catalog/domains` (catalogue **v1 historique**), alors que le parcours client v2 utilise `/services` (`listServices`). Double catalogue incohérent. |
| `Frontend/src/lib/api/technician-service.ts` | Toutes | Routes technicien supprimées côté backend → appels en 404. |
| `Frontend/src/lib/api/referrals-service.ts` | Toutes | Parrainage v1 retiré du schéma Prisma. |
| `Frontend/src/app/client/technicien/[id]/page.tsx` | Toutes | Affiche un profil technicien qui n'existe plus en base. |

### 3.3 Backend — incohérences contrôleurs / sécurité

| Fichier | Ligne | Problème |
|---------|-------|----------|
| `Backend/src/auth/auth.types.ts` | 1 | `UserRole` inclut `'TECHNICIAN'`, absent du schéma Prisma (enums retirés). Risque de cast côté client/frontend. |
| `Backend/src/admin/admin.controller.ts` | Toutes | Toutes les routes `/admin/*` protégées par `@Roles('ADMIN')` + `JwtAuthGuard` + `RolesGuard` — **correct**, mais aucun contrôle sur `GET /admin/disputes` (liste complète, pas de pagination forcée). |
| `Backend/src/catalog/catalog.controller.ts` | Toutes | `GET /services` avec `OptionalJwtAuthGuard` — public, correct pour le catalogue v2. |
| `Backend/src/saspay/saspay-webhook.controller.ts` | POST `/webhooks/saspay` | Route **publique** (pas de JWT) — attendable pour un webhook, mais vérification HMAC critique à confirmer. |

### 3.4 Déploiement — robustesse pipeline

| Fichier | Ligne | Constat |
|---------|-------|---------|
| `Backend/railway.json` | `preDeployCommand` | `npm run db:migrate:deploy && npm run db:seed:catalog` — présent et correct. |
| `Backend/package.json` | `build` | `prisma generate && nest build` — client Prisma régénéré à chaque build. Correct. |
| `Backend/package.json` | `db:seed:catalog` | `node prisma/seed-catalog.ts` — **idempotent** (upsert par `Service.code`, `CatalogCategory(serviceId, code)`, `CatalogItem(categoryId, code)` selon `seed-catalog-data.ts`). Correct. |
| `Backend/prisma.config.ts` | datasource.url | Fallback localhost si `DATABASE_URL` absent — uniquement pour `prisma generate` au build. Runtime exige une vraie `DATABASE_URL`. Correct. |

**Risque résiduel :** Si Railway exécute un build sans `DATABASE_URL` ou si le pre-deploy échoue silencieusement, le conteneur démarre avec un client Prisma désynchronisé (cas observé : erreur `proAvailable`).

---

## 4. Refonte du Backoffice Administrateur

### 4.1 État actuel — pages `/admin/*`

| Page | Chemin | Statut | Commentaire |
|------|--------|--------|-------------|
| Tableau de bord | `/admin` | Actif | Vue générale, à conserver |
| Missions | `/admin/missions` | Actif | Recherche par référence, à conserver |
| Litiges | `/admin/litiges`, `[id]` | Actif | À conserver |
| KYC | `/admin/kyc`, `[technicianId]` | **MORT v1** | Plus de techniciens → à supprimer |
| Catalogue | `/admin/catalog`, `[domainId]`, ... | Actif (v1 legacy) | CRUD domaines/marques/modèles — historique v1, à garder en lecture ou migrer |
| Familles | `/admin/catalog/familles` | Actif | Parcours "Autre appareil" v2 |
| Prix courants | `/admin/catalog/baremes` | Actif | À garder |
| Villes & zones | `/admin/catalog/villes` | Actif | À garder |
| Utilisateurs | `/admin/utilisateurs` | Actif | Gestion clients — à adapter (supprimer références technicien) |
| Finances | `/admin/finances` | Actif | Ledger + fonds Relio — à garder |
| Communication | `/admin/communication` | **MORT v1** | Message admin → technicien, destinataire inexistant |
| Assistant | `/admin/assistant` | Actif | Agent IA backoffice, à conserver |

### 4.2 Nouveau besoin métier (opérateur unique)

L'admin (Relio opérateur) doit pouvoir :
1. **Voir les demandes entrantes** et changer leur statut (SUBMITTED → ACCEPTED → SCHEDULED → IN_PROGRESS → COMPLETED).
2. **Gérer le catalogue v2** : Services > Catégories > Items, avec prix public et prix pro (`proAvailable`).
3. **Créer et gérer les comptes B2B** : génération de code unique, attribution `clientType = B2B`, prix pro.
4. **Voir les factures et l'état des fonds en escrow** (wallet) : `FinancialTransaction`, `FundsHold`, `WithdrawalRequest`.

### 4.3 Proposition d'architecture `/admin/`

```
/admin/
├── layout.tsx                 (RoleGuard ADMIN, navigation 5 sections)
├── page.tsx                   (Dashboard opérateur — demandes en attente, KPIs)
├── demandes/
│   ├── page.tsx               (Liste toutes les demandes, filtres statut/ville)
│   └── [id]/page.tsx          (Détail demande + changement de statut)
├── catalogue/
│   ├── page.tsx               (Liste services v2 : SYSTEME, SMARTPHONE, FRP)
│   ├── [serviceId]/page.tsx   (Détail service : catégories, items, prix)
│   └── items/[id]/page.tsx    (Édition item : prix public/pro, active, proAvailable)
├── clients/
│   ├── page.tsx               (Liste clients + comptes B2B)
│   └── [id]/page.tsx          (Détail compte, activation/désactivation, historique)
└── finances/
    ├── page.tsx               (Vue ledger, fonds en escrow, retraits)
    └── transactions/[id]/page.tsx (Détail transaction financière)
```

**Composants UI nécessaires :**
- `AdminDemandTable` — tableau demandes avec statuts et actions de changement
- `AdminCatalogTree` — arborescence Service > Catégorie > Item avec édition inline des prix
- `AdminClientCard` — fiche client avec badge B2B/CLIENT et solde wallet
- `AdminFinanceSummary` — KPIs fonds en escrow, transactions récentes
- `RoleGuard` ADMIN — déjà présent, à conserver

---

## 5. Accès et Sécurité de l'Admin

### 5.1 Devenir admin

- **Pas de route d'inscription admin publique** dans le code audité.
- L'attribution du rôle `ADMIN` se fait soit par **seed de base de données** (à confirmer dans `seed-catalog.ts` ou un seed admin dédié), soit par **modification manuelle en base** (`UPDATE "User" SET role = 'ADMIN' WHERE id = ...`).
- Aucun script de création d'admin via API n'a été trouvé dans `Backend/src/auth/` ou `Backend/src/admin/`.

### 5.2 Route racine du backoffice

- `/admin` (protégé par `RoleGuard expectedRole="ADMIN"` dans `Frontend/src/app/admin/layout.tsx`).
- Toutes les routes `/admin/*` sont protégées côté backend par `JwtAuthGuard` + `RolesGuard` + `@Roles('ADMIN')` (`Backend/src/admin/admin.controller.ts`, `catalog-admin.controller.ts`, etc.).

### 5.3 Failles d'accès potentielles

| Fichier | Ligne | Failles |
|---------|-------|---------|
| `Frontend/src/components/auth/role-guard.tsx` | Toutes | Protection **côté client uniquement** (React). Si un utilisateur manipule le JWT local, il peut accéder aux pages admin tant que le backend ne vérifie pas le rôle. |
| `Backend/src/admin/admin.controller.ts` | Toutes | Toutes les routes protégées par `@Roles('ADMIN')` — **correct**. Aucune route admin sans guard trouvée. |
| `Backend/src/catalog/catalog-admin.controller.ts` | Toutes | CRUD admin catalogue — protégé par `JwtAuthGuard` + `RolesGuard` + `@Roles('ADMIN')`. Correct. |
| `Backend/src/saspay/saspay-webhook.controller.ts` | POST `/webhooks/saspay` | Route **publique** (pas de JWT) — normal pour un webhook, mais la vérification de signature HMAC est la seule protection. À auditer séparément. |

**Verdict :** Le backend protège correctement les routes admin. La faiblesse est côté frontend (protection React uniquement, contournable si le JWT est compromis, mais le backend rejette néanmoins).

---

## 6. Plan d'Action Priorisé

### Sprint Nettoyage (suppression immédiate, sans casser le cœur v2)

1. Supprimer `Frontend/src/app/client/technicien/[id]/page.tsx` + `technician/public/*` (composants technicien v1).
2. Supprimer `Frontend/src/components/mission/dispatch-sonar-widget.tsx` + références dispatch.
3. Supprimer `Frontend/src/lib/api/technician-service.ts` + `referrals-service.ts` + `referrals-rules.ts`.
4. Supprimer `Frontend/src/app/admin/kyc/*` + `components/admin/kyc/*` + `app/admin/communication/page.tsx`.
5. Supprimer `Backend/src/technician/*` (sauf `supabase-storage.service.ts` si encore utilisé par médias/diagnostics — à vérifier).
6. Nettoyer `Backend/src/auth/auth.types.ts` : retirer `'TECHNICIAN'` de `UserRole` (cohérent avec le schéma Prisma).
7. **Ne pas supprimer** : `catalog-admin.*` (catalogue v1 historique nécessaire aux demandes existantes), `reviews/reputation.controller.ts` (à réviser, réfère `technicianId`).

### Sprint Fixes (priorité haute)

1. **Appliquer la migration** `20261106020000_add_service_pro_available` sur Railway (`prisma migrate deploy`). → Résout l'erreur 500 `/api/services`.
2. **Corriger le wizard** `demande-wizard.tsx` : étape 0, `canContinue` doit accepter un `selectedServiceLabel` même sans `selectedItemId`, ou rendre l'item optionnel.
3. **Aligner les appels API frontend** : remplacer `/catalog/domains` par `/services` dans les composants client qui affichent le catalogue v2.
4. **Vérifier Railway** : `preDeployCommand` bien exécuté, `startCommand` = `npm run start:prod` (pas `nest start` en dev).

### Sprint Admin (reconstruction backoffice)

1. **Semaine 1** : Dashboard `/admin` + Liste demandes `/admin/demandes` + détail avec changement de statut.
2. **Semaine 2** : Catalogue v2 `/admin/catalogue` (services, catégories, items, prix public/pro, `proAvailable`).
3. **Semaine 3** : Comptes B2B `/admin/clients` (activation/désactivation, code unique).
4. **Semaine 4** : Finances `/admin/finances` (ledger, escrow, retraits).
5. **En parallèle** : Supprimer pages KYC, Communication, technicien.

---

## Résumé exécutif — 5 plus gros problèmes

1. **Désynchronisation DB/Prisma (`proAvailable`)** : migration existante mais non appliquée en production Railway → `GET /api/services` renvoie 500. Cause : pre-deploy non exécuté ou build cache obsolète.
2. **Code mort v1 massivement présent** : pages technicien, dispatch, KYC, parrainage, communication admin ≈ 2 200-3 200 lignes à supprimer sans casser le cœur v2.
3. **Wizard client bloquant** : `canContinue` à l'étape 0 exige `selectedItemId` même quand seul un service v2 est sélectionné → utilisateur bloqué sur "Votre appareil".
4. **Double catalogue incohérent** : frontend appelle à la fois `/catalog/domains` (v1 historique) et `/services` (v2) sans distinction claire, générant de la confusion côté client.
5. **Absence de processus d'attribution du rôle ADMIN** : aucun script seed ni route API pour créer un compte admin ; le rôle doit être assigné manuellement en base, ce qui est fragile en production.

---

**Fichier enregistré :** `RAPPORT-AUDIT-V2-CLEANUP.md` à la racine du dépôt `projet`. Aucun autre fichier n'a été modifié.
