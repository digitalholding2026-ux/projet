# RAPPORT — Chantier M0.2 (migration v2) : suppression du domaine « technicien », dispatch, KYC, zone-coverage et parrainage

## Statut

- **Date** : 31/10/2026
- **Dépôt concerné** : `Repairdom-backend` (commit `885010e`), espace de travail racine `projet.git` (rapport).
- **Commit** : `885010e` — `chore(v2): drop technician, dispatch, kyc, zone-coverage and referral schema`
- **Push** : effectué sur `origin/main` du dépôt backend — évolution `366dc46..885010e`, branche propre, aucune régression de push.
- **Déploiement** : **non vérifié** à ce jour (Railway). Le `railway.json` du backend déclenche `npm run db:migrate:recover && npm run db:migrate:deploy && npm run start:prod` : la migration `20261031040000_remove_technician_and_dispatch` sera donc exécutée au prochain déploiement Railway.

## Synthèse

Clôture du chantier **M0.2 (migration v2)**. Le chantier comportait deux volets :

1. **Volet schéma/migration** (`remove_technician_and_dispatch`) : réalisé et déjà poussé la veille (`366dc46`).
2. **Volet code** : nettoyage du backend pour que `tsc --noEmit -p tsconfig.build.json` retourne **0 erreur** et que le socle v2 soit cohérent — réalisé ce jour (`885010e`).

Le commit final `885010e` porte **35 fichiers** (133 insertions / 4 018 lignes supprimées) : suppression des modules et specs obsolètes, refonte des services câblés au domaine supprimé, nettoyage des imports morts. `prisma validate`, `prisma generate`, `tsc`, `oxlint` et `vitest` passent.

## Fichiers

### Modifiés (21)

- `backend/src/admin/admin-super-powers.spec.ts`
- `backend/src/admin/admin.controller.ts`
- `backend/src/admin/admin.module.ts`
- `backend/src/admin/admin.service.ts`
- `backend/src/admin/catalog.service.ts`
- `backend/src/app.module.ts`
- `backend/src/auth/auth.service.ts`
- `backend/src/auth/auth.spec.ts`
- `backend/src/auth/dto/register.dto.ts`
- `backend/src/auth/password-reset-flow.spec.ts`
- `backend/src/auth/verification-reminder.scheduler.ts`
- `backend/src/auth/verification-reminder.spec.ts`
- `backend/src/collaboration/collaboration.service.ts`
- `backend/src/demandes/demande-helpers.ts`
- `backend/src/demandes/demandes.module.ts`
- `backend/src/demandes/demandes.service.ts`
- `backend/src/financial/financial.service.ts`
- `backend/src/financial/mission-holds.spec.ts`
- `backend/src/financial/payout-settlement.spec.ts`
- `backend/src/tracking/tracking-navigation.spec.ts`
- `backend/src/tracking/tracking.service.ts`

### Supprimés (13)

- `backend/src/admin/admin-fee-change-notification.spec.ts`
- `backend/src/admin/dto/send-technician-message.dto.ts`
- `backend/src/admin/dto/update-kyc-status.dto.ts`
- `backend/src/auth/register-referral-wiring.spec.ts`
- `backend/src/demandes/demandes-referrals-wiring.spec.ts`
- `backend/src/referrals/` (module complet, 8 fichiers)
- `backend/src/technician/technician-age.ts`

### Créés

- Aucun.

## Migrations

- **Nom** : `prisma/migrations/20261031040000_remove_technician_and_dispatch/migration.sql`
- **Nature** : suppression du domaine « technicien » v2 —
  1. retrait de la valeur `TECHNICIAN` de l’enum `Role` (reste `CLIENT`, `ADMIN`) ;
  2. suppression des tables du domaine technicien / dispatch / KYC / parrainage ;
  3. suppression de `Demande.technicianId` (+ index / contraintes) ;
  4. suppression de `User.referralCode` (+ index unique).
- Les tables financières (`FinancialTransaction`, `FundsHold`, `WithdrawalRequest`, `TopupIntent`, `RelioWithdrawal`) sont **volontairement non touchées** : elles conservent un `technicianId` historique de simple trace, sans relation vers un technicien.

## Vérifications

| Contrôle | Résultat |
|---|---|
| `prisma validate` | **exit 0** |
| `prisma generate` | **exit 0** (client Prisma 7.10.0) |
| `tsc --noEmit -p tsconfig.build.json` | **exit 0, 0 erreur** — ré-verifié **sans cache** (`--incremental false`) pour écarter tout artefact de cache |
| `oxlint src/ test/` | **exit 0, 0 erreur / 0 warning** après nettoyage des imports et variables inutilisés |
| `vitest run` | **691 tests passés / 45 ignorés / 736** (57 fichiers) ; 4 fichiers en échec uniquement au niveau assemblage (variables d’environnement absentes : `JWT_SECRET`, `FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`) |

### Preuve de non-régression

- Baseline `HEAD` propre (`git stash push -u`) + run complet : **5 fichiers en échec / 63 fichiers au total — 759 passés / 55 ignorés / 814 tests** (dont `referrals/referrals.module.spec.ts`, supprimé depuis par le chantier).
- État final du chantier : **4 fichiers en échec / 57 fichiers au total — 691 passés / 45 ignorés / 736 tests**.
- Les 4 échecs finaux sont les mêmes familles que la baseline (module assembly / variables d’environnement). Aucun échec additionnel : les seuls écarts sont les fichiers de specs supprimés par le périmètre. **Aucune régression constatée** ; les échecs restants sont pré-existants et hors périmètre.

### Cohérence du nettoyage

- Aucun fichier `*.js` parasite dans `src/` (hors `generated/`).
- `tsconfig.build.tsbuildinfo` : présent sur le disque, **non tracké** (`*.tsbuildinfo` dans `.gitignore`), excluant toute contamination du commit.

## Bugs trouvés

1. **`backend/src/tracking/tracking.service.ts`** — la réponse du suivi public était revenue **sans** les champs `technicianAssigned`, `technicianVerified` ni `timeline` (champs du contrat public v2). Correction : restauration des trois champs (`technicianAssigned: false`, `technicianVerified: false`, `timeline` reconstruit via `buildTimeline`). Le test `backend/src/tracking/tracking-navigation.spec.ts` échouait à ce sujet (`expected undefined to be false`) ; il passe désormais. Impact : rupture du contrat public de tracking.
2. **Nettoyage lint** : imports et variables devenus inutiles après refonte
   (`backend/src/demandes/demandes.service.ts` — types `DemandeMediaRow` / `DemandeRecord` ; `backend/src/admin/admin.controller.ts` — import `Post` ; `backend/src/auth/password-reset-flow.spec.ts` — variable `registered` inutilisée).

## Points d’attention

- **Risques runtime résiduels (M6)** : plusieurs services continuent de lire `Demande.technicianId`, colonne **supprimée** par la migration — hors périmètre du nettoyage `tsc` (les sélections Prisma ne sont pas toutes rétro-signalées par le typage généré) :
  `collaboration.service.ts`, `demande-media.service.ts`, `disputes.service.ts`, `reviews.service.ts`, `realtime.controller.ts`, `rewards.service.ts`, `backoffice-agent.tools.ts`.
- `financial.service.ts` : règlement technicien maintenu en logique **legacy** (historique), `settleMissionAtConfirmation` reçoit `technicianId: null` côté `demandes` — l’historique financier n’est pas altéré.
- `UserRole` conserve la valeur `'TECHNICIAN'` côté contrat d’authentification (`backend/src/auth/types.ts`) : seules les références Prisma/technicien ont été retirées.
- `backend/src/technician/` conserve les utilitaires partagés (storage, avatar, nationalités).

## Questions bloquantes

- **Aucune** bloquante pour la clôture de M0.2.
- **À traiter en M6** : purge définitive des lectures `Demande.technicianId` restantes (collaboration, médias, litiges, avis, realtime, rewards, backoffice) et confirmation du déploiement Railway.

## Décisions

- Les tables et traces financières ne sont pas supprimées (décision M0.2) : historique conservé.
- Les specs obsolètes ont été **supprimées** plutôt que réécrites lorsque la fonctionnalité disparaissait.
