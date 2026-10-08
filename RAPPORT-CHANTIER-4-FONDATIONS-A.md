# RAPPORT — CHANTIER 4-FONDATIONS-A

> Nouveau barème de commission technicien + seuil minimum d'intervention.
> Déployé sur Railway puis Vercel. La notification aux techniciens **n'a pas été
> déclenchée** : c'est l'objet du chantier 4-FONDATIONS-B.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Backend | `Repairdom-backend` — commit **`09cc75f`** `feat(financial): new technician fee schedule (500 XAF + 4%) and 5 000 XAF minimum quote` — poussé `00d8090..09cc75f` sur `main` |
| Frontend | `Repairdom-frontend` — commit **`7dc1818`** `feat(fee): show technician commission on quotes and enforce the 5 000 XAF minimum` — poussé `e920495..7dc1818` sur `main` |
| Déploiement | **Railway vert** : `GET https://repairdom-backend-production-4922.up.railway.app/api/health` → `200`, `status=ok`, `database=up`, uptime stable et croissant (218 s au dernier contrôle, sans redémarrage) |
| | **Vercel vert** : `https://www.relioo.space/` → `200` ; `https://relioo.space/` → `308` vers `www` (redirection apex, préexistante) ; `/technicien/demandes` → `200` |
| Notification | **NON déclenchée** — aucun appel à `POST /api/admin/notify-technicians/fee-change`, aucun scheduler, aucun appel au démarrage |

## Synthèse

La commission Relio prélevée sur chaque mission confirmée passe de **2 % du brut
technicien** (réparation + 2 000 de transport) à **500 FCFA + 4 % du montant du
devis** (réparation seule). Le transport reste un pass-through intégralement
reversé au technicien et n'est plus commissionné. Un seuil minimum de **5 000
FCFA** est appliqué à la création de tout devis, y compris aux devis automatiques
du catalogue. L'endpoint `POST /api/admin/notify-technicians/fee-change` (ADMIN)
est prêt mais volontairement jamais appelé automatiquement. Le technicien voit
désormais sa commission et son net dans le récapitulatif de son devis, avant et
après envoi.

Table canonique implémentée et verrouillée par tests :

| Devis | Client paie | Commission | Technicien reçoit |
|---|---|---|---|
| 5 000 | 7 000 | 700 | 6 300 |
| 10 000 | 12 000 | 900 | 11 100 |
| 15 000 | 17 000 | 1 100 | 15 900 |
| 25 000 | 27 000 | 1 500 | 25 500 |
| 100 000 | 102 000 | 4 500 | 97 500 |

## Fichiers

### Backend (`Repairdom-backend`) — 3 créés, 11 modifiés

| Type | Chemin |
|---|---|
| Créé | `src/financial/fee-calculator.ts` (source unique du barème) |
| Créé | `src/financial/fee-calculator.spec.ts` |
| Créé | `src/admin/admin-fee-change-notification.spec.ts` |
| Modifié | `src/financial/financial-fees.ts` |
| Modifié | `src/financial/financial.service.ts` |
| Modifié | `src/financial/mission-holds.spec.ts` |
| Modifié | `src/collaboration/collaboration.service.ts` |
| Modifié | `src/collaboration/dto/create-quote.dto.ts` |
| Modifié | `src/collaboration/quotes.spec.ts` |
| Modifié | `src/admin/admin.controller.ts` |
| Modifié | `src/admin/admin.service.ts` |
| Modifié | `src/auth/email-templates.ts` |
| Modifié | `src/auth/email.service.ts` |
| Modifié | `src/realtime/realtime.types.ts` |

### Frontend (`Repairdom-frontend`) — 2 créés, 13 modifiés

| Type | Chemin |
|---|---|
| Créé | `src/lib/technician-quote.ts` (aperçu avant envoi) |
| Créé | `src/lib/technician-quote.test.ts` |
| Modifié | `src/app/technicien/demandes/[id]/page.tsx` |
| Modifié | `src/app/technicien/revenus/page.tsx` |
| Modifié | `src/app/admin/finances/page.tsx` |
| Modifié | `src/components/technician/diagnostic/use-free-diagnostic.ts` |
| Modifié | `src/components/technician/diagnostic/free-diagnostic-desktop-view.tsx` |
| Modifié | `src/components/technician/diagnostic/free-diagnostic-mobile-view.tsx` |
| Modifié | `src/components/technician/revenus/revenue-overview.tsx` |
| Modifié | `src/components/admin/finances/relio-funds-section.tsx` |
| Modifié | `src/lib/api/technician-service.ts` |
| Modifié | `src/lib/api/finance-service.ts` |
| Modifié | `src/lib/diagnostic-libre.test.ts` |
| Modifié | `package.json` (script `test:unit` : 34 → 35 fichiers) |
| Modifié | `tsconfig.json` (ajout de l'exclude du nouveau test) |

### Dépôt racine (`projet`)
`RAPPORT-CHANTIER-4-FONDATIONS-A.md` (ce fichier, remplace l'audit-only du commit `396adc7`).

## Migrations

**Aucune migration Prisma.** Le chantier réutilise le type de notification
`ADMIN_MESSAGE` déjà présent dans l'enum `NotificationType` (déjà écrit par
`AdminService.sendTechnicianMessage`). Le dossier `prisma/migrations/` est
inchangé.

## Vérifications

| Commande | Backend | Frontend |
|---|---|---|
| `npx tsc --noEmit` | **OK** (`-p tsconfig.build.json`) | **OK** |
| `npm run lint` (oxlint) | **OK**, 0 warning | **OK**, 0 warning |
| Tests | **984 / 984** des tests aplicatifs | **380 / 380** dans le périmètre vérifiable |
| Non-régression | **prouvée** | **prouvée** |

### Détail des tests backend

- `HEAD` propre : `947 tests, 3 failed`
- Après chantier : `987 tests, 3 failed` (**+40 tests**)
- Comparaison des **noms** d'échecs via `git stash -u` puis `diff` : **identiques**
  (seuls les timings et le décompte changent)
- Les 3 échecs sont **pré-existants** et documentés : `src/demandes/demande-multimedia.spec.ts`
  (`Cannot read properties of undefined (reading 'trim')` sur `dto.description`,
  `demandes.service.ts:96`). Cf. `docs/UX-BACKLOG.md:38-42`.

### Détail des tests frontend — réserve méthodologique

`npm run test:unit` **n'a pas pu être lancé tel quel** : Node local = **20.20.2**,
le runner `.ts` natif exige **Node 22+** (limitation **pré-existante**, signalée
par `AGENTS.md` RÈGLE 4 et contredite par le `README.md` qui annonce « Node >= 20 »).

Vérification effectuée selon la procédure RÈGLE 4 : compilation `tsc` vers un
**miroir hors dépôt** (`/tmp/opencode/fe-mirror`, aucun fichier écrit dans le
dépôt), puis `node --test` sur les 35 fichiers.

- `HEAD` propre : **9 échecs**
- Après chantier : **9 échecs, noms identiques** (les indices de numérotation
  glissent de +19, le nouveau test en ajoutant 19)
- Ces 9 échecs sont des **artefacts du miroir**, pas des régressions : le miroir
  ne contient que `src/`, pas `public/sw.js`, `docs/`, `logo/`, `animatio json/`,
  `package.json` ni `.git`. Test concernés : `lottie`, `design-system`,
  `push/push-client`, `verification-confirm`, `demande-draft-sync` (alias `@/`
  non résoluble par `node`).
- **Sur Node 22 (Vercel / poste de dev), `npm run test:unit` lancera réellement
  les 35 fichiers.** Ce point n'a **pas** été vérifié en conditions réelles.

**Non exécuté volontairement** (interdit par la mission) : `next build`,
`npm run dev`, Docker, Postgres local, tests e2e exigeant une base.

## Bugs trouvés

| # | Constat | Impact | Correctif |
|---|---|---|---|
| 1 | **Régression de test détectée par mes propres tests** : `src/lib/diagnostic-libre.test.ts` asserait `Number.isInteger` dans `use-free-diagnostic.ts`. Le refactor vers la source unique `@/lib/technician-quote` avait supprimé cette chaîne du hook. | `npm run test:unit` serait **rouge** sur Node 22 → déploiement Vercel bloqué. | Test mis à jour : il vérifie désormais la délégation (`quoteAmountError(amount)`) **et** que `Number.isInteger` + `MIN_QUOTE_AMOUNT_XAF = 5_000` vivent dans `technician-quote.ts`. Corrigé **avant** le push. |
| 2 | **Violation de la règle FCFA** : `technicien/demandes/[id]/page.tsx:65-66` formatait le montant du devis avec `quote.amount.toLocaleString('fr-FR')`, hors `formatFCFA`. | Affichage non conforme et divergent du reste de l'app. | `formatAmount()` passe désormais par `formatFCFA` ; la devise n'est affichée que si elle diffère de XAF. Verrouillé par `technician-quote.test.ts`. |
| 3 | **11 libellés « 2 % » hardcodés** dans 6 fichiers (technicien revenus, admin finances, fonds Relio), devenus faux après le changement de barème. | L'admin et le technicien auraient affiché une commission erronée. | Remplacés par `TECHNICIAN_FEE_LABEL` (« Commission Relio (500 FCFA + 4 %) ») ou par les valeurs fournies par le backend. |

## Points d'attention

1. **Réconciliation historique** : les missions réglées **avant** ce chantier
   portent une commission à 2 % du brut. Le ledger n'est **jamais** réécrit.
   `isTechnicianFeeReconciled()` accepte donc **les deux barèmes** ; sinon
   l'écran `/admin/finances` aurait affiché « écart » sur toutes les missions
   historiques. `computeLegacyRelioCommission()` est conservé **uniquement** pour
   cette lecture, et documenté comme tel.

2. **Seuils catalogue** : un `pricing.referencePrice < 5 000` **bloque** le devis
   automatique avec un message renvoyant vers le diagnostic libre. **Aucun
   `Pricing` n'a été modifié** (décision admin). Anomalie constatée dans le seed
   smartphone : `reboot-force-reset` a `minPrice: 3 000` pour un
   `referencePrice: 5 000` (`backend/src/admin/catalog.service.ts:1561`). **À
   signaler** — non corrigé.

3. **Les `Pricing` de production n'ont pas pu être inventoriés** : le chantier
   interdit toute infrastructure (pas de Postgres local, pas d'e2e). Le contrôle
   reste visuel sur `/admin/catalog/baremes`, puis
   `/admin/catalog/[domainId]/[problemId]/[diagnosticId]` → « Tarification ».

4. **Commission à 5 000 FCFA = 14 % du devis** (700 / 5 000), contre 2,9 % avant.
   C'est la conséquence arithmétique directe de « 500 fixes + 4 % », validée par
   le commanditaire, et non un défaut d'implémentation. Le technicien le verra
   dans l'aperçu avant envoi.

5. **Doublon de règle assumé** : `src/lib/technician-quote.ts` (frontend) duplique
   la formule du backend. Raison : au moment de la saisie, **aucun devis n'existe
   en base**, donc aucun appel API n'est possible à chaque frappe. Après envoi,
   l'affichage utilise les montants **renvoyés par le backend**
   (`commission`, `netTechnician`) et ne recalcule rien. Le risque de divergence
   est couvert par les montants canoniques écrits en dur dans
   `technician-quote.test.ts` : tout changement de barème doit être répercuté
   dans le même commit.

6. **`Reconciliation` / `diagnostic-libre`** : `reconcileRepairDomFees()` reste du
   code **mort** (aucun appelant, constaté à l'audit) — hors périmètre, non
   corrigé. Les routes publiques `/api/health/db` et `/api/health/migrations`
   restent en place malgré le marqueur « À SUPPRIMER » — hors périmètre.

7. **Identité du build Railway non prouvable sans authentification** : le seul
   endpoint public `/api/health` ne distingue pas l'ancien du nouveau build. Le
   statut « Railway vert » repose sur `200 / status=ok / database=up` et un
   uptime croissant sans redémarrage après le push. La preuve fonctionnelle du
   nouveau barème devra venir du scénario de test en production.

8. **Divergence documentaire pré-existante** : `frontend/README.md` annonce
   « Node.js >= 20 » alors que `npm run test:unit` exige Node 22+. Non corrigée
   (hors périmètre).

## Questions bloquantes

**Aucune.**

---

## Scénario de test production (à exécuter par le commanditaire)

**Test 1 — Calcul de commission**
1. CLIENT : créer une demande, technicien accepte, devis de **25 000** accepté.
2. Client valide la fin de mission.
3. `/technicien/revenus` : commission **1 500 FCFA**, gain net **25 500 FCFA**.

**Test 2 — Seuil minimum**
1. TECHNICIAN : saisir un devis de **3 000** → bouton « Proposer » **désactivé**,
   aide « Minimum 5 000 FCFA ».
2. Envoi direct de l'API avec `amount: 3000` → `400`
   « Le montant minimum d'une intervention est de 5 000 FCFA. »

**Test 3 — Devis automatique catalogue sous le seuil**
1. TECHNICIAN KYC validé : sélectionner une intervention de catalogue dont le
   `referencePrice` < 5 000 → refus renvoyant vers le diagnostic libre.

**Test 4 — Notification (chantier 4-FONDATIONS-B, PAS ENCORE FAIT)**
1. ADMIN : `POST /api/admin/notify-technicians/fee-change`
2. Attendu : chaque technicien actif reçoit email + notification in-app.
3. Réponse attendue : `{ sent, failed, total, failedChannels }`.

**Test 5 — Non-régression missions en cours**
1. Mission avec un devis de 3 000 accepté **avant** le déploiement : elle doit
   pouvoir se terminer et se confirmer normalement (le seuil ne bloque que la
   création).