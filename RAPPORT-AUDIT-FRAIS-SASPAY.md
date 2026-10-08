# RAPPORT — AUDIT : transparence des frais SasPay (Option A, Relio absorbe)

> Audit préalable **OBLIGATOIRE** de la mission « Transparence des frais :
> SasPay affiché + Option A ». Lecture seule : **aucun code écrit, aucun
> commit applicatif, aucun push applicatif**.
>
> **Verdict : 3 points bloquants** avant toute implémentation (voir
> « Questions bloquantes »). Le plus important remet en cause un chiffre de la
> spécification : **le client ne paie aujourd'hui AUCUN frais SasPay sur le
> montant d'une mission**.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Nature | **Audit lecture seule stricte** — aucun fichier `backend/` ni `frontend/` modifié, aucun commit applicatif, aucun push applicatif |
| Dépôts concernés | `Repairdom-backend` (`da71861`), `Repairdom-frontend` (`50002d9`) — **inchangés** |
| Dépôt racine | `projet` — ce rapport seul |
| Verdict | **NON IMPLÉMENTABLE TEL QUE SPÉCIFIÉ** sans arbitrage sur 3 points (frais client, plafond de retrait, garantie du taux 3.5 %) |

## Synthèse

Deux constats changent la nature du chantier :

1. **Côté technicien (Option A), la majoration est techniquement faisable** et
   vérifiée : `ceil(net / (1 − 0.035))` garantit bien que le technicien reçoit
   le net attendu. Mais elle **casse le plafond des retraits** (10 000 000 net
   → 10 362 695 brut, au-delà de `MAX_WITHDRAWAL_AMOUNT`), et le plancher de
   450 XAF transforme un retrait de 100 XAF en 550 XAF bruts.

2. **Côté client, les frais SasPay 4.5 % existent déjà — mais uniquement sur
   les RECHARGES, pas sur les missions.** Le client saisit 2 000, SasPay
   débite 2 090, Relio crédite 2 000 : les 90 sont jamais affichés. Le montant
   d'une mission, lui, ne subit AUCUN frais : le client paie 17 000 pour un
   devis de 15 000, pas 17 765.

   L'exemple de la spécification (« Frais de transaction 765 FCFA » sur un
   devis de 17 000) correspond donc à **une ligne de facturation qui n'existe
   pas**. L'afficher reviendrait à indiquer au client un montant qu'il ne paie
   pas — ce qui est l'inverse de l'Option A, dont le but est de supprimer des
   frais, pas d'en ajouter.

3. **La Partie E (mentions e-mail) est sans objet** : aucun template
   d'e-mail du dépôt ne porte de montant de mission.

---

## 1. Flow payout SasPay (Option A)

| Élément | Emplacement |
|---|---|
| Envoi de l'argent au technicien | `backend/src/saspay/saspay-payout.service.ts:131-141` — `initializePayout({ amountMinor: request.amount, … })` |
| Construction du body HTTP | `backend/src/saspay/saspay-api.client.ts:404-418` — `amount: toSasPayDecimal(input.amountMinor)` |
| Ce que SasPay renvoie | `saspay-payout.service.ts:282-285` et `:307-309` — **`feeMinor`, `chargedAmountMinor`, `netAmountMinor`, `feeChargeMode`** |
| Écriture ledger | `backend/src/financial/financial.service.ts:2342-2400` — `settleWithdrawalSuccess` |
| Montant débité au compte | `financial.service.ts:2385-2392` — `debitAmount = saspay.chargedAmount ?? request.amount` |
| Routage webhook payout vs pay-in | `backend/src/saspay/saspay-webhook.service.ts` — via `WithdrawalRequest.saspayTransactionId` |

**Réponses aux quatre questions de l'audit :**

- **Comment le montant est-il calculé ?** Le backend envoie `request.amount`,
  c'est-à-dire le NET voulu ; SasPay déduit ses frais et renvoie
  `netAmountMinor`. Le ledger débite le `chargedAmount` réellement constaté.
- **Où les frais SasPay sont-ils lus ?** **Nulle part : ils ne sont pas
  calculés par Relio.** Ils sont *retournés* par l'API SasPay. Vérifié :
  `grep "0.045|0.035|4.5|3.5|RATE"` sur `src/saspay/*.ts` → **aucun
  résultat**.
- **Existe-t-il une notion de « frais Relio absorbe » ?** **Non.** Aucune trace
  dans le dépôt. Le ledger enregistre déjà le brut réellement débité, ce qui est
  la première moitié du travail d'Option A.
- **Le payout envoie-t-il déjà le net ?** **Oui** — c'est exactement ce qui
  produit l'écart constaté en production (2 000 envoyés → 1 930 reçus).

### Vérification chiffrée de la formule de majoration

| Net voulu | Brut à envoyer | Net réellement reçu (après 3.5 %) | Garanti ≥ net ? |
|---|---|---|---|
| 15 900 | 16 477 | 15 900 | ✅ |
| 2 000 | 2 450 | 2 364 | ✅ |
| 100 | 550 (plancher 450 appliqué) | 530 | ✅ |
| 10 000 000 | **10 362 695** | 10 000 000 | ⚠️ **dépasse `MAX_WITHDRAWAL_AMOUNT`** |

---

## 2. Flow d'encaissement client — conflit avec la spécification

| Élément | Emplacement |
|---|---|
| Création d'une recharge | `backend/src/saspay/saspay-topup.service.ts` |
| Crédit effectif du solde | `backend/src/financial/financial.service.ts:1931-2010` — `confirmTopupFromSasPay` |
| Ce qui est crédité | `financial.service.ts:2000-2010` — le **`netAmountMinor` constaté**, jamais le `requested` |
| Débit du montant d'une mission | `financial.service.ts` — `recordClientMissionDebit`, débite `finalAmount` = réparation + 2 000 |

**Les frais SasPay 4.5 % existent DÉJÀ en production**, et ce depuis le
chantier SASPAY-01 : le montant crédité est le net réellement encaissé, donc
les frais sont de fait **supportés par le client** — et jamais affichés.

**Le montant d'une mission ne subit aucun frais de transaction.** Le débit au
CONFIRMED est le brut exact (`réparation + transport`).

---

## 3. Affichage frontend actuel

| Écran | Fichier / lignes | Ce qui est affiché aujourd'hui |
|---|---|---|
| Client — devis à accepter | `frontend/src/app/client/demandes/[id]/page.tsx:520-645` | lignes réparation + déplacement + `totalToDebit` (brut) |
| Client — confirmation de devise (modal) | `client/demandes/[id]/page.tsx:785-800` | libellé seul, aucun détail de montant |
| Client — écran de confirmation | `frontend/src/app/client/confirmation/page.tsx` | montant de la mission, pas de détail de frais |
| Technicien — devis envoyé | `frontend/src/app/technicien/demandes/[id]/page.tsx:870-945` | réparation, déplacement, total client, commission, « Vous recevrez » |
| Technicien — formulaire de devis (aperçu) | `technicien/demandes/[id]/page.tsx:959-1010` | devis, déplacement, commission, net prévisionnel |
| Technicien — revenus | `frontend/src/app/technicien/revenus/page.tsx:200-248` (`MissionEarningCard`) | réparation, déplacement, commission, gain net |

---

## 4. Constantes de taux SasPay

**Aucune constante de taux n'existe dans le dépôt.** Les seuls seuils liés aux
montants SasPay sont les bornes de recharge et de retrait, dans
`backend/src/financial/financial-fees.ts` :

| Constante | Ligne | Valeur |
|---|---|---|
| `MIN_TOPUP_AMOUNT` / `MAX_TOPUP_AMOUNT` | 92-93 | 100 / 10 000 000 |
| `MIN_WITHDRAWAL_AMOUNT` / `MAX_WITHDRAWAL_AMOUNT` | 97-98 | 100 / 10 000 000 |

Revalidation runtime : `financial.service.ts:2599-2601`.

---

## 5. Templates e-mail (Partie E)

**`buildQuoteAcceptedEmail` n'existe pas.** Les gabarits existants sont dans
`backend/src/auth/email-templates.ts` : vérification de compte, réinitialisation
de mot de passe, mission disponible, KYC vérifié, KYC refusé, palier de
fidélité, relance de vérification.

**Aucun de ces gabarits ne porte de montant de mission.** La Partie E est donc
**SANS OBJET** en l'état ; la préserver de mentions supposerait d'écrire un
nouveau template, ce qui sort du périmètre annoncé.

---

## Points d'attention

1. **Le plafond des retraits est le vrai risque technique d'Option A.** La
   majoration porte le montant demandé au-delà de `MAX_WITHDRAWAL_AMOUNT`. Les
   bornes sont-elles un garde-fou *utilisateur* (à valider sur le net) ou une
   *limite SasPay* (à respecter sur le brut) ? La réponse change le correctif.

2. **Le plancher de 450 XAF gonfle les petits retraits.** Un retrait de 100 XAF
   (autorisé aujourd'hui) deviendrait 550 XAF bruts, soit 530 XAF nets. Le
   technicien reçoit plus que demandé — cohérent avec l'Option A, mais à
   documenter.

3. **Le taux de 3.5 % n'est garanti par aucun code.** Il est déduit de deux
   mesures en production. Si SasPay applique un **minimum de 450 XAF** plutôt
   qu'un taux sur les petits montants, la formule `net / (1 − r)`
   **sous-paierait** le technicien — exactement ce que l'Option A promet
   d'interdire. À confirmer sur un montant moyen (≈ 5 000 XAF) avant de figer
   la formule.

4. **Afficher des frais client non appliqués est un risque juridique et de
   confiance.** Un client qui voit « Frais de transaction 765 FCFA » sur un
   écran de devis, puis est débité de 17 000, constatera l'écart. C'est
   l'inverse de la transparence recherchée.

5. **Aucun montant de mission dans les e-mails** : le seul filet de
   transparence existant est l'UI. Une divergence backend/UI sur le net ne
   serait rattrapée par aucun canal.

6. **Le test 3 du scénario de production (« missions terminées avant ce
   chantier s'affichent correctement — pas de recalcul ») est implicitement
   satisfait** : l'affichage lit des montants déjà figés dans le ledger, aucun
   nouveau calcul ne serait appliqué aux missions passées. À confirmer quand
   même, car la majoration ne s'applique qu'aux **retraits** (nouveaux), pas aux
   missions.

## Questions bloquantes

### 1. Frais client 4.5 % — que signifie « Option A » côté client ?

Il n'existe aujourd'hui **aucun** frais sur le montant d'une mission. Trois
lectures :

- **(a) Affichage informatif** : mentionner dans le récap que les recharges
  Mobile Money sont majorées de 4.5 % par le prestataire. Honnête, ne change
  rien à la facture, **ne correspond pas** aux 765 FCFA de l'exemple.
- **(b) Ne rien afficher côté client** : le récap reste devis + transport +
  total, et la transparence porte sur le **technicien** — ce que l'Option A
  protège réellement. **Recommandé.**
- **(c) Appliquer réellement 4.5 % aux missions** : changement produit,
  contredit « Ne PAS toucher aux autres flows SasPay » **et** l'Option A.

### 2. Plafond de retrait après majoration

- (i) Valider les bornes sur le **montant net demandé** et laisser le brut
  dépasser — **recommandé** : les bornes protègent l'utilisateur, pas SasPay.
- (ii) Plafonner le net à `MAX / 1.035`, donc refus d'un retrait de 10 000 000.

### 3. Le 3.5 % est-il un taux ou un minimum ?

Les mesures (2 000 → 1 930) sont cohérentes avec un taux. **À confirmer sur un
montant moyen** avant de figer `ceil(net / (1 − 0.035))`. Si SasPay applique
un minimum de 450 XAF, la formule doit être `max(net / (1 − r), net + 450)` —
ce que la spécification propose déjà, mais il faut savoir lequel des deux
l'API applique réellement.

---

**Aucun code écrit.** En attente de vos réponses sur les 3 points.