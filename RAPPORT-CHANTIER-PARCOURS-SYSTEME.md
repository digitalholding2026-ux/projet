# RAPPORT-CHANTIER-PARCOURS-SYSTEME

## Statut

- Date : 2026-11-13
- Dépôts concernés : `Repairdom-backend` (main), `Repairdom-frontend` (main), `projet.git` (main — ce rapport)
- État push : tout poussé sur les trois dépôts, arbres de travail propres
- Déploiement : backend en attente de déploiement Railway (migration + seed), frontend prêt sur Vercel

## Synthèse

Mise en place complète du parcours "Système > Réinstallation OS" :
- Backend : enum `TargetDevice`, colonnes `targetDevice` (CatalogItem) et `extraRequirements` (Demande), migration Prisma, seed avec 7 items OS (PC/Mac), DTOs mis à jour, routes CRUD admin existantes.
- Frontend : wizard client (choix Mac/PC → OS filtrés → prix de base → exigences libres), page succès et suivi client (affichage cible, exigences, prix), refonte UI admin catalogue (tableau, modale d'édition/création, switch activation, bouton ajout, suppression), correction du mappage DTO frontend ↔ backend.
- Frontend : `tsc --noEmit` OK, `oxlint` OK (0 erreur). Backend : `tsc --noEmit` OK.

## Fichiers

Créés :
- `RAPPORT-CHANTIER-PARCOURS-SYSTEME.md` (ce rapport, à la racine)

Backend (`Repairdom-backend`) :
- `prisma/schema.prisma` — enum `TargetDevice`, champ `targetDevice` sur `CatalogItem`, champ `extraRequirements` sur `Demande`
- `prisma/migrations/20261113060000_add_target_device_and_extra_requirements/migration.sql` — création des colonnes en base
- `prisma/seed-catalog-data.ts` — 7 items OS (Windows 10/11, Ubuntu, Linux, macOS Ventura/Sonoma/Monterey) avec `targetDevice` et `active: true`
- `src/catalog/dto/catalog-item.dto.ts` — `targetDevice` sur Create/Update/UpdatePrices DTO
- `src/catalog/catalog-admin.service.ts` — `targetDevice` géré dans createItem/updateItem/updatePrices

Frontend (`Repairdom-frontend`) :
- `src/components/client/demande-wizard.tsx` — parcours Système : type machine, items OS filtrés, prix de base, textarea exigences
- `src/app/demande/succes/page.tsx` — affichage cible + exigences sur la page succès
- `src/app/client/demandes/[id]/page.tsx` — affichage cible + exigences + prix sur la page suivi
- `src/lib/api/catalog-service.ts` — `targetDevice` sur CatalogItemLite + filtre dans listItems
- `src/lib/api/request-service.ts` — `targetDevice` et `extraRequirements` sur CreateDemandeInput
- `src/app/admin/catalogue/page.tsx` — refonte UI (tableau, modale, switch, ajout, suppression)
- `src/lib/api/admin-service.ts` — `createCatalogItem`, `deleteCatalogItem`, `updateCatalogItem`, `targetDevice` sur `updateCatalogItemPrices`

## Migrations

- `20261113060000_add_target_device_and_extra_requirements` — ajoute l'enum `TargetDevice`, la colonne `targetDevice` (défaut `ALL`) sur `CatalogItem`, et la colonne `extraRequirements` sur `Demande`.

## Vérifications

- Backend `tsc --noEmit -p tsconfig.build.json` : OK
- Frontend `tsc --noEmit` : OK
- Frontend `oxlint` : OK (0 erreur sur les fichiers modifiés)
- Push backend : `76219fc` → `main` (origin)
- Push frontend : `ad561b1` → `main` (origin)
- Arbres de travail backend et frontend : propres (`## main...origin/main`)

## Bugs trouvés

1. **Colonnes manquantes en base** : le seed Railway échouait car `targetDevice` n'existait pas en base (aucune migration n'avait été créée). Corrigé par la migration `20261113060000`.
2. **Items inactifs par défaut** : le seed créait les items OS avec `active: false`, donc le menu client était vide. Corrigé : `active: true` dans le seed.
3. **Interface admin illisible** : textes chevauchants, pas de boutons d'action. Corrigé par la refonte du composant `ItemRow` (tableau, badges, switch activation, boutons Modifier/Supprimer).
4. **Mappage DTO frontend → backend** : l'enregistrement de la modale d'édition échouait (400 `property price should not exist`) car le frontend envoyait `price` alors que le DTO `UpdateCatalogItemPricesDto` attend `pricePublic`. Corrigé par l'envoi de `pricePublic` et l'ajout de la fonction `updateCatalogItem` (PATCH `items/:id`) pour les champs texte.
5. **Prix frontend/backend** : le champ `price` du frontend remplacé par `pricePublic` dans `updateCatalogItemPrices` pour correspondre au DTO backend.
6. **Persistance admin** : après toute modification (prix, statut, suppression) ou création d'item, l'interface ne se rafraîchissait pas (seul `loadServices()` était appelé, qui ne recharge pas les items de catégorie). Ajout de `refreshExpandedCategory()` qui recharge les items de la catégorie actuellement ouverte après chaque écriture. Corrigé aussi l'appel `loadItems` qui ne passait que 2 args au lieu de 3 (clé `categoryId` indéfinie).

## Points d'attention

- Le déploiement Railway du backend doit passer au vert pour que la migration + seed s'appliquent en production. Sans cela, le menu client "Système d'exploitation" reste vide.
- Les `.env` de chaque dépôt applicatif doivent être créés depuis `.env.example` (jamais committés).
- Node 22+ requis en local pour `npm.cmd ci` (étape non exécutée sur cette machine lors de l'initialisation, mais installée depuis).

## Questions bloquantes

Aucune.
