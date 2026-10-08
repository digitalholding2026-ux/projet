# RAPPORT — CHANTIER 4-FONDATIONS-A (audit préalable)

> Audit préalable obligatoire, réalisé **avant toute modification de code**.
> Aucun fichier de `backend/` ni `frontend/` n'a été touché.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Dépôts concernés | `Repairdom-backend` (HEAD `00d8090`), `Repairdom-frontend` (HEAD `e920495`), dépôt racine `projet` |
| Commits | **aucun** (phase d'audit, avant code) |
| Push / déploiement | **aucun** — le chantier n'est pas implémenté |
| État des arbres de travail | `backend/` et `frontend/` propres, alignés sur `origin/main` |

## Synthèse

Audit en lecture seule du barème de commission technicien et des points d'insertion du seuil
minimum de 5 000 FCFA. La commission actuelle est un pourcentage du **brut technicien**
(réparation + 2 000 XAF de déplacement) ; elle est prélevée en **une seule fois**, au `CONFIRMED`
de la mission, via `settleTechnicianAtConfirmation()`. Deux blocages arbitrages ont été identifiés
et soumis au commanditaire avant écriture du code : la **base de calcul** de la nouvelle
commission, et le **sort des devis automatiques catalogue** sous 5 000 FCFA.

## Fichiers

| Type | Chemin |
|---|---|
| Créé (ce rapport) | `RAPPORT-CHANTIER-4-FONDATIONS-A.md` (racine) |
| Modifié | **aucun** |
| Supprimé | **aucun** |

## Migrations

**Aucune.** Le chantier réutilise le type de notification `ADMIN_MESSAGE` existant
(`prisma/schema.prisma`, enum `NotificationType`) : pas de changement de schéma nécessaire.

---

# 🔍 Audit préalable — Commission technicien & seuil minimum

## 1. Fichiers qui définissent / calculent la commission

### 1.1 Source unique des montants — `backend/src/financial/financial-fees.ts` (98 lignes)

| Ligne | Élément | Valeur actuelle |
|---|---|---|
| 1-18 | En-tête documentaire : **« commission Relio = 2 % du BRUT technicien »** + exemple `20 000 → brut 22 000 → commission 440 → net 21 560` | à réécrire |
| 19 | `STANDARD_TRANSPORT_FEE` | `2_000` |
| 21-23 | `RELIO_COMMISSION_RATE_NUMERATOR` / `_DENOMINATOR` | `2` / `100` |
| 25 | `FINANCIAL_CURRENCY` | `'XAF'` |
| 33-35 | `CLIENT_PLATFORM_FEE` / `TECHNICIAN_PLATFORM_FEE` / `TOTAL_PLATFORM_FEES` | `100` / `150` / `250` — **legacy, lecture seule** |
| **37-48** | **`computeRelioCommission(grossAmount)`** — arrondi entier demi-supérieur `(gross×2 + 50) ÷ 100` | **← le calcul à remplacer** |
| 50-56 | `computeGrossAmount(repairAmount)` = `repair + 2000` | définit « brut » |
| 58-61 | `computeTechnicianNet(grossAmount)` = `gross − commission` | à adapter |

### 1.2 Point exact où la commission est prélevée

**`backend/src/financial/financial.service.ts` → `settleTechnicianAtConfirmation()`, lignes 445-516.**

```
L459  grossAmount  = repairAmount + travelAmount      (27 000 pour un devis de 25 000)
L460  commission   = computeRelioCommission(grossAmount)   ← LE prélèvement
L464-480  écriture CREDIT TECHNICIAN_REPAIR_REVENUE
L482-498  écriture CREDIT TECHNICIAN_TRAVEL_REVENUE
L500-515  écriture DEBIT  TECHNICIAN_FEE  amount = commission
           metadata : { fee, gross, rateNumerator: 2, rateDenominator: 100, currency }
```

Appelé **uniquement** depuis `settleMissionAtConfirmation()` (L404-408), lui-même appelé dans
`demandes.service.ts` sur la transition `→ CONFIRMED`. Jamais à la création, au dispatch, à
l'acceptation technicien, au devis, à l'acceptation du devis ni en négociation
(commentaire L436-439).

### 1.3 Tous les autres usages de la commission (backend)

| Fichier | Ligne(s) | Usage |
|---|---|---|
| `financial.service.ts` | 20, 26-37 | imports des constantes |
| | 435 | commentaire « commission Relio 2 % du brut » |
| | 503-514 | écriture `TECHNICIAN_FEE` |
| | 775, 841, 860, 955, 1012, 1150, 1176, 1282, 1560 | agrégations `Σ TECHNICIAN_FEE` (fonds Relio, synthèse admin, totaux technicien) |
| | **992-1046** | `reconcileRepairDomFees()` — **code mort, aucun appelant**. L1023 calcule l'attendu avec `computeRelioCommission(clientDebit)` |
| | 1041-1042, 1064-1065, 1229-1230 | `expectedPerMission.commissionRateNumerator/Denominator` exposé à l'admin |
| | 1067-1069 | `expectedPerMission.clientFee/technicianFee/total` legacy |
| `collaboration.service.ts` | 18, 253-254, 435, 557, 897 | `STANDARD_TRANSPORT_FEE` (transport, **pas** la commission) |
| `common/format-fcfa.ts` | 10 | `formatFCFA` serveur (pousse / notifications) |

### 1.4 Tests existants qui figent l'ancienne valeur

| Fichier | Ligne | Valeur figée |
|---|---|---|
| `financial/mission-holds.spec.ts` | **377, 386** | `it('CONFIRMED → … fee (440 = 2 % de 22000)')` et `expect(TECHNICIAN_FEE).toBe(440)` → **devra passer à 1 380** (500 + 4 %×22 000) |
| `financial/relio-funds.spec.ts` | 15, 30-31, 51, 65, 77, 89 | fixtures de sommes (mock `_sum`), non liées à la formule → non impactés |

---

## 2. Création des devis — où poser le seuil de 5 000

### 2.1 Devis **manuel** — `backend/src/collaboration/collaboration.service.ts` → `createQuote()`, **lignes 378-470**

```
L379  rôle TECHNICIAN obligatoire
L382  requireAccess  (CLIENT propriétaire ou technicien assigné, sinon 404 masqué)
L383  assertOpen / L384 assertDiagnosticAllowed  → statut ACCEPTED obligatoire
L388-392  flux catalogue : interdit sans négociation
L394-395  description non vide
L401      $transaction : refus si un ACCEPTED existe (L406-408)
L410-413  les PENDING existants passent REJECTED
L435      travelAmount = STANDARD_TRANSPORT_FEE   (le `travelAmount` du DTO est IGNORÉ)
```

→ **C'est ici qu'il faut insérer le contrôle `amount < 5000 → BadRequestException`.**

DTO : `backend/src/collaboration/dto/create-quote.dto.ts` L7-10 → `@IsInt() @Min(1) @Max(1_000_000_000)`.
Aujourd'hui le minimum est **1 XAF**.

Route : `POST /api/demandes/:demandeId/quotes` (`collaboration.controller.ts`,
`@Roles('CLIENT','TECHNICIAN')`, `TECHNICIAN` requis par le service).

### 2.2 Devis **automatique catalogue** — ⚠️ contourne `createQuote()`

`collaboration.service.ts` → `selectCatalogDiagnostic()`, **lignes 855-931** :

```
L862-875  création du Diagnostic (mode CATALOG)
L877      const amount = pricing.referencePrice ?? 0;     ← montant issu du barème
L878-900  tx.quote.create({ amount, source: 'CATALOG', … })
```

→ **Un devis catalogue sous 5 000 resterait créable** même après le garde-fou de `createQuote()`.
Aucun `Pricing` n'est modifié (interdit par le cahier des charges), donc c'est une **règle
applicative** à arbitrer.

### 2.3 Acceptation — non touchée (conforme à la décision du commanditaire)

`respondToQuote()` L~540-580 ne ré-audite pas le montant. Un devis de 3 000 XAF accepté avant
déploiement reste donc acceptable et se confirme normalement (scénario Test 4). ✔

---

## 3. Toutes les constantes de montants

| Fichier | Ligne | Constante | Valeur |
|---|---|---|---|
| `financial-fees.ts` | 19 | `STANDARD_TRANSPORT_FEE` | `2_000` |
| `financial-fees.ts` | 22-23 | `RELIO_COMMISSION_RATE_*` | `2 / 100` |
| `financial-fees.ts` | 33-35 | `CLIENT/TECHNICIAN_PLATFORM_FEE`, `TOTAL_PLATFORM_FEES` | `100 / 150 / 250` (legacy) |
| `financial-fees.ts` | 64-65 | `DEFAULT_TEST_CREDIT_AMOUNT`, `MAX_TEST_CREDIT_AMOUNT` | `50_000` / `1_000_000` |
| `financial-fees.ts` | 72 | `RELIO_WITHDRAWAL_MAX_AMOUNT` | `1_000_000_000` |
| `financial-fees.ts` | 92-93 | `MIN_TOPUP_AMOUNT` / `MAX_TOPUP_AMOUNT` | `100` / `10_000_000` |
| `financial-fees.ts` | 97-98 | `MIN_WITHDRAWAL_AMOUNT` / `MAX_WITHDRAWAL_AMOUNT` | `100` / `10_000_000` |
| `financial-fees.ts` | 79-86 | préfixes de références + `IDEMPOTENCY_KEY_MAX_LENGTH` | — |
| `dto/create-quote.dto.ts` | 3, 8-9 | `QUOTE_MAX_AMOUNT`, `@Min(1)` | `1_000_000_000` / **1** ← à passer à 5 000 |
| `dto/create-topup-intent.dto.ts` | 15-16 | `@Min` / `@Max` | 100 / 10 M |
| `dto/create-withdrawal-request.dto.ts` | 17-18 | `@Min` / `@Max` | 100 / 10 M |
| `dto/relio-withdraw.dto.ts` | 13 | `@Max` | 1 Md |
| `financial.service.ts` | 2572, 2581 | revalidations runtime topup / retrait | — |
| `rewards/rewards.config.ts` | 101 | `MIN_MISSION_AMOUNT_XAF` | `1_500` — **règle fidélité #4A, NON touchée** |

---

## 4. Formatage / affichage des montants côté frontend

Helper principal : **`frontend/src/lib/format-fcfa.ts`** (12 lignes, `formatFCFA`, entier XAF →
`"5 000 FCFA"` avec U+00A0).
Helper secondaire : **`frontend/src/lib/format.ts` L39-46 `formatCurrency`** + **L48-54
`formatCurrencySigned`** — utilisés par les pages finances (le ledger expose `currency`,
`formatFCFA` est sans paramètre).

### 4.1 Libellés « 2 % » à corriger — **9 occurrences frontend**

| Fichier | Ligne | Libellé |
|---|---|---|
| `app/technicien/demandes/[id]/page.tsx` | **882** | `Commission Relio (2 % du brut) prélevée sur ce tarif…` ← **la zone C.1** |
| `app/technicien/demandes/[id]/page.tsx` | **957** | `…déclenche le calcul du règlement (brut, commission Relio 2 %, net)` |
| `app/technicien/revenus/page.tsx` | **39** | `TECHNICIAN_FEE: 'Commission Relio (2 %)'` ← **zone C.3** |
| `app/technicien/revenus/page.tsx` | **225** | `Commission Relio (2 %)` (carte « Gains par mission ») |
| `components/technician/revenus/revenue-overview.tsx` | **38** | `label="Commission Relio (2 %)"` |
| `components/admin/finances/relio-funds-section.tsx` | 24, 86, 112 | `(2 % du brut …)`, `Commissions acquises (2 % du brut validé)` |
| `app/admin/finances/page.tsx` | **38, 129, 310, 435, 474** | `'Commission Relio (2 %)'`, `commissions 2 %`, `Commissions techniciens (2 %)`, … |

`app/admin/finances/page.tsx` L334-335 affiche déjà `expectedPerMission.commissionRateNumerator`
fourni par le backend → **se met à jour automatiquement** si le nouveau barème est exposé dans
`expectedPerMission`.

### 4.2 Deux zones de saisie de devis (deux UIs distinctes, toutes deux → `createDemandeQuote`)

| UI | Fichier / lignes | Validation actuelle |
|---|---|---|
| **A — devis simple** | `app/technicien/demandes/[id]/page.tsx` : state L108-110, handler `handleCreateQuote` **L286-312**, formulaire **L906-932** (bouton `disabled={!amountValue.trim() \|\| !quoteDescription.trim()}` L930) | `!Number.isInteger(amount) \|\| amount <= 0` |
| **B — diagnostic libre** | `components/technician/diagnostic/use-free-diagnostic.ts` **L57-65** (`amountError`, `canSubmit`), vues `free-diagnostic-desktop-view.tsx` / `-mobile-view.tsx` | `!isInteger \|\| < 1` |

### 4.3 Anomalie FCFA détectée au passage

`app/technicien/demandes/[id]/page.tsx` **L65-66** :

```ts
function formatAmount(quote) { return `${quote.amount.toLocaleString('fr-FR')} ${quote.currency}`; }
```

→ formatage **sans `formatFCFA`**, contrairement à la règle FCFA du dépôt. À corriger dans le
périmètre C.1.

---

## 5. Barèmes catalogue `Pricing` — anomalies < 5 000

Seed smartphone (`backend/src/admin/catalog.service.ts` L1500-1730), **9 interventions** :

| Ligne | Intervention | `minPrice` | `referencePrice` |
|---|---|---|---|
| 1516 | (démarrage) | 10 000 | 15 000 |
| 1535 | | 20 000 | 35 000 |
| **1561** | **`reboot-force-reset` — « Reset forcé / reboot »** | **3 000** ⚠️ | **5 000** |
| 1587 | | 12 000 | 22 000 |
| 1613 | | 8 000 | 15 000 |
| 1639 | | 8 000 | 15 000 |
| 1665 | | 10 000 | 18 000 |
| 1691 | | 5 000 | 10 000 |
| 1717 | | 8 000 | 12 000 |

**Une seule anomalie** : `minPrice = 3 000` sur « Reset forcé / reboot ». Le `referencePrice`
utilisé pour le devis automatique vaut 5 000 → le devis auto passe le seuil. **Non modifié**
(décision admin).

⚠️ **Les `Pricing` réellement présents en base de production n'ont pas pu être inventoriés** :
le chantier interdit toute infrastructure (pas de Postgres local, pas d'e2e). Les prix saisis
par l'admin depuis le seed ne sont pas accessibles en lecture seule depuis le dépôt. Le contrôle
reste visuel : `/admin/catalog/baremes`, ou
`/admin/catalog/[domainId]/[problemId]/[diagnosticId]` → section « Tarification ».

---

## 6. Points d'insertion de l'endpoint admin (B.3)

| Élément | Fichier / ligne |
|---|---|
| Contrôleur à étendre | `admin/admin.controller.ts` — `@Controller('admin')`, `@UseGuards(JwtAuthGuard, RolesGuard)`, `@Roles('ADMIN')` (L23-25) |
| Service | `admin/admin.service.ts` — ctor L42-55, `realtime?` / `push?` / `email?` / `config?` **optionnels** (convention pour les tests unitaires) |
| Base URL des e-mails | `admin.service.ts` **L60-63 `frontendUrl()`** (repli `https://relioo.space`) |
| Modèle 4 canaux à copier | `admin.service.ts` **L329-380** (KYC) : `realtime.publishToUser(..., 'notification.created')` L329, push `sendToUser` L354, `email.sendKycVerifiedEmail` L374, chacun isolé |
| Notification existante à réutiliser | `Notification` type **`ADMIN_MESSAGE`** — déjà écrit par `sendTechnicianMessage()` **L869-877** (aucune migration nécessaire) |
| Gabarits e-mail | `auth/email-templates.ts` (529 l.) — `emailLayout` L74, `paragraph` L126, `infoBox` L122, `escapeHtml` L48, `escapeAttr` L57, `buildRewardTierReachedEmail` L381 (modèle le plus proche : CTA + aucun montant formaté) |
| Service d'envoi | `auth/email.service.ts` — ajouter `sendFeeChangeEmail()` à côté de `sendRewardTierReachedEmail()` L230-258 (ne throw pas si `!isConfigured`) |
| ⚠️ Règle FCFA | `auth/email-templates.ts` + `notifications/notification-metadata.ts` : **jamais de montant pré-formaté dans `title`/`message`**. Le barème « 500 + 4 % » sera écrit **en toutes lettres dans le corps de l'e-mail** (texte rédigé, comme `buildRewardTierReachedEmail`), **pas** dans la notification in-app. |

---

## Vérifications

| Commande | Résultat |
|---|---|
| `git status` (racine, backend, frontend) | arbres propres, alignés sur `origin/main` |
| `tsc --noEmit` | **non exécuté** — aucun code modifié |
| `oxlint` | **non exécuté** — aucun code modifié |
| `vitest run` | **non exécuté** — aucun code modifié |
| Non-régression | **non applicable** — phase d'audit |

> Conformément à la mission, aucune commande exigeant une infrastructure n'a été lancée
> (`next build`, `npm run dev`, Docker, Postgres local, e2e).

## Bugs trouvés

| # | Constat | Emplacement | Impact | Suite |
|---|---|---|---|---|
| 1 | `formatAmount()` formate sans `formatFCFA` (viole la règle FCFA du dépôt) | `frontend/src/app/technicien/demandes/[id]/page.tsx:65-66` | incohérence d'affichage des montants sur la page mission technicien | sera corrigé dans C.1 |
| 2 | `reconcileRepairDomFees()` n'a **aucun appelant** (code mort) | `backend/src/financial/financial.service.ts:992` | aucun ; dette de maintenabilité | hors périmètre |
| 3 | Routes `/health/db` et `/health/migrations` publiques, marquées « À SUPPRIMER », **toujours en place** | `backend/src/health/health.controller.ts:53-57` | exposition de la liste des migrations appliquées en public | hors périmètre (déjà au backlog) |
| 4 | `test/app.e2e-spec.ts` = squelette Nest CLI par défaut référençant `AppController`/`AppService` **inexistants** | `backend/test/app.e2e-spec.ts` | fichier mort ; ramassé par `vitest run` (`**/*.spec.ts`) | hors périmètre |
| 5 | Seuil 5 000 > `MIN_MISSION_AMOUNT_XAF` (1 500) de la fidélité client | `backend/src/rewards/rewards.config.ts:101` | aucun — les deux règles coexistent sans contradiction | aucune |
| 6 | `mission-holds.spec.ts` fige l'ancienne commission (440) | `backend/src/financial/mission-holds.spec.ts:377,386` | le test devra être mis à jour (attendu 1 380) — **changement volontaire, pas une régression** | sera mis à jour |

## Points d'attention

1. **Le seuil de 5 000 rend la commission très lourde sur les petites missions** : à 5 000 FCFA
   la commission vaut 700 XAF, soit **14 %** du devis (au lieu de 2,9 % aujourd'hui). C'est la
   conséquence mathématique directe de « 500 fixes + 4 % », pas une erreur d'implémentation.
2. **Le « 23 500 FCFA » du scénario de production est arithmétiquement incompatible avec la règle
   de déplacement** : la plateforme crédite au technicien `TECHNICIAN_TRAVEL_REVENUE = 2 000 XAF`.
   Aucun choix de base de commission ne donne 23 500 pour un devis de 25 000.
3. **`Pricing` de production non inventoriables** sans base de données (voir §5).
4. **Règle FCFA** : le barème ne doit apparaître dans aucun champ `title`/`message` de
   notification ni dans aucun SMS/push ; formatage par `formatFCFA` uniquement à l'affichage.
5. **`README.md` (frontend) annonce « Node.js >= 20 »** alors que `npm run test:unit` exige
   Node 22+ (runner `.ts` natif). Pré-existant, hors périmètre.
6. **Le push backend est déjà poussé** (`00d8090`) et le frontend aussi (`e920495`) : le chantier
   4-A n'est pas un redéveloppement d'un chantier livré.

## Questions bloquantes

### 🔴 POINT 1 — Sur quoi la commission est-elle calculée ?

Aujourd'hui : `computeRelioCommission(grossAmount)` où **brut = réparation + 2 000 de
déplacement** (`financial-fees.ts` L7, L55).

**Devis de 25 000 FCFA :**

| | Commission | Net technicien | vs net actuel |
|---|---|---|---|
| **actuel** (2 % du brut 27 000) | 540 | 26 460 | — |
| **Option A** — base = **brut** 27 000 | 500 + 4 %×27 000 = **1 580** | **25 420** | −1 040 |
| **Option B** — base = **devis** 25 000 | 500 + 4 %×25 000 = **1 500** | **25 500** | −960 |

- **Option A** conserve la base de la règle financière existante (et reste compatible avec
  `reconcileRepairDomFees`, qui compare au `clientDebit`).
- **Option B** est la seule qui rende les **tests unitaires B.5 et le scénario de production
  cohérents entre eux** (`calculateTechnicianFee(5000)=700` ⇒ base = l'argument ;
  scénario « 25 000 ⇒ 1 500 » ⇒ base = le devis).

Nombres exacts à afficher dans `/technicien/revenus` selon l'option :

- A → réparation 25 000 / déplacement 2 000 / commission −1 580 / **net 25 420**
- B → réparation 25 000 / déplacement 2 000 / commission −1 500 / **net 25 500**

### 🔴 POINT 2 — Devis automatique catalogue sous 5 000 : bloqué ou pas ?

Le devis auto est créé dans `selectCatalogDiagnostic()` **L877-900**, chemin qui **ne passe pas**
par `createQuote()`. Les `Pricing` ne doivent pas être modifiés.

- **Option A** — bloquer aussi le devis auto : si `pricing.referencePrice < 5 000`, refus
  explicite renvoyant le technicien vers le diagnostic libre (devis manuel, seuil également
  appliqué). **Aucun `Pricing` modifié** — c'est une règle applicative. Rend l'énoncé
  « tout devis technicien ≥ 5 000 » réellement vrai.
- **Option B** — ne bloquer que les devis manuels : les devis auto sous 5 000 passent, avec une
  commission de 500 + 4 % prélevée dessus. Le seed actuel produit un devis auto à 5 000 pile,
  donc **rien ne change en l'état** ; mais toute intervention dont un admin baisserait le
  `referencePrice` sous 5 000 deviendrait un devis à commission > 10 %, ce qui va à l'encontre
  de l'objectif « marge insuffisante sur les petites missions ».

---

**Aucun code écrit, aucun fichier de code modifié.** En attente de l'arbitrage sur les points 1
et 2.