# RAPPORT — CHANTIER 4-FONDATIONS-C : Refonte LTV des récompenses client

> Remplacement du programme #4A (nombre de missions) par un modèle fondé sur la
> **marge cumulée** générée par le client. Déployé Railway puis Vercel.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Backend | `Repairdom-backend` — commit **`da71861`** `feat(rewards): replace mission-count programme with a margin-based LTV scheme` — poussé `0984c2e..da71861` |
| Frontend | `Repairdom-frontend` — commit **`50002d9`** `feat(rewards): rebuild the loyalty page around margin-based credits, badges and nature rewards` — poussé `0e1c284..50002d9` |
| Railway | ✅ **vert** : `status=ok`, `database=up`, uptime 490 s croissant sans redémarrage |
| **Migrations en production** | ✅ **PRUVÉES** : `GET /api/health/migrations` → 49 migrations, dont `20261016010000_rewards_ltv_notification_types` et `20261016020000_refonte_rewards_ltv` toutes deux **`applied`** |
| Vercel | ✅ `https://www.relioo.space/client/recompenses` → `200` (22 687 o), stable sur 5 contrôles |

## Synthèse

Le programme #4A récompensait un **nombre de missions** (paliers à 15 / 50 / 150
/ 500 missions). Avec le barème « 500 FCFA + 4 % » du chantier 4-A, ce modèle
était économiquement insoutenable : le coût du programme n'était borné par rien.

Il est remplacé par un modèle **LTV** : la progression porte sur la **marge
réellement générée** par le client, c'est-à-dire la commission Relio
prélevée sur chacune de ses missions confirmées. Le coût devient alors borné
**par construction** :

```
crédits = floor(marge cumulée / 10 000) × 500   soit exactement 5 % de la marge à vie
```

La marge est cumulée avec la **vraie fonction de commission** du chantier 4-A
(`calculateTechnicianFee`), donc la progression ne peut pas diverger de la
comptabilité. Le transport n'est jamais commissionné : il reste un
pass-through intégralement reversé au technicien.

Les 4 anciens paliers (`BRONZE` / `ARGENT` / `OR` / `PLATINE`, sur missions)
deviennent 3 paliers badge (`FIDELE` / `OR` / `PLATINE`, sur marge), et
s'ajoutent 3 récompenses nature cumulables. Crédits et nature se **réclament**
par le client ; leur versement reste manuel côté admin pour la nature.

## Fichiers

### Backend — 13 modifiés, 2 créés

| Type | Chemin |
|---|---|
| Modifié | `prisma/schema.prisma` — `RewardTier`, `NatureRewardTier`, `ClientRewardProgress`, `CLIENT_REWARD_CREDIT`, 2 `NotificationType` |
| **Créé** | `prisma/migrations/20261016010000_rewards_ltv_notification_types/migration.sql` |
| **Créé** | `prisma/migrations/20261016020000_refonte_rewards_ltv/migration.sql` |
| Modifié | `src/rewards/rewards.config.ts` — **réécrit** (source unique des seuils) |
| Modifié | `src/rewards/rewards.service.ts` — **réécrit** |
| Modifié | `src/rewards/rewards.controller.ts` — 3 routes client |
| Modifié | `src/rewards/rewards-notifications.service.ts` — 3 nouvelles notifications |
| Modifié | `src/notifications/notification-metadata.ts` — contrat metadata |
| Modifié | `src/auth/email-templates.ts` — gabarit récompenses |
| Modifié | `src/auth/email.service.ts` — `sendRewardTierReachedEmail` |
| Modifié | `src/mission-events/mission-events.ts` — types + libellés par défaut |
| Modifié | `src/rewards/{rewards.service,rewards.config,rewards-notifications,rewards.module}.spec.ts` |

### Frontend — 7 modifiés

| Type | Chemin |
|---|---|
| Modifié | `src/lib/api/rewards-service.ts` — **réécrit** (contrat + 3 appels) |
| Modifié | `src/lib/rewards-view.ts` — **réécrit** (logique pure) |
| Modifié | `src/app/client/recompenses/page.tsx` — **page refondue** |
| Modifié | `src/components/client/reward-badge.tsx` — API inchangée, table de présentation |
| Modifié | `src/lib/api/notifications-service.ts` — types metadata |
| Modifié | `src/lib/notifications/notification-mapping.ts` — 2 types + actions |
| Modifié | `src/lib/rewards-view.test.ts` — **réécrit** |

## Migrations

**2 fichiers SQL, écrits à la main** — `prisma migrate dev` exige une base de
données, interdite par la règle « aucun test nécessitant une infrastructure ».
La validation repose sur `prisma validate` (schéma valide) et sur
`prisma generate` (client TypeScript régénéré).

| Migration | Contenu |
|---|---|
| `20261016010000_rewards_ltv_notification_types` | 3 × `ALTER TYPE … ADD VALUE IF NOT EXISTS` (2 `NotificationType`, 1 `FinancialTransactionType`). Isolé dans son **propre** fichier, sans `BEGIN`/`COMMIT` : c'est la convention du dépôt, `ALTER TYPE ADD VALUE` n'étant pas compatible transaction sur les anciennes versions de PostgreSQL. |
| `20261016020000_refonte_rewards_ltv` | `DROP TABLE` puis `DROP TYPE` de l'ancien modèle, `CREATE TYPE` des deux nouveaux enums, `CREATE TABLE` + index unique + clé étrangère. |

**Ordre impératif documenté dans le SQL** : la table est supprimée en premier
(elle référence l'ancien enum par une colonne simple *et* deux colonnes
tableau), l'enum ne peut être recréé qu'ensuite. `CASCADE` n'est volontairement
**pas** utilisé sur le `DROP TYPE` : une dépendance oubliée doit faire échouer
la migration plutôt que supprimer autre chose par surprise.

**Suppression sans perte** : aucun client en base ne dispose de progression #4A
(programme jamais utilisé en production), donc ni la table ni l'enum ne
portent de données. Ni `ALTER COLUMN`, ni backfill.

## Vérifications

| Commande | Backend | Frontend |
|---|---|---|
| `prisma validate` | ✅ schéma valide | — |
| `prisma generate` | ✅ client régénéré | — |
| `tsc --noEmit` | ✅ (`-p tsconfig.build.json`) | ✅ |
| `npm run lint` | ✅ 0 warning | ✅ 0 erreur, 5 warnings préexistants hors périmètre |
| Tests | **980**, 3 échecs préexistants | **390**, 9 artefacts miroir |
| Non-régression | **prouvée** | **prouvée** |

### Backend
- Baseline `0984c2e` : `987 tests, 3 failed`
- Après `da71861` : `980 tests, 3 failed`
- `diff` des **noms** d'échecs : **identiques** — les 3 échecs sont
  pré-existants (`demande-multimedia.spec.ts`, `docs/UX-BACKLOG.md:38-42`).
- `src/rewards/` seul : **81/81** (contre 91 tests dont 54 échecs avant, l'API
  #4A ayant été remplacée).
- `rewards.module.spec.ts` assemble réellement l'application (`app.init()`) :
  les 3 routes client et les 2 routes admin répondent, et l'ancienne route
  `POST /client/rewards/:tier/claim` renvoie désormais **404**.

### Frontend
`npm run test:unit` **n'a pas pu être lancé tel quel** : Node local =
**20.20.2**, le runner `.ts` natif exige **Node 22+** (limitation
pré-existante, `AGENTS.md` RÈGLE 4). Vérification par compilation `tsc` vers un
**miroir hors dépôt** (`/tmp/opencode/fe-mirror`), puis `node --test`.

- Baseline : 9 échecs · Après : **9 échecs, noms identiques** → **0 régression**.
- Ces 9 échecs sont des **artefacts du miroir** (il ne contient que `src/`, pas
  `public/sw.js`, `docs/`, `logo/`, `package.json` ni `.git`) : identiques sur
  `HEAD` propre, donc sans rapport avec les modifications.
- `rewards-view.test.ts` : **15/15**.
- **Sur Node 22 (Vercel / poste de dev), `npm run test:unit` lancera réellement
  les 35 fichiers.** Non vérifié en conditions réelles.

> **Non exécuté volontairement** (interdit par la mission) : `next build`,
> `npm run dev`, Docker, Postgres local, tests e2e.

## Bugs trouvés

| # | Constat | Emplacement | Impact | Suite |
|---|---|---|---|---|
| **1** | **Bug dans le service écrit pour ce chantier** : `newlyReachedTiers` recevait `natureReached` (paliers NATURE) au lieu de la liste des BADGES déjà atteints. Conséquence : `FIDELE` était notifié **à chaque mission** après le franchissement (3 fois sur 12 missions). | `src/rewards/rewards.service.ts` (`accumulateMargin`) | Notifications en doublon, `notifyCreditsEarned`/badge bruyants | **Corrigé** + verrouillé par le test « ne NOTIFIE PAS deux fois un palier déjà atteint » |
| **2** | **Dérive de `NotificationType`** : les 2 nouvelles valeurs manquaient dans le type local de `mission-events.ts` (dérive déjà signalée à l'audit du 08/10 sur les types #4A). | `src/mission-events/mission-events.ts` | Le backend ne compilait pas | **Corrigé** — types + libellés par défaut dans `buildNotification` |
| **3** | **Lien de navigation perdu** : la page `/client/recompenses` référençait `/client/parrainage` (« Inviter un ami »). La réécriture l'a supprimé → `navigation.test.ts` échoué. | `src/app/client/recompenses/page.tsx` | Perte d'une entrée de parcours | **Corrigé** — lien restauré en action de `PageHeader` |
| 4 | `useToast` importé du mauvais module (`@/components/ui/toast` ne l'exporte pas). | page récompenses | erreur de compilation | Corrigé (`@/lib/toast-context`) |
| 5 | Icônes `gift` et `award` inexistantes dans le design system. | page récompenses | erreur de compilation | Corrigé (`sparkles`, `badge-check`) |
| 6 | Deux assertions de test qui attendaient une sémantique que la fonction n'avait pas (`creditProgressPercent(10_000)`). | `rewards-view.test.ts` | test incohérent | Test aligné sur la fonction, dont la sémantique est désormais documentée |

## Points d'attention

1. **`claimCredits` écrit dans le ledger** via `currentMode()`, qui lit
   `process.env.FINANCIAL_MODE`. **À VÉRIFIER en production** que cette
   variable est bien définie sur Railway, sinon les crédits seraient écrits en
   `SIMULATION` alors que le solde du client est lu en `REAL`.

2. **`currentMode()` est DUPLIQUÉ** : `FinancialService` a sa propre logique de
   mode. Si les deux divergent, un crédit pourrait être écrit dans un mode
   différent de celui de la lecture du solde. À faire converger plus tard.

3. **Aucun backfill** : les clients ayant confirmé des missions avant ce
   chantier démarrent avec une marge à 0. Choix assumé (le #4A ne produisait
   rien d'exploitable à reprendre, et un backfill imposerait de rejouer
   l'historique des règlements, avec un risque de double comptage).

4. **Le solde affiché ne bouge pas immédiatement** : `claimCredits` renvoie
   bien `newBalanceXAF`, mais la page `/client/recompenses` recharge
   `getRewardsProgress()`. La page `/client/solde` ne verra le nouveau solde
   qu'à son prochain chargement.

5. **Barème à 14 % sur les petites missions** : à 5 000 FCFA de devis, la
   commission vaut 700 XAF, soit 14 % du devis. C'est la conséquence
   arithmétique de « 500 fixes + 4 % » validée au chantier 4-A, combinée au
   plancher de 5 000. Le programme de fidélité ne change rien à ce point.

6. **`notifyCreditsEarned` n'envoie PAS d'e-mail** (décision assumée) : un
   crédit est un avantage mineur et fréquent, il s'afficherait dans l'app et en
   push. Badge et nature restent notifiés par e-mail.

7. **Progression dans la tranche** : à un multiple exact de 10 000, la barre
   affiche 100 % (tranche complétée) et non 0 %, pour ne pas retomber à zéro à
   l'instant où le crédit est gagné. Sémantique documentée dans la fonction.

8. **`npm run lint` frontend** : les 5 warnings `exhaustive-deps` restants sont
   **préexistants** et portent sur des fichiers non modifiés
   (`admin/catalog/**`, `technicien/profil/page.tsx`). Non corrigés.

9. **Le test structurel de hooks** (`technician-quote.test.ts`) tourne aussi
   sur les fichiers du chantier 4-A : aucun hook après return dans
   `/client/recompenses`.

10. **`Notification.metadata` : 2 clés supprimées** (`rewardMissions`,
    `rewardValueXAF`) et 7 ajoutées. Une notification #4A déjà en base reste
    lisible (le constructeur tolère les clés absentes) mais ses champs de
    seuil ne sont plus interpretés côté UI.

## Questions bloquantes

**Aucune.**

---

## Scénario de test production

**Test 1 — Progression**
1. Client avec plusieurs missions CONFIRMED → `/client/recompenses`.
2. **Attendu** : marge cumulée affichée, barre de progression correcte vers la
   tranche de 10 000, 3 badges et 3 récompenses nature listés avec leur statut.

**Test 2 — Crédits**
1. Atteindre 10 000 de marge cumulée.
2. **Attendu** : notification « De nouveaux crédits de fidélité ! ».
3. `/client/recompenses` → « Ajouter à mon solde ».
4. **Attendu** : solde +500, `creditsClaimed` = 500, bouton désactivé ensuite.

**Test 3 — Badge**
1. Atteindre 10 000 de marge.
2. **Attendu** : badge 🥉 Fidèle sur `/client/profil`.

**Test 4 — Nature**
1. Atteindre 50 000 de marge.
2. **Attendu** : notification « Récompense Petit électroménager débloquée ! ».
3. Cliquer « Réclamer ma récompense » → statut « En cours de traitement ».

**Test 5 — Anti-fraude**
1. Deux missions CONFIRMED consécutives, même technicien, moins de 48 h.
2. **Attendu** : signalement créé côté admin, la 2ᵉ mission ne cumule pas de marge.

**Test 6 — Réclamations rejetées**
1. « Ajouter à mon solde » sans crédit disponible → bouton désactivé.
2. Réclamer une nature non atteinte → bouton absent.