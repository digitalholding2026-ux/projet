# RAPPORT — Audit complet du frontend Relio

> Audit **en lecture seule**. Aucun fichier de code n'a été modifié : les
> anomalies sont documentées, chiffrées et localisées, pas corrigées.
> Le backend n'est **pas** audité ; il n'a servi qu'à vérifier les miroirs de
> contrats (mot de passe, frais, bornes financières).

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-30 |
| Périmètre | `frontend/` (`Repairdom-frontend`) — **backend hors périmètre** |
| Commit audité | `main` de `Repairdom-frontend`, état local propre (`git status` sans modification) |
| Commit de ce rapport | racine `projet.git`, branche `main` (voir § 12) |
| Modifications de code | **aucune** |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |
| Verdict global | **Socle solide, dette concentrée sur 5 axes** : duplication réseau, poids du bundle Lottie, accessibilité des contrastes, couverture de tests réelle, observabilité nulle |

---

## 2. Méthode — et ce qui n'a **pas** pu être vérifié

### 2.1 Ce qui a été fait

- Lecture intégrale de la configuration : `package.json`, `tsconfig.json`,
  `next.config.ts`, `vercel.json`, `eslint.config.mjs`, `oxlint.json`,
  `.nvmrc`, `.gitignore`, `.env.example`, `public/sw.js`, `docs/*.md`,
  `ARCHITECTURE.md`, `README.md`.
- Lecture ciblée mais exhaustive des 17 modules `src/lib/api/`, des 38
  composants `src/components/ui/`, des points d'entrée `app/layout.tsx`,
  `app/error.tsx`, `app/not-found.tsx`, `app/loading.tsx`, `app/manifest.ts`
  et des 4 `layout.tsx`.
- Balayage mécanique de **328 fichiers TS/TSX** (grep/Select-String) sur :
  `any`, `@ts-ignore`, `TODO/FIXME`, `dangerouslySetInnerHTML`, `eval`,
  `localStorage`, `fetch(`, `AbortController`, `useState/useEffect/useMemo`,
  `key={index}`, `console.*`, `<img>`, `next/image`, `aria-*`, `htmlFor`,
  `target="_blank"`, `NEXT_PUBLIC_*`, `text-primary`, `text-slate-400`,
  `window.innerWidth`, hooks de nettoyage (`removeEventListener`,
  `clearInterval`, `revokeObjectURL`).
- Vérification d'orphanité : pour chaque module non-`page`, recherche de son
  nom dans tous les autres fichiers du dépôt → 3 orphelins réels.
- Vérification des 38 liens internes statiques contre les 60 routes réelles :
  **0 lien cassé**.
- Vérification des miroirs backend : `assertPasswordStrong`
  (`backend/src/auth/auth.service.ts:71`), `SASPAY_COLLECT_RATE = 0.045`,
  `MIN/MAX_TOPUP/WITHDRAWAL` = 100 / 10 000 000.

### 2.2 Ce qui n'a **pas** pu être vérifié (et pourquoi)

| Non vérifié | Raison |
|---|---|
| `next build` / bundle réel | Interdit en local par la RÈGLE 3 du projet |
| `npm run dev`, rendu navigateur | Interdit en local (RÈGLE 3) |
| Lighthouse / Core Web Vitals réels | Nécessite un navigateur et un serveur |
| Tests e2e, parcours réels | Interdits (base non joignable, effet de bord prod) |
| Headers HTTP réellement servis par Vercel | Nécessite un déploiement ; l'analyse porte sur `next.config.ts` et `vercel.json` |
| Comportement à l'exécution des `useEffect` | Analyse statique uniquement, pas d'exécution React |

Les chiffres de poids (Ko/Mo) sont des **tailles de fichiers sur disque**, pas
des tailles de bundle après minification/gzip. Ils sont donc un **plancher**,
pas une estimation de transfert.

---

## 3. Synthèse

### 3.1 Chiffres clés

| Mesure | Valeur |
|---|---|
| Fichiers TS/TSX | **328** |
| Lignes TS/TSX (tests inclus) | **46 597** (dont **38 810** hors tests) |
| Pages (`page.tsx`) | **60** |
| Layouts | 4 (racine + client + technicien + admin) |
| Composants `.tsx` | 213 |
| Fichiers `'use client'` | **138** (65 % des `.tsx`) |
| Fichiers `'use server'` | **0** |
| `middleware.ts` | **0** (conforme à `ARCHITECTURE.md`) |
| `global-error.tsx` | **0** |
| Modules `src/lib/api/` | 17 |
| `apiFetch` distincts | **14** |
| Fichiers de test | **47** — **615 tests** |
| Assertions regex statiques | **1 195** |
| Tests de logique pure | **6** |
| JSON Lottie importés statiquement | **7 fichiers = 434 Ko** |
| Assets suivis et non référencés | **≈ 12,9 Mo** |
| `text-primary` (contraste 2,61:1) | **71** occurrences |
| `text-slate-400` (contraste 2,56:1) | **40** occurrences |
| Pages avec `metadata` propre | **11 / 60** |
| Liens d'évitement clavier | **0** |
| `<main>` déclarés | **8 fichiers / 60 pages** |

### 3.2 Verdict

Le code est **discipliné** : aucun `any`, aucun `@ts-ignore`, aucun
`dangerouslySetInnerHTML`, aucun `eval`, `strict: true`, aucun TODO/FIXME,
aucun lien interne cassé, aucune fuite de secret dans les URLs, `next/image`
correctement paramétré (`sizes` partout), client SSE avec backoff et repli
polling, focus trap dans `Modal`, `prefers-reduced-motion` respecté 29 fois.

La dette n'est pas dans la qualité ligne à ligne : elle est **structurelle**.

1. **14 copies d'un même client HTTP** sans timeout ni gestion de session :
   un seul point de vérité manque, et il manque partout à la fois.
2. **434 Ko de JSON dans un module importé par le layout public et par
   l'écran de chargement de toutes les pages authentifiées** — le contraire de
   ce que le commentaire du fichier promet.
3. **La couleur de marque échoue le contraste AA dans 71 endroits**, sur un
   produit mobile-first destiné à être lu en extérieur.
4. **1 195 assertions regex sur du texte source** : la suite protège les mots
   des fichiers, pas le comportement des écrans.
5. **Zéro observabilité** : une exception de production ne laisse aucune trace
   exploitable.

---

## 4. Constats — Bloquants (P0)

**Aucun.** Aucun défaut trouvé ne casse un parcours utilisateur de bout en
bout, n'expose un secret, ni ne rend une page inaccessible.

C'est un résultat, pas une esquive : les 3 pistes qui auraient pu l'être ont
été vérifiées et écartées.

| Piste P0 examinée | Résultat |
|---|---|
| XSS par injection HTML | `dangerouslySetInnerHTML` : **0** ; `innerHTML =` : **0** ; `eval`/`new Function` : **0** |
| Redirection ouverte après login | `safeRedirect` (`lib/api/auth-service.ts:144`) rejette `//`, `\`, et exige le préfixe du rôle — **correct** (durcissement mineur en § 7.10) |
| Cookie de session lisible en JS | Cookie `HttpOnly` posé par Railway, jamais lu côté Vercel ; `document.cookie` : **0 usage** |

---

## 5. Constats — Majeurs (P1)

### 5.1 Trois politiques de mot de passe, dont deux plus faibles que le backend

**Le backend refuse ce que le formulaire valide.**

| Emplacement | Règle appliquée |
|---|---|
| `backend/src/auth/auth.service.ts:71-82` | 8–128 car. + **1 majuscule + 1 minuscule + 1 chiffre** |
| `frontend/src/components/client/client-auth-form.tsx:73` | `password.length >= 8` **uniquement** |
| `frontend/src/components/technician/technician-auth-form.tsx:32` | `MIN_PASSWORD_LENGTH = 8` **uniquement** |
| `frontend/src/components/client/demande-auth-modal.tsx:72` | réimplémentation locale `isStrongEnough` |
| `frontend/src/lib/password-strength.ts` | **la bonne règle** — utilisée par **un seul** écran (`reset-password-form.tsx:12`) |

**Impact.** Un utilisateur saisit `motdepasse` (10 caractères, tout en
minuscules) : le champ affiche l'état « valide » (`client-auth-form.tsx:371-374`),
le bouton s'active, puis la soumission échoue en 400 avec
« Le mot de passe doit contenir au moins 8 caractères, 1 majuscule, 1 minuscule
et 1 chiffre. » Le message est correct, mais il arrive **après** que l'UI a dit
« c'est bon ». Sur le tunnel d'inscription, c'est exactement le moment où un
visiteur abandonne.

**Aggravant.** `demande-auth-modal.tsx` est une **quatrième** implémentation :
la même règle existe donc en 4 exemplaires, dont 3 peuvent diverger sans que
rien ne le signale.

**Correctif attendu.** `password-strength.ts` devient la seule source
(`passwordRules()` pour l'affichage, `isPasswordStrong()` pour la validation),
consommée par les 4 écrans ; un test statique verrouille l'absence de
`length >= 8` nu dans les formulaires.

---

### 5.2 `apiFetch` dupliqué 14 fois : aucun timeout, aucune gestion de session

**14 définitions distinctes** de la même fonction :

`admin-service.ts:98` · `auth-service.ts:74` · `catalog-service.ts:48` ·
`chronologies-service.ts:37` · `finance-service.ts:231` ·
`mission-events-service.ts:19` · `notifications-service.ts:90` ·
`referrals-service.ts:20` · `request-service.ts:140` ·
`review-service.ts:32` · `rewards-service.ts:97` · `summary-service.ts:70` ·
`technician-service.ts:161` · `tracking-service.ts:36`

Ce qui manque dans **les 14** :

| Mécanisme | État |
|---|---|
| `AbortController` / timeout | **absent partout** — les 2 seules occurrences du dépôt sont dans `lib/realtime/sse-client.ts:43,174` |
| Retry sur erreur réseau | absent |
| Traitement global du `401` (session expirée) | absent — chaque écran affiche « erreur » au lieu de renvoyer vers la connexion |
| Annulation au démontage | absent — un `setState` peut arriver après démontage |
| Point unique de log/observabilité | absent |

**Impact concret.** Sur un réseau mobile dégradé (le contexte d'usage du
produit), un `fetch` qui ne répond jamais laisse l'écran en `loading` **sans
fin** : pas de timeout, donc pas de message d'erreur, donc pas de bouton
« Réessayer » qui s'affiche. L'utilisateur ne peut que recharger la page.
Second effet : quand le cookie expire, l'utilisateur voit « Une erreur est
survenue » sur chaque écran, un par un, au lieu d'être redirigé une fois.

**Correctif attendu.** Un `lib/api/client.ts` exportant `apiFetch` (timeout
`AbortSignal.timeout`, injection de `signal`, hook `onUnauthorized` branché sur
`AuthProvider.refresh`, un seul `instanceof ApiError`), puis suppression des 14
copies. `api-error.ts` prouve déjà que le dépôt sait faire cette
mutualisation — il a été créé exactement pour ça.

---

### 5.3 434 Ko de JSON Lottie chargés sur toutes les pages

`frontend/src/components/lottie/lottie-animations.tsx:4-10` importe
**statiquement les 7 animations** :

| Fichier | Poids |
|---|---|
| `src/components/lottie/connexion-animation.json` | **257,9 Ko** |
| `animatio json/demande envoyer.json` | 79,6 Ko |
| `animatio json/erreur 404.json` | 37,5 Ko |
| `animatio json/social media whatsapp.json` | 24,6 Ko |
| `animatio json/menu nav.json` | 15,7 Ko |
| `animatio json/code  USSD Envoyer pour recharge.json` | 10,2 Ko |
| `animatio json/recherche de technicien.json` | 8,7 Ko |
| **Total** | **434 Ko** |

**Le commentaire du lecteur affirme l'inverse.**
`src/components/ui/lottie-animation.tsx:8-11` :

> « chaque usage passe par un wrapper de `src/components/lottie/` qui importe
> UN seul JSON → code-splitting par page, jamais de chargement global »

C'est **faux** : il n'y a qu'**un** wrapper (`lottie-animations.tsx`) et il
importe les 7 fichiers. Le `dynamic(() => import('lottie-react'))` ne
code-split que la **librairie**, pas les données.

**Pourquoi c'est grave.** Ce module est importé par 9 fichiers, dont trois
sont sur le chemin critique de *toutes* les pages :

| Importateur | Portée |
|---|---|
| `components/auth/loading-screen.tsx:3` | rendu par `RoleGuard` → **toutes les pages client / technicien / admin** |
| `components/public/public-header.tsx:10` | **toutes les pages publiques** |
| `app/not-found.tsx:6` | **toute 404** |

**Impact.** 434 Ko de JSON sont téléchargés et parsés pour afficher, dans le
cas le plus fréquent, une seule petite animation — et 258 Ko d'entre eux
(`connexion-animation.json`) ne servent qu'au formulaire de connexion.

**Correctif attendu.** Un fichier par animation
(`lottie/whatsapp.tsx`, `lottie/connexion.tsx`, …), chacun important **son**
JSON, et un import dynamique côté appelant. Le commentaire doit être réécrit
en même temps : c'est lui qui a masqué le problème.

---

### 5.4 Partage social cassé : OG/Twitter en SVG et `metadataBase` localhost

`frontend/src/app/layout.tsx` :

| Ligne | Problème |
|---|---|
| `:28` | `metadataBase: new URL(siteConfig.url)` — si `NEXT_PUBLIC_APP_URL` est absent, l'URL devient `http://localhost:3000` et **toutes** les URLs OG absolues pointent vers localhost |
| `:36-37` | `icons.apple` = `/brand/relio-mark.svg` — **iOS ignore les `apple-touch-icon` SVG** (PNG 180×180 requis) |
| `:44` | `openGraph.images` = `/brand/relio-logo-light.svg` — **Facebook, WhatsApp, LinkedIn, X n'acceptent pas le SVG** pour `og:image` |
| `:52` | idem pour `twitter.images` |

**Impact.** Tout partage du lien Relio sur WhatsApp — le canal de diffusion
principal au Cameroun — s'affiche **sans vignette**. Sur un produit qui vit du
bouche-à-oreille, c'est une perte directe d'acquisition.

**Aggravant.** `new URL()` est évalué au chargement du module : une valeur
malformée de `NEXT_PUBLIC_APP_URL` fait échouer le **build**, avec un message
qui ne désigne pas la variable. Aucun garde-fou (`try/catch` ou validation au
démarrage) n'existe.

**Correctif attendu.** PNG 1200×630 pour `og:image` (et 180×180 pour
`apple-touch-icon`), `metadataBase` calculé avec repli explicite, et un
`assertEnv()` qui échoue **tôt** avec un message nommant la variable.

---

### 5.5 Aucun en-tête de sécurité déclaré

`frontend/next.config.ts` fait 11 lignes :

```ts
const nextConfig: NextConfig = {
  reactStrictMode: true,
  poweredByHeader: false,
  images: { formats: ['image/avif', 'image/webp'] },
};
```

Aucun `headers()`. Absents du dépôt : `Content-Security-Policy`,
`X-Frame-Options` / `frame-ancestors`, `Referrer-Policy`,
`Permissions-Policy`, `X-Content-Type-Options`.

**Impact.** Vercel applique ses valeurs par défaut (dont HSTS), donc le site
n'est pas nu — mais **rien dans le dépôt ne garantit** ces en-têtes, et une
CSP est la seule défense en profondeur si un jour une dépendance injecte du
HTML. Le dépôt stocke par ailleurs un **token de brouillon** dans
`localStorage` (`lib/demande-draft-storage.ts:12`) : c'est précisément le
scénario où une CSP compte.

**Nuance honnête.** Sans `dangerouslySetInnerHTML` ni `eval`, l'absence de CSP
n'est pas une faille exploitable **aujourd'hui**. C'est une absence de filet.

---

### 5.6 Observabilité nulle

| Élément | État |
|---|---|
| Capture d'erreurs (Sentry, OpenTelemetry, Datadog) | **aucune dépendance** |
| `app/error.tsx:18` | `console.error(error)` — seul reporting du dépôt |
| `app/global-error.tsx` | **absent** → une erreur dans `RootLayout` n'a aucun écran de repli |
| `console.warn` en production | 5 occurrences (`demande-wizard.tsx:216,229,417,448`, `technicien/demandes/[id]/page.tsx:238`) |
| Analytics / Web Vitals | absents |

**Impact.** Une exception de rendu en production ne laisse de trace que dans la
console d'un utilisateur qui ne la lira jamais. Le `digest` est affiché à
l'écran (`error.tsx`) mais **personne ne le collecte**. Le diagnostic se fait
donc par capture d'écran envoyée par WhatsApp.

**Correctif attendu.** Un `instrumentation.ts` + un adaptateur minimal
(`@sentry/nextjs` ou un `POST /telemetry` maison) branché sur
`app/error.tsx`, `global-error.tsx` et `onUnhandledError`.

---

### 5.7 La suite de tests protège les mots, pas les comportements

| Mesure | Valeur |
|---|---|
| Fichiers de test | 47 |
| Fichiers utilisant `readFileSync` | **41** |
| Assertions `assert.match` / `doesNotMatch` | **1 195** |
| Assertions `equal` / `deepEqual` | 536 |
| Tests de logique pure (sans lecture de source) | **6** : `sse-client`, `onboarding-steps`, `demande-draft-sync`, `format-fcfa`, `payment-helpers`, `rewards-view` |
| Couverture de code | **aucune** (`/coverage` ignoré, aucun `c8`/`istanbul`) |
| `jsdom` / `@testing-library` | **absents** |

**Impact.** Les 613 tests verts prouvent que certains **mots** sont présents ou
absents dans certains fichiers. Ils ne prouvent pas qu'un formulaire soumet, ni
qu'un écran affiche l'erreur attendue. La RÈGLE 3 du projet documente déjà le
symptôme inverse : un commentaire anodin fait échouer un test parce qu'il
contient le mot surveillé. C'est le signe d'une suite qui teste la **forme du
code source**, pas son **effet**.

**Aggravant.** Aucun test ne couvre les zones les plus risquées :
`demande-wizard.tsx` (1 650 lignes, tunnel d'acquisition),
`withdrawal-panel.tsx` (538 lignes, mouvement d'argent),
`demande-auth-modal.tsx` (390 lignes, authentification en cours de tunnel).

---

### 5.8 Deux tests échouent systématiquement sous Windows

`frontend/src/lib/demande-draft-routing.test.ts:21` :

```ts
const SRC = new URL('../', import.meta.url).pathname;
```

Sous Windows, `.pathname` vaut `/C:/Users/.../src/`. `join()` sur ce préfixe
produit `\C:\Users\...`, résolu relativement au lecteur courant →
`C:\C:\Users\...` → `ENOENT`.

```
✖ D2 : aucun lien ne pointe plus vers /client/demande
✖ D2 : les routes /client/demandes (pluriel) sont INTACTES
  Error: ENOENT: scandir 'C:\C:\Users\Docker\Desktop\Relio\frontend\src\'
```

**Résultat mesuré** : `pass 613 / fail 2`.

**Impact.** Ce n'est pas une régression applicative — les deux tests
passeraient sous Linux/CI. Mais sur la machine de développement Windows, la
RÈGLE 3 (« la non-régression doit être **prouvée** ») devient inapplicable :
il faut trier à la main, à chaque exécution, deux échecs permanents.

**Correctif attendu.** `fileURLToPath(new URL('../', import.meta.url))`.

---

### 5.9 `demande-wizard.tsx` : 1 650 lignes, 36 `useState` dans un seul composant

| Mesure | Valeur |
|---|---|
| Fichier | **1 650 lignes** |
| Composant `DemandeWizard` | **1 380 lignes** (l. 121 → 1501) |
| `useState` | **36** |
| `useEffect` | **8** |
| `useMemo` | **8** |
| `useRef` | **3** |
| Sous-composants définis après | 6 (`DeviceChips`, `SelectChip`, `DomainTile`, `StepProgress`, `StickyRecap`, `SummaryRow`) |

C'est le **plus gros fichier du dépôt** et il porte le tunnel d'acquisition
public (anonyme **et** authentifié), la persistance de brouillon, l'upload
média, la modale d'authentification et la conversion.

**Impact.** 36 états indépendants dans un composant unique : toute évolution du
parcours anonyme touche le parcours authentifié, et inversement. C'est
exactement le couplage que `docs/UI-UX-ARCHITECTURE.md` interdit par ailleurs.
Les trois fichiers suivants (`technicien/demandes/[id]` 1 389 l.,
`client/demandes/[id]` 1 155 l.) présentent le même profil.

---

### 5.10 Contrastes : la couleur de marque échoue AA dans 71 endroits

Calculs WCAG 2.1 sur les tokens de `src/app/globals.css` :

| Combinaison | Ratio | Seuil AA (4,5:1) | Occurrences |
|---|---|---|---|
| `--primary` `#f97316` sur `--background` `#f6f7f9` | **2,61:1** | ❌ | **71** (`text-primary`) |
| `--primary` `#f97316` sur blanc `#fff` | 2,80:1 | ❌ | (idem) |
| `--muted-foreground` `#64748b` sur `#f6f7f9` | **4,44:1** | ❌ (de 0,06) | **487** |
| `text-slate-400` `#94a3b8` sur blanc | **2,56:1** | ❌ | **40** |
| `--primary` sur `--background` sombre `#0b0d12` | 6,84:1 | ✅ | — |

L'orange Relio échoue **aussi** le seuil AA-large (3:1) : il n'est conforme
pour **aucune** taille de texte sur fond clair.

**Impact.** Le produit est mobile-first, utilisé dehors, sur des écrans
moyenne gamme, par des utilisateurs qui ne sont pas tous à 20/20. Un ratio de
2,61:1 sur des libellés de navigation et des montants est un défaut de
lisibilité réel, pas théorique. `#f97316` reste utilisable en **fond** (avec
texte blanc dessus, 2,80:1 — insuffisant aussi) ou en **bordure/icône
décorative**, pas en texte.

**Correctif attendu.** Une variante `--primary-ink` (par ex. `#c2410c`, ≈ 4,6:1
sur `#f6f7f9`) pour tout ce qui est **texte**, `--primary` conservé pour les
fonds décoratifs, et un remplacement de `text-slate-400` par
`text-muted-foreground`.

---

### 5.11 Structure de page et métadonnées incomplètes

| Constat | Mesure |
|---|---|
| Lien d'évitement clavier (« Aller au contenu ») | **0** |
| `<main>` déclaré | **8 fichiers** pour **60 pages** (3 layouts + 5 pages) |
| Pages sans `<h1>` ni `PageHeader` | **13 / 60** |
| Pages avec `metadata` propre | **11 / 60** |
| `robots.txt` | **absent** |
| `sitemap.ts` | **absent** |
| Icône PWA 192/512 PNG `maskable` | **absente** (SVG `sizes: any` seulement) |

Les 13 pages sans titre de niveau 1 incluent des écrans majeurs :
`client/page.tsx` (tableau de bord), `client/demandes/page.tsx`,
`client/connexion`, `client/inscription`, `technicien/connexion`,
`technicien/inscription`, `mot-de-passe-oublie`, `reinitialiser-mot-de-passe`.

**Impact.** Pour un lecteur d'écran, une page sans `<h1>` ni `<main>` n'a ni
titre ni région principale : la navigation par titres et par repères est
inutilisable. Côté SEO, 49 pages sur 60 n'ont aucun titre propre — elles
héritent toutes du même `<title>` global.

---

## 6. Constats — Mineurs (P2)

### 6.1 Code mort : 3 fichiers, 77 lignes

| Fichier | Lignes | Preuve |
|---|---|---|
| `src/components/client/parrainage/parrainage-overview.tsx` | 43 | `ParrainageOverview` : 1 seule occurrence, sa définition |
| `src/components/client/recompenses/recompenses-skeleton.tsx` | 17 | `RecompensesSkeleton` : 1 seule occurrence |
| `src/lib/catalog-quote.ts` | 17 | `parseCatalogQuoteDescription` : **0** import |

`lib/dead-code-guard.test.ts` existe mais ne couvre pas ces trois fichiers
(il verrouille une liste fermée de composants déjà supprimés).

### 6.2 Quatre imports inutilisés (remontés par oxlint)

```
src/app/client/demandes/[id]/page.tsx:55   formatFCFA
src/app/technicien/demandes/[id]/page.tsx:7  EmptyState
src/app/client/parrainage/page.tsx:15      REFERRAL_MAX_REFERRALS
src/app/client/parrainage/page.tsx:19      isReferralExpired
```

### 6.3 `tsconfig.json` : deux `exclude` fantômes

```json
"src/lib/ai-classification.test.ts",
"src/components/admin/ai/ai-dashboard-helpers.test.ts"
```

Ces deux fichiers **n'existent pas**. Ils restent dans la liste d'exclusion :
un futur test portant ce nom serait silencieusement exclu du type-check.

### 6.4 ≈ 12,9 Mo d'assets suivis et jamais référencés

| Emplacement | Poids | Références dans `src/` |
|---|---|---|
| `recompense/` (6 PNG, racine) | **10,86 Mo** | **0** |
| `hero/hero.png` (racine) | **1,64 Mo** | **0** |
| `public/recompense/rewards-hero.png` | 422 Ko | **0** |
| `public/logo/relio-logo.svg`, `relio-pin.svg` | ~2 Ko | **0** |
| `MTN&ORANGE/` (2 fichiers) | 30 Ko | **0** |
| `logo/relio-logo.svg` (racine) | 1 Ko | **0** |

Attention : `public/recompense/petit-electromenager.png`,
`electromenager-moyen.png` et `smartphone.png` **sont** utilisés
(`components/landing/landing-sections.tsx:228-238`) — ne pas les confondre avec
le dossier `recompense/` de la racine, qui est un reliquat.

**Impact.** Chaque clone paie 12,9 Mo inutiles ; le dépôt grossit à chaque
version. Aucun impact sur le bundle servi (ces fichiers ne sont pas dans
`public/` pour la plupart), donc **aucun impact utilisateur**.

### 6.5 `referrals-service.ts` contourne `siteConfig`

`src/lib/api/referrals-service.ts:21` :

```ts
const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL ?? ''}${path}`, { ... });
```

C'est le **seul** module de `lib/api` à ne pas passer par `siteConfig`
(`lib/site-config.ts:6`). Deux sources pour la même URL de base : si demain la
résolution change (proxy, versionnage), ce module restera sur l'ancienne.

### 6.6 `/auth/me` appelé 8 fois en plus du `AuthProvider`

`AuthProvider` (`components/auth/auth-provider.tsx:30`) charge déjà la session.
Huit consommateurs refont l'appel :

| Fichier | Ligne |
|---|---|
| `app/client/parrainage/page.tsx` | 134 |
| `app/client/profil/page.tsx` | 45 |
| `app/technicien/historique/page.tsx` | 26 |
| `app/technicien/page.tsx` | 118 |
| `components/mission/conversation-section.tsx` | 46 |
| `components/client/dashboard/use-client-dashboard-data.ts` | 110 |
| `components/client/demande-wizard.tsx` | 247 |
| `components/auth/verification-panel.tsx` | 284 |

**Impact.** Requête supplémentaire par écran, et risque de désynchronisation
entre le contexte et l'état local. `useAuth()` expose déjà `user`.

### 6.7 `Field` ne relie pas l'erreur à son champ

`src/components/ui/field.tsx` affiche l'erreur dans un `<p role="alert">` mais
ne transmet ni `aria-invalid` ni `aria-describedby` au champ enfant. Un lecteur
d'écran annonce l'erreur **globalement**, puis ne sait pas **quel** champ est
fautif quand l'utilisateur revient dessus.

### 6.8 Navigation inférieure : cibles < 44 px et libellés 11 px

`src/components/ui/bottom-nav.tsx` — libellés en `text-2xs` (11 px, **24**
occurrences dans le dépôt) et `text-[11px]` (15 occurrences). Les cibles
tactiles de la barre sont sous les 44 × 44 px recommandés par les WCAG 2.5.5
(niveau AAA) et sous les 48 px de Material.

### 6.9 `key={index}` sur 8 listes

`app/admin/assistant/page.tsx:102` · `app/admin/page.tsx:99` ·
`components/client/dashboard/client-dashboard-skeleton.tsx:30` ·
`components/client/demande-wizard.tsx:984` ·
`components/mission/celebration.tsx:49` ·
`components/technician/dashboard/dashboard-skeleton.tsx:27` ·
`components/ui/breadcrumbs.tsx:40` · `components/ui/rating-stars.tsx:27`

Sur les squelettes et les étoiles, c'est sans conséquence (liste statique).
Sur `demande-wizard.tsx:984`, la liste est **dynamique** : un retrait d'élément
en milieu de liste fait réutiliser l'état du DOM de l'élément suivant.

### 6.10 `safeRedirect` : comparaison de préfixe sans frontière

`src/lib/api/auth-service.ts:147-152` accepte `target.startsWith('/client')`,
donc aussi `/clientXYZ`. Aucune route de ce nom n'existe : **non exploitable
aujourd'hui**. Durcissement : `target === p || target.startsWith(p + '/')`.

### 6.11 Le token de vérification e-mail est persisté dans `localStorage`

`src/lib/demande-draft-storage.ts:52-95`. Le commentaire (l. 60) affirme :

> « Une clé en **sessionStorage** permet au composant remonté de reconstituer »

Or `rememberVerifiedToken` utilise `storage()` qui retourne
**`window.localStorage`** (l. 14-24). Le token de vérification — que la
RÈGLE 5 classe parmi les **secrets** — survit donc à la fermeture du
navigateur, indéfiniment, jusqu'à ce qu'un autre token le remplace. Le
commentaire l. 65-67 affirme « Ce n'est PAS un secret » alors que
`readVerifiedToken()` retourne le token lui-même.

**Impact.** Faible en pratique (le token est consommé côté backend après
usage), mais c'est une divergence **documentation / code** sur un secret, et
elle contredit la RÈGLE 5.

### 6.12 `package.json` sans `engines`

Le README et `.nvmrc` exigent **Node ≥ 22** (runner `.ts` natif de
`node --test`). `package.json` ne déclare aucun `engines` : un `npm install`
sur Node 20 passe sans avertissement, puis `npm run test:unit` échoue à la
résolution des modules TypeScript.

### 6.13 `<img>` sans `alt`

`src/app/admin/kyc/[technicianId]/page.tsx:360` — aperçu d'un document KYC sans
texte alternatif. Les 5 autres `<img>` du dépôt ont un `alt` (ou en héritent).

### 6.14 `target="_blank"` avec `rel="noreferrer"` seul (×3)

`components/client/demande-auth-modal.tsx:366,375` et
`app/technicien/kyc/page.tsx:588`. `noreferrer` implique `noopener` dans tous
les navigateurs modernes : **non exploitable**, mais l'explicite
`noopener noreferrer` est déjà la convention ailleurs dans le dépôt
(`verification-panel.tsx:439`).

### 6.15 28 couleurs hexadécimales en dur dans les composants

`components/mission/celebration.tsx:8-14` (7 couleurs),
`components/mission/dispatch-sonar-widget.tsx:93,99,100`,
`components/client/parrainage/parrainage-overview.tsx:6` (`#0F172A`),
`app/layout.tsx:61` (`themeColor: '#F97316'`). Le design system vit dans
`globals.css` ; ces valeurs y échappent et ne suivront pas un changement de
charte.

### 6.16 Un seul `loading.tsx`, un seul `error.tsx`

| Fichier | Portée |
|---|---|
| `app/loading.tsx` | global |
| `app/error.tsx` | global |

Aucun `loading.tsx` ni `error.tsx` par segment. Conséquence : une erreur dans
un écran client fait remonter le repli **au niveau racine**, donc l'utilisateur
perd la navigation de sa section. Et aucune route ne bénéficie du streaming
Suspense de l'App Router.

### 6.17 `console.warn` en production

`components/client/demande-wizard.tsx:216,229,417,448` et
`app/technicien/demandes/[id]/page.tsx:238` écrivent dans la console de
l'utilisateur final, y compris en production.

---

## 7. Ce qui est solide — vérifié, pas supposé

Ces points ont été **testés par recherche**, pas supposés bons. Ils comptent
autant que les défauts : ils délimitent le périmètre des corrections.

| Point | Preuve |
|---|---|
| Aucun `any` implicite ou explicite | `strict: true`, `tsc --noEmit` **exit 0** |
| Aucun `@ts-ignore` / `@ts-expect-error` | 0 occurrence |
| Aucun `dangerouslySetInnerHTML`, `innerHTML =`, `eval`, `new Function` | 0 occurrence |
| Aucun TODO/FIXME/HACK réel | 12 correspondances, toutes des `XXXXX` de format (référence `RD-XXXXXX`, `RELIO-XXXXX`) |
| Aucun `catch {}` totalement vide | 0 correspondance — les `catch` silencieux sont commentés |
| Aucun lien interne cassé | 38 hrefs statiques × 60 routes → **0 orphelin** |
| Aucun secret dans une URL | token de brouillon et token de vérification restent en `localStorage`, jamais en query de tracking |
| `next/image` correctement utilisé | 5 usages, **tous** avec `sizes` ; `priority` sur le hero |
| `prefers-reduced-motion` respecté | 29 occurrences (`motion-reduce`, `motion-safe`, `matchMedia`) |
| Focus trap + ESC + scroll lock | `components/ui/modal.tsx` complet |
| Client SSE robuste | backoff 1→30 s, bascule `EventSource` → `fetch` streaming après 2 échecs, statut `polling` après 5 |
| Compteur de non-lus mutualisé | `lib/use-unread-notifications.ts` : un seul intervalle par onglet, arrêté au dernier abonné |
| Nettoyage des hooks | `removeEventListener` présent partout où un listener est posé ; `clearInterval` présent partout où un timer est créé ; `revokeObjectURL` présent sur les 2 `createObjectURL` |
| Miroirs financiers exacts | `0.045` (SasPay), `100` / `10 000 000` (bornes) — identiques au backend |
| Miroir de la politique mot de passe | `lib/password-strength.ts` **est** exact ; il est simplement sous-utilisé |
| Architecture sans middleware | `ARCHITECTURE.md` + `no-middleware.test.ts` verrouillent la décision cross-domain |
| `poweredByHeader: false` | `next.config.ts:5` |
| `lang="fr"` | `app/layout.tsx:71` |

---

## 8. Fichiers

### 8.1 Créés

| Chemin | Nature |
|---|---|
| `RAPPORT-AUDIT-FRONTEND.md` (racine `Relio`) | ce rapport |

### 8.2 Modifiés

**Aucun.** Audit en lecture seule.

### 8.3 Supprimés

**Aucun.**

---

## 9. Migrations

**Aucune.** Aucun changement de schéma, aucune dépendance ajoutée ou retirée.

---

## 10. Vérifications exécutées

| Commande | Résultat |
|---|---|
| `npm exec -- tsc --noEmit` (frontend) | **exit 0**, aucune sortie |
| `npm run lint` (frontend) | **exit 0** — `oxlint` : **4 warnings** `no-unused-vars` ; `eslint` : **5 warnings** `react-hooks/exhaustive-deps` ; **0 erreur** |
| `npm run test:unit` (frontend) | **613 pass / 2 fail** sur 615 |
| `git status` (frontend) | propre, aucune modification |
| `git status` (racine) | `?? backend/`, `?? frontend/`, `?? RAPPORT-AUDIT-FRONTEND.md` |

### 10.1 Les 5 warnings `react-hooks/exhaustive-deps`

```
src/app/admin/catalog/[domainId]/[problemId]/[diagnosticId]/page.tsx:115  'load' manquant
src/app/admin/catalog/[domainId]/[problemId]/page.tsx:67                  'load' manquant
src/app/admin/catalog/[domainId]/[brandId]/page.tsx:70                    'load' manquant
src/app/admin/catalog/[domainId]/page.tsx:68                              'load' manquant
src/app/technicien/profil/page.tsx:254                                    'activity.activityType', 'specialtiesList' manquants
```

### 10.2 Les 2 échecs de test

Identiques et reproductibles : `src/lib/demande-draft-routing.test.ts:93` et
`:102`, cause unique `demande-draft-routing.test.ts:21` (chemin Windows, § 5.8).

**Aucun autre échec.** Aucun test n'a été marqué `skip` ni exclu pour obtenir
ce résultat.

### 10.3 Preuve de non-régression

**Sans objet** : aucun fichier de code n'a été modifié. Le rapport ne peut donc
introduire aucune régression applicative.

---

## 11. Plan de correction proposé (non appliqué)

Ordre choisi par **rapport impact / effort**, pas par sévérité brute.

| # | Action | Effort | Effet |
|---|---|---|---|
| 1 | `fileURLToPath` dans `demande-draft-routing.test.ts:21` | 1 ligne | Rend la RÈGLE 3 applicable sous Windows (615/615) |
| 2 | `password-strength.ts` comme source unique des 4 formulaires | ~1 j | Supprime un échec de soumission évitable sur le tunnel d'inscription |
| 3 | `lib/api/client.ts` : un `apiFetch` avec timeout + `onUnauthorized`, suppression des 14 copies | ~2 j | Supprime les spinners infinis et l'expérience « erreur par écran » à l'expiration de session |
| 4 | Un fichier Lottie par animation | ~0,5 j | −434 Ko sur toutes les pages publiques et sur `LoadingScreen` |
| 5 | PNG 1200×630 + 180×180, `metadataBase` avec repli, `assertEnv()` | ~0,5 j | Vignette de partage WhatsApp rétablie |
| 6 | `--primary-ink` + remplacement des 71 `text-primary` textuels et des 40 `text-slate-400` | ~1 j | Conformité AA sur tout le texte |
| 7 | `headers()` dans `next.config.ts` (CSP, frame-ancestors, Referrer-Policy, Permissions-Policy) | ~0,5 j | Filet de sécurité, à faire **après** avoir listé les origines réellement appelées |
| 8 | `instrumentation.ts` + `global-error.tsx` + reporting | ~1 j | Une erreur de production devient diagnosticable |
| 9 | `<main>` + lien d'évitement dans les 3 layouts, `<h1>` sur les 13 pages | ~1 j | Navigation lecteur d'écran utilisable |
| 10 | Suppression des 3 fichiers morts, des 4 imports, des 2 `exclude`, des 12,9 Mo d'assets | ~1 h | Dette visible supprimée |
| 11 | `metadata` sur les 49 pages restantes + `robots.ts` + `sitemap.ts` | ~1 j | SEO |
| 12 | Découpage de `demande-wizard.tsx` | ~3 j | À planifier, pas à improviser |

---

## 12. Points d'attention

1. **Les poids annoncés sont des tailles disque.** Le bundle réel après
   minification/gzip n'a pas été mesuré (build interdit en local). L'ordre de
   grandeur et le mécanisme (import statique de 7 JSON) sont, eux, certains.
2. **Les contrastes sont calculés, pas mesurés à l'écran.** Les ratios WCAG
   sont exacts pour les tokens de `globals.css`, mais un `text-primary` posé
   sur un fond `bg-primary/5` ou une image donne un ratio différent de celui
   annoncé. Le chiffre de 71 occurrences est le nombre d'usages de la classe,
   pas le nombre d'échecs certains.
3. **Le rendu réel n'a pas été observé.** Aucun écran n'a été ouvert. Un défaut
   purement visuel (chevauchement, débordement) ne peut pas figurer ici.
4. **`framer-motion: ^13.4.4`** est déclaré dans `package.json`. La version
   n'a pas été vérifiée sur le registre npm ; si elle n'existe pas, `npm ci`
   échouerait — mais `node_modules` est présent et le lint passe, donc la
   dépendance est bien installée sur cette machine.
5. **Le dossier `frontend/` est non suivi à la racine** (`?? frontend/`) : il
   s'agit d'un dépôt imbriqué, conformément à la RÈGLE 2. Il ne doit jamais
   être ajouté à l'index racine.
6. **Les 5 warnings `exhaustive-deps` ne sont pas tous des bugs.** Les quatre
   `'load' manquant` sont le schéma classique « fonction définie dans le corps,
   appelée depuis un `useEffect` » : la correction propre est un `useCallback`,
   pas un ajout de dépendance.
7. **Le point 5.2 (absence de timeout) est le seul défaut dont l'impact est
   directement proportionnel à la qualité du réseau** — donc au marché visé.
   C'est le premier chantier que je recommanderais après le correctif de test.

---

## 13. Questions bloquantes

Aucune question ne bloque la livraison de ce rapport.

Deux décisions appartiennent au porteur de projet et conditionnent la suite :

1. **Faut-il corriger, et dans quel ordre ?** Ce rapport ne corrige rien. Le
   plan du § 11 est une proposition ; les points 1, 2 et 4 sont les seuls dont
   le bénéfice est immédiat et le risque quasi nul.
2. **La CSP (§ 5.5) exige la liste des origines réellement appelées**
   (Railway, OSM/OpenTopoMap pour les tuiles, éventuel domaine SasPay). Elle ne
   peut pas être écrite sans cette liste, sous peine de casser la carte ou les
   paiements.

Rappels de la tâche précédente, toujours ouverts et **hors périmètre de cet
audit** :

- pousser le correctif du bug de chemin Windows sur `Repairdom-frontend` ?
- renseigner les vraies valeurs de `backend/.env` et `frontend/.env` ?

---

## 14. Push de ce rapport

| Élément | Valeur |
|---|---|
| Dépôt | racine `projet.git` (`https://github.com/digitalholding2026-ux/projet.git`) |
| Branche | `main` |
| Fichier | `RAPPORT-AUDIT-FRONTEND.md` (unique fichier ajouté à l'index) |
| `backend/` et `frontend/` | **jamais ajoutés** (dépôts imbriqués, RÈGLE 2) |
| Secrets | aucun |
