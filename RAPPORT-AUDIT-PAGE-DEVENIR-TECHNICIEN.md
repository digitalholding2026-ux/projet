# RAPPORT — AUDIT : page `/devenir-technicien`

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Nature | **AUDIT en lecture seule stricte** — aucune modification de code, aucun commit applicatif, aucun push applicatif |
| Dépôts consultés | `Repairdom-frontend` (`5c29502` → `d6d5c49`), `Repairdom-backend` (`9edfbb2`) — **lecture seule, inchangés** |
| Seul commit | dépôt racine `projet`, ce rapport |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |
| Méthode | lecture de sources, `grep`/`wc`, un sous-agent d'exploration sur le parcours d'inscription |

## Synthèse

La page `/devenir-technicien` est un **composant serveur de 214 lignes, sans
donnée dynamique et sans appel API**. Tout son contenu est constitué de quatre
tableaux statiques déclarés dans le fichier lui-même, mappés en JSX.

Elle ne contient **aucun chiffre**. Ni revenu, ni commission, ni volume, ni
délai. Les seuls nombres visibles par l'utilisateur sont « Étape 1 » à
« Étape 8 », dérivés de la longueur d'un tableau.

Elle est référencée à 5 endroits en production (header desktop, header mobile,
footer, hero de la landing, bloc « Vous êtes technicien ? » de la landing) et
par aucun e-mail.

**Deux écarts de fond avec le produit réel**, qui ne relèvent pas du design :

1. **Le parcours décrit n'est pas le parcours implémenté.** La page présente
   8 étapes linéales dont « Faites vérifier votre identité » est la 3ᵉ. Le code
   réel (`onboarding-steps.ts:38-72`) définit 4 étapes, dans un autre ordre
   (profil → KYC → zones → disponibilité), et **aucune n'est bloquante** :
   l'ordre vient d'un hook d'affichage, pas d'une machine à états.
2. **Le paiement, qui est la question numéro un d'un technicien candidat, est
   absent des 8 étapes.** Il n'apparaît que dans le panneau du tunnel
   d'inscription (`auth-split.tsx:78`), que la page ne reprend pas.

Le barème réel existe dans le code — « Commission Relio (500 FCFA + 4 %) »,
`technician-quote.ts:46` — et n'est repris nulle part sur la page.

## 1. PAGE ACTUELLE

### a) Fichier

`frontend/src/app/devenir-technicien/page.tsx` — **214 lignes**.

Composant **serveur** (RSC) : aucune directive `'use client'` dans le fichier.

### b) Structure — blocs de haut en bas

| Lignes | Bloc | Contenu |
|---|---|---|
| 8-10 | `metadata` | `{ title: 'Devenir technicien' }` — rien d'autre |
| 12-24 | Constante `steps` | 8 étapes, tableau statique en module |
| 26-47 | Constante `benefits` | 4 arguments, tableau statique en module |
| 49-54 | Constante `requirements` | 4 prérequis, tableau statique en module |
| 56-62 | Constante `framework` | 5 labels, tableau statique en module |
| 66-67 | Chrome | `<div>` racine + `<PublicHeader />` |
| 69 | Conteneur | `<main className="mx-auto w-full max-w-6xl …">` |
| 70-102 | **HERO** | badge « Espace professionnel », `<h1>`, chapeau, CTA principal, lien « Déjà technicien ? » |
| 104-128 | **Comment ça fonctionne ?** | `<ol>` de 8 étapes, `aria-labelledby="how-title"` |
| 130-146 | **Pourquoi rejoindre Relio ?** | grille 4 cartes, `aria-labelledby="benefits-title"` |
| 148-167 | **Ce qu'il faut pour commencer** | `<ul>` 4 prérequis + paragraphe de clôture, `aria-labelledby="requirements-title"` |
| 169-185 | **Un fonctionnement encadré** | 5 pastilles, `aria-labelledby="framework-title"` |
| 187-208 | **CTA final** | `<h2>` « Prêt à rejoindre Relio ? » + 2 boutons |
| 211 | Chrome | `<PublicFooter />` |

Le rendu est 100 % déclaratif : quatre tableaux statiques mapnés en JSX. Aucun
état, aucun effet, aucune condition hors `index < steps.length - 1` (l. 112).

### c) Contenu

**Titres**

- `<h1>` l. 77-79 : « Devenez technicien Relio »
- `<h2>` l. 106 : « Comment ça fonctionne ? »
- `<h2>` l. 132 : « Pourquoi rejoindre Relio ? »
- `<h2>` l. 150 : « Ce qu'il faut pour commencer »
- `<h2>` l. 171 : « Un fonctionnement encadré »
- `<h2>` l. 192 : « Prêt à rejoindre Relio ? »

**Chapeau hero** (l. 80-83) : « Rejoignez la plateforme qui met en relation les
clients ayant besoin d'un dépannage avec des techniciens qualifiés près de chez
eux. »

**Les 8 étapes** (l. 12-24) — libellés exacts :

| # | Libellé | Icône |
|---|---|---|
| 1 | « Créez votre compte » | `user` |
| 2 | « Complétez votre profil professionnel » | `file` |
| 3 | « Faites vérifier votre identité » | `shield-check` |
| 4 | « Recevez des demandes correspondant à votre zone et vos compétences » | `pin` |
| 5 | « Échangez avec le client » | `chat` |
| 6 | « Établissez votre diagnostic » | `search` |
| 7 | « Proposez votre tarif » | `badge-check` |
| 8 | « Réalisez l'intervention » | `wrench` |

**Les 4 arguments** (l. 26-47) — titre + texte exacts :

| Titre | Texte | Icône |
|---|---|---|
| « Des demandes près de chez vous » | « Vous recevez les demandes correspondant à votre ville d'intervention et à vos compétences. » | `pin` |
| « Vous proposez votre tarif » | « C'est vous qui établissez le diagnostic et le tarif avant toute intervention. » | `badge-check` |
| « Un échange direct avec le client » | « Discutez avec le client avant de vous engager, via la messagerie de la mission. » | `users` |
| « Votre réputation s'affiche » | « Clients et techniciens s'évaluent après chaque mission. » | `star` |

**Les 4 prérequis** (l. 49-54) : « Un compte technicien » · « Un profil
professionnel complet (zone et compétences) » · « Vos informations
professionnelles (téléphone, ville, catégories) » · « Une vérification d'identité
par l'équipe Relio ».

Clôture l. 163-166 : « Une fois votre compte créé, votre dossier est revu par
l'équipe Relio avant que vous receviez vos premières demandes. »

**Les 5 labels d'encadrement** (l. 56-62) : « Techniciens vérifiés » · «
Diagnostic clair » · « Tarif avant intervention » · « Suivi de mission » ·
« Historique des interventions ».

**Chiffres et statistiques**

**ABSENT.** Recherche sur tout le fichier : aucune occurrence de `FCFA`, `XAF`,
`%`, aucun montant, aucun compteur. Les seules suites de chiffres sont des
classes utilitaires Tailwind (`text-2xl`, `sm:p-8`, `bottom-[19px]`) et
`index + 1` (l. 122), qui produit un libellé d'étape.

Il n'y a donc rien à qualifier de hardcodé ou de réel : **la page ne contient
aucune statistique**.

**CTA**

| Position | Label | href | Lignes |
|---|---|---|---|
| Hero, principal | « Devenir technicien » | `/technicien/inscription` | 85-92 |
| Hero, secondaire | « Se connecter » (lien texte, précédé de « Déjà technicien ? ») | `/technicien/connexion` | 97-99 |
| CTA final, principal | « Devenir technicien » | `/technicien/inscription` | 197-201 |
| CTA final, secondaire | « Déjà technicien ? Se connecter » (`variant="secondary"`) | `/technicien/connexion` | 202-206 |

Deux libellés identiques pour la même destination, écrits en dur à chaque
occurrence — aucune constante partagée.

### d) Composants utilisés

| Composant | Import | Occurrences |
|---|---|---|
| `Button` | l. 3, `@/components/ui/button` | 3 balises (l. 86, 198, 203) |
| `Icon` + type `IconName` | l. 4, `@/components/ui/icon` | 7 balises (l. 74, 95, 119, 139, 157, 180, 190) + 5 usages du type |
| `PublicHeader` | l. 5 | 1 (l. 67) |
| `PublicFooter` | l. 5 | 1 (l. 211) |
| `Link` | l. 2, `next/link` | 4 |

Aucun composant métier. La page n'utilise **ni** `Card`, **ni** `Section`, **ni**
`ProgressionBar`, **ni** `EmptyState` : tout le JSX est inline, avec des classes
qui dupliquent celles de `landing-sections.tsx`.

Les 15 icônes utilisées (`user`, `file`, `shield-check`, `pin`, `chat`,
`search`, `badge-check`, `wrench`, `users`, `star`, `phone`, `clock`,
`briefcase`, `check-circle`, `sparkles`) font partie des **56** `IconName`
disponibles dans `icon.tsx`. Contrainte ferme pour une refonte : aucun nom hors
de cette liste.

### e) Appels API

**ABSENT.** Aucun `fetch`, aucun import de `@/lib/api/`, aucun `useEffect`, aucun
`useState`, aucune fonction `async`. `DevenirTechnicienPage` est synchrone (l. 64).

### f) Routes liées

**6 références** — 5 en production, 1 en test :

| Fichier | Ligne | Contexte |
|---|---|---|
| `components/public/public-header.tsx` | 87 | Lien desktop, « Devenir technicien » |
| `components/public/public-header.tsx` | 157 | Lien menu mobile, `onClick={closeMenu}` |
| `components/public/public-footer.tsx` | 52 | Section « Professionnels », sous-titre « Vous êtes professionnel ? » |
| `components/landing/hero.tsx` | 51 | CTA secondaire du hero (contour blanc, `Icon briefcase`) |
| `components/landing/landing-sections.tsx` | 292 | Bloc « Vous êtes technicien ? », CTA `bg-slate-900` |
| `components/auth/technician-auth-split.tsx` | 43 | Pied du tunnel : `href={isSignUp ? '/technicien/connexion' : '/devenir-technicien'}` |
| `lib/technician-auth.test.ts` | 192 | Assertion sur le pied du tunnel (cf. §6) |

**Émails : ABSENT.** Aucun template ne pointe vers cette route. `EmailService`
expose 10 méthodes, aucune n'est un message de bienvenue technicien : les
e-mails technicien sont `sendVerificationEmail`, `sendKycVerifiedEmail`,
`sendKycRejectedEmail`, `sendFeeChangeEmail`, `sendVerificationReminderEmail`,
`sendMissionAvailable`.

### g) Données chargées

**Données dynamiques : néant.** Zéro requête, zéro prop, zéro paramètre de
route, zéro context.

**Données statiques :** les 4 tableaux (`steps`, `benefits`, `requirements`,
`framework`), tous déclarés dans le fichier, tous typés par annotation inline
(`Array<{ icon: IconName; … }>`). Aucune constante importée, aucun contenu
externalisé.

La seule donnée non triviale d'exécution est `steps.length`, utilisée deux fois :
l. 112 (trait de liaison verticale) et l. 122 (« Étape N »).

## 2. PARCOURS D'INSCRIPTION TECHNICIEN

**Le flux n'est pas la séquence linéaire de l'énoncé.** Il n'y a pas de tunnel
multi-étapes bloquant : une inscription unique, puis un tunnel d'onboarding
**libre** où l'ordre n'est pas imposé.

Source de vérité : `frontend/src/lib/technician/onboarding-steps.ts` (144 lignes),
critères `done` l. 96-105.

| Rang | Étape | Route | Critère « terminé » |
|---|---|---|---|
| 1 | Compléter mon profil | `/technicien/profil` | `categories.length > 0 && cityId != null` (l. 98) |
| 2 | Vérifier mon identité | `/technicien/kyc` | `kycStatus === 'VERIFIED'` (l. 100) |
| 3 | Définir mes zones | `/technicien/zones` | `coverage.length > 0` (l. 102) |
| 4 | Me rendre disponible | `/technicien` | `isAvailable === true` (l. 104) |

Aucune étape ne redirige vers la suivante. Le dashboard affiche une bannière et
une checklist ; chaque page est atteignable directement par URL et par le menu
latéral.

| # | Route | Fichier | Lignes | Champs réels | Endpoint principal |
|---|---|---|---|---|---|
| 1 | `/technicien/inscription` | `app/technicien/inscription/page.tsx` | 12 | **7** | `POST /auth/register` |
| 2 | `/technicien/verification` | `app/technicien/verification/page.tsx` | 15 | **0** | `POST /auth/verify-email` |
| 3 | `/technicien/profil` | `app/technicien/profil/page.tsx` | 485 | **12** | `PATCH /technician/profile` + `PATCH /auth/me` |
| 4 | `/technicien/kyc` | `app/technicien/kyc/page.tsx` | 723 | **7** | `POST /technician/kyc/submit` |
| 5 | `/technicien/zones` | `app/technicien/zones/page.tsx` | 490 | **2** | `PUT /technician/coverage` |
| — | `/technicien/onboarding` | `app/technicien/onboarding/page.tsx` | 185 | **0** | *(GET profil + couverture, via hook)* |
| — | `/technicien` (dashboard) | `app/technicien/page.tsx` | 509 | **0** | `PATCH /technician/profile` (disponibilité) |
| — | `/technicien/connexion` | `app/technicien/connexion/page.tsx` | 13 | **2** | `POST /auth/login` |

Les pages 1, 2 et 8 sont des coquilles : leurs champs sont dans
`components/technician/technician-auth-form.tsx` (403 lignes) et
`components/auth/verification-panel.tsx`.

**Les 7 champs d'inscription**, avec lignes vérifiées :

| Label | Contrôle | Obligatoire | Ligne |
|---|---|---|---|
| `Prénom *` | Input | oui | 181 |
| `Nom *` | Input | oui | 190 |
| `Téléphone *` | Input | oui | 203 |
| `Adresse e-mail *` | Input `type=email` | oui | 215 |
| `Mot de passe *` | Input password | oui, min 8 | 226 |
| `Ville d'intervention *` | Select alimenté par `GET /cities` | oui | 281 |
| `Catégories de réparation *` | groupe de `<button role="checkbox">` | oui (≥ 1) | 337 |

Second appel au montage : `listCities()` → `GET /cities`
(`technician-auth-form.tsx:85-88`). `GET /cities` est public, sans JWT.

**Profil, 12 champs** : photo (upload), téléphone, WhatsApp, type d'activité,
années d'expérience (0-70), bio (max 600), catégories d'appareils
**obligatoires**, types d'équipements, spécialités (max 20), ville
d'intervention **obligatoire**, notifications push. Nom et email en lecture seule.
Les deux requêtes de sauvegarde sont lancées en `Promise.all` (l. 219-235) : pas
d'atomicité, l'échec de `PATCH /auth/me` fait échouer l'enregistrement visible du
profil.

**KYC, 7 champs** — sauvegarde auto **debouncée 600 ms** sur l'identité
(l. 219-230), pas de bouton d'enregistrement :

| Label | Obligatoire |
|---|---|
| Date de naissance (majeur 18 ans) | oui |
| Nationalité (ISO 3166-1) | oui |
| Nature de votre pièce | oui |
| Recto / Verso (CNI) ou page unique (passeport) | oui, conditionnel |
| Justificatif professionnel | **non (facultatif)** |

Endpoints : `GET /technician/kyc`, `POST /technician/kyc/documents`,
`DELETE /technician/kyc/documents/:id`, `GET /technician/kyc/documents/:id/url`
(URL signée 5 min), `POST /technician/kyc/submit`.

Déposer un fichier ne change pas le statut : seule la soumission explicite
passe le dossier à `PENDING`.

**Zones, 2 champs** : ville de référence (**bloquante** — sans elle, aucune zone
n'est proposée, l. 322) et zone à ajouter. GPS via
`getCurrentTravelPosition()` (géolocalisation navigateur).

**Redirections** — `app/technicien/layout.tsx` (114 lignes) ne contient **aucun**
`router.push` ; il délègue à `RoleGuard` (l. 83-85) →
`lib/guard-decision.ts:45-89` :

| Condition | Destination |
|---|---|
| Technicien non vérifié, hors page de vérification | `/technicien/verification` |
| Rôle ≠ TECHNICIAN, ou route publique alors qu'il est connecté | `roleHomePath(role)` |
| Non authentifié + route privée | `/technicien/connexion?redirect=<path>` |

Trois routes sont publiques (`layout.tsx:30`) : `/technicien/connexion`,
`/technicien/inscription`, `/technicien/verification`. Deux sont « immersives »
(`layout.tsx:34`, header global masqué) : `/technicien/inscription` et
`/technicien/connexion`.

**Écart avec la page auditée** : la page annonce « Faites vérifier votre identité »
comme étape 3 d'un parcours linéaire. Le code réel est profil → KYC → zones →
disponibilité, et aucune étape n'est bloquante.

## 3. ARGUMENTS ACTUELS

**Présents sur la page** (« Pourquoi rejoindre Relio ? », l. 131-146) : les 4 du
tableau §1c, reproduits tels quels.

**Arguments supplémentaires existants ailleurs dans le dépôt, absents de cette
page** — `TECHNICIAN_GUARANTEES`, `components/auth/auth-split.tsx:61-82`,
panneau du tunnel `/technicien/inscription` :

| Titre | Texte |
|---|---|
| « Missions dans vos zones » | « Vous ne recevez que les demandes de dépannage de votre secteur. » |
| « Identité vérifiée une fois » | « Contrôlez votre pièce d'identité, puis intervenez sans autre formalité. » |
| « Devis Systematic » | « Le prix de chaque intervention est validé avec le client avant de commencer. » |
| **« Paiement après validation »** | « Vous êtes payé une fois l'intervention terminée et validée. » |

Les deux derniers sont absents de la page de recrutement alors qu'ils figurent
déjà dans le tunnel : matière existante, non réutilisée.

**Barème** : `src/lib/technician-quote.ts:46` définit
`TECHNICIAN_FEE_LABEL = 'Commission Relio (500 FCFA + 4 %)'`, avec
`TECHNICIAN_FEE_FIXED_XAF = 500` et `TECHNICIAN_FEE_RATE_PERCENT = 4`. Recherche
sur la page : **0 occurrence** de « 500 », « 4 % » ou « commission ».

Le libellé est exposé au technicien dans `/technicien/revenus` (l. 26) et dans
l'e-mail `sendFeeChangeEmail` (`backend/src/auth/email.service.ts:377`).

**Aucun argument chiffré de revenu, de revenu moyen, de volume de missions ou de
délai de paiement n'existe dans le code** — ni sur cette page, ni ailleurs.

## 4. SECTIONS MANQUANTES

| Section | Présente ? | Constat |
|---|---|---|
| **Témoignages techniciens** | **NON** | Aucun sur la page. Le dépôt n'en contient aucun : le bloc témoignages de la landing a été **supprimé** — `app/page.tsx:22` : « Le bloc témoignages a été SUPPRIMÉ (faux avis). » |
| **Calcul de revenus estimés** | **NON** | Aucun simulateur, aucun chiffre. Le barème existe dans le code mais n'est pas repris ici. |
| **FAQ technicien** | **NON** | Aucune FAQ sur la page. Seule FAQ du site : `components/landing/faq.tsx`, **7 questions toutes orientées client** (devis, choix du technicien, paiement, annulation, zones du client, suivi, récompense client). Aucune question adressée à un technicien candidat. |
| **Avantages concrets** | **PARTIELLEMENT** | 4 arguments (§1c), sans aucune quantification. Le panneau du tunnel en ajoute 4 autres (§3) que la page ne reprend pas. |
| **Explications du process** | **PARTIELLEMENT** | Les étapes 5-7 couvrent diagnostic, tarif et intervention en une ligne chacune. **Le paiement n'apparaît pas** dans les 8 étapes : il n'existe que dans `TECHNICIAN_GUARANTEES`, hors page. |
| **Programme de récompenses technicien** | **NON** | Aucune mention. Un programme existe côté **client** (`/client/recompenses`, 6 visuels dans `public/recompense/`). Côté technicien, `/technicien/revenus` est un tableau de bord financier, pas un programme de récompenses. |
| **Support / accompagnement** | **NON** | Aucune mention de canal de contact, d'accompagnement à l'inscription, de formation ni de numéro à appeler. Seule mention de l'équipe Relio : la revue de dossier (l. 164-165). |
| **Prérequis (KYC, zones)** | **PARTIELLEMENT** | 4 prérequis listés (l. 49-54), dont la vérification d'identité. **Le dossier KYC lui-même n'est pas décrit** : nature des pièces acceptées (CNI recto-verso, passeport), majeur 18 ans, transmission à Relio. Ces informations n'existent que dans `/technicien/kyc`. |

Blocs non listés dans l'énoncé mais également absents :

- **Durée du parcours** : aucune estimation.
- **Nombre de techniciens déjà actifs** : aucun.
- **Conditions d'éligibilité négatives** : aucune exclusion (activité principale, matériel, disponibilité).
- **Grille tarifaire en vigueur** : absente (§3).
- **Missions de démonstration** : aucun exemple de type d'intervention.
- **Candidature alternative** : aucun formulaire — le seul chemin est le tunnel `/technicien/inscription`.

## 5. CONCURRENCE / INSPIRATIONS DISPONIBLES

| Recherche | Résultat |
|---|---|
| Page « Pourquoi Relio » pour techniciens | **ABSENT.** La section « Pourquoi rejoindre Relio ? » de la page elle-même est la plus proche. Aucune page dédiée. |
| Email de bienvenue technicien | **ABSENT.** Voir §1f pour les 10 méthodes de `EmailService`. |
| Témoignages dans d'autres pages | **ABSENT** — et supprimés délibérément (`app/page.tsx:22`). |
| Preuves sociales chiffrées | **PRÉSENT** sur la landing, avec masquage à l'échec : `components/landing/trust-stats.tsx:8` — « Preuves sociales — CHIFFRES 100 % RÉELS, SANS AUCUNE VALEUR EN DUR ». Deux indicateurs, « Villes couvertes » et « Zones couvertes », alimentés par `GET /cities` (l. 33). Le bloc disparaît si `cities.length === 0` (l. 34-35). **Seul chiffre réellement affiché sur le site public.** |

**Infrastructure d'avis existante et exploitable** : le module `Reviews` est
bidirectionnel. `backend/src/reviews/reviews.service.ts:86` —
`const targetId = user.role === 'CLIENT' ? demande.technicianId : demande.clientId`.
Un technicien **peut** noter un client, et un client peut noter un technicien,
après mission. Une réputation est lue pour les techniciens
(`technician.service.ts:1047, 1070`). Mais **aucun avis n'est exposé
publiquement**.

## 6. CONTRAINTES

### i18n / traduisibilité

**ABSENT.** Aucune couche i18n dans le dépôt : `useTranslation`, `next-intl` et
`i18next` sont absents de tout `src/`. Aucun segment `[locale]` dans `app/`. Les
4 tableaux sont du texte en dur. Le manifeste PWA déclare `lang: 'fr'` en dur
(`app/manifest.ts`). Aucun dictionnaire à alimenter.

### SEO

| Élément | État |
|---|---|
| `metadata` de page | **Partielle** — `{ title: 'Devenir technicien' }` seul (l. 8-10) |
| `description` de page | **ABSENT** |
| `openGraph` de page | **ABSENT** — hérite des valeurs globales |
| `twitter` de page | **ABSENT** — hérite des valeurs globales |
| `keywords` / `robots` | **ABSENT** |
| `generateMetadata` | **ABSENT** dans tout le dépôt |
| JSON-LD / données structurées | **ABSENT** |
| `sitemap.xml` | **ABSENT** — aucun fichier dans `app/` |
| `robots.txt` | **ABSENT** — aucun fichier dans `app/` |
| Canonical | **ABSENT** |

Hérité du `layout.tsx` racine (l. 26-55) : `metadataBase`, template de titre
`%s · Relio`, description globale, `openGraph` et `twitter` globals, `icons`.
Image OG par défaut : `/brand/relio-logo-light.svg`.

`/devenir-technicien` est l'une des **11** pages portant `export const metadata`
(hors `layout.tsx` racine qui en porte une lui aussi), sur les **10 segments de
route** de `app/` (la racine + 9 sous-dossiers : `admin`, `client`,
`conditions-utilisation`, `demande`, `devenir-technicien`, `mot-de-passe-oublie`,
`reinitialiser-mot-de-passe`, `suivi`, `technicien`).

Sans metadata : `src/app/page.tsx` (landing), `src/app/suivi`,
`src/app/admin/*`, `src/app/client/*` (hors `connexion`, `inscription`,
`confirmation`) et `src/app/technicien/*` (hors `connexion` et `inscription`).

### Tests statiques

**Aucun test ne lit `devenir-technicien/page.tsx`.** La seule occurrence de la
chaîne dans les tests est `lib/technician-auth.test.ts:192`, qui assert sur
`technician-auth-split.tsx` :

```
assert.match(technicianSplit, /'\/devenir-technicien'/);
```

Ce test échouerait si le pied du tunnel d'inscription ne pointait plus vers la
page. Il ne lit pas la page elle-même — **la page peut donc être réécrite sans
casser de test existant**.

Rappel pour la refonte : ce dépôt n'a ni jsdom ni testing-library ; les tests
statiques font `readFileSync` puis `assert.match` sur le **texte entier du
fichier**, commentaires et chaînes compris. Un test futur sur cette page
surveillerait des mots du source, pas du rendu.

## 7. FICHIERS CLÉS

| # | Fichier | Lignes | Rôle |
|---|---|---|---|
| 1 | `frontend/src/app/devenir-technicien/page.tsx` | 214 | La page auditée : ses 4 tableaux de contenu et tout son JSX. |
| 2 | `frontend/src/components/public/public-header.tsx` | 188 | 2 liens vers la page (l. 87 desktop, l. 157 mobile). |
| 3 | `frontend/src/components/public/public-footer.tsx` | 65 | Section « Professionnels » (l. 45-58). |
| 4 | `frontend/src/components/ui/icon.tsx` | 320 | Les 56 `IconName`. Contrainte ferme de la refonte. |
| 5 | `frontend/src/components/ui/button.tsx` | 67 | Le `Button` des 4 CTA ; variantes `default`, `secondary`, `lg`. |
| 6 | `frontend/src/lib/technician/onboarding-steps.ts` | 144 | Source de vérité du parcours réel (4 étapes, critères `done` l. 96-105). Incohérent avec les 8 étapes de la page. |
| 7 | `frontend/src/components/auth/auth-split.tsx` | 204 | `TECHNICIAN_GUARANTEES` (l. 61-82) : 4 arguments du tunnel, dont 2 absents de la page. |
| 8 | `frontend/src/lib/technician-quote.ts` | 118 | Barème réel : `TECHNICIAN_FEE_LABEL = 'Commission Relio (500 FCFA + 4 %)'` (l. 46). |
| 9 | `frontend/src/components/landing/trust-stats.tsx` | 93 | Précédent d'implémentation des chiffres réels : masquage si API en échec, aucune valeur en dur. |
| 10 | `frontend/src/lib/technician-auth.test.ts` | — | Seul test référençant la route (l. 192). Aucun test ne lit la page. |

Compléments : `frontend/src/components/landing/hero.tsx:51` et
`landing-sections.tsx:292` (deux autres points d'entrée),
`frontend/src/components/technician/technician-auth-form.tsx` (403 lignes, les
7 champs d'inscription), `frontend/src/app/technicien/kyc/page.tsx` (723 lignes,
seule description du dossier KYC).

## 8. DONNÉES DISPONIBLES POUR LA REFONTE

| Recherche | Résultat |
|---|---|
| **Chiffres réels sur les techniciens** (nombre, revenus moyens) | **ABSENT dans le code.** Aucun endpoint `TechnicianController` n'expose de statistiques : les 9 routes (`backend/src/technician/technician.controller.ts`) sont toutes authentifiées et personnelles — `profile`, `coverage`, `kyc`, `kyc/documents/:id/url`, `available`, `my-demandes`, `my-demandes/history`, `demandes/:id`, `demandes/:id/medias/:mediaId/file`. **Aucune route publique de comptage.** À VÉRIFIER si l'information existe en base. |
| **Chiffres réels géographiques** | **PRÉSENT.** `GET /cities` → villes et zones. Déjà consommé par `trust-stats.tsx` avec masquage à l'échec. Seul chiffre réellement affiché sur le site public. |
| **Avis / notes techniciens** | **Structure présente, contenu non.** Le module `Reviews` est bidirectionnel (`reviews.service.ts:86`) et une réputation est lue pour les techniciens (`technician.service.ts:1047, 1070`). Mais **aucun avis n'est exposé publiquement** : ni composant ni endpoint de liste d'avis technicians. La landing a supprimé ses faux avis. |
| **Photos / vidéos de techniciens** | **ABSENT.** Voir §8-bis pour le détail des 14 assets. Aucun portrait, aucune vidéo, aucun avatar de démonstration. |
| **Assets visuels** | 14 fichiers dans `frontend/public/` (détail ci-dessous). |

### 8-bis. Contenu de `frontend/public/`

| Dossier | Fichiers | Utilisable pour une page de recrutement |
|---|---|---|
| `brand/` | `relio-logo-light.svg`, `relio-mark.svg` | oui |
| `logo/` | `relio-logo.svg`, `relio-pin.svg` | oui |
| `hero/` | `hero_main.webp` | oui |
| `operators/` | `orange.jpg`, `mtn.jpg` | non (opérateurs de paiement) |
| `recompense/` | `iron.png`, `tv.png`, `smartphone.png`, `free_repair.png`, `tshirt_cap.png`, `mystery_box.png` | programme **client** |
| racine | `sw.js` | non |

**Aucun visuel de technicien, aucun portrait, aucune vidéo.**

### Matière textuelle existante, réutilisable sans invention

- 4 arguments du tunnel (`auth-split.tsx:61-82`), dont 2 absents de la page :
  « Identité vérifiée une fois » et « Paiement après validation ».
- Barème exact et à jour (`technician-quote.ts:46`) : « Commission Relio
  (500 FCFA + 4 %) ».
- 4 étapes réelles du parcours et leurs critères (`onboarding-steps.ts:38-72`).
- 5 libellés d'encadrement (l. 56-62) et 4 prérequis (l. 49-54), déjà sur la page.
- Aucune des 7 questions de `landing/faq.tsx` n'est transposable telle quelle :
  elles sont toutes adressées à un client.

## Points d'attention

1. **Le parcours affiché et le parcours implémenté divergent.** 8 étapes
   linéales affichées contre 4 étapes réelles non bloquantes, dans un ordre
   différent. C'est le seul écart de fond entre la page et le produit ; le
   reste est une question de contenu.

2. **Le paiement manque aux 8 étapes**, alors qu'il figure déjà dans le panneau
   du tunnel d'inscription (`auth-split.tsx:78`). L'information existe dans le
   dépôt, à un seul écran de la page.

3. **Aucune donnée de conversion technicien n'est exposée publiquement.** Les 9
   routes `TechnicianController` sont toutes personnelles et authentifiées. Un
   chiffre « X techniciens actifs » n'est pas récupérable depuis le frontend
   aujourd'hui ; il faudrait une route publique côté backend, qui n'existe pas.

4. **Les avis ne sont pas publics.** Le module `Reviews` est bidirectionnel et
   produit des notes dans les deux sens, mais rien ne les expose hors du cadre
   d'une mission. La landing a supprimé ses faux avis : l'antécédent est
   explicite dans `app/page.tsx:22`.

5. **Le barème technicien a déjà changé** (`09cc75f` a posé 500 XAF + 4 % et le
   minimum de devis 5 000 XAF, et `sendFeeChangeEmail` existe pour notifier les
   techniciens). Le libellé `TECHNICIAN_FEE_LABEL` est la source de vérité et
   n'est repris nulle part sur la page de recrutement : une nouvelle page qui
   l'écrive en dur créerait un second point de vérité, exactement comme l'a fait
   le barème de parrainage du chantier 4B (`referrals.config.ts` côté backend /
   `referrals-rules.ts` côté frontend, comparés par un test qui lit les deux
   fichiers).

6. **Deux libellés CTA identiques sont écrits en dur deux fois** (l. 90 et 199).
   Sans constante partagée, une refonte partial laissera les deux occurrences
   diverger.

7. **La contrainte `IconName` est ferme** : 56 noms, aucun autre. Une icône
   existante ailleurs dans le produit mais absente de `icon.tsx` ne compilera
   pas.

## Limites de cet audit

- **Lecture seule.** Aucun test n'a été exécuté, aucune page n'a été rendue, aucun
  appel HTTP n'a été fait. Les constats « 0 occurrence » sont des recherches
  textuelles (`grep`) sur les sources, pas des observations de rendu.
- **Le parcours d'inscription a été audité par un sous-agent d'exploration.**
  Les chiffres de champs ont été recoupés sur les libellés d'inscription
  (l. 181, 190, 203, 215, 226, 281, 337 de `technician-auth-form.tsx`) ; les
  décomptes de champs des pages `profil`, `kyc` et `zones` **n'ont pas été
  recomptés ligne à ligne** et sont à VÉRIFIER avant de fonder une conception
  dessus.
- **Aucune donnée de production n'a été interrogée.** Les chiffres réels sur les
  techniciens (nombre, revenus) sont marqués ABSENT **au sens du code** : ils
  peuvent exister en base. Sans accès base, c'est la seule conclusion
  possible.
- **Les pages hors dépôt n'ont pas été consultées.** Pas d'analyse SEO externe,
  pas de mesure de performance, pas de rendu sur mobile réel.

## Questions bloquantes

Aucune.

Trois éléments restent à trancher **hors de la lecture seule**, et sont hors
périmètre de cet audit :

1. Faut-il exposer publiquement des statistiques de techniciens (nombre actif,
   revenu moyen) ? Cela demande une nouvelle route backend, aujourd'hui
   inexistante.
2. Faut-il exposer publiquement des avis de techniciens ? La donnée existe et
   est bidirectionnelle, mais n'est aujourd'hui visible d'aucune mission.
3. Le parcours à afficher est-il celui de `onboarding-steps.ts` (4 étapes, non
   bloquant) ou une projection commerciale en 8 étapes ? La page actuel et le
   code ne disent pas la même chose.
