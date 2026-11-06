# RAPPORT — Chantier M1-déploiement : sortie de la boucle de redémarrage Railway + seed du catalogue

## Statut

- **Date** : 06/11/2026
- **Dépôt concerné** : `Repairdom-backend`
  - `392deba` — `fix(db): order withdrawal DDL after the financial types and repair seed SQL`
  - `eb80e80` — `fix(deploy): run migrate deploy and catalog seed as a pre-deploy step`
  - poussés : `9d77e9c..eb80e80` sur `origin/main`, sans rejet ni rebase.
- **Frontend** : `c369c7d` (`feat(v2): wire wizard step 1-3 to new catalog API`) est **toujours local** (`ahead 1`). Poussée différée jusqu'à la validation du backend en environnement réel (RÈGLE 2).
- **Racine** : le présent rapport + `5be1716` (rapport M1) et `325d66f` (init machine) restent à pousser.
- **Base de données Railway** : 54 migrations appliquées, schéma à jour, catalogue seedé et re-seedé (preuves au § Preuves d'exécution).
- **Déploiement Railway** : **rouge.** `502` sur `/`, `/health`, `/api/health`, `/api/services` et `/api/services/FRP/categories`, avant comme après le push. Détail et diagnostic à § Déploiement.

## Diagnostic (exécuté avant toute correction)

Trois causes indépendantes, toutes lues dans les logs du redeploy fourni (`logs.1793958365146.log`) et dans la base réelle :

1. **La chaîne de démarrage commençait par le script de secours.**
   `railway.json` → `startCommand: "npm run db:migrate:recover && npm run db:migrate:deploy && npm run start:prod"`.
   Le script interroge `_prisma_migrations` dès la première ligne ; sur une base sans historique, PostgreSQL lève `42P01`, le script sort en `1`, `&&` coupe la chaîne, et `restartPolicyType: ON_FAILURE` rejoue jusqu'à `restartPolicyMaxRetries: 10`. Les logs montraient exactement ce motif : onze blocs `node scripts/migrate-recover.js`, et jamais un `migrate deploy` mené au bout.
   Le script lui-même se terminait par `process.exit(1)` y compris sur un cas normal (« migration not found » n'était pas le problème — c'était l'absence de la table).

2. **`20260922010000_admin_super_powers` était inapplicable depuis une base vide** : elle déclarait `ALTER TYPE "FinancialTransactionType" ADD VALUE 'RELIO_WITHDRAWAL'` et la table `RelioWithdrawal`, alors que les trois types financiers (`FinancialTransactionType`, `FinancialTransactionMode`, `FinancialTransactionStatus`) naissent dans `20260927010000_add_financial_ledger`, donc **après** elle. Erreur : `P3018` / `42704 — type "FinancialTransactionType" does not exist`. L'application partielle laissait la migration en échec dans l'historique → `P3009` sur tous les déploiements suivants.

3. **`20261009010000_equipment_families` contenait une apostrophe non échappée** : `('VENTILATION', 'Ventilateur / brasseur d'air', …)` → `P3018` / `42601 — syntax error at or near "air"`. Même mécanisme d'historique corrompu.

4. **Trouvé en route, hors logs Railway : le seed ne tournait pas.** `npm run db:seed:catalog` exécutait `node prisma/seed-catalog.ts`, qui importait `../src/generated/prisma/client.js`. Prisma 7 génère des fichiers **`.ts`** dont les imports internes s'écrivent en `.js` — spécificateur que Node ne réécrit pas vers `.ts` (vérifié : `ERR_MODULE_NOT_FOUND` sur `src/generated/prisma/enums.js` chargé depuis `client.ts`). Conséquence : `/api/services` aurait renvoyé `[]` même une fois l'application démarrée.

## Corrections

### Configuration de déploiement

- `Backend/railway.json`
  ```json
  "preDeployCommand": "npm run db:migrate:deploy && npm run db:seed:catalog",
  "startCommand": "npm run start:prod",
  ```
  Les migrations et le seed ne conditionnent plus le processus serveur : un échec de migration apparaît dans les logs de déploiement au lieu de relancer un conteneur déjà mort. `db:migrate:recover` reste au dépôt et reste invocabable à la main ; il a juste quitté le chemin de démarrage.

- `Backend/scripts/migrate-recover.js` — étape 0 : `SELECT to_regclass('public._prisma_migrations')`. Table absente = base neuve, message explicite, sortie `0`.

### Historique de migrations

- `Backend/prisma/migrations/20260922010000_admin_super_powers/migration.sql` — blocs financiers retirés ; il ne reste que `User.isActive` et la valeur `ADMIN_MESSAGE`, tous deux sous garde (`ADD COLUMN IF NOT EXISTS`, test `pg_enum`).
- `Backend/prisma/migrations/20260927020000_add_relio_withdrawal/migration.sql` — **nouveau**, placé après le moteur financier (`20260927010000`) et avant `20261001010000_saspay_foundations`, qui ajoute des colonnes à `RelioWithdrawal`. Contenu entièrement gardé (`pg_enum`, `IF NOT EXISTS`, `EXCEPTION WHEN duplicate_object`), donc rejouable.
- `Backend/prisma/migrations/20261009010000_equipment_families/migration.sql` — `d'air` → `d''air`.

Les deux fichiers modifiés étaient en échec dans l'historique et ont été **marqués rolled back puis réappliqués** (`prisma migrate resolve --rolled-back`), donc leur somme de contrôle enregistrée correspond désormais au contenu du dépôt. Une migration déjà appliquée normalement n'aurait pas dû être touchée.

### Seed

- `Backend/prisma/seed-catalog.ts` — le client Prisma est chargé depuis `dist/generated/prisma/client.js` (sortie de `npm run build`, déjà utilisée par `start:prod` et par `migrate-recover.js`), l'import des données passe en `./seed-catalog-data.ts`. Message d'erreur explicite si le build manque. Aucun fichier de `src/` n'a été modifié, aucun fichier ajouté à `src/`.

## Preuves d'exécution sur la base réelle

Commandes exécutées depuis ce poste, `DATABASE_URL` = l'URL publique du proxy Railway (jamais écrite dans un fichier du dépôt, seulement dans l'environnement du shell).

```
$ npm run db:migrate:status
54 migrations found in prisma/migrations
Database schema is up to date!

$ npx prisma migrate deploy        # après resolve --rolled-back sur les deux migrations cassées
All migrations have been successfully applied.

$ npx prisma migrate deploy        # seconde passe
No pending migrations to apply.

$ npm run db:seed:catalog          # première passe
Catalogue v2 seedé : 3 services, 9 catégories, 67 items.

$ npm run db:seed:catalog          # seconde passe (idempotence)
Catalogue v2 seedé : 3 services, 9 catégories, 67 items.
```

Comptages relus en SQL après chaque passe :

| Contrôle | Après seed n°1 | Après seed n°2 |
|---|---|---|
| `Service` | 3 | 3 |
| `CatalogCategory` | 9 | 9 |
| `CatalogItem` | **67** | **67** |
| doublons `(categoryId, code)` | 0 | 0 |
| doublons `Service.code` | 0 | 0 |
| items `TECNO` à `pricePublic = 3000` **et** `pricePro = 1500` | 59 | 59 |

Répartition par catégorie : `SYSTEME` = REINSTALL_OS, DRIVERS, SOFTWARE, DEBRIDAGE (1 item chacun, 5 000 / 2 500 / sur devis / sur devis) ; `SMARTPHONE` = SCREEN, BATTERY, CONNECTOR, OTHER (1 item chacun, tous « sur devis ») ; `FRP` = TECNO, **59** modèles.

**Écart avec le brief M1** : le brief annonçait « 55 Tecno + 8 autres », soit 63 items. La table `TECNO_BY_ANDROID` du seed produit 59 modèles (7 + 7 + 6 + 8 + 9 + 7 + 8 + 7), total réel **67**. Le chiffre 55 du brief ne correspond à aucune regroupement cohérent des versions Android ; le seed livré suit le barème, pas le compte annoncé.

Dérive résiduelle (`prisma migrate diff --from-config-datasource --to-schema prisma/schema.prisma`) — 7 écarts **cosmétiques et préexistants**, aucun bloquant :

- `AiWarning.pricingCheckId` : l'`onDelete` enregistré diffère du schéma ;
- `updatedAt` avec défaut en base sur `EquipmentFamily`, `RelioWithdrawal`, `TopupIntent`, `WithdrawalRequest` (Prisma gère `@updatedAt` sans défaut SQL) ;
- `ServiceCity.id` : défaut `gen_random_uuid()` côté base ;
- index `Problem(domainId, slug)` présent en base, absent du schéma.

## Vérifications (commit)

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit -p tsconfig.build.json` | **exit 0** |
| `npm run lint` (`oxlint src/ test/`) | **0 erreur / 0 warning**, 205 fichiers |
| `npx vitest run` | **58 fichiers, 740 tests passés, 2 skipped, 0 échec** |

Pas de comparaison `git stash` ici : les trois portes sont vertes dans l'absolu, il n'y a pas d'échec préexistant à distinguer d'une régression. Le spec du seed (`src/catalog/catalog.service.spec.ts`) tourne sur un client en mémoire, sans base.

## Déploiement

**Non vérifié — l'application ne répond toujours pas.** Après le push de `eb80e80`, sonde depuis ce poste (aucun CLI Railway, aucun token sur cette machine : je ne vois ni les logs de build, ni l'état du conteneur, ni le réglage du tableau de bord) :

| Sonde | Résultat |
|---|---|
| `GET /api/health` pendant ~9 min, 10 essais espacés de 30 s | **502 sur les 10 essais** (`curl` exit 22) |
| `GET /api/health` (reprise, après le délai) | **502** — `{"status":"error","code":502,"message":"Application failed to respond","request_id":"bJ5R9KRES-yKVurK9fVATg"}` |
| `GET /health` | **502** |
| `GET /` | **502** |
| `GET /api/services` | **502** (corps d'erreur du proxy, pas une réponse de l'API) |

Le 502 vient du bord Railway (`request_id` fourni par le proxy) : **aucun conteneur ne répond sur aucun chemin**, y compris avant ce chantier. Ce n'est donc pas le catalogue qui pose problème, c'est le processus serveur lui-même, ou l'absence de déploiement.

Trois explications possibles, dans l'ordre de probabilité, **aucune ne peut être tranchée sans les logs Railway** :

1. **Aucun nouveau déploiement n'a été déclenché.** `restartPolicyMaxRetries: 10` épuisé, le service reste éteint jusqu'à un prochain deploy. Si l'auto-deploy à chaque push GitHub n'est pas activé sur le service, `eb80e80` attend toujours son tour.
2. **Le `preDeployCommand` a échoué** (le déploiement est alors marqué en échec et le conteneur n'est jamais lancé). Deux causes candidates dans cet ordre :
   - le Node de l'image est antérieur à 22.18 et ne retire pas les types nativement → `node prisma/seed-catalog.ts` lèverait « Unknown file extension ".ts" » (`engines.node` indique `>=22`, pas une version précise) ;
   - `npm run db:migrate:deploy` ou le seed n'a pas accès à `DATABASE_URL` au moment du pre-deploy.
3. **La commande de démarrage saisie dans le tableau de bord prime sur `railway.json`** (DÉCISION 1, sans réponse) : dans ce cas la chaîne `recover && migrate deploy && start:prod` de l'ancienne image s'exécute encore. Avec les migrations désormais appliquées, elle devrait malgré tout atteindre `start:prod`, mais l'image en question ne contient pas la correction du seed.

À lire dans Railway → Deploys du dernier déploiement après `eb80e80` : (a) une ligne `pre-deploy command` et son code de sortie, (b) la version de Node choisie par Nixpacks, (c) la première ligne d'erreur s'il y en a une. Ces trois éléments déterminent laquelle des trois causes retenir.

## Variables d'environnement à synchroniser

Côté **Railway** (backend) — obligatoires en production :

| Variable | Valeur attendue | Contrôle dans le code |
|---|---|---|
| `NODE_ENV` | `production` | `env.validation.ts`, et `isProduction` pilote le cookie d'authentification |
| `DATABASE_URL` | fournie par Railway | `@IsNotEmpty`, **aucun repli localhost** au runtime |
| `PORT` | injecté par Railway | `IsNumberString`, défaut `3000` |
| `JWT_SECRET` | secret long | exigé en production, sinon le conteneur refuse de démarrer |
| `CORS_ORIGINS` | `https://www.relioo.space,https://relioo.space` (+ l'URL de preview Vercel si utile) | `main.ts` lève `CORS_ORIGINS is required in production` ; **sans ce réglage le navigateur bloque tous les appels** |
| `FRONTEND_URL` | `https://www.relioo.space` | liens des e-mails (vérification, réinitialisation) |

Facultatives mais nécessaires à des fonctions précises : `RESEND_API_KEY` / `EMAIL_FROM` (envoi d'e-mails), `SUPABASE_URL` + `SUPABASE_ANON_KEY` + `SUPABASE_SERVICE_ROLE_KEY` + `SUPABASE_BUCKET` (médias des demandes), `SASPAY_*` (payouts), `GROQ_*` (aide IA), `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_SUBJECT` (notifications push — la clé publique est relue par le frontend via `GET /api/push/vapid-public-key`, donc **aucune** variable `NEXT_PUBLIC_` correspondante n'est nécessaire côté Vercel), `FINANCIAL_MODE` (`SIMULATION` par défaut ; à ne passer en mode réel qu'avec les clés SasPay).

Côté **Vercel** (frontend) :

| Variable | Valeur attendue | Rôle |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `https://repairdom-backend-production.up.railway.app/api` | base de tous les appels (`site-config.ts`, `apiFetch`) |
| `NEXT_PUBLIC_APP_URL` | `https://www.relioo.space` | URLs absolues, partage, SEO |

Le cookie de session s'appelle `repairdom_token`, est posé par l'API avec `secure` quand `NODE_ENV=production` et `sameSite: 'none'` dans ce cas. Le contrat de synchronisation est donc : **HTTPS des deux côtés + `NODE_ENV=production` sur Railway + `CORS_ORIGINS` exactement l'origine du frontend + `credentials: true`** (déjà en dur dans `main.ts`). Si l'une des trois manque, l'authentification part en échec silencieuse (le cookie n'est jamais renvoyé au navigateur), pas en erreur visible.

## Points d'attention

1. **Le `railway.json` peut être écrasé par le réglage du tableau de bord.** La DÉCISION 1 (« y a-t-il une Start Command saisie dans Railway ? ») n'a pas reçu de réponse et je n'ai aucun accès au tableau de bord. Si une commande y est enregistrée, elle prime sur le dépôt et le seed ne sera pas exécuté en pre-deploy. À vérifier dans Railway → Settings → Deploy → **Build/Start Command** : rien ne doit y être saisi, les valeurs doivent venir de `railway.json`.
2. **Le seed dépend de `dist/`.** `npm run db:seed:catalog` exige que `npm run build` (`prisma generate && nest build`) ait tourné avant. C'est le cas dans une image Nixpacks (le script `build` existe, et `start:prod = node dist/main` fonctionne en production), mais en local il faut builder avant de seeder — message d'erreur explicite sinon.
3. **Node ≥ 22.18 pour le seed local** : le fichier d'entrée reste du TypeScript et repose sur la suppression native des types. Sans le drapeau, un Node 22.6–22.17 refuserait de le charger.
4. **Deux migrations de l'historique ont été réécrites** (voir § Corrections) : légitime uniquement parce qu'elles étaient en échec et ont été rollbackées puis rejouées. Un autre environnement qui les aurait appliquées **avant** la correction aura une somme de contrôle divergente ; le cas connu est cette base Railway, déjà traitée.
5. **Aucune commande destructrice n'a été exécutée** : pas de `migrate reset`, pas de `migrate dev`, pas de suppression de ligne. Le seed est un `upsert`.
6. **`prisma.config.ts` conserve un repli `localhost` pour `DATABASE_URL`**, mais uniquement pour la phase `generate` (le CLI a besoin d'une URL pour produire le client). Ce n'est pas le chemin d'exécution : `env.validation.ts` exige `DATABASE_URL` et `main.ts` échoue sans elle. La consigne « pas de repli localhost » est respectée là où elle compte.
7. **Le frontend M1 n'est pas poussé** : l'espace client consomme encore l'ancien chemin de diagnostic tant que `c369c7d` n'est pas en production.
8. **Non vérifié** : les logs de build Nixpacks (aucun CLI Railway sur ce poste), l'état réel du conteneur après `eb80e80`, et le comportement du wizard pas à pas dans un navigateur. La RÈGLE 3 interdit `next dev` / `npm run dev` ; les conséquences sont déduites du code et des sondeurs HTTP.

## Questions bloquantes

**Aucune** pour la clôture technique. Un seul point reste entre les mains du porteur de projet : le réglage « Start Command » du tableau de bord Railway (Point d'attention 1).
