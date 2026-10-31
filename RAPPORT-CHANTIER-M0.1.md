# RAPPORT — Chantier M0.1 (migration v2) : suppression des modules technician / dispatch / kyc / zone-coverage

## Statut

- **Date** : 2026-10-30
- **Dépôt concerné** : `Repairdom-backend` uniquement. `Repairdom-frontend` **non touché**.
- **Commit** : `c63fb77` — *refactor(v2): remove technician, dispatch, kyc and zone-coverage modules*
- **Push** : `9851cfd..c63fb77` sur `origin/main` (`main...origin/main`, propre, aucun retard).
- **Déploiement** : non vérifié (Railway). Aucun test d'exécution lancé.

## Synthèse

Les modules `technician`, `dispatch`, `kyc` et `zone-coverage` ont été retirés du backend : 32 fichiers supprimés, 11 modifiés, **-7101 / +21 lignes**. Le câblage Nest est nettoyé et `tsc --noEmit -p tsconfig.build.json` comme `oxlint src/ test/` passent **à 0 erreur**.

**Écart majeur avec la consigne** : il n'y a **aucune** erreur `tsc` résiduelle sur `TechnicianProfile`, `Demande.technicianId` ou `acceptedAt`. La consigne les attendait, mais `prisma/schema.prisma` n'a pas été modifié (c'est M0.2) : le client Prisma généré expose toujours le modèle et la colonne, donc toutes les références restantes compilent. Le nettoyage Prisma reste donc **entièrement à faire en M0.2**, sans dette de compilation à reprendre.

## Fichiers

### Supprimés (32)

`src/technician/` (22) :

- `technician.service.ts` (1757 lignes), `technician.controller.ts`, `technicians-public.controller.ts`, `technician.module.ts`
- `dto/travel-location.dto.ts`, `dto/update-coverage.dto.ts`, `dto/update-location.dto.ts`, `dto/update-status.dto.ts`, `dto/update-technician-profile.dto.ts`
- `identity-documents.ts`, `kyc-file.ts`
- specs : `technician-accept`, `technician-age`, `technician-kyc-documents`, `technician-kyc-security`, `technician-location`, `technician-media-preview`, `technician-profile`, `technician-profile-city`, `technician-travel`, `technician-travel-gps-chantier`, `geo-eligibility`

`src/dispatch/` (4) : `dispatch.module.ts`, `dispatch.service.ts` (511 lignes), `dispatch.scheduler.ts`, `dispatch.spec.ts` (595 lignes)

`src/geo/` (4 specs nommés « geo ») : `geo-distance.spec.ts`, `geo-matching.spec.ts`, `geo-normalize.spec.ts`, `geo-resolution.spec.ts`

Autres specs nommés dans le périmètre (2) : `src/admin/admin-kyc-notifications.spec.ts`, `src/auth/register-technician.spec.ts`

`src/kyc/`, `src/zones/`, `src/zone-coverage/` : **absents de ce dépôt**, aucune suppression à faire.

### Modifiés (11)

- `src/app.module.ts` — import et entrée `TechnicianModule` retirés
- `src/demandes/demandes.module.ts` — `DispatchModule` + `TechnicianModule` retirés ; `DemandeMediaService` et `SupabaseStorageService` déclarés en providers locaux
- `src/collaboration/collaboration.module.ts` — `TechnicianModule` retiré ; `DemandeMediaService` + `SupabaseStorageService` en providers locaux
- `src/admin/admin.module.ts` — `TechnicianModule` retiré ; `SupabaseStorageService` en provider local
- `src/demandes/demandes.service.ts` — import `DispatchService`, paramètre de constructeur et appel `dispatchWave1` retirés (la création de demande ne déclenche plus aucune vague)
- `src/demandes/demande-equipment.spec.ts` — instanciation alignée, test « dispatch en panne » supprimé (il testait un comportement qui n'existe plus)
- `src/demandes/demande-multimedia.spec.ts` — instanciation alignée
- `src/demandes/demandes-referrals-wiring.spec.ts` — 3 instanciations alignées
- `src/demandes/demandes-rewards-wiring.spec.ts` — 3 instanciations alignées
- `src/disputes/disputes.spec.ts` — 1 instanciation alignée
- `src/financial/mission-holds.spec.ts` — 1 instanciation alignée

### Conservés volontairement sous `src/technician/`

`avatar-file.ts`, `nationalities.ts`, `supabase-storage.service.ts` (+ son `supabase-storage.spec.ts`), `technician-age.ts`.

Ils ne sont pas des fichiers du domaine technicien : ils sont importés par `auth/` (`avatar-file`, `supabase-storage`), `admin/` (`nationalities`, `technician-age`, `supabase-storage`) et `demandes/` (`supabase-storage`). Les supprimer aurait cassé `auth` et `catalog`, explicitement hors périmètre. `supabase-storage.spec.ts` est conservé parce qu'il teste du code conservé : le supprimer aurait retiré la couverture d'un service toujours en production.

## Migrations

Aucune. `prisma/schema.prisma` n'est pas modifié (M0.2).

## Vérifications

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit -p tsconfig.build.json` | **exit 0, 0 erreur** |
| `oxlint src/ test/` | **exit 0, 0 erreur** |
| `tsc --noEmit -p tsconfig.json` (specs inclus) | 19 erreurs — **ensemble identique au HEAD propre** |
| `npm test` | **non exécuté** (consigne : base non joignable) |

### Preuve de non-régression

Comparaison faite en `git stash push -u` (HEAD propre `9851cfd`) puis `git stash pop`, mêmes options de compilation :

- **HEAD propre** : 20 erreurs dans `tsc -p tsconfig.json`
- **Après chantier** : 19 erreurs
- **Différence** : exactement une, dans `src/auth/register-technician.spec.ts` — fichier supprimé par le périmètre. Aucune erreur ajoutée, aucune erreur déplacée.

Les 19 erreurs restantes sont **toutes pré-existantes** et hors périmètre : `admin-fee-change-notification.spec.ts` (TS2769), `catalog-scales.spec.ts` (TS2698 ×2), `auth.spec.ts` (TS2554, TS2339, TS2532, TS2493), `password-reset.spec.ts` (TS2554), `quotes.spec.ts` (TS18047 ×2), `rewards.service.spec.ts` (TS2322), `supabase-storage.spec.ts` (TS2352 ×3), `test/*.e2e-spec.ts` (`supertest/types` introuvable ×4).

Elles n'apparaissent pas dans `tsconfig.build.json`, qui exclut `**/*spec.ts` et `test` : le contrôle demandé passe donc bien à zéro.

### Erreurs attendues sur `TechnicianProfile` / `technicianId` / `acceptedAt`

**Aucune.** Explication vérifiée : le client Prisma est généré depuis un `schema.prisma` inchangé, où `model TechnicianProfile` (ligne 818) et `Demande.technicianId` (ligne 634) existent toujours. `acceptedAt` n'existe même pas dans le schéma, et sa seule occurrence dans le code était dans `technician.service.ts`, supprimé. Rien ne peut donc échouer à la compilation sur ces symboles.

Conséquence pour M0.2 : la migration Prisma devra aussi retirer les usages **restants** de `Demande.technicianId`, qui sont nombreux et fonctionnels aujourd'hui — `financial.service.ts` (payouts, holds), `disputes.service.ts`, `reviews.service.ts`, `realtime.controller.ts`, `collaboration.service.ts`, `tracking.service.ts`, `backoffice-agent.tools.ts`. Ces fichiers n'ont pas été touchés ici (le périmètre demandait de ne pas toucher `financial`, et la colonne relève de M0.2).

## Bugs trouvés

Aucun bug introduit. Deux points relevés :

1. **`src/geo/geo-eligibility.ts` et `src/geo/geo-matching.ts` sont désormais du code mort** — `geo-eligibility.ts` n'a plus aucun importateur (ses seuls consommateurs étaient `technician.service.ts` et `dispatch.service.ts`), et `geo-matching.ts` n'est plus importé que par `geo-eligibility.ts`. Ils compilent, mais ne servent plus à rien. Non supprimés : le périmètre ne visait que les **tests** nommés « geo », pas le module `geo`, et `src/geo/city-reference.ts` et `geo-normalize.ts` restent utilisés (auth, demandes).
2. **Commentaires obsolètes** dans des fichiers conservés qui décrivent encore les modules supprimés : `demande-helpers.ts` (lignes 4-14, boucle circulaire dispatch), `demande-media.service.ts` (ligne 9, « fourni + exporté par TechnicianModule »), `auth.module.ts` (ligne 24), `auth/email.service.ts` (ligne 158), `geo/geo-eligibility.ts` (lignes 3-13), `rewards/rewards.module.ts` (ligne 23), `admin/admin.service.ts` (ligne 48). Laissés tels quels : ce sont des commentaires, et les réécrire n'entre pas dans « on supprime d'abord ».

## Points d'attention

- **`src/admin/` conserve toute la logique KYC** (`admin.controller.ts`, `admin.service.ts`, DTO `update-kyc-status.dto.ts`, `send-technician-message.dto.ts`). Les routes sont sous `/api/admin/*`, pas `/api/kyc/*` : elles sont donc hors du périmètre littéral. Mais leur objet — valider les documents d'un technicien — n'a plus de sens une fois les techniciens supprimés. À traiter dans un chantier dédié, pas ici.
- **`src/dispatch` étant supprimé, plus aucun e-mail « mission disponible » n'est envoyé** et `DispatchWave` n'est plus alimenté côté écriture (`admin.service.ts:966` fait encore un `count` en lecture). C'est l'effet attendu de la suppression, mais c'est un changement de comportement fonctionnel, non un simple nettoyage.
- **Deux specs supprimés couvraient du code conservé** : `admin-kyc-notifications.spec.ts` (admin KYC) et `register-technician.spec.ts` (inscription, donc `auth`). La règle « tout spec dont le nom contient kyc/technician » imposait leur suppression, mais la couverture de `admin/` et `auth/` baisse d'autant.
- **`src/technician/` subsiste** avec 5 fichiers d'infrastructure partagée. Le dossier porte encore le nom d'un domaine supprimé ; le déplacement vers un dossier `storage/` ou `common/` sera à faire quand le périmètre le permettra.
- **Non vérifié** : démarrage réel de l'application et résolution d'injection Nest (interdit sans base, RÈGLE 3). Les providers `SupabaseStorageService` / `DemandeMediaService` déclarés localement dans `admin`, `demandes` et `collaboration` n'ont donc été validés que par la compilation, pas par un bootstrap.
- **Déploiement Railway** non observé après le push.

## Questions bloquantes

Aucune. Trois décisions restent ouvertes :

1. Faut-il supprimer le code mort `geo-eligibility.ts` / `geo-matching.ts` dans un mini-chantier de nettoyage ?
2. Faut-il retirer la logique KYC de `src/admin/` (routes `/api/admin/*`, objet devenu caduc) ?
3. Faut-il déplacer les 5 utilitaires restants hors de `src/technician/` ?
