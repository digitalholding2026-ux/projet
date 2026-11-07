# RAPPORT-WIZARD-CATALOGUE-M1

## Statut
- Date : 2026-11-07
- Dépôt concerné : `Frontend` (`Repairdom-frontend`)
- Fichier modifié : `src/components/client/demande-wizard.tsx`
- Fichier modifié : `src/components/client/catalog-service-picker.tsx`
- Commit : non effectué (pas de git/node disponible dans l'environnement — voir « Vérifications »)
- Push : non effectué

## Synthèse
Adaptation du `DemandeWizard` au nouveau catalogue M1 (`/api/services`) et suppression des scories de l'ancienne marketplace (Domaines/Problèmes/Marques). Le parcours fonctionne désormais de bout en bout sur le nouveau catalogue : choix du Service, puis de la Catégorie, puis de l'Item (avec prix), activation du bouton « Continuer » dès qu'un item est sélectionné, et récapitulatif affichant Service / Modèle-Item / Prix.

## Fichiers

### `src/components/client/demande-wizard.tsx`
- Suppression des imports `listCatalogDomains`, `listCatalogBrands`, `listEquipmentFamilies`, `CatalogDomainLite`, `CatalogBrandLite`, `EquipmentFamilyLite`, `EmptyState`, `Skeleton`, `ICON_NAMES`, `ReactNode`.
- Suppression de l'état legacy : `families`, `equipmentFamily`, `domains`, `catalogLoading`, `domainId`, `domainName`, `brandId`, `brands`, `categoryId` (state).
- Suppression de l'effet de chargement de l'ancien catalogue et des effets de restauration de domaine/marque/famille du brouillon.
- Nouvel état : `selectedServiceCode`, `selectedServiceLabel`, `selectedCategoryId`, `selectedCategoryLabel`, `selectedItemId`, `selectedItemLabel`, `selectedItemPrice`.
- `draftFields` : `categoryId` = `selectedCategoryId`, `domainId/brandId/equipmentFamily` = `''`, `isOtherDomain` = `false`.
- Nouvel `onCatalogSelect` (appelé par `CatalogServicePicker` une fois service + catégorie + item choisis) qui renseigne l'état de sélection.
- `canContinue` (étape 0) : renvoie `true` dès que `selectedItemId !== ''` (plus de vérification `domainId`/`brandId`).
- `submitAsAuthenticatedClient` : utilise `selectedCategoryId` ; suppression de `domainId`/`brandId`/`equipmentFamily` du payload `createDemande`.
- `handleSubmit` : suppression des gardes legacy `domainId === OTHER_DOMAIN` / `brandId === ''`.
- `hasStarted` : prend en compte `selectedItemId`.
- Récapitulatif (étape 3) : lignes « Service », « Modèle / Item », « Prix » (format FCFA, « Sur devis » si null).
- Suppression des composants `DeviceChips`, `SelectChip`, `DomainTile`, `domainIcon`, `CATEGORY_ICONS`, constante `OTHER_DOMAIN`.
- Suppression du message « Catalogue indisponible » et du bouton « Décrire sans catégorie ».

### `src/components/client/catalog-service-picker.tsx`
- Ajout d'une fonction `retryServices` et d'un bouton « Réessayer » dans l'état d'erreur de chargement des prestations (catalogue propre + retry, sans impasse).

## Règle métier appliquée
- Le choix du Service (SYSTEME / SMARTPHONE / FRP) définit la nature de l'appareil.
- Plus de sélection séparée de « catégorie d'appareil » ou de « marque ».
- FRP : étape Appareil = choix du modèle Tecno (item dans la catégorie TECNO).
- SYSTEME / SMARTPHONE : étape Appareil = choix de l'item parmi les catégories du service.
- « Continuer » activé dès qu'un `selectedItemId` est défini, quel que soit le service.

## Vérifications
- `tsc --noEmit` : **NON EXÉCUTÉ** (Node indisponible dans l'environnement).
- `npm run lint` : **NON EXÉCUTÉ** (Node indisponible).
- `vitest run` / `test:unit` : **NON EXÉCUTÉ** (Node indisponible).
- Aucun test statique ne lit `demande-wizard.tsx` (pas de régression par commentaire identifiée).
- Preuve fonctionnelle attendue : après sélection « Déblocage FRP » → « Camon i », `selectedItemId` est renseigné, donc `canContinue` (étape 0) est vrai et le bouton « Continuer » s'active ; le récapitulatif affiche « FRP / Camon i / 3000 FCFA ».

## Bugs trouvés
- BUG 1 (catalogue indisponible) : corrigé — le wizard ne charge plus l'ancien catalogue, il délègue à `CatalogServicePicker` branché sur `/api/services`.
- BUG 2 (bouton Continuer bloqué) : corrigé — `canContinue` ne dépend plus de `deviceCategoryId`/`problemCategoryId` (legacy) mais uniquement de `selectedItemId`.

## Points d'attention
- Le backend n'a pas été touché (conforme à la contrainte).
- La synchronisation du brouillon anonyme utilise `selectedCategoryId` comme `categoryId` ; `domainId`/`brandId` ne sont plus envoyés.
- La non-régression n'a pas pu être prouvée par l'exécution des tests (environnement sans Node). À valider sur une machine disposant de Node 22+ avant push.
- Ordre de push (RÈGLE 2) : backend d'abord (déjà vert), puis frontend — ce chantier est frontend uniquement.

## Questions bloquantes
- Aucune.
