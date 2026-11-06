# RAPPORT — Chantier M1 : catalogue Relio v2 (prestations, catégories, items, tarifs) + seed Tecno

## Statut

- **Date** : 06/11/2026
- **Dépôts concernés** : `Repairdom-backend` (commit `9d77e9c`), `Repairdom-frontend` (commit `c369c7d`), espace de travail racine `projet.git` (le présent rapport).
- **Commits**
  - backend `9d77e9c` — `feat(v2): add catalog (services, categories, items, prices) + seed Tecno` (18 fichiers, +1 140 / −2)
  - frontend `c369c7d` — `feat(v2): wire wizard step 1-3 to new catalog API` (3 fichiers, +276 / −1)
- **Push** : **NON EFFECTUÉ.** Les deux commits sont locaux. Attendu la validation du porteur de projet avant poussée (RÈGLE 6), ordre backend puis frontend (RÈGLE 2).
- **Déploiement** : **NON VÉRIFIÉ.** La migration M1 n'est pas appliquée en base, donc rien de M1 n'est observable en production.
- **Environnement** : Node v22.23.4 et npm 10.9.2, bootstrappés dans `C:\Users\Docker\.qoder\bin\node` lors de l'initialisation machine du 06/11 (installation système volontairement évitée) ; git 2.52.0 portable, même raison.

## Synthèse

M1 construit le catalogue v2 **à côté** du catalogue historique, pas à la place. Le référentiel v1 (`ServiceDomain` / `DeviceBrand` / `DeviceModel` / `Problem` / `CatalogIntervention` / `Pricing`) et le module `src/admin/catalog` existant (2 203 lignes) sont intacts et continuent d'alimenter les demandes en cours.

**Côté backend** : trois modèles neufs, une migration SQL écrite à la main, un seed idempotent de 3 prestations / 9 catégories / **67 items**, trois lectures publiques dont le tarif est déduit de la session, et quatre écritures admin.

**Côté frontend** : trois fonctions d'API branchées sur `/api/services`, un écran de sélection à trois niveaux, et **deux lignes** touchées dans le wizard.

## Chiffres du seed (livrable demandé)

| Niveau | Volumétrie | Détail |
|---|---|---|
| Services | **3** | `SYSTEME` « Réparation système », `SMARTPHONE` « Réparation smartphone », `FRP` « Déblocage FRP » |
| Catégories | **9** | SYSTEME : `REINSTALL_OS`, `DRIVERS`, `SOFTWARE`, `DEBRIDAGE` — SMARTPHONE : `SCREEN`, `BATTERY`, `CONNECTOR`, `OTHER` — FRP : `TECNO` |
| Items | **67** | 4 (SYSTEME) + 4 (SMARTPHONE) + **59** (FRP / Tecno) |

**Grille tarifaire seedée, telle que sortie du module de données :**

| Item | `pricePublic` | `pricePro` | Affichage client |
|---|---|---|---|
| `REINSTALL_OS` | 5000 | null | « 5 000 FCFA » |
| `DRIVERS` | 2500 | null | « 2 500 FCFA » |
| `SOFTWARE` | null | null | « Sur devis — vous serez contacté » |
| `DEBRIDAGE` | null | null | « Sur devis — vous serez contacté » |
| `SCREEN`, `BATTERY`, `CONNECTOR`, `OTHER` | null | null | « Sur devis — vous serez contacté » |
| 59 items `TECNO_*` | 3000 | 1500 | « 3 000 FCFA » en public, « 1 500 FCFA » pour un compte B2B |

**Répartition des 59 Tecno par `androidVersion`** : 7.0 → 7 · 8.0 → 7 · 9 → 6 · 10 → 8 · 11 → 9 · 12 → 7 · 13 → 8 · 14 → 7. Total 59, **59 codes distincts** (contrôle fait par `new Set(codes).size`). Les `description` des items Tecno sont toutes `null`, les prix sont identiques dans tout le groupe (3000 / 1500) ; seul le badge « Android {version} » les distingue.

## Écarts par rapport à la demande

1. **55 vs 59 modèles Tecno.** Le brief dit « FRP — 55 modèles Tecno » et, en livrable, « 55 Tecno + 8 autres ». La liste qu'il énumère ensuite contient **59** modèles (7+7+6+8+9+7+8+7). Ce sont **les noms énumérés qui ont été seedés**, d'où 67 items au total et non 63, et le test d'idempotence porte le titre « seede les volumes attendus (**59 Tecno** + 8 autres) ». Si c'est 55 qui fait foi, il faut dire quels 4 modèles retirer.
2. **`User.clientType` ajouté à un modèle existant.** La contrainte était « aucune modification du schéma Prisma hors des 3 nouveaux modèles », mais la règle de tarif était « `forPro` est déduit du JWT : si `clientType === 'B2B'` ». Or ni le schéma (enum `Role` = `CLIENT | ADMIN` seulement), ni le payload du token (`{sub, email, role, tokenVersion}`), ni `RequestUser` (`{id, email, role, tokenVersion}`) ne portaient de `clientType` : zéro support dans le code. **Arbitrage demandé et accordé au lancement du sprint** : champ additif `clientType ClientType @default(CLIENT)`, donc comportement des comptes existants inchangé.
3. **`listItems` prend `serviceCode` en plus de `categoryId`.** Le brief décrivait `listItems(categoryId, { forPro })`. La lecture résout d'abord la prestation par son code, puis vérifie `catalogCategory.findFirst({ where: { id: categoryId, serviceId: service.id } })` : sans cela, une catégorie de `SMARTPHONE` appelée sous `/api/services/FRP/...` répondait 200 avec des items d'une autre prestation. Le contrat HTTP demandé reste strictement celui du brief (`:code/categories/:catId/items`) ; c'est la signature interne du service qui porte le paramètre supplémentaire.
4. **Le wizard est superposé, pas remplacé** (arbitrage accordé). Le brief demandait de « remplacer la première étape du demande-wizard ». `src/components/client/demande-wizard.tsx` est lu texte en texte par **7 fichiers de test statiques** qui assertionnent verbatim `STEPS`, `brandId !== ''`, `listEquipmentFamilies()` et la question « Quel appareil avez-vous ? ». Supprimer ce bloc faisait échouer la cible « `npm run test:unit` vert ». Résultat : `<CatalogServicePicker />` est **ajouté** en premier enfant de l'étape 0, au-dessus de la question appareil ; rien d'autre n'est touché. La refonte complète reste à faire en M5.
5. **`catalog-service.ts` complété, pas remplacé** (même raison). Le brief disait « remplacer le contenu actuel par 3 fonctions ». Les exports v1 (`listCatalogDomains`, `listCatalogBrands`, `listCatalogModels`, `listCatalogProblems`, `listEquipmentFamilies`, `listNationalities`) sont appelés par le wizard, des pages `/client/**` et `/admin/**` que le même brief impose de laisser fonctionnelles. Les 3 fonctions v2 sont ajoutées à leur suite, en réutilisant **le `apiFetch` déjà privé du fichier** — aucune nouvelle implémentation de fetch.
6. **Le seed n'a jamais été exécuté contre une vraie base.** RÈGLE 3 interdit Postgres local ; `npm run db:seed:catalog` est donc **resté non exécuté**. La preuve d'idempotence ci-dessous porte sur la logique d'upsert, pas sur un vrai Postgres.
7. **Ids en `cuid()`** sur les trois modèles neufs, alors que les modèles historiques de ce schéma utilisent `@default(dbgenerated("gen_random_uuid()"))`. C'est `cuid()` qui était demandé ; le mélange des deux conventions dans un même schéma est un choix du brief, pas une dérive.
8. **`createdAt` / `updatedAt` ajoutés sur `CatalogCategory` et `CatalogItem`.** Le brief ne les listait que sur `Service`. Ajoutés par cohérence avec le reste du schéma et pour que l'admin puisse trier ; aucun comportement n'en dépend.
9. **Un fichier d'auth neuf, hors catalogue** : `src/auth/optional-jwt-auth.guard.ts`. Une lecture publique doit rester publique tout en connaissant le visiteur quand il est connecté. Aucun guard « optionnel » n'existait ; `JwtAuthGuard` rejette à 401. Sur token invalide le guard **supprime `request.user` et laisse passer** : le tarif retombe sur public au lieu de casser la page.

## Fichiers

### Backend — créés (12)

- `prisma/migrations/20261106010000_add_catalog_v2_services/migration.sql` (108 lignes) — `CREATE TYPE` sous garde `EXCEPTION WHEN duplicate_object` pour les deux enums, 3 × `CREATE TABLE IF NOT EXISTS`, 6 contraintes/index nommés comme Prisma les attend (`CatalogCategory_serviceId_code_key`, `CatalogItem_categoryId_code_key`, `CatalogItem_categoryId_active_order_idx`…), 2 FK `ON DELETE CASCADE`, et `ALTER TABLE "User" ADD COLUMN IF NOT EXISTS "clientType" "public"."ClientType" NOT NULL DEFAULT 'CLIENT'`. **Non destructif et rejouable.**
- `prisma/seed-catalog-data.ts` (234 lignes) — données + `seedCatalogV2(prisma)`, séparées du CLI pour que le seed soit testable sans base.
- `prisma/seed-catalog.ts` (35 lignes) — point d'entrée `npm run db:seed:catalog` : `dotenv/config`, `PrismaPg` + `pg.Pool`, appel de `seedCatalogV2`, log de synthèse, `$disconnect()`. TypeScript natif Node 22 (ni `ts-node` ni `tsx` dans le projet).
- `src/catalog/catalog.types.ts` (37), `catalog.service.ts` (104), `catalog.controller.ts` (33), `catalog-admin.service.ts` (148), `catalog-admin.controller.ts` (38), `catalog.module.ts` (24), `dto/catalog-item.dto.ts` (87), `catalog.service.spec.ts` (160).
- `src/auth/optional-jwt-auth.guard.ts` (31).

### Backend — modifiés (6)

- `prisma/schema.prisma` (+80) — 2 enums, 3 modèles, 1 champ `User`.
- `package.json` (+1) — script `db:seed:catalog`, inséré après `db:generate`.
- `src/app.module.ts` (+2) — `CatalogModule` déclaré en dernière entrée des `imports`.
- `src/auth/auth.types.ts` (+5) — `UserClientType = 'CLIENT' | 'B2B'`, `clientType` dans `RequestUser`.
- `src/auth/auth.service.ts` (+11 −1) — `verifyToken` retourne `clientType`.
- `src/auth/auth.module.ts` (+4 −1) — `OptionalJwtAuthGuard` fourni et exporté.

### Frontend — créé (1)

- `src/components/client/catalog-service-picker.tsx` (220 lignes) — écran neuf, **lu par aucun test existant**.

### Frontend — modifiés (2)

- `src/lib/api/catalog-service.ts` (+52) — 3 types + `listServices`, `listCategories`, `listItems` ; le fichier gagnait au passage son saut de ligne final.
- `src/components/client/demande-wizard.tsx` (+3) — 1 import, 1 JSX, 1 ligne blanche. `STEPS`, `canContinue`, `brandId`, `listEquipmentFamilies()`, le brouillon, les médias et le reste de l'étape 0 sont inchangés.

## Décisions de conception qui méritent d'être relues

- **Le tarif n'est jamais choisi par le client.** Aucun endpoint n'accepte `forPro` en query, et le `ValidationPipe` global (`whitelist` + `forbidNonWhitelisted`) renverrait un 400 si quelqu'un l'envoyait. C'est la règle du brief, et le pipeline Nest existant la rend non contournable.
- **`clientType` est relu en base, pas lu dans le JWT.** `verifyToken` recall déjà l'utilisateur à chaque requête (c'est aussi ce qui rend `tokenVersion` et `isActive` opposables) ; `clientType` suit la même trajectoire. Un compte promu B2B par l'admin change de tarif à la requête suivante, sans ré-émission de token. L'inverse — faire confiance au payload — aurait permis un tarif pro volé à coup d'édition de cookie.
- **Soft delete admin.** `DELETE /api/admin/catalog/items/:id` passe `active = false`. Un item déjà cité par une demande ne disparaît donc pas rétroactivement, et un re-seed ne le réactive pas : les blocs `update` des upsert **omettent volontairement `active`**.
- **`projectPrice()` est une fonction pure exportée** (`{ price, surDevis }`), pour que la règle « prix null ⇒ sur devis » ait un seul endroit et un seul test.
- **Le wizard ne peut pas mélanger deux prestations** : changer de niveau réinitialise les niveaux aval (`chooseService` remet catégorie et item à null, `chooseCategory` remet item à null).

## Affichage prix / sur devis (livrable demandé — description, pas capture)

**Capture impossible** : `next build` et `npm run dev` sont interdits en local (RÈGLE 3), et l'API n'est pas déployée. Ce qui suit est lu dans le code, pas observé dans un navigateur.

Le picker rend trois niveaux dans la section `aria-label="Choix du service"` :

1. **Prestation** — 3 cartes (grille `sm:grid-cols-3`), titre `entry.label`, sous-titre `entry.description` si elle existe. Pendant le chargement : 3 `Skeleton` `h-20 rounded-xl` en `aria-hidden`. Carte retenue : `border-primary bg-primary/5`.
2. **Catégorie** — boutons `Button size="sm"`, `variant="primary"` pour la catégorie retenue, `secondary` sinon.
3. **Item** — liste `<ul>` à lignes séparées ; à gauche le label, suivi du badge `Android {androidVersion}` en `text-xs` quand le champ est non null (c'est le cas des 59 Tecno), à droite le prix.

**Le prix, exactement :**

- Dans la **liste d'items**, une entrée tarifée affiche `formatFCFA(entry.price)`, une entrée sans tarif affiche le mot court **« Sur devis »** en `text-muted-foreground`. Exemple réel pour un visiteur non connecté sur la catégorie TECNO : **« Spark 4 · Android 9 — 3 000 FCFA »** (`formatFCFA(3000)` produit « 3 000 FCFA » avec une espace insécable U+00A0, jamais une espace simple).
- Sous la liste, **la sélection complète** est rappelée en une ligne `aria-live="polite"` : `{item.label} : {priceLabel}` où `priceLabel` est soit `formatFCFA(item.price)`, soit la phrase longue **« Sur devis — vous serez contacté »**. Exemples : « Camon 30 Pro : 3 000 FCFA » (public), « Camon 30 Pro : 1 500 FCFA » (compte B2B), « Installation logiciel : Sur devis — vous serez contacté ».
- Le mot « FCFA » n'est écrit **nulle part ailleurs** dans le composant : le formatage reste confiné à `src/lib/format-fcfa.ts`, comme dans le reste de l'application. Un prix `null` transmis par erreur à `formatFCFA` rendrait « — », jamais « null FCFA ».
- Pour les 4 items SYSTEME/SMARTPHONE tarifés (`REINSTALL_OS`, `DRIVERS`) le client connecté verra le **tarif public**, `pricePro` étant `null` : un compte B2B qui choisit « Réinstallation OS » verra donc « Sur devis — vous serez contacté », pas 5 000 FCFA. C'est la conséquence directe de la grille seedée demandée ; **à confirmer si ce n'est pas voulu** (voir Points d'attention 1).
- `onSelect` ne remonte un objet `CatalogSelection` (code + label de chaque niveau, `itemId`, `price`, `surDevis`) **que quand les trois niveaux sont choisis**. Un parent ne peut pas recevoir de sélection partielle. Aujourd'hui **aucun parent ne passe `onSelect`** : le picker est monté sans callback et n'écrit donc rien dans la demande — volontaire, le câblage de la demande au catalogue v2 est un livrable M5.

## Vérifications

| Contrôle | Backend | Frontend |
|---|---|---|
| Types (porte imposée) | `tsc --noEmit -p tsconfig.build.json` → **exit 0** | `tsc --noEmit` → **exit 0** |
| Types (complètes) | `tsc --noEmit` → **15 erreurs dans 9 fichiers**, **identiques sur `HEAD` propre** : ce sont des specs/e2e que je n'ai pas touchés | — |
| Schéma | `prisma validate` → **« The schema at prisma\schema.prisma is valid »**, exit 0 | — |
| Lint | `oxlint src/` → **0 erreur, 0 warning** | `oxlint src/` → 6 warnings ; `eslint src` → 4 warnings (0 erreur) ; **compteurs identiques à `HEAD` propre**, et **0 warning sur les 3 fichiers touchés** |
| Tests | `vitest run` → **58 fichiers / 740 passés + 2 skipped / 0 échec** | `npm run test:unit` → **391 tests, 8 suites, 391 passés, 0 échec** |

### Preuve de non-régression

Conformément à RÈGLE 3, la comparaison a été faite sur **noms d'échecs**, pas sur des compteurs, en remisant le travail (`git stash push -u`) puis en le restaurant (`git stash apply`).

- **Backend** — les 15 erreurs du `tsc --noEmit` complet sont reproduites à l'identique sur `HEAD` propre, dans les mêmes fichiers de spec/e2e préexistants. La porte imposée par le brief (`tsconfig.build.json`) est à 0 dans les deux états. `vitest run` complet : 0 échec avec mes changements ; les 6 tests catalogue sont **additifs** (le total passe de 734 à 740 cas), aucun test M0 supprimé n'est réapparu.
- **Frontend** — 6 warnings oxlint et 4 warnings eslint sur `HEAD` propre, **les mêmes** avec mes changements (les 4 warnings eslint sont les `react-hooks/exhaustive-deps` des pages catalogue admin, déjà signalés en M0.3). 391 tests / 0 échec, soit exactement la baseline M0.3 : **aucun test ajouté, aucun retiré, aucun cassé**.
- **Garde-fous statiques** : les 7 fichiers de test qui lisent `demande-wizard.tsx` passent, ce qui démontre que le contrat `STEPS` / `brandId` / `listEquipmentFamilies` est intact. Aucun mot interdit par les tests statiques (`openDispute`, `window.innerWidth`, `saspay`/`SasPay`, `MediaGallery`, `suite.close`, `describe.skip`, `it.skip`, `.only`) n'a été introduit, et `useMemo` n'apparaît que dans le fichier neuf — la restriction `useMemo`/`useCallback` vise la page devis, pas le wizard.

### Preuve d'idempotence du seed (6 tests, tous verts)

`src/catalog/catalog.service.spec.ts` joue `seedCatalogV2` **deux fois de suite** contre un double en mémoire qui honore les contraintes uniques `code`, `serviceId_code` et `categoryId_code` :

| Test | Ce qu'il verrouille |
|---|---|
| `listServices renvoie les 3 prestations` | codes `[FRP, SMARTPHONE, SYSTEME]`, 3 lignes |
| `un visiteur non-B2B lit le tarif public (Tecno = 3000)` | `price: 3000`, `surDevis: false` |
| `un compte B2B lit le tarif professionnel (Tecno = 1500)` | `price: 1500`, `surDevis: false` |
| `un tarif null devient { price: null, surDevis: true }` | `SYSTEME → SOFTWARE`, unique item, `{ price: null, surDevis: true }` |
| `seede les volumes attendus (59 Tecno + 8 autres)` | 3 / 9 / 67, dont 59 sous `TECNO`, et `expect(SEED_TOTALS.items).toBe(67)` — le total annoncé ne peut pas dériver silencieusement |
| `est idempotent : rejouer ne crée aucun doublon et ne change aucun id` | après second passage : mêmes compteurs, **`TECNO_SPARK_20` garde le même `id`**, et l'ensemble `(categoryId, code)` reste sans doublon (`new Set(clefs).size === clefs.length`) |

**Limite à lire honnêtement** : cette preuve porte sur la logique d'upsert du seed, pas sur Postgres. Elle ne démontre pas que la migration SQL s'applique, ni que l'`upsert` de Prisma 7.10 se comporte comme le double sur une vraie base. La première exécution réelle de `npm run db:seed:catalog` reste à faire côté déploiement.

## Bugs trouvés

1. **Une collision de nom évitée de justesse** — `src/admin/catalog.service.ts` existe déjà (2 203 lignes, catalogue v1 admin). Le nouveau service porte donc le nom `CatalogV2Service` dans un module séparé `src/catalog/`. Le brief disait « `catalog.service.ts` » : pris au pied de la lettre dans `src/admin/`, il aurait écrasé le service v1 ; dans `src/catalog/`, il entrait en collision à l'import. Les deux fichiers existent désormais sous le même nom de fichier, dans deux dossiers distincts.
2. **Aucun mécanisme de tarif « pro » n'existait** — voir Écart 2. Ce n'est pas un défaut du code M0, c'est une exigence de M1 qui supposait une donnée absente du schéma.
3. **Aucun garde-fou contre une catégorie lue sous la mauvaise prestation** — la route `/api/services/:code/categories/:catId/items` aurait exposé les items d'une autre prestation avec un simple `where: { id: categoryId }`. Traité dans le service (Écart 3).

## Points d'attention

1. **`pricePro` n'est renseigné que sur les 59 Tecno.** Les 8 items SYSTEME/SMARTPHONE ont `pricePro: null` conformément au brief, donc **un client B2B voit « sur devis » sur la réinstallation OS (5000) et les pilotes (2500)**. Si l'intention était « le pro paie moins cher partout », il manque les tarifs pro de ces deux items.
2. **Le tarif pro reste accessible à un CLIENT qui lit la base.** `CatalogItem.pricePro` n'est renvoyé brut par aucune route publique — c'est voulu — mais la colonne est lisible côté admin, et `admin/catalog` v1 a ses propres endpoints de prix. Aucun risque immédiat, juste à savoir que la séparation public/pro est une règle de projection, pas une contrainte de stockage.
3. **Rien n'écrit `clientType` à `B2B`.** Le champ existe, il est lu, il n'a **aucun chemin d'écriture** : ni inscription (rôle limité à `CLIENT`), ni page admin, ni seed. Aucun compte ne peut être B2B aujourd'hui, donc la branche `forPro: true` est morte en pratique jusqu'à ce qu'on décide qui promeut.
4. **`GET /api/services` renvoie la prestation même si `description` est null.** Le seed ne renseigne pas de descriptions de service ; les cartes frontend n'affichent donc que le titre. Complétable par l'admin ou par un prochain seed.
5. **Aucun test ne couvre les contrôleurs.** La spec porte sur le service et le seed. `OptionalJwtAuthGuard` (anonyme / token valide / token falsifié), le rejet d'un `forPro` en query, et les 4 routes admin ne sont pas testés. À ajouter si M5 doit modifier ces contrats.
6. **Le picker n'est pas câblé à la demande.** `onSelect` n'est passé par personne : l'étape choisie n'est ni stockée, ni envoyée à `createDemande`, ni conservée dans le brouillon (`demande-draft-sync`). Deux parcours de sélection coexistent donc visuellement dans l'étape 0 (v2 en haut, v1 appareil/marque ci-dessous) jusqu'à M5.
7. **Non testé** : aucun build, aucun `next dev`, aucun navigateur, aucune base. Les conséquences d'affichage sont déduites du code.
8. **La migration M1 n'est pas appliquée** et le seed n'est pas joué. Le frontend de ce commit, s'il était déployé tel quel, appellerait `/api/services` sur un backend qui ne connaît pas encore la route (404) et afficherait l'`Alert` « Prestations momentanément indisponibles. » — le wizard historique en dessous resterait pleinement utilisable, c'est l'effet de bord voulu de la superposition. **Ne pas pousser le frontend avant d'avoir appliqué la migration et le seed côté serveur.**

## Questions pour le porteur de projet

1. Les 59 modèles énumérés font-ils foi, ou faut-il ramener à 55 (et dans ce cas, lesquels retirer) ?
2. `pricePro` à null sur `REINSTALL_OS` (5000) et `DRIVERS` (2500) : voulu, ou tarifs pro manquants ?
3. Qui promeut un compte en `B2B` — page admin M6, import, ou règle automatique ? En attendant, le tarif pro est inaccessible.
4. La superposition wizard v1 + bloc v2 est-elle acceptable en production entre M1 et M5, ou M1 part-il en attente de M5 (voir Point d'attention 8 sur l'ordre de déploiement) ?
