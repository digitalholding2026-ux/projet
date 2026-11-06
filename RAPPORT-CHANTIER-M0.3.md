# RAPPORT — Chantier M0.3 (migration v2) : résiduel backend mort + nettoyage frontend de l'espace prestataire

## Statut

- **Date** : 05/11/2026
- **Dépôts concernés** : `Repairdom-backend` (commit `229d613`), `Repairdom-frontend` (commit `65a891b`), espace de travail racine `projet.git` (le présent rapport).
- **Commits**
  - backend `229d613` — `chore(v2): drop dead geo-eligibility and geo-matching helpers` (3 fichiers, +2 / −138)
  - frontend `65a891b` — `refactor(v2): remove technician pages, services and tests` (97 fichiers, +188 / −12 212)
- **Push** : les deux effectués sur `origin/main`, sans rejet ni rebase.
  - backend : `885010e..229d613`
  - frontend : `4874951..65a891b`
- **Ordre de poussée respecté** : backend d'abord, puis frontend (RÈGLE 2).
- **Déploiement** : **NON VÉRIFIÉ.** Ni Railway ni Vercel ne sont observables depuis cette machine (aucun CLI installé, aucun accès log). La variable `NEXT_PUBLIC_API_URL` n'est pas en clair dans le dépôt : aucune sonde de santé n'a été tentée, pour ne pas deviner d'URL. Comme en M0.1 et M0.2, la vérification du déploiement reste à la charge du porteur de projet.
- **Environnement** : git 2.56.0.windows.2, Node v26.13.0, npm 11.20.0.

## Périmètre demandé, et ce qui en écarte

Le brief backend (« supprime le résiduel ») et le brief frontend ont été exécutés. Les **4 écarts** par rapport à la demande, avec la raison, sont listés en fin de rapport (§ Écarts). Aucun n'est une exécution manquée : ce sont des corrections de périmètre ou des critères impossibles à satisfaire tels qu'écrits.

## Synthèse

**Backend** — deux utilitaires `geo` n'avaient survécu à M0.1 que parce que leur nom ne figurait pas dans la liste de suppression : `isGeoEligible` (filtre d'éligibilité de zone) et `selectCandidatesForWave` (dernier consommateur de `DispatchWave` dans le code). Vérifié par recherche : **zéro appelant restant** dans `src/` et `test/`. Un commentaire de `nationalities.ts` affirmait encore que le code alpha-2 était « la valeur persistée (`TechnicianProfile.nationality`) » — faux depuis la migration `remove_technician_and_dispatch` ; réécrit en langage métier, sans changement de comportement.

**Frontend** — l'espace prestataire a été retiré : 16 routes, 23 composants d'écran, 5 modules `lib/technician`, 9 fichiers de test et 1 orphelin. Les deux modules de libellés et de barème ont été **débarrassés de leurs symboles prestataire puis renommés**, comme décidé par le porteur de projet. `GuardRole` et `NotificationRole` sont passés à `CLIENT | ADMIN`, et plus aucune référence ni aucun lienfrontend ne pointe vers une route supprimée.

## Fichiers

### Backend — supprimés (2)

- `backend/src/geo/geo-eligibility.ts` (78 lignes)
- `backend/src/geo/geo-matching.ts` (58 lignes)

### Backend — modifiés (1)

- `backend/src/technician/nationalities.ts` — commentaire sur la valeur persistée.

### Backend — conservés volontairement

- `backend/src/geo/geo-distance.ts`, `city-reference.ts`, `geo-normalize.ts` : consommateurs vivants (`demandes/demande-helpers.ts`, `auth/auth.service.ts`, `demandes/demandes.service.ts`, `city-reference.ts` lui-même).

### Frontend — supprimés (56)

- **Routes (18 fichiers)** : `src/app/technicien/**` — `page.tsx`, `layout.tsx`, `connexion`, `inscription`, `verification`, `demandes`, `demandes/[id]`, `historique`, `kyc`, `onboarding`, `profil`, `revenus`, `notifications`, `zones`, `chronologies`, `chronologies/[id]` ; plus `src/app/devenir-technicien/page.tsx` et `src/app/devenir-technicien/recrutement-stats.tsx`.
- **Composants (24 fichiers)** : `src/components/technician/**` hors `public/**` — `dashboard/` (5), `diagnostic/` (4), `historique/` (1), `onboarding/` (3), `profil/` (4), `revenus/` (5), `technician-auth-form.tsx`, `technician-demande-card.tsx` ; plus `src/components/auth/technician-auth-split.tsx` (orphelin après retrait de son écran).
- **Modules (3 fichiers)** : `src/lib/technician/onboarding-steps.ts`, `use-onboarding-state.ts`, `use-onboarding-banner-visibility.ts`.
- **Tests (10 fichiers)** : `devenir-technicien.test.ts`, `diagnostic-libre.test.ts`, `referrals-frontend.test.ts`, `technician-auth.test.ts`, `technician-dashboard-ui.test.ts`, `technician-kyc.test.ts`, `technician-quote.test.ts`, `technician-recruitment-ui.test.ts`, `technician/onboarding-steps.test.ts`, `technician/onboarding-components.test.ts`.

### Frontend — renommés (2)

- `src/lib/technician-profile.ts` → **`src/lib/profile-labels.ts`** (similarité 84 %)
- `src/lib/technician-quote.ts` → **`src/lib/quote-fee.ts`** (similarité 73 %)

### Frontend — nettoyés sans être supprimés

Source (13) :

- `src/lib/guard-decision.ts` — `GuardRole = 'CLIENT' | 'ADMIN'`, `roleHomePath` et `verificationPathFor` sans branche prestataire.
- `src/components/auth/role-guard.tsx` — `expectedRole: 'CLIENT' | 'ADMIN'`.
- `src/lib/api/auth-service.ts` — `role?: 'CLIENT'` à l'inscription, payload ville/adresse/code de parrainage plus conditionné par un rôle, `homePathForRole` réduit à `/admin` et `/client`.
- `src/components/ui/bottom-nav.tsx`, `src/components/auth/reset-password-form.tsx`, `src/components/auth/verification-panel.tsx` (`VerificationRole = 'CLIENT'`, routes de repli uniques).
- `src/components/landing/landing-sections.tsx` (section `BecomeTechnician` retirée) et `src/app/page.tsx` (son montage).
- `src/components/landing/hero.tsx`, `src/components/public/public-header.tsx`, `src/components/public/public-footer.tsx` — plus aucun lien `/devenir-technicien`.
- `src/lib/notifications/notification-mapping.ts` — voir § Décisions.
- `src/lib/technician-kyc-rules.ts` — suppression de `KYC_PAGE_HREF = '/technicien/kyc'` (route morte, zéro appelant).
- `src/components/mission/quote-lines.tsx`, `src/components/mission/mission-summary.tsx`, `src/app/client/demandes/[id]/page.tsx` — commentaires qui décrivaient un écran disparu ou un test supprimé, reformulés en langage métier ; import mis à jour pour `mission-summary.tsx`.
- Appels des modules renommés (7) : `admin/finances/page.tsx` (`RELIO_FEE_LABEL`), `admin/kyc/page.tsx`, `admin/kyc/[technicianId]/page.tsx`, `admin/missions/page.tsx`, `client/technicien/[id]/page.tsx`, `components/mission/mission-summary.tsx`.

Tests (15) et outillage (1) :

`role-guard.test.ts`, `demande-equipment.test.ts`, `demande-media.test.ts`, `dispute-status.test.ts`, `gps-helpers.test.ts`, `navigation.test.ts`, `responsive.test.ts`, `password-reset.test.ts`, `demande-draft-routing.test.ts`, `demande-draft-d2_5.test.ts`, `verification-confirm.test.ts`, `verification-panel-actions.test.ts`, `verification-loop-fix.test.ts`, `realtime/realtime-hooks.test.ts`, `notifications/notification-mapping.test.ts`, `mission-quote-convergence.test.ts` + `package.json` (`test:unit` ramené de 47 à 37 fichiers).

### Frontend — créés

Aucun.

## Migrations

Aucune. La migration `20261031040000_remove_technician_and_dispatch` reste **celle de M0.2**, inchangée, et son application en base demeure à la main du porteur de projet (RÈGLE : pas de `prisma migrate deploy` depuis ce poste).

## Vérifications

| Contrôle | Backend | Frontend |
|---|---|---|
| Types | `tsc --noEmit -p tsconfig.build.json` → **exit 0** | `tsc --noEmit` → **exit 0, 0 erreur** |
| Lint | `oxlint src/ test/` → **0 erreur / 0 warning**, 196 fichiers | `npm run lint` → **0 erreur, 4 warnings** (`react-hooks/exhaustive-deps`, 4 pages catalogue admin, pré-existants) |
| Tests | non exécutés — base de test non joignable (consigne du brief) | `npm run test:unit` → **391 tests, 391 passés, 0 échec** |

### Preuve de non-régression (frontend)

- **Baseline mesurée avant toute suppression**, sur l'arbre propre : **615 tests, 611 passés, 4 échecs pré-existants**. L'un de ces 4 échecs, `referrals-frontend.test.ts`, lisait `backend/src/referrals/referrals.config.ts` — fichier supprimé par le backend `885010e` ; le test ne pouvait plus passer et a été supprimé avec le périmètre. Le détail des 3 autres n'a **pas été revérifié** depuis, leurs fichiers étant supprimés ou réalignés.
- **Après les suppressions seules** : 387 tests, 351 passés, **36 échecs** — presque tous des `ENOENT` de tests statiques qui lisaient des pages `/technicien/**` supprimées, plus quelques assertions sur des branches réellement retirées.
- **Après réalignement des 15 tests conservés** : 391 tests, **0 échec**.
- Les 224 cas qui manquent entre 615 et 391 sont **les cas des fichiers de test supprimés par le périmètre**, pas des cas retirés d'un fichier conservé. Le chiffre attendu par le brief (« 615 pass / 0 fail ») n'était pas atteignable : 615 était le total **avant** suppression.

### Ce que les tests statiques ont coûté, et ce qui a été préservé

Le dépôt frontend n'a ni jsdom ni testing-library : ses tests sont **statiques** (`readFileSync` + `assert.match` sur le texte entier, commentaires inclus). Conséquence directe : un commentaire peut casser un test. Deux garde-fous menaçaient de disparaître avec leur cible ; ils ont été **re-ancrés sur une surface vivante** après vérification que la cible est propre :

- `demande-media.test.ts` : le contrôle des mots interdits (`Choisir un diagnostic`, `getDemandeSuggestions`, `selectDemandeDiagnostic`) suivait la page devis du prestataire. Ré-ancré sur `src/app/client/demandes/[id]/page.tsx`, vérifié sans aucun de ces mots et sans `MediaGallery` (2 occurrences de la section média cliente).
- `saspay-fees.test.ts` : la garde « aucun frais Mobile Money exposé au client » ciblait deux écrans ; réduite à `withdrawal-panel.tsx`, le second (`apercu-devis`) n'existant plus. Le test `mission-quote-convergence.test.ts` a perdu sa moitié prestataire, son garde-fou restant (`total: number | null`, pas de calcul dans le composant) est conservé.

## Bugs trouvés

1. **Chemin Windows cassé dans un test** — `src/lib/demande-draft-routing.test.ts` construisait sa racine avec `new URL('../', import.meta.url).pathname`, ce qui donne `/C:/…` sous Windows ; le `join()` suivant produisait `C:\C:\…` et l'`ENOENT` faisait échouer le test. Corrigé par `fileURLToPath`. **Ce défaut était pré-existant au chantier**, il bloquait la cible « 0 échec ».
2. **Un lien de notification pointait une route morte pour tout le monde** — dans `notification-mapping.ts`, `ADMIN_MESSAGE` renvoyait `{ label: 'Lire', href: '/technicien/notifications' }` **sans condition de rôle** : un client qui cliquait « Lire » depuis une notification admin arrivait sur une route qui n'existe plus (et n'a jamais existé chez lui). C'est un défaut visible en production, pas seulement un résidu de commentaire.
3. **Liste `test:unit` désynchronisée** — `package.json` énumérait encore 10 fichiers de test supprimés ; le runner Node `--test` échoue au chargement sur un fichier absent. Ramené de 47 à 37 entrées.
4. **Commentaires factuellement faux** — `nationalities.ts` (valeur persistée dans une table supprimée), `quote-lines.tsx` (formules de calcul attribuées à un écran disparu), `client/demandes/[id]/page.tsx` (pointait un test supprimé). Trois corrections de prose, aucun changement de comportement.

## Écarts par rapport à la demande

1. **Chemins inexistants dans le brief frontend.** `src/app/(auth)/technicien/**`, `src/lib/onboarding-technician.ts`, `src/lib/auth/guard-decision.ts` et `src/components/lottie/recherche-technicien.*` n'existaient pas. Réalités correspondantes traitées : `src/app/technicien/**` (les routes d'authentification y étaient, sans groupe `(auth)`), `src/lib/technician/**` (3 modules + 2 tests), `src/lib/guard-decision.ts` ; aucun fichier lottie `recherche-technicien.*` n'a été trouvé. Le `technician-auth-form.tsx` « isolé » du brief existait bien au chemin indiqué et a été supprimé.
2. **`src/app/layout.tsx` : aucune entrée de navigation prestataire.** Le brief demandait de les retirer ; il n'y en a pas. Le fichier ne contient que trois descriptions marketing identiques (§ Points d'attention 3), laissées telles quelles car ce n'est pas de la navigation.
3. **`diagnostic-libre.test.ts` ajouté au périmètre de suppression.** Son nom ne correspondait à aucun motif demandé, mais ses 5 cas portaient intégralement sur l'écran de diagnostic libre du prestataire, supprimé avec lui. Le conserver aurait laissé un test qui lit une page morte.
4. **Critère « 615 pass / 0 fail » impossible** — chiffré et remplacé par 391/391 ; voir § Preuve de non-régression.

## Décisions prises sur place (validées par le porteur de projet)

- **Conservés pour M6** : `src/lib/api/technician-service.ts` (685 lignes, 56 exports), `src/lib/api/referrals-service.ts`, `src/components/technician/public/**`, `src/app/client/technicien/[id]/page.tsx`, `src/app/admin/kyc/**`. Le brief demandait de supprimer les deux premiers ; la réponse « garde pour M6 » prime, donc les fichiers sont restés et **leurs importateurs n'ont pas eu à être corrigés**. Liste des importateurs : `client/parrainage/page.tsx`, `client/technicien/[id]/page.tsx`, `components/client/client-auth-form.tsx`, `components/mission/travel-section.tsx`, `components/technician/public/public-technician-hero.tsx`, `lib/referrals-rules.ts` (mention en commentaire).
- **`notification-mapping.ts` nettoyé au-delà de la liste demandée** : le fichier n'était pas nommé dans le brief, mais ses liens et ses branches étaient morts. Choix retenu : `NotificationRole = 'CLIENT' | 'ADMIN'`, suppression de `techMission`, et les 8 types qui ne décrivaient qu'une action prestataire (`QUOTE_ACCEPTED`, `QUOTE_REJECTED`, `NEGOTIATION_REQUESTED`, `PRICING_WARNING`, `CONFIRMED`, `MISSION_AVAILABLE`, `KYC_VERIFIED`, `KYC_REJECTED`) retombent sur le repli « aucun bouton », la même règle que pour un type inconnu. **Leur table d'urgence est inchangée** : ils s'affichent toujours, à la même place, seulement sans lien. `ADMIN_MESSAGE` mène désormais à `/client/notifications` pour le client et à rien pour l'admin (aucun centre de notifications admin).
- **Message de commit backend différent** : le brief imposait `refactor(v2): remove technician, dispatch, kyc and zone-coverage modules`, déjà utilisé par `c63fb77` (M0.1), et `chore(v2): drop … schema`, utilisé deux fois (`366dc46`, `885010e`). Nouveau message choisi pour le résiduel.

## Points d'attention

1. **Onglet « Techniciens » encore vivant côté admin** — `src/app/admin/utilisateurs/page.tsx:23` (`type RoleTab = 'CLIENT' | 'TECHNICIAN'`, bouton ligne 143). L'enum backend ne contient plus que `CLIENT` et `ADMIN` depuis M0.2 : cet onglet interroge un rôle supprimé et renverra une liste vide ou une erreur. Non touché (zone « NE PAS TOUCHER les pages admin » du brief). **Décision attendue.**
2. **Composant orphelin qui appelle des routes mortes** — `src/components/mission/travel-section.tsx` n'est monté par **aucune page** (son unique montage était l'écran prestataire) et appelle `startTravel`, `markTravelArrived`, `refreshTravelLocation`, routes `/api/technician/*` supprimées en M0.1. `gps-helpers.test.ts` le surveille encore (2 lectures), ce qui explique qu'il ait été conservé plutôt que supprimé ici.
3. **Copie marketing** — `src/app/layout.tsx:34`, `:46`, `:53` décrivent toujours « la mise en relation des clients et des techniciens locaux ». Trois occurrences, même phrase. hors périmètre strict (ce n'est pas de la navigation), à trancher produit.
4. `technician-kyc-rules.ts` **n'a pas été renommé** (le brief ne citait que deux fichiers) et son contenu reste v1 : `TECHNICIAN_MINIMUM_AGE`, et le message `MINIMUM_AGE_ERROR` « … pour exercer comme technicien sur Relio ». Les ré-exports morts de `profile-labels.ts` (`TECHNICIAN_MINIMUM_AGE`, `TechnicianActivityType`) ont été retirés, mais les symboles vivent toujours dans leur module d'origine.
5. **`quote-fee.ts` : quatre fonctions sans appelant** (`previewQuote`, `calculateRelioFee`, `isQuoteAmountAllowed`, `quoteAmountError`) depuis la suppression de l'écran de saisie, et **le test qui verrouillait l'alignement avec `backend/src/financial/fee-calculator.ts` n'existe plus** : plus rien ne contrôle la divergence des deux barèmes. Le risque est écrit noir sur blanc dans l'en-tête du module. Les constantes restent utilisées indirectement par `admin/finances` via `RELIO_FEE_LABEL`.
6. **`admin/finances/page.tsx` affiche encore « Net technicien »** (ligne 479) et lit `mission.technicianFee` / `f.netTechnician` : ce sont des **noms de champs de l'API financière**, pas des symboles frontend ; les renommer casserait le contrat avec un backend qui n'a pas encore fait M6.
7. **Non testé** : aucun test d'exécution (pas de `next dev`, pas de build, pas de navigateur) — conforme à la RÈGLE 3. Les conséquences visuelles du retrait (404 sur les anciens favoris `/technicien/*`, barre de navigation réduite à deux entrées) sont déduites du code, pas observées.
8. **Migration M0.2 toujours non appliquée** en base à ma connaissance (le rapport M0.2 la déclarait « non vérifiée »). Si elle n'est pas passée, le frontend v2 interroge un backend dont l'enum accepte encore `TECHNICIAN` : aucune casse côté frontend (plus aucun appel), mais l'inverse n'est pas vrai.

## Questions bloquantes

**Aucune** pour la clôture de M0.3. Quatre arbitrages à prendre, dans l'ordre où ils se présenteront :

1. Onglet « Techniciens » de `admin/utilisateurs` : retiré, ou conservé jusqu'à M6 avec une valeur vide ?
2. `travel-section.tsx` et les appels `technician-service` encore en place : suppression maintenant, ou remplacement du suivi de trajet en M4 ?
3. Barème de `quote-fee.ts` : faut-il un nouveau test d'alignement avec le backend alors qu'aucun écran ne le calcule plus, ou supprimer le module en M6 ?
4. Vocabulaire marketing (`layout.tsx`, `MINIMUM_AGE_ERROR`, `KYC_PAGE_*`) : refonte produit prévue, ou laissée à M6 ?
