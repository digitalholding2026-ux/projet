# RAPPORT — Audit complet du backend Relio

> Audit **en lecture seule**. Aucun fichier de code n'a été modifié, aucune
> dépendance ajoutée, aucune migration créée : les anomalies sont documentées,
> chiffrées et localisées, pas corrigées.
> Le frontend n'est **pas** audité ici ; il a fait l'objet de
> `RAPPORT-AUDIT-FRONTEND.md`. Les deux rapports sont indépendants.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-30 |
| Périmètre | `backend/` (`Repairdom-backend`) — **frontend hors périmètre** |
| Commit audité | `9851cfd` (`main`, `chore(tech): stab tests, lockfile et assemblage AppModule`), arbre de travail propre |
| Commit de ce rapport | racine `projet.git`, branche `main` (voir § 14) |
| Modifications de code | **aucune** |
| Migrations | **aucune** (51 migrations existantes auditées, aucune créée) |
| Dépendances ajoutées | **aucune** |
| Verdict global | **Architecture métier solide et finances sérieusement verrouillées ; la dette critique est ailleurs : authentification/anti-abus, vérification d'e-mail non appliquée par l'API, vulnérabilités de dépendances de production, observabilité nulle** |

---

## 2. Méthode — et ce qui n'a **pas** pu être vérifié

### 2.1 Ce qui a été fait

- Lecture intégrale de la configuration : `package.json`, `package-lock.json`
  (métadonnées), `tsconfig.json`, `tsconfig.build.json`, `nest-cli.json`,
  `oxlint.json`, `prisma.config.ts`, `railway.json`, `.gitignore`,
  `.env.example`, `README.md`, `docs/*.md`, `scripts/migrate-recover.js`.
- Lecture de `src/main.ts`, `src/app.module.ts`,
  `src/config/env.validation.ts`, du filtre global
  `src/common/filters/http-exception.filter.ts`, des 4 guards
  (`JwtAuthGuard`, `RolesGuard`, `DraftThrottleGuard`,
  `TrackingThrottleGuard`) et des 2 décorateurs.
- Cartographie exhaustive des **27 contrôleurs** et de leurs **184 routes** :
  chemin, `@UseGuards`, `@Roles` pour chacun.
- Lecture des services structurants : `auth.service.ts` (635 l.), les 5 guards
  et le flux de session, `demandes.service.ts` (619 l.),
  `demande-media.service.ts`, `demande-draft.service.ts`,
  `financial.service.ts` (2 546 l.), `fee-calculator.ts`,
  `financial-fees.ts`, `saspay-fees.ts`, `saspay-webhook.service.ts`,
  `saspay-topup.service.ts`, `saspay-payout.service.ts`,
  `saspay-api.client.ts`, `collaboration.service.ts`,
  `technician.service.ts`, `admin.service.ts`, `disputes.service.ts`,
  `reviews.service.ts`, `referrals.service.ts`, `rewards.service.ts`,
  `dispatch.service.ts`, `backoffice-agent.tools.ts`,
  `supabase-storage.service.ts`, `health.controller.ts`,
  `realtime.controller.ts`, `push.service.ts`, `email.service.ts`.
- Lecture du schéma Prisma (1 669 lignes) : modèles, enums, index,
  `@@unique` d'idempotence, `onDelete`.
- Lecture des **51 migrations** : recherche mécanique de
  `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, `DELETE FROM` ; contrôle des
  timestamps dupliqués.
- Balayage mécanique de **238 fichiers TS** (hors `src/generated/`) sur :
  `any`, `console.*`, `process.env`, `queryRaw`/`executeRaw`,
  `trust proxy`, `helmet`, `Throttler`, `rate-limit`, `@Cron`,
  `setInterval`, `logger.*`, `findMany` sans `take`, `emailVerified`,
  `FileInterceptor`, `limits`.
- `npm audit` complet et `npm audit --omit=dev` (réseau disponible).
- Exécution réelle de `tsc --noEmit`, `oxlint` et `vitest run`, dans deux
  configurations d'environnement (voir § 10).

### 2.2 Ce qui n'a **pas** pu être vérifié (et pourquoi)

| Non vérifié | Raison |
|---|---|
| Démarrage réel de l'API (`npm run start:prod`) | Interdit en local par la RÈGLE 3 (nécessite une base) |
| Tests e2e `*.e2e-spec.ts` | Exclus par la configuration ; nécessitent une base joignable |
| Comportement réel de Railway (migrations au boot, healthcheck) | Nécessite le déploiement ; analyse sur `railway.json` et le code |
| Exploitabilité effective des CVE | Pas de test d'intrusion ; `npm audit` seul, pas de validation en conditions réelles |
| Volumétrie réelle en production | Aucun accès à la base ; les constats de performance sont **structurels** (formes de requêtes), pas mesurés |
| Rotation / contenu réel des secrets Railway | Hors dépôt, non accessible |
| Efficacité du relais payout VPS | Le relais n'est pas dans ce dépôt ; seul le client est audité |

Les chiffres de lignes sont des **lignes de fichiers** (comptage
`Measure-Object -Line`), pas des lignes de code logique.

---

## 3. Synthèse

### 3.1 Chiffres clés

| Mesure | Valeur |
|---|---|
| Fichiers TS dans `src/` | **288** (dont **50 générés** par Prisma) |
| Fichiers TS hors `src/generated/` | **238** |
| Lignes TS totales | **142 918** (dont **42 330** hors générés) |
| Contrôleurs | **27** |
| Routes HTTP | **184** |
| Fichiers de test exécutés | **81** (`80` dans `src/` + `test/web-push-import.spec.ts`) |
| Lignes de tests dans `src/` | **16 562** |
| Fichiers e2e **non exécutés** | **5** (`test/*.e2e-spec.ts`) |
| Migrations Prisma | **51** |
| DTO | **36** |
| `any` explicite (hors tests) | **0** |
| `console.log/info/debug` (hors tests) | **0** |
| Vulnérabilités `npm audit` (prod) | **12** : 1 critique, 9 hautes, 2 modérées |
| Vulnérabilités `npm audit` (total) | **19** : 1 critique, 11 hautes, 7 modérées |

### 3.2 Verdict

Le socle est **nettement au-dessus de la moyenne** d'un projet de cette taille :
ledger financier en partie double avec verrous consultatifs PostgreSQL,
idempotence par `reference` unique, holds atomiques, transitions de statut
gardées, contrôle d'accès par objet (`requireAccess`), masquage 404 au lieu de
403 pour ne pas révéler l'existence d'une mission, secrets jamais journalisés
dans les flux d'e-mail et de brouillon, et **zéro `any`** dans le code de
production.

Les problèmes ne sont pas dans la logique métier : ils sont dans
**l'enveloppe** — ce qui se passe avant que la logique métier ne soit atteinte.

1. **La vérification d'e-mail n'est pas appliquée par l'API.** Le cookie posé
   à l'inscription donne un accès complet, alors que seule la route `login`
   refuse un client non vérifié.
2. **Aucune limitation de tentatives sur `POST /auth/login`.** Le seul
   rate-limit en base est celui de `forgot-password`.
3. **Le rate-limit par IP est structurellement faux derrière Railway** :
   `trust proxy` n'est pas activé, donc `req.ip` est l'IP de l'edge, et la
   limite « 10/heure/IP » de `forgot-password` devient **globale à toute la
   plateforme**.
4. **12 vulnérabilités de production**, dont `multer` (6 avis DoS, dont un
   contournement de limite de taille), `body-parser`, `nodemailer` (23 avis,
   dépendance directe **inutilisée**), et `proxy-addr` en critique.

---

## 4. Constats — Bloquants (P0)

### 4.1 P0-1 — La vérification d'e-mail est contournable : le cookie d'inscription vaut accès complet

**Où** : `src/auth/auth.controller.ts` (route `POST /api/auth/register`),
`src/auth/auth.service.ts` (`login`, `verifyToken`), `src/auth/jwt-auth.guard.ts`.

**Fait vérifié** :

- `register` pose **systématiquement** le cookie de session :
  `this.authService.setAuthCookie(res, this.authService.signToken(user))`,
  y compris pour un `CLIENT` dont `emailVerified === false`.
- `login` refuse bien un client non vérifié
  (`if (found.role === 'CLIENT' && !found.emailVerified) throw Unauthorized`).
- **Mais `verifyToken` ne vérifie jamais `emailVerified`** : il contrôle la
  signature, l'existence du compte, `isActive` et `tokenVersion`, rien d'autre.
- `JwtAuthGuard` et `RolesGuard` ne vérifient pas non plus `emailVerified`.

**Conséquence** : un visiteur peut s'inscrire avec **n'importe quelle adresse
e-mail qu'il ne possède pas**, recevoir immédiatement un cookie valide 7 jours,
et appeler **toutes** les routes `CLIENT` : création de demandes (donc
déclenchement du dispatch, des e-mails et des push vers les techniciens),
upload de médias, intentions de recharge SasPay, consultation de finances,
litiges, avis. Le commentaire du contrôleur affirme que « la vérification
d'e-mail reste obligatoire pour accéder au dashboard » : c'est vrai **côté
frontend** (`guard-decision.ts`), faux **côté API**. Un client direct de l'API
ignore ce garde-fou.

Le contrôle `emailVerified` n'existe qu'à deux endroits du backend :
`login` et la route de vérification elle-même. Aucun guard, aucun décorateur,
aucun middleware ne l'applique globalement.

**Aggravant** : `POST /auth/register` n'a **aucune limitation** (voir P0-2),
donc la création massive de comptes non vérifiés est triviale, et chaque
compte peut créer des demandes qui partent en dispatch.

**Correctif attendu (non appliqué)** : soit refuser le cookie à l'inscription
et ne le poser qu'après vérification, soit ajouter un guard
« e-mail vérifié » sur les routes `CLIENT` (avec une liste blanche explicite :
`/auth/me`, `/auth/resend-verification`, `/auth/verify-email`). La seconde
option est la seule qui protège réellement l'API.

---

### 4.2 P0-2 — `POST /api/auth/login` sans aucune limitation de tentatives

**Où** : `src/auth/auth.controller.ts` (route `login`),
`src/auth/auth.service.ts` (`login`).

**Fait vérifié** :

- La route `login` n'a **ni guard de throttle, ni compteur d'échecs, ni
  délai progressif**.
- Aucun `@nestjs/throttler` dans les dépendances ; aucun `APP_GUARD` global.
- Le seul rate-limit persistant du dépôt est `PasswordResetAttempt`
  (3/h/e-mail, 10/h/IP) et il ne couvre **que** `forgot-password`.
- `verifyPassword` utilise `scrypt` (N=16384, r=8, p=1) et
  `timingSafeEqual` : le hachage est bon, mais il ne protège pas contre
  l'essai massif de mots de passe, seulement contre la récupération hors ligne.
- `login` renvoie le même message pour « e-mail inconnu » et « mot de passe
  faux » — bon point, mais qui n'empêche pas le bourrage d'identifiants.

**Conséquence** : bourrage d'identifiants illimité. Un attaquant peut tester
des listes de mots de passe à la vitesse du serveur, sur tous les comptes,
sans jamais être ralenti ni bloqué. Le coût `scrypt` (≈ 16 Mio, ~50–100 ms)
limite le débit mais ne l'annule pas, et permet aussi un **déni de service
CPU** par saturation de requêtes `login`.

**Aggravant** : `register`, `resend-verification` et `reset-password` ne sont
pas non plus limités (voir P1-5).

---

### 4.3 P0-3 — Derrière Railway, le rate-limit par IP est global (et falsifiable)

**Où** : `src/main.ts` (aucun `trust proxy`), `src/auth/auth.controller.ts`
(`forgotPassword` lit `req.ip`), `src/demandes/demande-draft-throttle.guard.ts`,
`src/tracking/tracking-throttle.guard.ts` (lisent `x-forwarded-for`).

**Fait vérifié** :

- `main.ts` n'appelle **jamais** `app.set('trust proxy', ...)` ni
  `app.enable('trust proxy')`. Recherche exhaustive : **0 occurrence** de
  `trust proxy` dans tout `src/`.
- Sans `trust proxy`, Express laisse `req.ip` à l'adresse de la socket, c'est-
  à-dire **l'IP de l'edge Railway**, identique pour tous les utilisateurs.
- `requestPasswordReset(email, ip)` compte donc
  `PasswordResetAttempt` sur cette IP unique : la limite
  `PASSWORD_RESET_MAX_PER_IP_PER_HOUR` (10/h) est atteinte par **les 10
  premières demandes de toute la plateforme**, tous utilisateurs confondus.
  Au-delà, **plus personne** ne peut réinitialiser son mot de passe pendant
  une heure. Un attaquant peut maintenir ce blocage indéfiniment avec 10
  requêtes par heure.
- Les deux guards maison, eux, lisent **directement**
  `x-forwarded-for` (première valeur de la liste) : cet en-tête est fourni par
  le client et **falsifiable** si le proxy ne le réécrit pas. Ils sont donc à
  la fois contournables (fausse IP) et partagés (vraie IP de proxy).

**Conséquence double** : denial of service sur la réinitialisation de mot de
passe pour tous les utilisateurs légitimes, et rate-limit contournable sur les
routes publiques de brouillon et de suivi.

**Correctif attendu** : `app.set('trust proxy', 1)` (ou le nombre de sauts
Railway/Cloudflare exact) **et** lecture de `req.ip` uniquement, jamais de
`x-forwarded-for` brut.

---

### 4.4 P0-4 — 12 vulnérabilités de production, dont 1 critique et 9 hautes

**Commande** : `npm audit --omit=dev` dans `backend/`.

| Sévérité | Paquet | Direct | Nature |
|---|---|---|---|
| **CRITIQUE** | `proxy-addr` | non | Usurpation d'IP via sous-réseau de confiance IPv4-mapped IPv6 |
| HAUTE | `multer` | non (via `@nestjs/platform-express`) | 6 avis : DoS par noms de champs, fuite de descripteurs, **contournement de la limite de taille via une `fileFilter` asynchrone**, écritures orphelines |
| HAUTE | `@nestjs/platform-express` | **oui** | Cumul des avis `multer` |
| HAUTE | `body-parser` | non | DoS par octets résiduels après un corps compressé |
| HAUTE | `nodemailer` | **oui** | 23 avis cumulés (injection de commandes SMTP, contournement `disableFileAccess`/`disableUrlAccess`, SSRF, DoS quadratiques, fuite d'identifiants SMTP inter-transports) |
| HAUTE | `prisma` | **oui** | Via `@prisma/config` et `mysql2` |
| HAUTE | `@prisma/config` | non | Via `deepmerge-ts` |
| HAUTE | `deepmerge-ts` | non | Épuisement de pile sur graphes récursifs |
| HAUTE | `mysql2` | non | Fuite d'identifiants en clair, bombe de décompression |
| HAUTE | `source-map-js` | non | DoS par sections indexées |
| MODÉRÉE | `fast-uri` | non | Confusion d'hôte / de schéma |
| MODÉRÉE | `qs` | non | `TypeError` non capturé (DoS) |

**Deux constats aggravants, vérifiés** :

1. **`nodemailer` est une dépendance directe de production mais n'est
   importé nulle part.** Recherche exhaustive de `nodemailer` dans `src/` :
   **0 occurrence**. Les e-mails passent par l'API HTTP Resend (`fetch` dans
   `email.service.ts`). Retirer `nodemailer` de `dependencies` supprime à lui
   seul **23 avis** et une surface d'attaque entière.
2. **`prisma` (le CLI) est dans `dependencies`, pas `devDependencies`.** Cela
   embarque le CLI et ses dépendances (`mysql2`, `deepmerge-ts`,
   `@prisma/config`) dans l'image de production, alors que le runtime
   n'utilise que `@prisma/client`. C'est la source de 4 des avis hautes.

**Nuance honnête sur `proxy-addr` (critique)** : l'exploitabilité dépend de
`trust proxy`. Comme il n'est pas activé, Express ne dérive pas `req.ip` de
`X-Forwarded-For` via `proxy-addr`. La CVE reste néanmoins présente dans
l'image de production, et le correctif de P0-3 (`trust proxy`) est exactement
la condition qui la rendrait exploitable. **Les deux sujets doivent être
traités ensemble.**

---

## 5. Constats — Majeurs (P1)

### 5.1 P1-1 — `FINANCIAL_MODE` lu de deux façons incompatibles : crédits de récompense dans le mauvais mode

**Où** : `src/financial/financial.service.ts` (`getMode`) et
`src/rewards/rewards.service.ts` (`currentMode`).

```ts
// financial.service.ts
const raw = this.config.get<string>('FINANCIAL_MODE') ?? 'SIMULATION';
const value = raw.trim().toUpperCase();          // tolère "real", " Real "
if (value === 'REAL') return 'REAL';

// rewards.service.ts
return process.env.FINANCIAL_MODE === 'REAL' ? 'REAL' : 'SIMULATION';
//                                         ^ comparaison stricte, sensible à la casse
```

**Conséquence** : avec `FINANCIAL_MODE=real` (minuscules) ou ` Real ` :

- le ledger, les holds et les retraits travaillent en **REAL** ;
- `RewardsService.claimCredits` écrit `CLIENT_REWARD_CREDIT` en
  **SIMULATION**, sans passer par `FinancialService.record()` (donc sans
  `ensureModeAllowed`).

Le client réclame ses crédits, la transaction est créée, mais elle est
**invisible dans son solde REAL** et n'apparaît pas dans la supervision
financière REAL. Inversement, un environnement `REAL` correctement saisi ne
pose pas de problème — le risque est une faute de casse dans les variables
Railway, silencieuse.

**Aggravant** : `claimCredits` écrit **directement** dans
`financialTransaction` au lieu d'appeler `financial.record()`, ce qui contourne
le garde-fou de mode et la fabrique d'écritures idempotentes du service
financier. C'est le seul endroit du ledger qui n'utilise pas le chemin commun.

---

### 5.2 P1-2 — Tout technicien peut lire les médias d'une mission non assignée, sans contrôle de zone, de catégorie ni de KYC

**Où** : `src/demandes/demande-media.service.ts` (`getMediaFileUrl`),
`src/technician/technician.service.ts` (`getDemandeDetail`).

Règle d'accès actuelle :

```ts
(actor.role === 'TECHNICIAN' &&
  demande.technicianId === null &&
  (demande.status === 'SUBMITTED' || demande.status === 'PENDING'))
```

**Ce qui manque** : aucune vérification que le technicien est **éligible** à
cette mission (ville, zone de couverture, catégorie, `kycStatus === 'VERIFIED'`,
`isAvailable`). N'importe quel compte `TECHNICIAN` authentifié, même non
vérifié KYC, même dans une autre ville, peut obtenir une URL signée 15 minutes
s'il connaît `demandeId` + `mediaId`.

Les identifiants sont des UUID, donc non énumérables — c'est la seule
protection, et elle est réelle. Mais elle repose sur le secret de l'URL, pas
sur une autorisation. Un `demandeId` fuité (lien partagé, capture d'écran,
journal, ancien technicien) suffit si le `mediaId` accompagne.

**Aggravant assumé** : `getDemandeDetail` renvoie explicitement `medias` au
technicien éligible **avant** acceptation. Ce choix est documenté, mais il
transforme la liste d'opportunités en galerie photo de pannes domestiques
visibles par tous les techniciens de la catégorie.

**Recommandation** : appliquer dans `getMediaFileUrl` exactement la même
éligibilité que `acceptDemande` (`isGeoEligible` + catégorie + KYC vérifié),
ou restreindre les médias au technicien assigné et n'exposer qu'un aperçu
flouté/nombre de pièces avant acceptation.

---

### 5.3 P1-3 — Les médias de demande et les documents KYC ne sont pas validés sur leur contenu binaire

**Où** : `src/demandes/demande-media.service.ts` (`uploadMedia`,
`uploadAudio`), `src/technician/technician.service.ts` (`submitKycDocument`).

- Le type est déduit **uniquement** de `file.mimetype`, fourni par le client.
- Les avatars, eux, vérifient la signature binaire (`isImageBuffer`), ce qui
  prouve que le besoin est connu dans le dépôt.
- Un fichier HTML/JS/SVG renommé et déclaré `image/png` est accepté, stocké
  dans le bucket privé, puis servi via une URL signée avec le
  `Content-Type` déclaré à l'upload.

**Risque** : stockage de contenu arbitraire (25 Mo × 5 par demande), et
dépendance au comportement de Supabase sur le `Content-Type`/`Content-
Disposition`. Le bucket privé et les URL signées limitent fortement l'impact
(pas d'XSS direct si le `Content-Type` est respecté), mais ce n'est pas une
garantie de bout en bout.

**Recommandation** : réutiliser `isImageBuffer` pour les images, et vérifier
les magic bytes MP4/WebM/OGG côté backend avant l'upload.

---

### 5.4 P1-4 — Upload en mémoire : 25 Mo × 5 fichiers, plus `rawBody` global

**Où** : `src/main.ts` (`rawBody: true`), `src/demandes/demande-media.service.ts`
(`DEMANDE_MEDIA_MAX_BYTES = 25 * 1024 * 1024`,
`DEMANDE_MEDIA_MAX_FILES = 5`), `src/collaboration/collaboration.controller.ts`
(audio 25 Mo), `src/technician/technician.controller.ts` (KYC).

- `FileInterceptor` de Nest utilise `multer` en **memoryStorage** : chaque
  fichier est entièrement en RAM avant l'appel à Supabase.
- Une demande peut porter **5 × 25 Mo = 125 Mo** de buffers simultanés pour un
  seul utilisateur.
- `rawBody: true` est activé **globalement** (nécessaire au webhook SasPay) :
  Nest conserve `req.rawBody` pour **toutes** les requêtes, ce qui double la
  mémoire du corps sur chaque appel.
- Aucune limite globale de corps JSON n'est déclarée (`app.use(json({limit}))`
  absent) : on dépend du défaut Express (100 ko), non documenté dans le code.

**Conséquence** : un compte authentifié (ou quelques-uns en parallèle) peut
faire monter la mémoire du conteneur jusqu'à l'OOM. Les avis `multer`
(contournement de limite de taille via `fileFilter` asynchrone, descripteurs
orphelins) rendent le bornage encore moins fiable.

**Recommandation** : streaming vers Supabase (ou disque temporaire), limite
par requête, `rawBody` restreint au webhook (vérification de route), et limite
JSON explicite.

---

### 5.5 P1-5 — `register` et `resend-verification` sans limitation : bombardement d'e-mails et création massive de comptes

**Où** : `src/auth/auth.controller.ts` (`register`, `resendVerification`),
`src/auth/auth.service.ts` (`resendVerification`).

- `POST /auth/register` : aucune limite. Chaque appel crée un compte, hache un
  mot de passe (`scrypt`, coût CPU), et envoie un e-mail Resend.
- `POST /auth/resend-verification` : aucune limite. Chaque appel **génère un
  nouveau token** et **envoie un e-mail** à l'adresse ciblée. Un attaquant peut
  bombarder la boîte d'un tiers (ou la sienne) et consommer le quota Resend.
- `POST /auth/reset-password` : aucune limite non plus (la limite est sur la
  *demande*, pas sur la consommation du token).

**Aggravant** : combiné à P0-1, `register` devient un générateur de comptes
utilisables immédiatement, chacun pouvant déclencher un dispatch.

---

### 5.6 P1-6 — Tokens de vérification et de réinitialisation stockés en clair, et token de reset exposé en `GET`

**Où** : `prisma/schema.prisma` (`User.emailVerificationToken`,
`User.passwordResetToken`), `src/auth/auth.controller.ts`
(`GET /auth/reset-password/:token/validate`).

- Les deux tokens sont stockés **en clair** en base. Une fuite de dump SQL
  (ou un accès en lecture à la base) permet de réinitialiser n'importe quel
  mot de passe non encore réinitialisé, et de vérifier n'importe quel e-mail.
- Les tokens sont comparés par `findFirst({ where: { passwordResetToken: token } })`
  — donc en clair, non hachés.
- `GET /auth/reset-password/:token/validate` place le token **dans l'URL** :
  il atterrit dans les journaux d'accès, les historiques de proxy et les
  en-têtes `Referer`.

**Nuance** : les tokens sont à usage unique, expirent (24 h pour la
vérification, TTL borné pour le reset) et sont supprimés après usage. Le
risque est donc limité à la fenêtre de validité, mais il est réel en cas de
compromission de la base.

**Recommandation** : stocker `sha256(token)` et comparer le hash ; passer la
validation de token en `POST`.

---

### 5.7 P1-7 — `JWT_SECRET` validé seulement « non vide » en production

**Où** : `src/config/env.validation.ts` (`validate`),
`src/auth/auth.service.ts` (constructeur).

```ts
if (validatedConfig.NODE_ENV === Environment.Production && !validatedConfig.JWT_SECRET) {
  throw new Error('JWT_SECRET is required in production.');
}
```

- Aucune longueur minimale, aucune entropie minimale.
- `JWT_SECRET=a` passe la validation et signe tous les JWT.
- Aucune rotation, aucun `kid`, aucune procédure de révocation globale autre
  que `tokenVersion` par utilisateur.

**Nuance** : hors production, une clé aléatoire de 32 octets est générée au
démarrage, ce qui est correct et documenté.

---

### 5.8 P1-8 — Dispatch : tous les techniciens actifs chargés en mémoire à chaque vague

**Où** : `src/dispatch/dispatch.service.ts` (`loadCandidates`, `runWave`).

`loadCandidates()` fait un `findMany` **sans `take`** sur tous les
`User` `role = TECHNICIAN`, `isActive`, `technicianProfile.isAvailable`,
avec leurs couvertures de zone, puis filtre et trie en mémoire.

**Conséquence** : le coût est linéaire en nombre de techniciens, **à chaque
vague et pour chaque demande**. À 1 000 techniciens, chaque création de
demande charge 1 000 profils + toutes leurs couvertures. Les e-mails partent
ensuite séquentiellement dans la même boucle.

**Nuance** : `selectCandidatesForWave` borne le nombre de destinataires par
vague, mais le **chargement** n'est pas borné.

---

### 5.9 P1-9 — Deux N+1 et une troncature silencieuse dans `technician.service.ts`

**Où** : `src/technician/technician.service.ts`
(`listMineHistory`, `listAvailable`).

**`listMineHistory`** (l. 1029) : `take: 200`, puis **pour chaque demande** :

```ts
const client = await this.prisma.user.findUnique(...);
const clientReputation = await this.reviews.getReputation(client.id); // agrégat SQL
```

Soit jusqu'à **400 requêtes** pour un seul écran d'historique.

**`listAvailable`** (l. 972) : `take: 200` **puis** filtrage géographique et
par catégorie **en mémoire**, puis `slice(0, 50)`.

Conséquence : si les 200 demandes les plus récentes sont hors zone/catégorie,
le technicien voit **moins de 50 opportunités alors que des missions éligibles
existent plus loin dans la table**. La borne `take` s'applique **avant** le
filtre métier, ce qui rend la liste incomplète de façon non déterministe.

**Recommandation** : pousser le filtre ville/zone/catégorie dans le `where`
Prisma (les index `@@index([cityId])`, `@@index([zoneId])`,
`@@index([status])` existent), et faire l'agrégat de réputation en une requête
groupée.

---

### 5.10 P1-10 — Aucun en-tête de sécurité, aucun garde-fou global de routes

**Où** : `src/main.ts`, `src/app.module.ts`.

- Pas de `helmet` : aucun `Strict-Transport-Security`,
  `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
  `Permissions-Policy`.
- Pas d'`APP_GUARD` global : la protection est **déclarative par contrôleur**.
  Une nouvelle route ajoutée sans `@UseGuards` est **publique par défaut**.
  C'est exactement ce qui s'est produit pour le catalogue (corrigé depuis,
  mais le mécanisme reste).
- Trois contrôleurs sont entièrement publics : `cities`, `health`,
  `webhooks/saspay` (signé HMAC) — plus `tracking` (throttlé) et les routes
  publiques de `auth` et `demandes/drafts`.
- `catalog-public.controller.ts` documente explicitement ce piège : « ajouter
  une nouvelle route GET ici sans y ajouter les guards la rendra PUBLIQUE par
  défaut ». Le risque est donc connu mais non structurellement neutralisé.

**Recommandation** : `APP_GUARD` global qui refuse par défaut, avec un
décorateur `@Public()` explicite — l'inverse de la convention actuelle.

---

### 5.11 P1-11 — L'agent backoffice envoie des données personnelles à un LLM tiers

**Où** : `src/backoffice-agent/backoffice-agent.tools.ts`,
`src/backoffice-agent/backoffice-agent.controller.ts`.

- L'agent est bien `ADMIN` uniquement, en lecture seule (aucun
  `create/update/delete/queryRaw`, vérifié par test statique), avec `select`
  explicites excluant les secrets.
- **Mais** `USER_SELECT` inclut `email` et `phone`, `searchMessages` renvoie
  le **contenu intégral des conversations client↔technicien**, `getDemande`
  renvoie `description`, `neighborhood`, devis, litiges.
- Ces données sont envoyées à **Groq** (`groq.client.ts`), prestataire
  externe, pour inférence.

**Conséquence** : des données personnelles (identité, téléphone, adresse de
quartier, messages privés, montants) quittent l'infrastructure Relio à chaque
question d'un admin. Aucune anonymisation, aucun consentement tracé, aucune
mention dans les politiques. C'est un sujet RGPD, pas un bug technique.

**Nuance** : l'agent est désactivé sans `GROQ_API_KEY`, et il est
strictement admin. Le risque est donc **organisationnel** (qui interroge,
quand, sur qui) plus que technique.

---

### 5.12 P1-12 — Retrait des fonds Relio : un seul admin, aucune double validation

**Où** : `src/financial/financial.service.ts` (`withdrawRelioFunds`),
`src/financial/finance-admin.controller.ts`.

Le code est **techniquement solide** : montant entier > 0, verrou consultatif
`pg_advisory_xact_lock('relio_withdrawal')`, contrôle du disponible sous
verrou, référence unique, écriture ledger atomique, historique immuable.

**Mais** : un seul `ADMIN` authentifié suffit pour retirer de l'argent réel.
Pas de 2FA, pas de seconde validation, pas de plafond, pas de délai de
refroidissement, pas d'alerte. Un cookie de session admin volé (ou un compte
admin compromis) suffit à vider le disponible.

**Nuance** : le mode par défaut est `SIMULATION` ; le risque ne devient réel
qu'avec `FINANCIAL_MODE=REAL`. C'est précisément le mode de production visé.

---

## 6. Constats — Mineurs (P2)

### 6.1 `saspay-webhook.controller.ts` : caractères `+` parasites dans le commentaire

Lignes 14 à 17, le commentaire contient des `+` en début de ligne (résidu de
diff mal nettoyé) :

```
+ *  - CHANTIER ERREURS P1 : un body non-JSON (signature valide mais contenu
+ *    malformé) → 400 `Corps webhook non-JSON` ...
```

Sans effet fonctionnel (c'est un commentaire), mais c'est le seul fichier du
dépôt dans cet état et il porte sur le composant le plus sensible (webhook
paiement).

### 6.2 Deux migrations partagent le même timestamp

`prisma/migrations/` contient **deux** dossiers préfixés `20260929010000` :

- `20260929010000_add_zone_and_city_refs`
- `20260929010000_demande_multimedia`

Prisma applique les migrations par ordre **lexicographique** du nom complet.
L'ordre actuel tient parce que `add_...` < `demande_...`, mais il est
**implicite** : renommer l'un des deux, ou en insérer un troisième, peut
inverser l'ordre et casser un déploiement neuf. Les 49 autres timestamps sont
uniques.

### 6.3 Une migration détruit une table de données

`20261016020000_refonte_rewards_ltv/migration.sql` :

```sql
DROP TABLE IF EXISTS "ClientRewardProgress";
```

La refonte du programme de récompenses (marge LTV) supprime l'ancienne table
de progression. C'est **assumé et documenté** par le chantier, mais c'est
irréversible : un `prisma migrate deploy` sur une base contenant des données
de récompenses historiques les perd définitivement, sans sauvegarde.

Les deux autres opérations destructrices sont des `DROP COLUMN` sur
`Pricing` (`technicianPrice`, `customerPrice`), cohérentes avec la règle
métier (aucun prix côté technicien).

### 6.4 `scripts/migrate-recover.js` : `$executeRawUnsafe` et `execSync` dans le startCommand

- Le script utilise `prisma.$executeRawUnsafe(...)` avec une **chaîne SQL
  littérale** (pas d'interpolation de données utilisateur) : pas d'injection,
  mais l'API `Unsafe` est évitable.
- Il appelle `execSync('npx prisma migrate resolve --applied ...')` avec
  `cwd: process.cwd()`.
- Il est exécuté **à chaque démarrage** par `railway.json` :
  `npm run db:migrate:recover && npm run db:migrate:deploy && npm run start:prod`.
- Si le seed échoue, `process.exit(1)` fait échouer le déploiement — ce qui est
  le comportement voulu, mais rend le boot dépendant d'un script codé pour
  **une seule migration nommée en dur** (`20261009010000_equipment_families`).

**Conséquence** : dette opérationnelle. Ce script ne sert plus à rien une fois
la migration récupérée, mais reste dans le chemin critique de chaque
déploiement.

### 6.5 `prisma.config.ts` : fallback de connexion localhost

```ts
url: process.env.DATABASE_URL ?? 'postgresql://postgres:postgres@localhost:5432/repairdom'
```

Le commentaire dit « fallback uniquement pour `prisma generate` ». Mais
`prisma migrate deploy` lit la même configuration : lancé sans
`DATABASE_URL`, il viserait **localhost** et échouerait avec un message
trompeur. En production la variable est toujours injectée par Railway, donc
le risque est faible — mais le fallback devrait être limité à `generate`.

### 6.6 `CORS_ORIGINS` vide hors production = toutes origines avec credentials

`main.ts` :

```ts
origin: corsOrigins?.length ? corsOrigins : true,
credentials: true,
```

Hors production, toute origine est reflétée **avec** `credentials: true`.
C'est acceptable en développement local, mais si un environnement de
préproduction est lancé sans `CORS_ORIGINS` et avec `NODE_ENV` ≠ `production`,
n'importe quel site peut appeler l'API avec le cookie de l'utilisateur.

### 6.7 `process.env` lu directement dans deux services

- `src/rewards/rewards.service.ts` (`currentMode`) — voir P1-1.
- `src/admin/catalog.controller.ts` (`if (process.env.NODE_ENV === 'production')`).

Le reste du dépôt passe par `ConfigService`. Ces deux accès directs
contournent la validation centralisée et rendent les tests dépendants de
l'environnement du processus.

### 6.8 Le contrôle de réputation d'un technicien est ouvert à tout compte authentifié

`src/reviews/reviews.service.ts` (`getTechnicianReputation`) : la méthode
ignore complètement `_user` et renvoie l'agrégat. Le commentaire l'assume
(« publique dans le contexte authentifié »). Un compte `CLIENT` quelconque
peut donc interroger la réputation de n'importe quel technicien — ce qui est
probablement voulu, mais **aucune limite de débit** ne borne l'énumération.

### 6.9 Troncatures silencieuses sans pagination

| Endpoint | Limite | Pagination |
|---|---|---|
| `GET /demandes` (actives) | `take: 100` | aucune |
| `GET /demandes/my/history` | `take: 100` | aucune |
| `GET /technician/my-demandes/history` | `take: 200` | aucune |
| `GET /notifications/mine` | `take: 50` | aucune |
| `GET /admin/finances/...` | filtres, pas de pagination | — |
| `listKycFolders` (admin) | **aucun `take`** | — |
| `listRelioWithdrawals` | `take: 200` | aucune |

Un utilisateur au-delà de la borne ne voit **jamais** ses éléments les plus
anciens, sans indication. Seuls les litiges admin sont réellement paginés
(`page`/`limit`, bornés à 50).

### 6.10 `/api/health/db` et `/api/health/migrations` sont publics

`src/health/health.controller.ts` expose sans authentification :

- la liste des colonnes attendues sur `User` et leur présence ;
- **le nom et le statut de chacune des 51 migrations**.

C'est un choix documenté (diagnostic Railway sans tunnel Postgres). La fuite
est limitée à de la métadonnée d'infrastructure, mais elle révèle la version
exacte du schéma, donc les vulnérabilités potentielles d'une migration non
appliquée. Le commentaire du fichier prévoit de les retirer « si le produit
installe un référentiel utilisateur » — la condition est floue.

### 6.11 5 fichiers de tests e2e ne sont jamais exécutés

`vitest.config.ts` inclut `**/*.spec.ts`. Les fichiers
`test/*.e2e-spec.ts` (`admin.module`, `app`, `auth-demandes`,
`push.module`, `realtime.module`) ne correspondent pas au motif et ne sont
lancés que par `npm run test:e2e`, qui **n'est appelé par aucun script de CI**.
Il n'y a d'ailleurs **aucun fichier CI** dans le dépôt (`.github/` absent à la
racine, dans `backend/` et dans `frontend/`).

### 6.12 Un test « anti-401 » passe alors que la base est injoignable

`src/admin/catalog-public-access.spec.ts` vérifie l'**absence de 401** sur les
routes publiques du catalogue. Exécuté seul avec une `DATABASE_URL` factice,
il **passe (19/19)** tout en laissant Prisma journaliser des `P1001`
(`DatabaseNotReachable`) : le contrôleur renvoie un succès vide, et
l'assertion « pas de 401 » ne fait pas la différence entre « catalogue servi »
et « catalogue en échec ».

C'est exactement le type de test vert à vide que la RÈGLE 3 du projet
interdit. Il prouve la frontière d'authentification, pas le fonctionnement.

### 6.13 Deux tests explicitement `it.skip`

`src/demandes/demande-multimedia.spec.ts:190` et `:197` :

- « ni texte ni média → 400 explicite (état inatteignable via HTTP :
  description obligatoire) »
- « média seul → SUCCESS, description null (état inatteignable via HTTP) »

Les deux états sont documentés comme inatteignables via HTTP. Le `skip` est
donc honnête, mais il laisse deux branches du service non couvertes.

### 6.14 Les logs contiennent des adresses e-mail en clair

`src/auth/email.service.ts` journalise `to` dans chaque succès et chaque
échec (`E-mail de vérification envoyé à ${to}.`,
`Échec d'envoi Resend pour ${to}`). Les tokens ne sont **jamais** journalisés
(bon point, explicitement codé), mais l'adresse e-mail est une donnée
personnelle qui se retrouve dans les logs Railway.

### 6.15 `rawBody` et `ValidationPipe` : `transform` sans conversion implicite

`main.ts` active `transform: true` mais pas `enableImplicitConversion`. Les
DTO de requête qui déclarent `@IsNumber()` sur un `@Query()` reçoivent donc
une **chaîne** : la validation échoue ou passe selon le type attendu. Les
contrôleurs qui parsent `Number(query.page)` à la main (admin, litiges)
contournent le problème, mais le comportement n'est pas homogène.

### 6.16 `getReputation` et `getClientReputation` : deux politiques différentes

`getTechnicianReputation` est ouverte à tout authentifié ;
`getClientReputation` exige d'être le client lui-même ou son technicien. La
dissymétrie est probablement voulue (le technicien est un professionnel
public, le client non), mais elle n'est pas écrite comme une règle unique et
testable — elle vit dans deux méthodes voisines.

---

## 7. Ce qui est solide — vérifié, pas supposé

Cette section existe parce qu'un audit qui ne relève que les défauts est
inutilisable. Les points suivants ont été **lus dans le code**, pas supposés.

### 7.1 Ledger financier

- Écritures **immuables** (`status: VALIDATED` à la création), jamais mises à
  jour.
- **Idempotence par `reference` UNIQUE** : `record()` vérifie l'existence puis
  rattrape `P2002` en relisant l'écriture existante.
- Montants **entiers XAF**, arithmétique entière, jamais de flottant
  (`fee-calculator.ts`, `financial-fees.ts`, `saspay-fees.ts`).
- Barème unique et documenté : `commission = 500 + arrondi(4 % × devis)`,
  transport 2 000 XAF pass-through, collecte SasPay 4,5 % ajoutés au client,
  payout 3,5 % déduits.
- **Verrous consultatifs PostgreSQL** par utilisateur
  (`pg_advisory_xact_lock(hashtext('user_funds:<id>'))`) et pour les retraits
  (`'relio_withdrawal'`) : lecture du disponible + réservation/débit
  sérialisées, anti-TOCTOU.
- **Holds** : `holdClientAtAcceptance` crée un `FundsHold ACTIVE` dans la même
  transaction que l'acceptation du devis ; le débit définitif n'a lieu qu'à la
  confirmation (`settleMissionAtConfirmation`). Jamais de mission `ACCEPTED`
  sans hold, jamais de hold sans acceptation.
- Solde **toujours calculé** (`ledger − holds ACTIVE`), jamais stocké.
- Litige `RESOLVED` → libération du hold sans écriture, confirmation
  définitivement bloquée (`isConfirmationBlocked`), ce qui évite un débit sans
  réservation.
- Webhook SasPay : HMAC-SHA256 sur `${timestamp}.${rawBody}` **octets
  exacts**, tolérance 300 s, `timingSafeEqual`, JSON parsé **après** la
  signature, idempotent, 200 rapide.

### 7.2 Authentification

- `scrypt` (N=16384, r=8, p=1, keylen 64) avec sel aléatoire 16 octets et
  format versionné `scrypt$N$r$p$salt$hash`.
- Comparaison en temps constant (`timingSafeEqual`).
- Cookie `httpOnly`, `secure` en production, `sameSite: 'none'` en production
  (nécessaire au cross-site Vercel/Railway), `maxAge` 7 jours.
- `tokenVersion` incrémenté à la réinitialisation de mot de passe : toutes les
  sessions existantes sont invalidées.
- Compte désactivé (`isActive: false`) : connexion refusée **et** sessions
  existantes révoquées dans `verifyToken`.
- `forgot-password` répond toujours `{ ok: true }` (anti-énumération) et
  journalise sans le token.
- Message identique pour e-mail inconnu et mot de passe faux.

### 7.3 Contrôle d'accès par objet

- `CollaborationService.requireAccess` : client propriétaire ou technicien
  assigné, sinon **404** (pas 403) — l'existence de la mission n'est pas
  révélée.
- `RealtimeController.streamMission` applique exactement la même règle et le
  documente.
- `DemandesService.findForClient` : `where: { id, clientId }` — un client ne
  peut pas lire la demande d'un autre.
- `DisputesService.openDispute` : vérifie `demande.clientId === clientId`,
  statut `COMPLETED`, technicien assigné, unicité.
- `ReviewsService.createReview` : auteur issu du JWT, cible déduite de la
  mission, auto-évaluation refusée, `@@unique([demandeId, authorId])`.
- KYC : `storagePath` jamais renvoyé au client, consultation uniquement par
  URL signée 300 s, contrôle de propriété.
- Médias : URLs signées 15 min, jamais d'URL publique persistée, 404 sans
  fuite d'existence.

### 7.4 Transactions et concurrence

- Acceptation de mission : `updateMany` gardé
  (`status IN (SUBMITTED, PENDING)`, `technicianId: null`) — deux techniciens
  ne peuvent pas accepter la même mission ; le second reçoit un `409` explicite.
- Acceptation de devis : `updateMany` gardé sur `status: 'PENDING'` +
  rattrapage `P2002` sur l'index partiel « un seul ACCEPTED par mission ».
- Récompenses : `updateMany` avec `where` portant les valeurs lues
  (`creditsEarned`, `creditsClaimed`) — réclamation concurrente impossible.
- Parrainage : `referredId` UNIQUE, auto-parrainage refusé, limite de filleuls
  avec ligne `EXPIRED` pour empêcher le contournement.
- Dispatch : `DispatchWave` avec `@@unique([demandeId, wave])`, relecture
  systématique avant envoi, idempotent et réentrant.

### 7.5 Qualité statique

- **0 `any` explicite** hors tests, alors même qu'`oxlint` désactive
  `@typescript-eslint/no-explicit-any`.
- **0 `console.*`** hors tests : tout passe par `Logger`.
- `tsc --noEmit -p tsconfig.build.json` : **exit 0**.
- `oxlint src/ test/` : **exit 0, 0 warning**.
- Validation globale `whitelist: true` + `forbidNonWhitelisted: true` :
  aucun champ inconnu n'atteint les services.
- 36 DTO, tous validés par `class-validator` (le seul cas sans décorateurs
  propres, `CreateDemandeDraftDto`, hérite de `BaseDemandeDto`).
- Filtre d'exception global : jamais de stack trace renvoyée au client,
  champs machine (`code`, `required`, `available`, `currency`) transmis
  proprement, `VALIDATION_ERROR` pour les erreurs `ValidationPipe`.
- Secrets : les tokens de brouillon, de vérification et de reset ne sont
  **jamais** journalisés (codé explicitement, commentaires à l'appui).

---

## 8. Fichiers

### 8.1 Créés

| Fichier | Nature |
|---|---|
| `RAPPORT-AUDIT-BACKEND.md` (racine) | Ce rapport |

### 8.2 Modifiés

**Aucun.** L'audit est en lecture seule.

### 8.3 Supprimés

**Aucun.**

---

## 9. Migrations

**Aucune migration créée, modifiée ou appliquée.**

État audité : **51 migrations**, du `20260908010000_add_users_demandes` au
`20261017030000_add_referral_notification_types`.

Points relevés :

| Migration | Constat |
|---|---|
| `20260929010000_add_zone_and_city_refs` et `20260929010000_demande_multimedia` | **Timestamp identique** — ordre lexicographique implicite |
| `20261016020000_refonte_rewards_ltv` | `DROP TABLE IF EXISTS "ClientRewardProgress"` — destruction irréversible de données |
| `20260910020000_remove_technician_customer_price` | `DROP COLUMN "technicianPrice"`, `DROP COLUMN "customerPrice"` — cohérent avec la règle métier |
| Toutes les autres | Aucun `DROP TABLE`, `TRUNCATE` ou `DELETE FROM` |

Le schéma Prisma (1 669 lignes) porte des index ciblés (`clientId`,
`technicianId`, `status`, `zoneId`, `domainId`, `brandId`, `modelId`,
`problemId`, `equipmentFamily`) et des `@@unique` d'idempotence
(`FinancialTransaction.reference`, `FundsHold.reference`,
`DemandeDispute.demandeId`, `Review.[demandeId, authorId]`,
`Referral.referredId`, `DispatchWave.[demandeId, wave]`,
`User.referralCode`).

---

## 10. Vérifications exécutées

### 10.1 Compilation et lint

```text
npx tsc --noEmit -p tsconfig.build.json   → exit 0
npm run lint (oxlint src/ test/)          → exit 0, 0 warning
```

Aucune erreur, aucun avertissement.

### 10.2 Tests — deux exécutions, résultat honnête

**Sans variables d'environnement** (état par défaut de la machine, `.env`
byte-identique à `.env.example`, donc secrets vides) :

```text
Test Files  5 failed | 76 passed (81)
Tests       1019 passed | 55 skipped (1074)
```

Les 5 échecs sont **exclusivement** des erreurs de validation d'environnement
(`ConfigModule.forRoot` → `validate()` lève sur `JWT_SECRET`,
`FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM` vides) dans les specs
d'assemblage NestJS :

- `src/admin/catalog-public-access.spec.ts`
- `src/auth/verification-reminder.module.spec.ts`
- `src/demandes/demande-draft.module.spec.ts`
- `src/referrals/referrals.module.spec.ts`
- `src/rewards/rewards.module.spec.ts`

**Avec les variables injectées** (`NODE_ENV=test`, `JWT_SECRET`,
`FRONTEND_URL`, `RESEND_API_KEY`, `EMAIL_FROM`, `DATABASE_URL` factice) :

```text
Test Files  81 passed (81)
Tests       1072 passed | 2 skipped (1074)
Duration    69.42s
exit        0
```

**Preuve de non-régression** : les mêmes 5 fichiers, relancés isolément avec
les variables, passent (38/38 puis 12/12 puis 19/19). Les échecs sont donc
**100 % environnementaux**, pas des régressions de code. Aucun test n'a été
modifié, aucun `skip` ajouté, aucun `.only`.

**Ce que cette preuve ne dit pas** : elle ne prouve pas que la suite couvre
les chemins critiques. Elle prouve seulement que le code actuel passe ses
propres tests. Les § 6.11 à 6.13 documentent les angles morts.

### 10.3 Audit des dépendances

```text
npm audit --omit=dev   → 12 vulnérabilités : 1 critique, 9 hautes, 2 modérées
npm audit              → 19 vulnérabilités : 1 critique, 11 hautes, 7 modérées
```

Détail en § 4.4. Toutes les vulnérabilités « prod » ont un correctif
disponible (`fixAvailable: true` ou montée `prisma@7.10.1`).

### 10.4 Ce qui n'a pas été exécuté

- `npm run test:e2e` : nécessite une base joignable, interdit en local.
- `npm run start:prod` : idem.
- `prisma migrate status` : nécessite la base de production.
- Test de charge, test d'intrusion, analyse SAST/DAST.

---

## 11. Plan de correction proposé (non appliqué)

Par ordre de gravité décroissante. Aucune de ces actions n'a été réalisée.

### Immédiat (P0)

1. **Appliquer la vérification d'e-mail côté API** : guard global
   `EmailVerifiedGuard` sur les routes `CLIENT`, avec liste blanche
   (`/auth/me`, `/auth/verify-email`, `/auth/resend-verification`,
   `/auth/logout`). Alternative plus simple : ne pas poser le cookie à
   l'inscription.
2. **Limiter `POST /auth/login`** : compteur par e-mail + par IP, délai
   progressif, verrouillage temporaire. Réutiliser le modèle
   `PasswordResetAttempt` ou ajouter `@nestjs/throttler` avec un stockage
   partagé.
3. **Activer `trust proxy`** avec le nombre de sauts exact, puis remplacer
   toute lecture de `x-forwarded-for` par `req.ip` dans les deux guards maison.
4. **Traiter `npm audit`** : retirer `nodemailer` (inutilisé), déplacer
   `prisma` en `devDependencies`, monter `prisma`/`@prisma/client` à `7.10.1`,
   mettre à jour `@nestjs/platform-express`/`multer`, `body-parser`,
   `proxy-addr`.

### Court terme (P1)

5. **Unifier `FINANCIAL_MODE`** : injecter `ConfigService` dans
   `RewardsService`, réutiliser `FinancialService.getMode()`, et faire passer
   `claimCredits` par `financial.record()`.
6. **Restreindre les médias non assignés** à l'éligibilité réelle
   (zone/catégorie/KYC) ou au technicien assigné.
7. **Valider les magic bytes** des médias et documents KYC.
8. **Passer les uploads en streaming** ou borner la mémoire par requête ;
   limiter `rawBody` au webhook ; déclarer une limite JSON explicite.
9. **Limiter `register` et `resend-verification`.**
10. **Hacher les tokens** de vérification/réinitialisation en base ; passer
    `validateResetToken` en `POST`.
11. **Exiger une longueur minimale de `JWT_SECRET`** (≥ 32 caractères) en
    production.
12. **Borner `loadCandidates`** et pousser les filtres géographiques dans
    Prisma.
13. **Corriger les N+1** de `listMineHistory` et le `take` avant filtre de
    `listAvailable`.
14. **Ajouter `helmet`** et un `APP_GUARD` global « deny by default ».
15. **Encadrer l'agent backoffice** : anonymiser e-mail/téléphone/contenu
    avant l'appel Groq, ou restreindre les outils.

### Moyen terme (P2)

16. Nettoyer le commentaire `+` de `saspay-webhook.controller.ts`.
17. Fusionner ou renuméroter les deux migrations `20260929010000_*`.
18. Retirer `migrate-recover.js` du `startCommand` une fois la migration
    récupérée.
19. Restreindre le fallback `DATABASE_URL` de `prisma.config.ts`.
20. Ajouter une pagination par curseur aux listes tronquées.
21. Ajouter une CI (lint + `tsc` + `vitest`) et exécuter les e2e sur une base
    éphémère.
22. Rendre `catalog-public-access.spec.ts` non vacuant (asserter le contenu,
    pas seulement l'absence de 401).
23. Anonymiser les adresses e-mail dans les logs.

---

## 12. Points d'attention

- **Le mode financier de production n'est pas vérifiable depuis le dépôt.**
  Si `FINANCIAL_MODE=REAL` est actif, les constats P1-1 et P1-12 deviennent
  des sujets d'argent réel ; s'il est `SIMULATION`, ils restent latents.
  Cette information est dans Railway, pas dans le code.
- **La vérification d'e-mail n'est peut-être pas exploitée en pratique** si
  le frontend est le seul client et applique `guard-decision.ts`. Mais l'API
  est publique, et un client direct n'est pas empêché.
- **Les constats de performance sont structurels, pas mesurés.** Aucune
  volumétrie de production n'était accessible. `take: 200` peut être
  parfaitement suffisant aujourd'hui et devenir un bug à 5 000 techniciens.
- **Les CVE `multer`/`body-parser`/`nodemailer` sont des DoS**, pas des
  exécutions de code à distance, à l'exception des avis `nodemailer` sur
  `disableFileAccess`/`disableUrlAccess` — inexploitables ici puisque
  `nodemailer` n'est pas utilisé.
- **`proxy-addr` critique** : la CVE est présente mais la condition
  d'exploitation (`trust proxy`) est aujourd'hui absente. Le correctif de P0-3
  la rendrait exploitable s'il était appliqué sans mise à jour.
- **Le test `catalog-public-access.spec.ts` passe avec une base injoignable** :
  ne pas le compter comme une preuve de fonctionnement du catalogue.
- **`prisma` en dépendance de production** alourdit l'image et expose le CLI
  (et `mysql2`) inutilement. Ce n'est pas une faille en soi, mais une surface.
- **Les e2e ne tournent nulle part** : aucun CI, et `npm test` les exclut. Les
  cinq fichiers existent mais ne protègent rien aujourd'hui.
- **Les logs Railway contiennent des e-mails en clair** ; la politique de
  rétention Railway n'est pas connue.

---

## 13. Questions bloquantes

1. **`FINANCIAL_MODE` en production vaut-il `REAL` ?** Si oui, P1-1 (crédits de
   récompense dans le mauvais mode) et P1-12 (retrait admin sans double
   validation) sont à traiter avant toute nouvelle mise en production.
2. **La vérification d'e-mail doit-elle être une barrière API ou seulement
   UX ?** Le code actuel a choisi « UX seulement ». Confirmer ce choix, ou
   l'inverser (P0-1).
3. **Le dispatch doit-il accepter des demandes d'un compte non vérifié ?**
   Aujourd'hui oui, via le cookie d'inscription. C'est la porte d'entrée du
   spam de techniciens.
4. **`nodemailer` peut-il être retiré ?** Aucun import dans `src/` ; les
   e-mails passent par Resend en HTTP. Confirmer avant de le sortir de
   `dependencies`.
5. **Les endpoints `/api/health/db` et `/api/health/migrations` doivent-ils
   rester publics ?** Le code les assume pour le diagnostic Railway ; ils
   exposent la version du schéma.
6. **L'agent backoffice doit-il envoyer des messages privés à Groq ?** Si oui,
   une base légale et une politique de rétention doivent exister ; sinon, il
   faut anonymiser avant l'appel.

---

## 14. Push de ce rapport

```text
Dépôt   : https://github.com/digitalholding2026-ux/projet.git
Branche : main
Fichier : RAPPORT-AUDIT-BACKEND.md (racine uniquement)
```

Aucun fichier de `backend/` ni de `frontend/` n'est ajouté à l'index racine :
ce sont des dépôts imbriqués (gitlinks). Seul le rapport est commité.

---

*Fin du rapport. Audit réalisé le 2026-10-30 sur `backend@9851cfd`,
arbre de travail propre, sans modification de code.*
