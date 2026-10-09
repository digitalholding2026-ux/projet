# RAPPORT — TRANSPARENCE SASPAY : recharge client + payout technicien (Option A)

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-08 |
| Dépôts | `Repairdom-backend` → **`2e03366`**, `Repairdom-frontend` → **`95f7b8e`** |
| Commits | **1 par dépôt**, poussés dans l'ordre backend → frontend (RÈGLE 2) |
| Déploiement | **Railway** ✅ redéployé et vérifié. **Vercel** ✅ redéployé et vérifié. |
| Node | 22.20.0 installé dans `/tmp/opencode/node-v22.20.0-linux-x64` (la machine reste en 20.20.2 par défaut) |
| Validation | Utilisateur — les 3 arbitrages confirmés avant push |

### Preuve de déploiement

**Railway** — `GET /api/health` a renvoyé `uptime: 63079` puis `uptime: 4.02`
sur deux appels distants : l'instance a bien redémarré après le push. Réponse
courante : `{"status":"ok","database":"up"}`.

**Vercel** — inspection du JavaScript réellement servi, pas seulement du code
HTTP :

| Page | Chunk vérifié | Marqueur attendu | Constat |
|---|---|---|---|
| `/client/solde/recharger` | `app/client/solde/recharger/page-e55a3cf….js` | `Confirmer et payer`, `Frais Mobile Money (4,5 %)`, `Math.ceil(.045*e)` | ✅ présents |
| `/technicien/revenus` | chunk partagé `9889-d19f….js` | `Frais de transfert Mobile Money pris en charge par Relio.` | ✅ présente |
| tous écrans technicien | idem | `Frais SasPay`, `Débité de votre solde`, `Reçu bénéficiaire`, `0.035` | ✅ **absents** |

Les Build IDs de chunks diffèrent de ceux d'avant le push. L'absence des
anciens libellés de frais dans le bundle servi confirme que le nouveau code est
en production, et pas l'ancien.

## Synthèse

Deux règles opposées, appliquées séparément :

- **Côté client**, les 4,5 % de l'encaissement sont désormais annoncés AVANT
  validation, sur l'écran de recharge comme sur l'écran de résultat.
- **Côté technicien**, l'Option A est appliquée : Relio envoie le brut majoré
  pour que le technicien encaisse exactement son net, et aucun montant de
  frais ne lui est exposé — uniquement une mention.

**Un défaut de la spécification a été corrigé en cours de route** : le point
A.2 affirmait que le ledger pouvait continuer à débiter le `chargedAmount`
SasPay. C'est faux dès qu'on envoie un brut majoré — cela porterait les frais
sur le technicien. Détail en « Bugs trouvés » §1.

## Fichiers

### Backend (`Repairdom-backend`)

Créés :

- `src/financial/saspay-fees.ts` — barème SasPay (source unique serveur)
- `src/financial/saspay-fees.spec.ts` — 12 tests

Modifiés :

- `src/saspay/saspay-payout.service.ts` — envoie `computeSaspayPayoutCharged(request.amount)`
- `src/financial/financial.service.ts` — `settleWithdrawalSuccess` débite le **net** ; metadata enrichie
- `src/saspay/withdrawal.controller.ts` — commentaires (plafond sur net, absence de breakdown)
- `src/financial/payout-settlement.spec.ts` — 1 test inversé + 1 test Option A ajouté
- `src/financial/finance-payment-chantier.spec.ts` — test ADD_ON aligné
- `src/saspay/saspay-payout.service.spec.ts` — 1 assertion mise à jour + 4 tests Option A

### Frontend (`Repairdom-frontend`)

Créés :

- `src/lib/saspay-fees.ts` — aperçu de recharge (miroir client)
- `src/lib/saspay-relio-absorbs-fees.ts` — mention technicien (module pur, sans montant)
- `src/lib/saspay-fees.test.ts` — 34 tests

Modifiés :

- `src/app/client/solde/recharger/page.tsx` — encart 3 lignes + bouton « Confirmer et payer »
- `src/app/client/solde/recharge/result/page.tsx` — encart « Solde crédité / Total payé »
- `src/app/technicien/demandes/[id]/page.tsx` — mention sous « Vous recevrez » (devis envoyé) et sous l'aperçu avant envoi
- `src/app/technicien/revenus/page.tsx` — mention sous « Gain net »
- `src/components/finance/withdrawal-panel.tsx` — mention, récap « Vous recevrez », breakdown vidé de ses montants
- `package.json` — `test:unit` : ajout du nouveau fichier
- `tsconfig.json` — nouveau test ajouté aux exclusions

## Migrations

Aucune. Aucun changement de schéma.

## Vérifications

Node 22.20.0 (installé hors du dépôt, RÈGLE 4).

| Contrôle | Baseline `HEAD` | Après chantier | Verdict |
|---|---|---|---|
| Backend `tsc --noEmit` | ✅ | ✅ | pas de régression |
| Backend `oxlint` | ✅ | ✅ | pas de régression |
| Backend `vitest run` | 977/980 | **994/997** | **+17, 0 régression** |
| Frontend `tsc --noEmit` | ✅ | ✅ | pas de régression |
| Frontend `oxlint` + eslint | ✅ | ✅ | pas de régression |
| Frontend `test:unit` | 386/390 | **420/424** | **+34, 0 régression** |

**Preuve de non-régression** : les deux suites ont été rejouées sur `HEAD` propre
via `git stash -u`. Les 3 échecs backend (`demande-multimedia.spec.ts`) et les
4 échecs frontend (`demande-draft-sync`, `design-system` logo, 2 ×
`verification-confirm`) sont **les mêmes fichiers, les mêmes noms de tests**
qu'avant le chantier. Ils sont déjà consignés au backlog
(`backend/docs/UX-BACKLOG.md`) comme pré-existants. Aucun test n'est présenté
comme vert s'il ne l'est pas.

**Partie E (hooks React)** : `eslint` ne remonte **aucun** diagnostic
`rules-of-hooks` sur les 5 fichiers modifiés. Les hooks y restent tous avant
le `return` conditionnel — vérifié par lecture de la position de chaque hook
et de chaque sortie anticipée.

## Bugs trouvés

### 1. La spec disait que le ledger pouvait débiter le `chargedAmount` — c'est faux

**Impact — grave.** Le hold est créé sur `request.amount` (le **net**), mais
`settleWithdrawalSuccess` débite le `chargedAmount` **si SasPay le fournit**.
Avant Option A, `chargedAmount == request.amount` : la différence était
invisible. Dès qu'on envoie le brut majoré, elle apparaît :

- Retrait net 10 000 → hold 10 000, brut envoyé 10 363
- SasPay débite 10 363 au Prestataire, le technicien reçoit 10 000 ✅
- Ledger débite **10 363** du solde Relio ❌ → solde du technicien à **−363**

Le technicien verrait son solde Relio devenir négatif alors qu'il a reçu son
net exact. Les frais seraient à sa charge : l'inverse exact de l'Option A.

**Correctif** — le ledger débite **toujours `request.amount`**. Les montants
constatés (`chargedAmount`, `fee`, `netAmount`, `feeChargeMode`) restent
tracés en metadata avec `saspayFeesAbsorbedByRelio: true`, donc le coût réel
de Relio demeure disponible pour la comptabilité. Deux tests existants
encodaient l'ancien comportement et ont été mis à jour en conséquence
(`payout-settlement.spec.ts`, `finance-payment-chantier.spec.ts`) ; un test
non-régression vérifie explicitement qu'un retrait de 10 000 laisse le solde à
10 000 et non à 9 637.

### 2. `WithdrawalFeeBreakdown` exposait des montants devenus faux

**Impact.** Le composant affichait « Frais SasPay », « Débité de votre solde »
(`chargedAmount`) et « Reçu bénéficiaire ». Sous Option A, le `chargedAmount`
n'a plus aucun rapport avec le débit du solde (qui reste le net) : ces trois
lignes seraient à la fois interdites par la règle « aucun montant de frais
côté technicien » et **factuellement fausses**.

**Correctif** — le composant est réduit à la mention. Le libellé de la
confirmation « Montant » devient « Vous recevrez ». `feeChargeModeNote`
reste en place (testé, documenté, disponible pour l'admin) mais n'a plus
d'appelant UI.

## Écarts explicites par rapport à la demande

| Point demandé | Réalisé | Pourquoi |
|---|---|---|
| A.3 — endpoint `GET /technician/withdrawals/:reference/breakdown` | **NON livré** | La spec le marque « optionnel », mais sa charge utile (`fraisSaspay`, `brutEnvoye`) **contredit frontalement** la Partie C (« ne pas exposer le montant de frais SasPay au technicien »). Les deux exigences sont incompatibles. J'ai suivi C, qui porte l'engagement client, et documenté le refus dans l'en-tête du contrôleur. Le coût réel reste tracé côté serveur. |
| A.2 — « vérifier que SasPay accepte les brut > 10M » | **NON vérifiable** | Exige une transaction réelle. Décision validée : livrer tel quel. En cas de refus, le payout part en `FAILED`, le hold est libéré, aucun débit — pas de perte de fonds. |
| A.4 — « plafond sur net : un retrait de 10M net accepté » | Vérifié par test | Le plafond `MAX_WITHDRAWAL_AMOUNT` porte sur `request.amount`, donc sur le net. Confirmé aussi sur le DTO (`@Max`) et sur `assertWithdrawalAmount`. Un test vérifie que 10 000 000 net est envoyé à 10 362 695. |
| B.1 — aperçu temps réel | Livré | `previewTopupFees` recalculé à chaque frappe, sans aller-retour HTTP. |
| C — 4 écrans technicien | Livré | Mention identique sur les 4, extraite dans un module unique pour qu'aucune formulation ne diverge. |

## Points d'attention

1. **Le plafond technique de SasPay sur le BRUT reste inconnu.** Le test 4 de
   production (retrait de 10 000 000 net) est le vrai probe : si SasPay
   refuse, on verra un `FAILED` propre. Le correctif serait de borner le net
   à `MAX / 0,965`, ce qui contredirait la décision validée n°3.
2. **Node 22 installé hors du dépôt**, dans `/tmp/opencode/`. La machine est
   toujours en 20.20.2 par défaut : `npm run test:unit` échoue sans cette
   variable PATH. À installer durablement.
3. **`npm ci` est cassé sur le backend** (`Missing: typescript@5.9.3` — dette
   connue du backlog). J'ai utilisé `npm install --no-package-lock
   --legacy-peer-deps` ; **le `package-lock.json` est resté intact** (vérifié
   par `git status`). Le fichier de lock reste donc désynchronisé en amont.
4. **Aucun e-mail touché**, conformément à la Partie D : aucun template ne porte
   de montant de mission. À reconsidérer si un template technicien avec
   montant est créé plus tard.
5. **L'aperçu client est un calcul frontend.** Si SasPay change de taux, le
   backend et le frontend divergent jusqu'à la prochaine mise à jour. Les deux
   jeux de constantes sont testés séparément, et un test frontend vérifie que
   le fichier backend porte les mêmes valeurs — mais aucune alerte runtime
   n'existe. Le montant réellement crédité reste la vérité serveur
   (`netAmountMinor`), et l'écran de résultat affiche ce qui a été constaté.
6. **3 échecs backend et 4 échecs frontend sont pré-existants** et hors périmètre
   de ce chantier. Ils ne sont pas introduits ici mais restent à traiter.

## Questions bloquantes

Aucune.

## Suite — tests en production

Cinq scénarios à faire tourner par l'utilisateur (déjà précisés dans la
spécification) :

1. **Recharge client** — saisir 2 000 → l'aperçu doit afficher 2 000 / 90 /
   2 090, SasPay débiter 2 090, le solde être crédité de 2 000.
2. **Mission client** — devis 15 000 → payer 17 000, **aucun frais visible**.
3. **Payout technicien** — solde 15 900, retrait de 15 900 → il doit recevoir
   **exactement 15 900**, et la mention « pris en charge par Relio » doit être
   visible sur l'écran de retrait.
4. **Plafond** — retrait net de 10 000 000 → brut envoyé 10 362 695.
5. **Non-régression** — les missions antérieures affichent les mêmes montants.

### Plan de correction si le test 4 échoue

Si SasPay refuse un payout au-delà de 10 M **brut** (hypothèse non vérifiable
sans transaction réelle, cf. Points d'attention §1), le scénario observable est
un `FAILED` propre : le hold est libéré, aucun débit, le technicien peut
reposer une demande. Aucun fonds n'est perdu et le ledger n'est pas faussé.

Le correctif tient en une ligne — borner le NET au plafond technique :

```ts
// backend/src/financial/saspay-fees.ts
export const MAX_PAYOUT_GROSS_XAF = 10_000_000;
export function computeMaxWithdrawableNetXAF(): number {
  return Math.floor(MAX_PAYOUT_GROSS_XAF * (1 - SASPAY_PAYOUT_RATE));
}
```

à appliquer dans `assertWithdrawalAmount` (`financial.service.ts`) **et** dans
le `@Max` du DTO `create-withdrawal-request.dto.ts`, avec la même constante
miroir côté frontend (`finance-limits.ts`) — sinon l'UI proposerait un montant
que le serveur refuse.

À noter : ce correctif contredit la décision validée n°3 (plafond à 10 M sur le
net). Il ne doit être appliqué que sur confirmation factuelle du refus SasPay,
pas par précaution.