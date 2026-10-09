# RAPPORT — CHANTIER 4B : parrainage client

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôts | `Repairdom-backend`, `Repairdom-frontend`, dépôt racine `projet` |
| Commits | backend `9edfbb2` · frontend `d6d5c49` · racine `RAPPORT-CHANTIER-4B-PARRAINAGE.md` |
| Base | `main` poussée dans les deux dépôts, dans l'ordre backend → frontend (RÈGLE 2) |
| Déploiement | **Railway vert, Vercel vert** — vérifié factuellement (voir §Déploiement) |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |

## Synthèse

Un client partage un lien `?ref=RELIO-XXXXX`. L'inscription du filleul crée un
`Referral` en `REGISTERED`. **500 FCFA de crédit** sont versés au parrain **et**
au filleul à la première mission `CONFIRMED` du filleul — jamais à l'inscription,
qui rendrait le mécanisme inépuisable (créer un compte, encaisser, supprimer).

Le versement est **deux écritures de ledger distinctes**, pas un solde stocké
ailleurs : le ledger reste la seule vérité financière, comme pour le programme
de récompenses.

**Le cycle de modules était le vrai obstacle de ce chantier**, pas le modèle de
données. Il est résolu sans `forwardRef` — voir Bugs §1.

## Fichiers

### Backend (`Repairdom-backend`)

Créés :

- `src/referrals/referrals.config.ts` — barème, alphabet, validation (pur, 100 l.)
- `src/referrals/referrals.service.ts` — 4 méthodes (386 l.)
- `src/referrals/referrals-notifications.service.ts` — 4 canaux (216 l.)
- `src/referrals/referrals.controller.ts` — 2 endpoints
- `src/referrals/referrals.module.ts`
- `src/referrals/referrals.service.spec.ts` — **29 tests** (règles métier)
- `src/referrals/referrals-notifications.spec.ts` — **8 tests** (4 canaux)
- `src/referrals/referrals.module.spec.ts` — **10 tests** (assemblage `app.init()`)
- `src/auth/register-referral-wiring.spec.ts` — **11 tests** (rattachement à l'inscription)
- `src/auth/email-referral-templates.spec.ts` — **11 tests** (gabarits e-mail)
- 3 dossiers de migration (voir §Migrations)

Modifiés : `prisma/schema.prisma`, `src/app.module.ts`, `src/auth/auth.service.ts`,
`src/auth/dto/register.dto.ts`, `src/auth/email.service.ts`,
`src/auth/email-templates.ts`, `src/demandes/demandes.service.ts`,
`src/demandes/demandes.module.ts`, `src/mission-events/mission-events.ts`,
`src/notifications/notification-metadata.ts`, `src/realtime/realtime.types.ts`.

### Frontend (`Repairdom-frontend`)

Créés : `src/lib/referrals-rules.ts` (pur), `src/lib/api/referrals-service.ts`,
`src/lib/referrals-frontend.test.ts` (**15 tests**).

Modifiés : `src/app/client/parrainage/page.tsx` (**refondu**),
`src/components/client/client-auth-form.tsx`, `src/lib/api/auth-service.ts`,
`src/lib/api/notifications-service.ts`, `src/lib/notifications/notification-mapping.ts`,
`package.json`, `tsconfig.json`.

## Migrations

**3 migrations**, comme demandé :

| Nom | Nature |
|---|---|
| `20261017010000_add_referrals` | `CREATE TYPE ReferralStatus`, `CREATE TABLE Referral`, 4 index dont 2 uniques, `ALTER TABLE User ADD COLUMN referralCode`, 2 clés étrangères |
| `20261017020000_add_referral_transaction_types` | `ALTER TYPE FinancialTransactionType ADD VALUE` × 2 |
| `20261017030000_add_referral_notification_types` | `ALTER TYPE NotificationType ADD VALUE` × 2 |

Les deux `ALTER TYPE` sont dans des fichiers **séparés** : `ALTER TYPE ... ADD
VALUE` est incompatible avec un bloc de transaction sur PostgreSQL 11 et
antérieurs. C'est la convention déjà appliquée par les chantiers #4A et
4-FONDATIONS-C.

`prisma generate` a été exécuté : le client est à jour, `tsc` passe.

## Vérifications

| Contrôle | Baseline | Après | Verdict |
|---|---|---|---|
| Backend `tsc --noEmit` | ✅ | ✅ | pas de régression |
| Backend `oxlint` | ✅ | ✅ | pas de régression |
| Backend `vitest run` | 994/997 | **1 071/1 074** | **+77, 0 régression** |
| Frontend `tsc --noEmit` | ✅ | ✅ | pas de régression |
| Frontend `oxlint` + eslint | 5 warnings, 0 erreur | identique | pas de régression |
| Frontend `test:unit` | 438/442 | **453/457** | **+15, 0 régression** |

Les échecs restants sont **les mêmes** qu'en base : 3 côté backend
(`demande-multimedia.spec.ts`, déjà au backlog) et 4 côté frontend
(`demande-draft-sync`, `design-system` logo, `verification-confirm` ×2).
Aucun test existant modifié ou supprimé.

**Le test d'assemblage `app.init()` A pu être écrit et passe** (10 tests) :
`PrismaService` n'implémente pas `OnModuleInit`, donc le graphe complet se
construit sans base — le motif existe déjà (`rewards.module.spec.ts`). C'est le
test qui prouve que le cycle `Auth ↔ Referrals` ne fait pas tomber l'API au
démarrage sur Railway. `DATABASE_URL` y est un dummy, aucun port n'est touché.

## Bugs trouvés

### 1. Le cycle de modules que la spécification ne voyait pas

La spec disait « Imports : PrismaModule, RealtimeModule, PushModule, EmailModule »
en A.9 — **or aucun module `EmailModule` n'existe** : `EmailService` est
exporté par `AuthModule`. Et `ReferralsController` a besoin de `JwtAuthGuard` et
`RolesGuard`, qui sont aussi des providers d'`AuthModule`.

Donc `ReferralsModule → AuthModule` est inévitable. Mais A.6 demande que
`AuthService.register` appelle `ReferralsService`, ce qui crée
`AuthModule → ReferralsModule`. **Cycle.**

Deux contournements possibles, un écarté :

- `forwardRef` — **exclu par convention** dans ce dépôt (« aucun `forwardRef`
  dans tout le dépôt »). Non envisagé.
- Extraire `EmailModule` + un module de guards — **insuffisant** : `JwtAuthGuard`
  dépend d'`AuthService`, donc le cycle persisterait par les guards.

**Solution retenue : résolution lazy via `ModuleRef`.** `AuthService` ne déclare
**aucune** dépendance sur `ReferralsModule` ; il résout le service au moment de
l'appel (`this.moduleRef.get(ReferralsService, { strict: false })`), dans un
`try/catch`. Aucun module n'a besoin de l'autre pour démarrer : le cycle
n'existe pas, seule une recherche au runtime, dont l'échec est absorbé — un
compte sans parrainage doit s'inscrire normalement.

Un second cycle,a été évité de la même façon : le mode financier du
parrainage est lu via `ConfigService` (`FINANCIAL_MODE`) et **non** via
`FinancialService`, car `FinancialModule → DemandesModule → ReferralsModule`.
Le mode reste une décision serveur dans les deux cas.

### 2. L'alphabet de code ne correspond pas à son commentaire

La spec fournit l'alphabet `ABCDEFGHJKMNPQRSTUVWXYZ23456789` en annonçant
« sans 0/O/I/L ambigus ». C'est vrai — mais seulement pour **5 caractères** :
`0`, `1`, `I`, `L`, `O`. Ma première version du commentaire affirmait aussi
que `2/Z`, `5/S`, `8/B` étaient exclus : **c'est faux**, ils sont présents dans
l'alphabet fourni. Le commentaire a été corrigé et le test vérifie désormais les
5 exclusions réelles — le premier test que j'avais écrit échouait, et c'est le
test qui avait raison.

### 3. Le test d'inscription a d'abord échoué pour une raison hors sujet

Trois échecs successifs de `register-referral-wiring.spec.ts`, aucun lié au
parrainage : `register` exige une adresse précise, puis résout la ville via
`ServiceCity`, puis crée un `TechnicianProfile`. Le double Prisma ne fournissait
rien de tout cela. Un test d'inscription qui passe sans ces trois briques
testerait... un stub qui n'inscrit personne. Le double a été complété, et le test
« technicien non rattaché » n'a pu être écrit qu'une fois `TechnicianProfile`
présent — c'est le test qui a corrigé le double, pas l'inverse.

### 4. `ASSIGNED → CONFIRMED` n'existe pas

Un test vérifiant « aucun versement sur une transition non-confirmante » a d'abord
supposé un passage à `TERMINATED`. La machine à états client ne connaît que
`SUBMITTED/PENDING/ACCEPTED/SCHEDULED → CANCELED` et `COMPLETED → CONFIRMED`.
Le test porte désormais sur `ASSIGNED → CONFIRMED`, qui est une transition
REFUSÉE — et qui prouve davantage : la demande est rejetée avant toute écriture.

### 5. Un test vert à vide, évité de justesse

Le double de test initial attribuait aux lignes créées l'identifiant `ref-1`,
qui **écrasait** une ligne seedée dans la Map du double : le test « limite de 5 »
comptait 5 au lieu de 6 et échouait. Le double est aujourd'hui fidèle sur ce
point : identifiant préfixé à part, unicité `code` et `referredId` reproduites,
`updateMany` gardé (c'est lui qui porte le verrou anti-double-versement).

## Écarts explicites par rapport à la demande

| Demandé | Réalisé | Pourquoi |
|---|---|---|
| A.3 — `id String @id @default(uuid())` | `gen_random_uuid()` | Convention du dépôt : toutes les clés primaires sont `dbgenerated("gen_random_uuid()")`. `uuid()` de Prisma génère en JavaScript, pas en SQL. |
| A.1 — `User.referralCode` « généré à la 1ᵉʳᵉ demande » | idem | Choixeconomique : pré-remplir la colonne unique pour tout le monde ne ferait que grossir l'index. Le code est créé à la 1ʳᵉʳᵉ visite de `/client/parrainage`. |
| A.3 — limite « 5 filleuls REGISTERED ou REWARDED » | idem | Un `PENDING` (lien partagé sans inscription) ne consomme pas d'emplacement : sinon partager son lien à droite et à gauche épuiserait les 5 places avant qu'un seul ne s'inscrive. |
| A.3 — `PENDING` créé à l'inscription | **Jamais** | Le modèle le prévoit (lien partagé, invitation), mais rien ne le crée : le flux est « lien → inscription directe ». Créer un `PENDING` sans usage serait de la surface morte. |
| Limite atteinte → refus | Silencieux, ligne `EXPIRED` | Le parcours d'inscription ne doit jamais échouer. Mais la ligne EST créée : sans elle, le filleul pourrait être « offert » à un **autre** parrain via un second code, ce qui contournerait la limite. |
| `Math.random` pour le code | Conservé, documenté | Le code n'est **pas un secret** : il n'accorde aucun accès et ne déclenche aucun paiement. `crypto` serait de la complexité sans contrepartie — c'est dit dans le fichier. |
| B.6 — test « bouton Copier appelle `navigator.clipboard` » | Test statique (présence de l'appel) | Le dépôt n'a ni jsdom ni mock de `navigator`. Le comportement réel sera vérifié par le scénario de production. |

## Points d'attention

1. **`getMyReferrals` crée le code au besoin.** Afficher la page pour un client
   qui n'a jamais parrainé déclenche une écriture (`getOrCreateMyCode`). C'est
   cohérent avec « le code est généré à la première demande », mais cela veut
   dire que la lecture de la page n'est pas purement une lecture.

2. **Le reward est versé sur la première mission CONFIRMED du filleul** — pas
   sur *sa* première mission, mais sur la première mission qui suit le
   rattachement. Un filleul qui a déjà fait 3 missions avant d'être parrainé est
   récompensé dès sa mission suivante. C'est le plus simple à raisonner, et
   cela évite de savoir si l'inscription a précédé ou suivi les missions.

3. **`referredEmail` n'est lu par personne côté serveur.** Il est stocké pour
   que le parrain se souvienne de qui il a invité ; aucune notification n'en
   dépend (elles partent au `referredId`). Le test frontend vérifie qu'il
   n'apparaît pas dans une notification.

4. **Le taux de conversion du parrainage n'est pas mesuré.** Rien ne dit combien
   de liens partagés mènent à une inscription. Si le mécanisme se révèle
   coûteux à 1 000 FCFA par filleul validé sans volume, la limite de 5 est le
   seul frein. Un suivi simple (`PENDING` créé à l'ouverture de la page)
  allowrait de le mesurer — non fait ici.

5. **Le `campaign` `client.referrals_updated` (SSE)** est émis pour rafraîchir
   la page du parrain. Le frontend ne s'y abonne **pas** encore : la page se
   recharge au montage et propose « Réessayer » en cas d'erreur. Le canal est
   câblé côté serveur pour qu'un futur rafraîchissement sans rechargement soit
   gratuit.

6. **Aucune migration n'a pu être exécutée.** Pas de base de données accessible
   ici. Les trois fichiers SQL sont écrits à la main et cohérents avec le
   schéma Prisma, mais **seul un déploiement Railway les validera**. Le
   `startCommand` Railway les applique via `npm run db:migrate:deploy`.

## Scénarios de test en attente

Ceux de la spécification, plus deux points d'attention spécifiques :

| # | Scénario | Point de vigilance |
|---|---|---|
| 1 | Parcours complet A → B | Le champ doit être **pré-rempli** par `?ref=` **et** l'inscription doit succeed malgré un code valide. |
| 2 | Limite de 5 | Le 6ᵉ filleul s'inscrit **normalement** et apparaît côté parrain en « Non éligible », pas en erreur. |
| 3 | Auto-parrainage | Inscription réussie, aucun bonus, une ligne `EXPIRED` n'est **pas** créée (le service refuse). |
| 4 | Code invalide | Inscription réussie. Vérifier que **rien** n'apparaît côté parrain. |
| 5 | Filleul sans mission | Aucune écriture ledger, notification, ni push. Le parrain voit « Inscrit ». |
| 6 | Notifications | 4 canaux pour les **deux** acteurs. Le push et l'e-mail ne peuvent être vérifiés qu'en production. |

**Scénario à ajouter, non prévu par la spec** : un filleul qui s'inscrit **sans**
mission puis en fait une → récompense versée au deuxième moment. C'est le
comportement réel du déclencheur, et il n'est pas évident à l'usage.

## Déploiement

Vérifié par requête, pas supposé. Aucune CLI Railway ni Vercel n'est installée
ici : c'est le comportement observable de la production qui a servi de preuve.

**Railway** (`repairdom-backend-production-4922.up.railway.app`)

| Contrôle | Avant | Après |
|---|---|---|
| `GET /api/health` | 200 | 200 |
| `GET /api/client/referrals/me` | **404** | **401** |
| `POST /api/client/referrals/code` | 404 | **401** |

`404 → 401` est la preuve attendue : la route est enregistrée et le guard
s'applique. Un `404` aurait voulu dire que le build n'avait pas picked les
routes ; un `200` aurait voulu dire qu'elles sont ouvertes à tous. **Les trois
migrations se sont donc appliquées sans erreur** — un échec de migration fait
échouer le `startCommand` Railway, et l'API ne répondrait plus du tout.

**Vercel** (`relioo.space`)

| Contrôle | Résultat |
|---|---|
| `GET /client/parrainage` | 200 |
| `GET /client/inscription` | 200 |

Contrôle du contenu servi, et pas seulement du code HTTP : le chunk
`app/client/parrainage/page-*.js` contient bien `role:"progressbar"` avec
`aria-label:"Progression du parrainage"`, le libellé « X places restantes »,
« Aucune invitation », « l'invitation seule ne rapporte rien », `navigator.clipboard`
et « Réessayer ». Le chunk d'inscription contient « Code parrainage (facultatif) »
et `RELIO-XXXXX`.

Une absence a été vérifiée et **n'est pas un problème** : `client.referrals_updated`
n'apparaît dans aucun chunk frontend. C'est le canal SSE que le *backend* émet
pour rafraîchir la page du parrain ; le frontend ne s'y abonne pas encore, comme
annoncé au §Points d'attention 5. Le nom `ProgressionBar` est également absent
du chunk, simple conséquence du renommage par le minifieur — l'implémentation
est bien là, sous le nom `x`.

## Scénarios restant à faire manuellement

Les 6 scénarios de la spécification restent à faire en production (ils exigent
deux comptes réels et une mission confirmée). S'y ajoute le cas découvert pendant
le chantier : un filleul inscrit **sans** mission, puis qui en fait une. Les
notifications push et e-mail ne sont vérifiables qu'avec les clés VAPID et Resend
réelles.

## Questions bloquantes

Aucune pour le push.

Une décision produit à trancher plus tard : **faut-il mesurer le taux de
conversion** (créer un `PENDING` à l'ouverture de la page) ? C'est ce qui
permettrait de savoir si le mécanisme est rentable avant d'élargir la limite
de 5 filleuls.