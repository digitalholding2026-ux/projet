# RAPPORT — FIX URGENT : CATALOGUE ACCESSIBLE AUX ANONYMES

**Date** : 2026-10-06
**Dépôts concernés** : `Repairdom-backend` (`fb3c587`, puis `e467fdd`), `Repairdom-frontend` (`7da6999`)
**Nature** : correctif d'urgence — le tunnel public du chantier D2 était inutilisable

---

## 1. SYNTHÈSE

Le chantier D2 avait rendu le wizard de demande **public** (`/demande`, hors du
`RoleGuard`), mais les endpoints du catalogue étaient **restés protégés**. Un
visiteur sans compte recevait donc 401 et le wizard affichait « Catalogue
indisponible » : il ne pouvait littéralement pas choisir son appareil, donc pas
décrire sa panne.

Le tunnel était ouvert à l'entrée et **muré à la première étape**. Les
référentiels de lecture sont désormais anonymes ; le back-office est intact.

---

## 2. LE CONSTATTEUR

Le flux des échecs était **totalement muet** :

```ts
listCatalogDomains().catch(() => setDomains([]))
```

Un 401 et une panne réseau produisaient exactement la même image, sans la
moindre trace exploitable. Impossible de distinguer « catalogue
indisponible » de « visiteur non authentifié » — ce qui a retardé le
diagnostic.

---

## 3. FICHIERS MODIFIÉS

| Dépôt | Chemin | Nature |
|---|---|---|
| backend | `src/admin/catalog-public.controller.ts` | Retrait du `@UseGuards` / `@Roles` de **classe** ; guards reposés au niveau **méthode** sur `nationalities` (+39 l.) |
| backend | `src/admin/catalog-public-access.spec.ts` | **Nouveau** — 19 tests |
| backend | `src/admin/catalog-public-access.spec.ts` | Correctif d'une assertion vacueuse (commit `e467fdd`) |
| frontend | `src/components/client/demande-wizard.tsx` | Point 8 optionnel : 2 `console.warn` (statut HTTP seul) |

Aucune migration. Aucune dépendance ajoutée. Aucun autre contrôleur touché.

---

## 4. FRONTIÈRE PUBLIC / PRIVÉ

### 4.1 Devenus publics (lecture seule)

| Route | Service | Champs réellement exposés |
|---|---|---|
| `GET /api/catalog/domains` | `listPublicDomains()` | `id, name, slug, icon, category, _count` |
| `GET /api/catalog/domains/:id/brands` | `listPublicBrands()` | `id, name, slug, _count` |
| `GET /api/catalog/brands/:id/models` | `listPublicModels()` | `id, name, slug` |
| `GET /api/catalog/domains/:id/problems` | `listPublicProblems()` | `id, name, slug` |
| `GET /api/catalog/families` | `listPublicFamilies()` | `code, label, icon` |
| `GET /api/catalog/cities` | `listPublicCities()` | identique à `/api/cities`, déjà public |

### 4.2 Restent protégés

| Route | Guard | Justification |
|---|---|---|
| `GET /api/catalog/nationalities` | `JwtAuthGuard + RolesGuard + @Roles` — **méthode** | Consommée uniquement par `/technicien/kyc`, page déjà protégée |
| Tout `admin/catalog/*` | inchangé (classe) | Back-office |
| `/api/demandes`, `/api/notifications/*`, `/api/finances/topup/*`, `/api/client/rewards` | inchangés | Vérifié par test anti-glissement |

### 4.3 Analyse de sensibilité — l'exposition est sans risque

Ce n'est pas une évidence, c'est vérifié dans les deux sens :

| Contrôle | Résultat |
|---|---|
| PII dans les `select` publics | **Aucune** — pas d'email, téléphone, adresse ni identifiant de compte |
| Données tarifaires | **Aucune** — ni prix, ni min/max, ni marge ; le pricing reste hors de ce controller |
| `include` large | **Aucun** — le contrat public repose sur des `select` explicites |
| Filtrage `isActive` | Présent sur `listPublicDomains/Brands/Models` |
| Verbes d'écriture dans le controller public | **Aucun** (`@Post`/`@Put`/`@Patch`/`@Delete` absents) |

Ces contrôles sont **automatisés** (`catalog-public-access.spec.ts`), pas
seulement documentés : une régression future qui ajouterait un `pricing` dans un
`select` public fait échouer la suite.

---

## 5. VÉRIFICATIONS

### 5.1 Backend

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit -p tsconfig.build.json` | **OK** |
| `oxlint src/ test/` | **propre** |
| Nouveau fichier de test | **19/19 verts** |
| Suite complète | **941 passés** |

Les 3 échecs de `src/demandes/demande-multimedia.spec.ts` sont
**pré-existants** — référence établie au chantier D1 sur `HEAD` propre, **noms
identiques**, donc **0 régression**.

### 5.2 Frontend

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit` | **OK** |
| `oxlint src/` | **propre** |
| Tests | **60 verts**, aucun fichier `.js` parasite dans `src/` |

### 5.3 Scénario de production — exécuté après déploiement

| # | Test | Attendu | Obtenu |
|---|---|---|---|
| 1 | `GET /catalog/domains` sans cookie | 200 | **200**, 1 domaine (`Smartphones`), champs `['_count','category','icon','id','name','slug']` |
| 2 | `GET /catalog/domains` avec cookie invalide | identique | **200** — le cookie n'a plus d'effet, route réellement publique |
| 3 | `GET /catalog/families` | 200 | **200** |
| 4 | `GET /catalog/cities` et `GET /cities` | 200 | **200** / **200** |
| 5 | `GET /catalog/nationalities` | 401 | **401** |
| 6 | `POST /admin/catalog/domains` sans cookie | 401 | **401** |
| 7 | Railway déploiement | — | ~120 s (un 502 transitoire pendant la bascule) |
| 8 | Vercel `/demande` | 200 + wizard | **200**, `Quel appareil avez-vous ?` présent |

---

## 6. BUGS ET DÉFAUTS TROUVÉS PAR MES PROPRES TESTS

| # | Défaut | Impact | Correction |
|---|---|---|---|
| 1 | **Assertion vacuous** : mon test « avec cookie » affirmait `not 200`, qui ne passait que parce qu'aucune base n'est jointe en local (500). En production, 200 est la bonne réponse | Test **faux et vide** : il décrivait le harnais, pas le comportement | Corrigé en `not-401`, cohérent avec les autres lectures publiques. Commit `e467fdd` |
| 2 | Chemin de test **inventé** : `GET /api/finances/topup/intents` n'existe pas (seul `POST` et un `GET /intents` distinct) | 404 au lieu du 401 attendu | Route réelle vérifiée dans le controller avant de l'écrire |
| 3 | `import { readFileSync }` **dupliqué** en bas de fichier | Erreur de parsing, suite non chargée | Import remonté en tête |

Le défaut **n° 1 est le plus important** : c'est précisément le piège qui produit
des tests verts à vide. Il n'aurait été visible qu'en comparant le résultat
attendu à la réalité de production — ce que le scénario de test a fait.

Le code de production n'a été affecté par aucun de ces trois défauts.

---

## 7. POINTS D'ATTENTION

### 7.1 Le risque structurel du correctif

Retirer le `@UseGuards` de la **classe** rend **toute nouvelle route GET
publique par défaut**. C'est voulu pour les référentiels, dangereux pour une
donnée de compte. C'est écrit noir sur blanc dans l'en-tête du controller, et
un test verrouille qu'aucun verbe d'écriture n'y est introduit — mais un
futur `GET` portant une donnée de compte y échapperait.

### 7.2 Le wizard ne se remettra pas tout seul

Les domaines éventuellement déjà en cache dans un navigateur ne sont pas
rechargés automatiquement. Un visiteur ayant ouvert `/demande` **avant** le
correctif doit **recharger la page** (F5) pour voir le catalogue.

### 7.3 Deux routes publiques non utilisées par le wizard

`GET /catalog/brands/:id/models` et `GET /catalog/domains/:id/problems` ne sont
appelées par aucun parcours public actuel (le wizard s'arrête au domaine +
marque). Elles ont été rendues publiques parce qu'elles figuraient dans la
demande initiale. **Question restée sans réponse** : les laisser publiques
(préparation D3, coût nul) ou les re-protéger (surface minimale).

### 7.4 `nationalities` reste protégé

Vérifié que seul `/technicien/kyc` l'appelle, page déjà protégée → aucune
cassée. La garde au niveau méthode est un rappel : c'est le seul endroit du
controller où les guards sont encore posés explicitement.

### 7.5 Le `console.warn` ne remplace pas une alerte

Il rend le diagnostic possible, mais un 401 sur un endpoint public reste
invisible côté monitoring. Rien n'a été mis en place pour alerter : hors
périmètre.

---

## 8. QUESTIONS BLOQUANTES

**Aucune.**

Un arbitrage reste ouvert, sans bloquer (§ 7.3) : faut-il re-protéger
`brands/:id/models` et `domains/:id/problems` ?

---

## 9. SCÉNARIO DE TEST PRODUCTION RESTANT À FAIRE

| # | Test | Attendu |
|---|---|---|
| 1 | Navigation privée → `https://www.relioo.space/demande`, **F5** | Les tuiles de domaines s'affichent, plus de « Catalogue indisponible » |
| 2 | Cliquer un domaine | Les marques se chargent |
| 3 | Parcours « Autre appareil » | Les familles d'équipement se chargent |
| 4 | Connexion ADMIN → `/admin/catalog` | Le CRUD fonctionne normalement |
| 5 | Remplir et envoyer une demande | Le tunnel complet va jusqu'à la conversion |

Le test **1** est celui qui valide le correctif : c'est exactement ce qui
échouait en production.