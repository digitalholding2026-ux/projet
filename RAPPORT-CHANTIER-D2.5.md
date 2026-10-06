# RAPPORT — CHANTIER D2.5 : COOKIE SYSTÉMATIQUE + RELANCES EMAIL

**Date** : 2026-10-06
**Dépôts concernés** : `Repairdom-backend` (`dea7651`), `Repairdom-frontend` (`00f7e49`)
**Règle projet** : `AGENTS.md` créé et poussé dans ce commit de session
**Dépend de** : D1 (backend brouillon `8177aa6`), D2 (frontend wizard public `08f4edb`)

---

## 1. SYNTHÈSE

Le blocage décrit dans `RAPPORT-CHANTIER-D2.md` § 2 est levé des deux côtés.

**Backend** — le cookie de session est désormais posé **sans condition** à
l'inscription, y compris pour un CLIENT non vérifié. C'était la cause de la
mort du tunnel : sans cookie, `POST /demandes/drafts/:token/convert` répondait
401. Un **scheduler de relances** (J+1, J+3, J+7) rattrape les comptes dont
l'e-mail n'est jamais confirmé.

**Frontend** — le brouillon est converti même si l'e-mail reste à vérifier, et
l'utilisateur atterrit sur `/client/verification?from=demande` (puis sur la
liste de ses demandes après validation) au lieu d'un échec.

Le tunnel « demande d'abord, inscription à la fin » **fonctionne désormais de
bout en bout**.

---

## 2. LE BLOCANT, ET COMMENT IL A ÉTÉ LEVÉ

| Constat (D2) | Correction D2.5 |
|---|---|
| `auth.controller.ts` conditionnait la pose du cookie à `user.emailVerified`, qui vaut `false` pour un CLIENT | Pose inconditionnelle |
| `convert` répondait donc 401 | Le wizard convertit avant toute condition sur l'e-mail |
| Le panneau de vérification déconnectait après validation | `logout()` sauté quand `?from=demande` |

**Séparation préservée** : le cookie donne l'**identité**, la vérification
e-mail donne l'**accès au dashboard**. `guard-decision.ts` n'a **pas** été
touché — un compte non vérifié est toujours redirigé vers
`/client/verification`.

Vérifié en production après déploiement :

```
POST /api/auth/register → HTTP 201 + set-cookie: repairdom_token=…
user.emailVerified = false · role = CLIENT
```

Le cookie est posé **et** le compte reste non vérifié : les deux contrôles sont
bien séparés.

---

## 3. FICHIERS — BACKEND

### Créés

| Chemin | Lignes |
|---|---|
| `backend/src/auth/verification-reminder.scheduler.ts` | 236 |
| `backend/src/auth/verification-reminder.spec.ts` | 361 (23 tests) |
| `backend/src/auth/verification-reminder.module.spec.ts` | 84 (4 tests) |

### Modifiés

| Chemin | Nature |
|---|---|
| `src/auth/auth.controller.ts` | Retrait du `if (user.emailVerified)` |
| `src/auth/email-templates.ts` | `buildVerificationReminderEmail`, `verificationReminderSubject`, `MAX_VERIFICATION_REMINDERS` (+97 l.) |
| `src/auth/email.service.ts` | `sendVerificationReminderEmail` (+42 l.) |
| `src/auth/auth.module.ts` | Scheduler enregistré, provider **interne** (non exporté) |
| `src/auth/auth.spec.ts` | +5 tests du cookie |
| `prisma/schema.prisma` | 2 champs sur `User` |
| `test/auth-demandes.e2e-spec.ts` | Commentaire corrigé + test JWT décodable |
| `docs/UX-BACKLOG.md` | 3 suivis |

---

## 4. FICHIERS — FRONTEND

### Créé

| Chemin | Lignes |
|---|---|
| `frontend/src/lib/demande-draft-d2_5.test.ts` | 161 (16 tests) |

### Modifiés

| Chemin | Nature |
|---|---|
| `src/components/client/demande-wizard.tsx` | `handleAuthSuccess` réécrite, type `ConversionOutcome`, 401 → « Se connecter » (+96 l.) |
| `src/components/client/demande-auth-modal.tsx` | `showSignInCta`, `signInRequestId` |
| `src/components/auth/verification-panel.tsx` | `?from=demande` : `logout()` conditionnel, redirect, bouton « Voir mes demandes » |
| `package.json` / `tsconfig.json` | Enregistrement du test |

---

## 5. MIGRATION

`add_verification_reminders` — `prisma/migrations/20261015010000_add_verification_reminders/`

| Élément | Contenu |
|---|---|
| Colonnes | `verificationReminderCount Int NOT NULL DEFAULT 0`, `lastVerificationReminderAt TIMESTAMP(3)` |
| Idempotence SQL | `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` (convention du dépôt) |
| Index | `User_pending_verification_idx` sur `createdAt` **partiel** `WHERE "emailVerified" = false` |
| Justification de l'index partiel | Seul le balayage le traverse ; les autres requêtes sur `User` (par `email`, `cityId`) sont éliminées d'emblée par le préfixe `emailVerified` |

---

## 6. VÉRIFICATIONS

### Backend

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit -p tsconfig.build.json` | **OK** |
| `oxlint src/ test/` | **propre** |
| Nouveaux tests | **27/27 verts** (23 scheduler + 4 assemblage) |
| Suite complète | **922 passés** |

**Non-régression prouvée** : les 3 échecs de
`src/demandes/demande-multimedia.spec.ts` sont **pré-existants** — vérifié par
`git stash` lors du chantier D1 (mêmes noms, sur `HEAD` propre).

### Frontend

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit` | **OK** |
| `oxlint src/` | **propre** |
| Nouveaux tests | **60/60 verts** (16 D2.5 + 25 sync + 19 routage) |
| Tests existants | **5 échecs, noms identiques** à la référence D2 → **0 régression** |

### Déploiement

| Contrôle | Résultat |
|---|---|
| Railway `/api/health/db` | **HTTP 200** |
| Railway migrations | **47**, dernière `20261015010000_add_verification_reminders` → **`applied`** |
| Vercel `/demande` | **HTTP 200** |
| Prod : `POST /auth/register` | **201 + `set-cookie: repairdom_token`** |
| Prod : `emailVerified` retourné | **`false`** (séparation identité/accès intacte) |

---

## 7. BUGS ET DÉFAUTS TROUVÉS PAR MES PROPRES TESTS

| # | Défaut | Impact | Correction |
|---|---|---|---|
| 1 | **`toDraftPayload` oubliait `brandId`** (trouvé en D2) | Demande inatteignable à la reprise | Corrigé en D2 |
| 2 | **Le compteur rendait le compte éligible à la fenêtre suivante dans le même balayage** | 3 relances au lieu d'1 | Restructuré en **une requête** groupée par fenêtre |
| 3 | **Le double de test ignorait le `where`** | Mesurait la mécanique du mock, pas la logique métier | Double rendu fidèle (`AND` + Fenêtres) |
| 4 | **Le double n'écrivait pas en mémoire** | Test d'idempotence vert à vide | `update` reflète désormais l'écriture |
| 5 | **Ancre de test devenue obsolète** après factorisation | Faux négatif | Ré-ancrée + invariance maintenue |
| 6 | **Le harnais local écrivait des `.js` dans le dépôt** via un lien symbolique | Fichier source supprimé une fois | Nettoyé, harnais corrigé, RÈGLE 4 écrite dans `AGENTS.md` |

Le bug **n° 2 est le plus important** : il n'aurait pas cassé la production
(l'âge du compte l'empêchait), mais il révélait une **dépendance cachée à la
donnée** plutôt qu'une garantie structurelle.

---

## 8. POINTS D'ATTENTION

### 8.1 Le token est régénéré à chaque relance — non demandé, mais indispensable

Le token de vérification d'origine expire à **24 h**. Un rappel J+3 porterait
donc un **lien déjà mort** : un e-mail qui ne fonctionne pas, le pire des deux
mondes. Chaque relance crée un token neuf (`randomBytes(24)`), exactement comme
`resendVerification`.

### 8.2 Idempotence multi-instance : imparfaite

La relecture « juste avant l'envoi » n'est **pas atomique**. Deux réplicas
pourraient doubler sur la fenêtre exacte. Correct en instance unique (Railway
actuel). Solution si scale → colonne `verificationReminderClaimedAt` + un
`updateMany` conditionnel. **Noté au backlog.**

### 8.3 Un compte non vérifié peut déposer une demande

Conséquence directe de « cookie = identité / vérification = accès ». La demande
est créée et part en dispatch ; le client ne peut simplement pas encore la
suivre. C'est l'interprétation de la consigne « ne pas toucher à
`guard-decision.ts` », mais c'est un choix à confirmer.

### 8.4 Cookie à 7 jours pour un compte non vérifié

Le TTL du cookie est `7d` pour tout le monde. Un compte qui ne vérifie jamais
conserve donc une session valide 7 jours tout en restant bloqué côté dashboard.
Durée réduite pour les comptes non vérifiés = changement d'une ligne.

### 8.5 Fenêtres de 2 h

Balayage horaire, fenêtre large de 2 h : un balayage tardif ou manqué est
rattrapé au suivant. En revanche un compte peut recevoir son rappel avec
**jusqu'à 1 h de retard**, et **2 h de retard** s'il est sur le chemin entre
deux balayages (tombant hors des deux fenêtres). Tolérable, à surveiller.

### 8.6 Reprise du brouillon pour un utilisateur revenu connecté

Non traitée (inchangé depuis D2) : après vérification, l'utilisateur revient
connecté sur `/demande`, mais le mode authentifié ne relit pas le brouillon.
Ici le problème est **atténué** : la demande a déjà été convertie et est
visible dans `/client/demandes`, que la redirection de `?from=demande` cible
directement.

### 8.7 Effet de bord de ma vérification en production

Le contrôle du déploiement a créé **2 comptes de test** en base :
`controle.d25.20261006@relio-test.invalid` et `controle.d25.b@relio-test.invalid`
(non vérifiés, domaine `.invalid` qui ne reçoit aucun e-mail). Ils sont
inertes mais **insupprimables via l'API publique** (seul
`DELETE admin/users/:id` existe, protégé par `ADMIN`). **À supprimer depuis le
back-office admin.**

---

## 9. QUESTIONS BLOQUANTES

**Aucune.**

Deux arbitrages restent ouverts, sans bloquer :

1. **Durée du cookie pour un compte non vérifié** (§ 8.4) : 7 j comme tout le
   monde, ou réduit ?
2. **Un compte non vérifié peut-il déposer une demande ?** (§ 8.3) : c'est
   l'état actuel ; à confirmer ou à restreindre côté backend.

---

## 10. SCÉNARIO DE TEST PRODUCTION

| # | Test | Attendu |
|---|---|---|
| 1 | Navigation privée → `/demande`, remplir, « Envoyer », s'inscrire | Upload + **conversion** + redirection `/client/verification?from=demande` |
| 2 | Suivre le lien de l'e-mail | `/client/demandes` avec la demande fraîchement créée |
| 3 | Compte test avec `createdAt` à J+1 | E-mail de relance reçu (attendre le balayage, ou appeler `sweep()`) |
| 4 | `POST /auth/register` (curl) | `set-cookie: repairdom_token=…` présent |
| 5 | `GET /auth/me` avec ce cookie | **200**, `emailVerified: false` |

Le test 1 est **le** test de validation du chantier : c'est le parcours qui
était cassé depuis D2.