# RAPPORT — AUDIT : écrans mission client & technicien (lecture seule)

> Préparation d'une refonte UX de `/client/demandes/[id]` et
> `/technicien/demandes/[id]`. Audit **factuel, exhaustif, sans recommandation ni
> jugement**.
>
> **Mode : lecture seule stricte.** Aucun fichier modifié, aucun commit
> applicatif, aucun push applicatif, aucun build, aucun test, aucun serveur
> local.

---

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Nature | **Audit lecture seule** — 0 fichier modifié dans `backend/` ni `frontend/` |
| Frontend | `Repairdom-frontend` `95f7b8e` — inchangé |
| Backend | `Repairdom-backend` `2e03366` — non modifié (lecture du schéma Prisma seule) |
| Dépôt racine | `projet` — ce rapport seul |
| Vérification | `git status --porcelain` vide dans les deux dépôts applicatifs |

## Synthèse

Deux écrans de détail de mission écrits en JSX monolithique : **931 lignes**
côté client, **1 085 lignes** côté technicien. Le fichier technicien ne contient
**aucun sous-composant** ; sa card principale concentre 21 blocs
conditionnels. Six composants de `components/mission/*` sont partagés, quatre
sont utilisés par un seul écran, quatre ne le sont par aucun. 6 tests statiques
lisent ces pages, dont 3 verrouillent l'absence de hook après un `return`
conditionnel. **Il n'existe aucun système d'ancrage** (`#chat`, `#map`) ni dans
le code, ni dans les URLs push.

## Fichiers

Créés : ce rapport.

Modifiés : aucun. Supprimés : aucun.

Lus : `frontend/src/app/client/demandes/[id]/page.tsx` (931 l.),
`frontend/src/app/technicien/demandes/[id]/page.tsx` (1 085 l.),
`frontend/src/components/mission/*` (18 fichiers, 2 454 l.),
`frontend/src/components/ui/*` (39 fichiers),
`frontend/src/lib/*`, `backend/prisma/schema.prisma`,
`backend/src/demandes/demandes-lifecycle.ts`,
`backend/src/collaboration/collaboration.service.ts`,
`backend/src/technician/technician.service.ts`.

## Migrations

Aucune.

## Vérifications

- `git status --porcelain` (racine, backend, frontend) → **aucune modification de code**.
- Aucun build, aucun test, aucun serveur n'ont été lancés : les constats sont
  issus de la **lecture des sources**, et les tests sont décrits d'après leur
  code, pas d'après leur exécution.

## Questions bloquantes

Aucune.

---

# 1. ÉCRAN CLIENT `/client/demandes/[id]`

## a) Métadonnées

| | |
|---|---|
| Fichier | `frontend/src/app/client/demandes/[id]/page.tsx` |
| **Lignes** | **931** |
| `'use client'` | oui (l.1) |

**Imports de composants (26)** — UI : `Button` (l.6), `Skeleton`/`SkeletonCard`/`SkeletonRow` (l.7), `Avatar` (l.8), `Icon` (l.9), `Alert` (l.10), `Badge` (l.11), `ConfirmDialog` (l.12), `Field` (l.13), `Select` (l.14), `Textarea` (l.15), `SectionHeader` (l.16), `DemandeStatusBadge`/`QuoteStatusBadge` (l.17).
Métier : `DemandeProgress` (l.18), `MissionTimeline` (l.19), `DispatchSonarWidget` + `partitionDispatchWaves` (l.20), `FloatingChat` (l.21), `TravelBanner` (l.22), `MissionMap` (l.23), `RechercheTechnicienAnimation` (l.24), `DemandeMediaSection` (l.26), `DiagnosticAudioPlayer` (l.27), `RatingSection` (l.29).

**Composants enfants créés dans le fichier**

| Nom | Lignes | Rôle |
|---|---|---|
| `ClientDemandeDetailPage` | 71–865 | composant principal |
| `TravelMapSection` | 871–931 | carte de mission + légende GPS |

Helpers locaux : `formatPrice` (l.67–69), constante `POLL_INTERVAL_MS = 5000` (l.65).

## b) Structure visuelle actuelle — ordre top→bas

| # | Bloc | Ligne | Hauteur est. | Contenu | Condition | Composant responsable |
|---|---|---|---|---|---|---|
| 1 | Fil d'Ariane | 344–353 | ~24 px | « ← Mes missions » | toujours | `<nav>` + `Link` + `Icon` |
| 2 | En-tête `h1` | 356–364 | ~70 px | référence, badge statut, catégorie, date | toujours | `DemandeStatusBadge` |
| 3 | Bandeau déplacement | 368 | ~48 px | statut + fraîcheur + distance | toujours (`travel` peut être null) | `TravelBanner` |
| 4 | Carte localisation | 371–378 | ~380 px | Leaflet + légende | `travel?.enRoute \|\| travel?.arrived` | `TravelMapSection` → `MissionMap` |
| 5 | Alerte d'erreur | 380 | variable | — | `error` | `Alert` |
| 6 | Actions principales | 383–403 | ~48 px | Confirmer / Annuler | `COMPLETED` ou `canCancel` | `Button` ×2 |
| 7 | Litige | 406–448 | ~120 px | badge, catégorie, description, décision | `status === 'COMPLETED'` | `Badge` + `Alert` + `Button` |
| 8 | **Grille 2/3 + 1/3** | 450 | — | — | toujours | `<div grid lg:grid-cols-3>` |

**Colonne principale (`lg:col-span-2`, l.452)**

| # | Bloc | Ligne | Hauteur est. | Condition | Composant |
|---|---|---|---|---|---|
| 9 | Diagnostic & Solution | 454–521 | ~280 px | `demande.technician` | `SectionHeader` + `blockquote` bruts + `DiagnosticAudioPlayer` |
| 10 | Proposition d'intervention | 524–649 | ~340 px | `demande.technician` | `dl` brut + `ConfirmDialog` |
| 11 | Éléments multimédia | 652–658 | ~150–350 px | toujours | `DemandeMediaSection` |
| 12 | Chronologie | 661–673 | ~300 px | toujours | `DispatchSonarWidget` (≥2 vagues) ou `MissionTimeline` ou `DemandeProgress` |
| 13 | Avis sur le technicien | 675–681 | ~200 px | `technician && status === 'CONFIRMED'` | `RatingSection` |

**Colonne secondaire (`lg:col-span-1`, l.685)**

| # | Bloc | Ligne | Hauteur est. | Condition |
|---|---|---|---|---|
| 14 | Synthèse de la demande | 687–715 | ~300 px | toujours (`dl` brut : appareil, problème, lieu, contact) |
| 15 | Technicien assigné | 718–759 | ~240 px | avatar + ville + 2 boutons, **ou** animation Lottie de recherche |

**Hors grille**

| # | Bloc | Ligne | Condition |
|---|---|---|---|
| 16 | `FloatingChat` | 763–773 | `showChat` (portail `document.body`) |
| 17 | `ConfirmDialog` (action) | 775–808 | `confirmAction !== null` — 4 variantes de titre/description |
| 18 | `ConfirmDialog` (litige) | 810–862 | `disputeDialogOpen` — contient `Select` + `Textarea` |

**Hauteur cumulée estimée** en mission confirmée avec devis, diagnostic audio et
4 médias : ≈ 2 400 px desktop (≈ 4,5 écrans à 800 px de fenêtre), ≈ 3 600 px
mobile.

## c) États conditionnels

Gestion **par `if`/ternaires dans le JSX**, sans `switch`. Les variables
dérivées sont calculées **après** les returns anticipés, l.308–339 :

| Variable | Ligne | Règle |
|---|---|---|
| `canCancel` | 308 | `['SUBMITTED','PENDING','ACCEPTED','SCHEDULED'].includes(status)` |
| `baseCanDiscuss` | 309 | `status !== 'CANCELED' && status !== 'CONFIRMED'` |
| `catalogFlow` | 310 | `quotes.some(q => q.source === 'CATALOG')` |
| `negotiationUnlocked` | 311 | `negotiationRequestedAt` **ou** devis `ACCEPTED` |
| `canDiscuss` | 313 | `baseCanDiscuss && (!catalogFlow \|\| negotiationUnlocked)` |
| `showChat` | 314 | `technician` **et** `(!catalogFlow \|\| negotiationUnlocked)` |
| `showDispatchRadar` | 338 | `dispatchWaves.length >= 2` |

**Répartition par statut**

| Statut | Blocs rendus |
|---|---|
| `SUBMITTED`/`PENDING` | 1,2,3,4?,5,6(annuler),11,12,14,15(Lottie recherche) |
| `ACCEPTED` | idem, `canCancel` vrai |
| `SCHEDULED` | idem, `canCancel` vrai |
| `IN_PROGRESS` | 1,2,3,5,11,12,14,15 — **aucune action principale** |
| `COMPLETED` | 1,2,3,5,6(confirmer),7,9,10,11,12,14,15 |
| `CONFIRMED` | 1,2,3,5,9,10,11,12,13,14,15 — **ni annuler ni confirmer** |
| `CANCELED` | 1,2,5,11,12,14,15 — **ni actions, ni litige, ni chat** |

## d) Actions utilisateur

| Label | Ligne | Action | Endpoint (service) |
|---|---|---|---|
| « Confirmer l'intervention » | 386 | `confirmAction{confirm}` → `handleStatusChange('CONFIRMED')` (l.180) | `updateDemandeStatus` |
| « Annuler la demande » | 394 | `confirmAction{cancel}` | `updateDemandeStatus` |
| « Contester l'intervention » | 437 | `setDisputeDialogOpen(true)` | — |
| « Accepter le devis » | 595 | `confirmAction{accept,quoteId}` | `respondToQuote` |
| « Refuser » | 602 | `confirmAction{reject,quoteId}` | `respondToQuote` |
| « Négocier avec le technicien » | 612 | `handleNegotiate` (l.231) | `requestQuoteNegotiation` |
| « Recharger mon solde » | 577 | `<Link href="/client/solde/recharger">` | — |
| « Voir le profil » | 736 | `<Link href={/client/technicien/${id}}>` | — |
| « Discuter » | 743 | `setChatOpen(true)` | — |
| « Envoyer la contestation » | 818 | `handleOpenDispute` (l.254) | `openDispute` |

**Formulaires inline** : aucun sur le corps de la page. Le litige est dans un
`ConfirmDialog` (l.824–861, `Select` + `Textarea`). Le chat est dans
`FloatingChat`.

**Modales** : 2 `ConfirmDialog`. Le premier est polymorphe — 4 `kind` avec
titre / description / `confirmLabel` calculés en ternaires imbriqués
(l.780–807).

**Garde de solde** : `handleQuoteResponse` l.205–215 compare
`balance.balance < totalToDebit` **avant** l'appel API et affiche un encart
« solde insuffisant » (l.568–590) avec le déficit et 2 CTA.

## e) Données chargées

**Appels API au montage** — `load(true)` l.131–163, `Promise.all` sur 4 :

| Appel | Ligne | Phase |
|---|---|---|
| `getDemande(params.id)` | 134 | montage + refetch |
| `listDemandeDiagnostics` | 135 | montage + refetch |
| `listDemandeQuotes` | 136 | montage + refetch |
| `listMissionEvents` | 137 | montage + refetch (`.catch(() => [])` silencieux) |
| `getDispute` | 146 | **si `status === 'COMPLETED'`** |
| `getClientFinanceSummary` | 153 | **si `initial` uniquement** (jamais refetché) |

**SSE** : `subscribe(missionStreamUrl(params.id), …)` l.111. Filtre
`mission.message_created` (l.112) ; tout autre événement incrémente `sseTick`
(l.113).

**Refetch** : `useEffect` dépend de `[params?.id, sseTick]` (l.178).
`setSseTick` déclenche un rechargement complet des 4 ressources.

**Polling** : `setInterval` 5 000 ms (l.169–173), sauté si `document.hidden`
(l.170) **et** si `realtimeStatusRef.current === 'sse'` (l.171). Le chat gère
ses messages lui-même.

## f) État local

| Hook | Nombre | Lignes |
|---|---|---|
| `useState` | **19** | 74–104 |
| `useEffect` | **2** | 109, 127 |
| `useMemo` | **0** | — |
| `useCallback` | **0** | — |
| `useRef` | **1** | 106 (`realtimeStatusRef`) |

**Hooks après un `return` conditionnel : AUCUN.** Les 3 returns anticipés sont
l.281 (`loading`), l.295 (`error && !demande`), l.306 (`!demande`). Le dernier
hook est l.115. Tous les calculs dérivés (l.308–339) sont des expressions
pures. Verrouillé par `technician-quote.test.ts` l.233.

## g) Longueur des sous-composants

| Composant | Lignes |
|---|---|
| `TravelMapSection` | 871–931 (**61**) |

## h) Problèmes structurels identifiés

**Duplication de logique avec l'écran technicien** — mêmes données, deux rendus :

| Donnée | Client | Technicien |
|---|---|---|
| Diagnostic | l.466–507 (`blockquote` + `border-l-2`) | l.819–851 (`Alert` info/neutral) |
| Devis détaillé | l.536–566 (`<dl>` bordé) | l.890–951 (`<div>` + commission + net) |
| Médias | l.652–658 `DemandeMediaSection` | l.529–533 (**même composant**) |
| Audio diagnostic | l.471–479 | l.822–830 (**même composant, même prop shape**) |
| Statut de devis | l.531 | l.886 (**même composant**) |
| Litige | l.406–448 (lecture + action) | l.727–744 (lecture seule) |

**Sections qui pourraient être conditionnelles** (constat) :

- Bloc 14 « Synthèse » (l.687) s'affiche à l'identique sur une mission
  `CANCELED` comme sur une `CONFIRMED`.
- Bloc 11 « Éléments transmis » (l.652) est **non conditionnel** : rendu même
  sans technicien assigné, même en `CANCELED`.
- Bloc 12 « Chronologie » (l.661) est non conditionnel : rendu en `CANCELED` où
  il n'y a plus rien à suivre.
- `TravelBanner` (l.368) est rendu inconditionnellement, y compris en
  `CANCELED`.

**Blocs orphelins (jamais rendus) : aucun.** `travelMap` n'est lu que dans
`TravelMapSection` (l.876), `travel.fresh` également (l.887) — tous deux
alimentés par `demande.travel`.

**Divergence de vocabulaire** : l'écran client n'utilise pas `MissionInfo` ; il
réécrit l'équivalent en `<dl>` (l.689–714). `MissionInfo` n'est importé que par
l'écran technicien.

---

# 2. ÉCRAN TECHNICIEN `/technicien/demandes/[id]`

## a) Métadonnées

| | |
|---|---|
| Fichier | `frontend/src/app/technicien/demandes/[id]/page.tsx` |
| **Lignes** | **1 085** (1 086 avec la ligne de fin) |
| `'use client'` | oui (l.1) |

**Imports de composants (25)** — UI : `Card`/`CardContent`/`CardHeader`/`CardTitle`
(l.6), `Button` (l.7), `Skeleton*` (l.8), `Avatar` (l.9), `Icon` (l.10),
`Alert` (l.11), `Badge` (l.12), `ConfirmDialog` (l.13), `Field` + `Input` via
le barrel `@/components/ui` (l.14), `PageHeader` + `SectionHeader` (l.15),
`DemandeStatusBadge`/`QuoteStatusBadge` (l.16), `RatingStars` (l.30),
`PushNotificationCard` (l.18).
Métier : `MissionInfo` (l.17), `DiagnosticAudioPlayer` (l.19),
`FreeDiagnosticSection` (l.20), `DemandeMediaSection` (l.21), `DemandeProgress`
(l.23), `TravelSection` (l.24), `MissionMap` (l.25), `MissionSummaryCard`
(l.27), `ConversationSection` (l.28), `RatingSection` (l.29).

**Composants enfants créés dans le fichier** : **AUCUN**. Tout est dans le
composant principal. Helpers : `formatAmount` (l.75–78), `formatPrice`
(l.80–82), constante `FALLBACK_KYC_BLOCKER` (l.86–92).

## b) Structure visuelle actuelle — ordre top→bas

| # | Bloc | Ligne | Hauteur est. | Contenu | Condition |
|---|---|---|---|---|---|
| 1 | `PageHeader` « Détail de la demande » | 485 | ~56 px | retour `/technicien` | toujours |
| 2 | **Card principale** | 487–746 | **~1 400 px** | 21 sous-blocs ci-dessous | toujours |
| 3 | Lien « Voir la chronologie » | 748–762 | ~64 px | dernière activité + chevron | `events.length > 0` |
| 4 | `FreeDiagnosticSection` | 766–771 | ~500 px | formulaire diagnostic + devis | `canChooseDiagnostic && canProposeManualQuote` |
| 5 | `MissionSummaryCard` | 773–775 | ~300 px | récap complet | statut ∈ ACCEPTED…CONFIRMED |
| 6 | Card « Discussion avec le client » | 777–793 | ~400 px | `ConversationSection` inline | `baseCanDiscuss && (catalogFlow ? negotiationUnlocked : true)` |
| 7 | Card « Diagnostic » | 795–858 | ~400 px | contenu + audio + 3 champs | toujours |
| 8 | Card « Proposition tarifaire » | 860–1062 | ~500 px | devis + formulaire inline | toujours |
| 9 | `RatingSection` | 1064–1070 | ~200 px | avis sur le client | `status === 'CONFIRMED'` |
| 10 | `ConfirmDialog` | 1072–1083 | — | « Marquer comme terminée » | `confirmFinish` |

**Sous-blocs de la card principale (l.487–746), dans l'ordre :**

| # | Sous-bloc | Ligne | Hauteur est. | Condition |
|---|---|---|---|---|
| 2.1 | Référence + badge + catégorie | 488–494 | ~60 px | toujours |
| 2.2 | Bandeau appareil catalogue | 496–501 | ~48 px | `deviceLabel` non vide |
| 2.3 | Bandeau « Autre appareil » | 505–516 | ~48 px | `!deviceLabel && (equipmentFamily \|\| equipmentType)` |
| 2.4 | `MissionInfo` | 518–525 | ~140 px | toujours |
| 2.5 | Médias | 529–533 | ~150–350 px | toujours |
| 2.6 | Avancement (`DemandeProgress`) | 535–542 | ~130 px | `status !== 'CANCELED'` |
| 2.7 | `TravelSection` (boutons GPS) | 544–546 | ~160 px | `demande.technicianId` |
| 2.8 | Carte mission | 550–582 | ~380 px | `technicianId && hasInterventionCoords` |
| 2.9 | Card Client + réputation | 584–605 | ~90 px | `demande.client` |
| 2.10 | Alerte `actionError` | 607 | variable | `actionError` |
| 2.11 | Bandeau KYC + bouton désactivé | 609–644 | ~180 px | `canAccept && kycRequired` |
| 2.12 | Bouton « Accepter la demande » | 646–655 | ~48 px | `canAccept && !kycRequired` |
| 2.13 | `PushNotificationCard` | 659 | ~90 px | `status === 'ACCEPTED'` |
| 2.14 | Champ date + « Planifier » | 661–681 | ~130 px | `ACCEPTED && hasAcceptedQuote` |
| 2.15 | Alerte « en attente du tarif » | 683–687 | ~56 px | `ACCEPTED && !hasAcceptedQuote` |
| 2.16 | « Démarrer l'intervention » | 689–698 | ~48 px | `SCHEDULED` |
| 2.17 | « Marquer comme terminée » | 700–709 | ~48 px | `IN_PROGRESS` |
| 2.18 | Alerte « en attente de confirmation » | 711–715 | ~56 px | `COMPLETED` |
| 2.19 | Alerte « confirmée » | 717–719 | ~56 px | `CONFIRMED` |
| 2.20 | Alerte « annulée » | 721–723 | ~56 px | `CANCELED` |
| 2.21 | Litige (lecture seule) | 727–744 | ~120 px | `dispute` |

**Hauteur cumulée estimée** (mission `IN_PROGRESS`, 4 médias, chat ouvert) :
≈ 3 800 px desktop, ≈ 5 500 px mobile.

## c) États conditionnels

**Trois returns anticipés** — l.327 (`loading`), l.341 (`error && !demande`),
l.389 (`!demande`). Le cas KYC de l.349–368 est **imbriqué dans le 2ᵉ return**.

Variables dérivées calculées après, l.391–481 :

| Variable | Ligne | Règle |
|---|---|---|
| `canAccept` | 391 | `status === 'SUBMITTED' \|\| 'PENDING'` |
| `hasInterventionCoords` | 394 | lat/long `typeof number` |
| `ownTravelPoint` | 396–403 | `travel.fresh` **et** coords non nulles |
| `hasStaleTravelPoint` | 406–411 | `!fresh` **et** coords présentes |
| `isPreAcceptance` | 416 | `=== canAccept` |
| `kycRequired` | 417 | `canAccept && !kycVerified` |
| `kycBlocker` | 422–426 | `kycAcceptanceBanner(...)` ?? `FALLBACK_KYC_BLOCKER` |
| `canChooseDiagnostic` | 436 | `status === 'ACCEPTED'` |
| `canProposeManualQuote` | 437–439 | `ACCEPTED` **et** (`!catalogFlow` **ou** négociation demandée **et** pas de devis accepté) |
| `canSubmitQuote` | 464–469 | `!actionBusy && amount valide && !erreur && ≥5 000 && description non vide` |

**Répartition par statut**

| Statut | Blocs rendus |
|---|---|
| `SUBMITTED`/`PENDING` | 1,2(.1–.9),.10?,.11 ou .12,.15,.16–.21? ; **jamais** 4,5,7,8,9 |
| `ACCEPTED` | 1,2(.1–.9),.13,.14 ou .15,.16–.21?,**4**,**5**,**6**,7,8 |
| `SCHEDULED` | 1,2(.1–.9),.16–.21?,**5**,**6**,7,8 |
| `IN_PROGRESS` | idem + .17 |
| `COMPLETED` | idem + .18 |
| `CONFIRMED` | idem + .19 + **9** |
| `CANCELED` | 1,2(.1–.5),.20 — **pas de .6, .7, .8**, ni 4, 5, 6, 7, 8, 9 |

## d) Actions utilisateur

| Label | Ligne | Action | Endpoint |
|---|---|---|---|
| « Terminer ma vérification » | 619, 356 | `<Link href="/technicien/kyc">` | — |
| « Accepter la demande » (désactivé) | 636 | `disabled`, aucun handler | — |
| « Accepter la demande » | 647 | `handleAccept` (l.248) | `acceptDemande` |
| « Planifier l'intervention » | 671 | `handleSchedule` (l.278) | `updateTechnicianDemandeStatus(…, 'SCHEDULED', ISO)` |
| « Démarrer l'intervention » | 690 | `handleStatusChange('IN_PROGRESS')` | `updateTechnicianDemandeStatus` |
| « Marquer comme terminée » | 701 | `setConfirmFinish(true)` → ConfirmDialog | via `handleStatusChange('COMPLETED')` |
| « Proposer un tarif » | 868 | `setShowQuoteForm(v => !v)` | — |
| « Proposer » (devis) | 1051 | `handleCreateQuote` (l.297) | `createDemandeQuote` |

**Formulaires inline** : 2 — `datetime-local` (l.664–669) et le devis complet
(l.980–1060 : montant `number` + aperçu live + description + bouton).

**Modales** : 1 `ConfirmDialog` (l.1072–1083).

## e) Données chargées

**4 `useEffect` distincts**

| # | Lignes | Objet | Appels |
|---|---|---|---|
| 1 | 133–139 | abonnement SSE mission | `subscribe(missionStreamUrl(id))` |
| 2 | 141–153 | libellés familles « Autre appareil » | `listEquipmentFamilies()` — **montage unique** |
| 3 | 160–177 | profil KYC | `getTechnicianProfile()` — dépend de `params.id` |
| 4 | 200–246 | mission | `Promise.all([getTechnicianDemande, listDemandeDiagnostics.catch, listDemandeQuotes.catch, listMissionEvents.catch])` + `getDispute` |

**SSE** : 2 flux.
- `missionStreamUrl(id)` l.135 → filtre `mission.message_created` (l.136),
  sinon `setRefreshKey` (l.137).
- `useUserStream(...)` l.182 → sur `technician.kyc_verified` /
  `technician.kyc_rejected`, refetch **du seul profil** (l.189–194), jamais de
  la mission.

**Refetch** : `useEffect` l.246 dépend de `[params?.id, refreshKey]`.
`refreshKey` est incrémenté par le SSE (l.137) et par
`FreeDiagnosticSection onDone` (l.769).

**Polling** : identique au client — 5 000 ms, sauté si `document.hidden`
(l.238) et si SSE actif (l.239).

## f) État local

| Hook | Nombre |
|---|---|
| `useState` | **20** (l.97–126) |
| `useEffect` | **4** |
| `useMemo` | **0** |
| `useCallback` | **0** |
| `useRef` | **1** (l.130) |

**Hooks après un `return` conditionnel : AUCUN.** Commentaire explicite
l.447–459 documentant la régression `0e1c284` et interdisant tout hook au bloc
l.460–469. Deux tests verrouillent cette règle (`technician-quote.test.ts`
l.233 et l.253).

## g) Longueur des sous-composants

**AUCUN créé dans le fichier.**

## h) Points spécifiques

### Vue « mission disponible » (non assignée) vs « mission assignée »

Une seule vue, pas de branching global. La différenciation se fait par
`demande.technicianId` et `canAccept` :

| Différence | Non assignée (`canAccept`) | Assignée |
|---|---|---|
| Avancement | absent (l.535 requiert `!== 'CANCELED'` — en fait présent aussi) | présent |
| `TravelSection` | **absent** (l.544 requiert `technicianId`) | présent |
| Carte mission | **absent** (l.550 idem) | présent |
| Bandeau / bouton Accepter | l.609 ou l.646 | absent |
| Push `MissionSummaryCard` | absent (statut requis) | l.773 |
| Diagnostic / devis | cartes 7 et 8 **présentes mais en état verrouillé** | contenu réel |

Le bloc 2.6 `DemandeProgress` s'affiche dans les deux cas : sa condition est
`status !== 'CANCELED'` (l.535), pas `technicianId`.

### Blocage « Accepter » si KYC non vérifié — où est-ce géré

Quatre points de contrôle, pas un :

1. **Variable** `kycRequired = canAccept && !kycVerified` — l.417.
2. **Bandeau** l.609–644 : `Alert` variant issu de `kycBlocker` (l.422–426 →
   `kycAcceptanceBanner`, `technician-kyc-rules.ts` l.187–205), avec motif du
   refus éventuel, CTA `KYC_PAGE_HREF`, et **bouton visible mais `disabled`**
   (l.636).
3. **Écran d'erreur 404** l.349–368 : si `error && !demande` **et**
   `technicianProfile !== null && !kycVerified`, remplace l'erreur générique
   par un message métier « Vérification d'identité requise » + 2 CTA.
4. **Intermédiaire** l.371–378 : si `!profileLoaded`, un squelette remplace
   l'erreur le temps que le profil arrive — évite le flash.

État KYC par défaut `useState(true)` (l.108) — optimiste, corrigé dès que
`getTechnicianProfile` répond (l.168).

### Diagnostic libre + audio + catalogue — où sont ces sous-sections

| Élément | Lignes | Condition |
|---|---|---|
| `FreeDiagnosticSection` | 766–771 | `canChooseDiagnostic && canProposeManualQuote` = `ACCEPTED` **et** (pas de flux catalogue **ou** négociation demandée **et** pas de devis accepté) |
| Audio | 821–830 | `latestDiagnostic.hasAudio` |
| Badge « Non référencé » | 802–804 | `latestDiagnostic.mode === 'MANUAL'` |
| Badge « Catalogue » | 805–807 | `latestDiagnostic.mode === 'CATALOG'` |
| Alerte « Diagnostic verrouillé » | 811–817 | `isPreAcceptance && !latestDiagnostic` |
| Contenu diagnostic | 818–851 | `latestDiagnostic` |

Le composant `FreeDiagnosticSection` (40 l.) délègue à `useFreeDiagnostic`
(156 l.) et `ResponsiveView` (Desktop 98 l. / Mobile 99 l.). Le **hub catalogue a
été retiré** — le catalogue ne subsiste que comme `source: 'CATALOG'` sur le
devis (`catalogFlow`, l.429) et l'indicateur l.876–881.

---

# 3. COMPOSANTS PARTAGÉS

## `frontend/src/components/mission/*` — 18 fichiers, 2 454 lignes

| Fichier | Lignes | Exports | Rôle | Hooks | Client | Technicien |
|---|---|---|---|---|---|---|
| `demande-media-section.tsx` | 350 | `DemandeMediaItem`, `DemandeMediaSection` | dépôt pièces jointes + visionneuse | useState ×2, useCallback, useEffect ×3, `useMediaUrl` | ✅ l.653 | ✅ l.529 |
| `travel-section.tsx` | 245 | `TravelSection` | GPS V3 technicien (3 boutons) | useState ×6, useRef | ❌ | ✅ l.545 |
| `mission-card.tsx` | 239 | `MissionCardPart`, `deviceLabelFor`, `MissionCard`, `LiveMissionCard` | carte mission des **listes** | aucun | ❌ | ❌ |
| `conversation-section.tsx` | 216 | `ConversationSection` | fil de discussion | useState ×6, useRef ×2, useEffect ×3 | ⚠️ indirect (via `FloatingChat` l.764) | ✅ l.786 |
| `dispatch-sonar-widget.tsx` | 196 | `isDispatchWaveEvent`, `partitionDispatchWaves`, `DispatchSonarWidget` | hub radar des vagues de dispatch | useState | ✅ l.664 | ❌ |
| `tracking-hero.tsx` | 171 | `TrackingHeroTechnician`, `TrackingHero` | hero de suivi (page `/suivi`) | aucun | ❌ | ❌ |
| `mission-summary.tsx` | 155 | `MissionSummaryCard` | récap complet de mission | useState ×2, useEffect | ❌ | ✅ l.774 |
| `mission-map-inner.tsx` | 125 | `MissionMapInner` | rendu Leaflet réel | useRef, useEffect ×2, `useMap` | ✅ indirect | ✅ indirect |
| `mission-stepper.tsx` | 120 | `MissionStepper` | stepper vertical animé | aucun | ❌ | ❌ |
| `rating-form.tsx` | 110 | `REVIEW_COMMENT_MAX_LENGTH`, `RatingForm` | formulaire d'avis | useState ×5 | ⚠️ via `rating-section` | ⚠️ via `rating-section` |
| `rating-section.tsx` | 98 | `RatingSection` | carte avis + liste | useState ×3, useCallback, useEffect | ✅ l.676 | ✅ l.1065 |
| `diagnostic-audio-player.tsx` | 82 | `DiagnosticAudioPlayer` | lecture note vocale diagnostic | useState ×3, useCallback, useEffect | ✅ l.471 | ✅ l.822 |
| `celebration.tsx` | 81 | `Celebration` | confettis CSS | aucun | ❌ | ❌ |
| `mission-info.tsx` | 64 | `MissionInfo` | infos demande (description, ville, dates) | aucun | ❌ | ✅ l.518 |
| `mission-map.tsx` | 68 | `MissionMap` | enveloppe dynamic-import Leaflet | useState | ✅ l.900 | ✅ l.553 |
| `travel-banner.tsx` | 60 | `TravelBanner` | bannière « en route » client | aucun | ✅ l.368 | ❌ |
| `mission-timeline.tsx` | 44 | `MissionTimeline` | chronologie événements | aucun | ✅ l.668 | ❌ |
| `demande-progress.tsx` | 30 | `PROGRESS_STEPS`, `DemandeProgress` | avancement 6 paliers | aucun | ✅ l.671 | ✅ l.539 |

**Bilan d'usage** : 6 composants utilisés par les **deux** écrans
(`demande-media-section`, `rating-section`, `diagnostic-audio-player`,
`demande-progress`, `mission-map` + `mission-map-inner`), 4 uniquement technicien
(`travel-section`, `mission-summary`, `mission-info`), 3 uniquement client
(`dispatch-sonar-widget`, `travel-banner`, `mission-timeline`), 1 partagé mais
monté indirectement (`conversation-section`), 4 utilisés par aucun des deux
(`mission-card`, `tracking-hero`, `mission-stepper`, `celebration`).

## Composants annexes

| Fichier | Lignes | Client | Technicien |
|---|---|---|---|
| `components/client/chat/floating-chat.tsx` | 176 | ✅ l.764 | ❌ |
| `components/technician/diagnostic/free-diagnostic-section.tsx` | 40 | ❌ | ✅ l.767 |
| `…/free-diagnostic-desktop-view.tsx` | 98 | ❌ | ✅ (via `ResponsiveView`) |
| `…/free-diagnostic-mobile-view.tsx` | 99 | ❌ | ✅ (via `ResponsiveView`) |
| `…/use-free-diagnostic.ts` | 156 | ❌ | ✅ |
| `components/client/demande-wizard.tsx` | **1 742** | ❌ (écran de création) | ❌ |

---

# 4. LOGIQUE MÉTIER DANS L'UI

| Règle | Fichier | Lignes |
|---|---|---|
| **Machine à états** | `backend/src/demandes/demandes-lifecycle.ts` | `CLIENT_TRANSITIONS` 16–22, `TECHNICIAN_TRANSITIONS` 24–28, `assertTransition` 32–66 |
| | `frontend/src/lib/request-status.ts` | `STATUS_BY_CONTEXT` 18–54 (3 dictionnaires : client 22–31, technician 32–41, history 44–53) |
| | `frontend/src/lib/mission-progress.ts` | `STATUS_PROGRESS` 3–12, `MISSION_STEPS` 25–31 |
| | `frontend/src/components/mission/demande-progress.tsx` | `PROGRESS_STEPS` 10–17 |
| **Conditions par rôle** | client `page.tsx` | l.308–314 (`canCancel`, `baseCanDiscuss`, `catalogFlow`, `negotiationUnlocked`) |
| | technicien `page.tsx` | l.391, 416–439 |
| | `ui/status-badge.tsx` → `request-status.ts` | `demandeStatusConfig` 56–64 |
| **Devis (accept / refuse / négocie)** | client | `handleQuoteResponse` 198–229, `handleNegotiate` 231–246 |
| | technicien | `handleCreateQuote` 297–325 |
| | verrou flux catalogue | client l.611–621, technicien l.876–881 |
| | seuil 5 000 / commission | `frontend/src/lib/technician-quote.ts` — `previewTechnicianQuote` 74–85, `quoteAmountError` 93–118 |
| | source de vérité serveur | `backend/src/financial/fee-calculator.ts` — `calculateTechnicianFee` |
| **Chat** | composant | `mission/conversation-section.tsx` — `useEffect` 44/56/92, SSE l.71 |
| | modal client | `client/chat/floating-chat.tsx` — portal l.120 |
| | scroll | **ABSENT** — aucun `scrollIntoView`, aucun `location.hash` dans `src/` |
| **GPS envoi (technicien)** | `mission/travel-section.tsx` | `startTravel` / `refreshTravelLocation` / `markTravelArrived`, `TRAVELABLE_STATUSES` l.36, throttle 30 s l.41 |
| | helpers | `frontend/src/lib/travel-location.ts` — `getCurrentTravelPosition`, `formatTravelDistance`, `formatTravelRecency` |
| **GPS affichage (client)** | `mission/travel-banner.tsx` | 11–60 |
| | carte | `mission/mission-map.tsx` 46–68 → `mission-map-inner.tsx` |
| | backend fraîcheur | `backend/src/geo/*` |
| **SSE** | `frontend/src/lib/realtime/sse-context.tsx` | `RealtimeProvider` 32–62, `subscribe` 52–55 |
| | client | `useEffect` 109–115 + garde polling l.171 |
| | technicien | `useEffect` 133–139 + garde polling l.239 + `useUserStream` l.182–195 |
| | sanitation erreurs | `lib/ui-error-message.ts` — `toUserErrorMessage` 154–181 |

---

# 5. PROBLÈMES IDENTIFIÉS (factuels)

## Blocs où l'utilisateur doit scroller > 3 écrans

| Écran | Bloc | Estimation |
|---|---|---|
| technicien | card principale l.487–746 | ~1 400 px |
| technicien | parcours complet `ACCEPTED` (1→8) | ~3 800 px desktop / ~5 500 px mobile |
| client | parcours complet `COMPLETED` (1→15) | ~2 400 px desktop / ~3 600 px mobile |

## Actions secondaires placées avant les principales

**Technicien, l'ordre est inverse du parcours utilisateur** : la card principale
contient, dans l'ordre — bouton « Accepter » (l.646), carte mission (l.550),
fiche client (l.584), `TravelSection` (l.544), médias (l.529), `MissionInfo`
(l.518) — et **ensuite seulement**, hors card : `FreeDiagnosticSection`
(l.766), `MissionSummaryCard` (l.773), discussion (l.777), diagnostic (l.795),
devis (l.860). Le récap de mission (l.773) est rendu après la discussion et le
diagnostic.

**Client** : `TravelBanner` (l.368) et la carte (l.371) précèdent les actions
principales (l.383). La chronologie (l.661) précède l'avis (l.675).

## Doublons d'information (même info affichée 2 fois)

| Information | Où |
|---|---|
| Diagnostic | client l.466–507 **et** technicien l.819–851 |
| Devis détaillé | client l.536–566 **et** technicien l.890–951 |
| Description du problème | technicien : `MissionInfo` l.518 **et** `DemandeMediaSection` l.529 |
| Lieu | technicien : `MissionInfo` l.520 **et** carte l.550 **et** `TravelSection` l.545 |
| Avancement | technicien : `DemandeProgress` l.539 **et** `MissionSummaryCard` l.774 **et** lien chronologie l.748 **et** bannière d'état l.711–723 |
| Statut | technicien : badge l.491 **et** `MissionSummaryCard` l.90 **et** bannière l.717–723 |
| Médias | `DemandeMediaSection` monté des deux côtés avec la même donnée |

`MissionSummaryCard` (rendu côté technicien) rend son badge avec
`context="client"` (l.90) — libellé issu du vocabulaire client.

**Deux taxonomies d'avancement concurrentes** : `PROGRESS_STEPS` (6 paliers) vs
`MISSION_STEPS` (5 paliers). Les deux écrans de détail utilisent la première ;
`MissionStepper` utilise la seconde.

## États vides

| Cas | Client | Technicien |
|---|---|---|
| Pas de technicien | animation Lottie + texte l.750–758 | n/a (page inaccessible sans acceptation, sauf erreur) |
| Pas de diagnostic | texte l.515–519 | texte l.852–856 |
| Pas de devis | texte l.643–647 | texte l.974–978 |
| Pas d'événements | `DemandeProgress` l.671 | `lastActivityLabel` synthétisé l.470–473 |
| Pas de média | `DemandeMediaSection` interne | idem |
| Pas de KYC résolu | n/a | squelette l.371–378 |
| Pas de position GPS | `Alert` l.893–897 et l.922–926 | `Alert` l.557–561 + span l.575–579 |

`EmptyState` (design system) **n'est utilisé sur aucun des deux écrans**.

## Erreurs silencieuses

| Cas | Ligne | Traitement |
|---|---|---|
| `listMissionEvents` échoue | client l.137, technicien l.214 | `.catch(() => [])` — chronologie vide |
| `getDispute` échoue | client l.148, technicien l.224 | `.catch(() => undefined)` — litige invisible |
| `getClientFinanceSummary` échoue | client l.155 | `.catch(() => undefined)` — solde `null` → la garde l.209 (`balance &&`) laisse passer l'acceptation vers un 400 backend |
| `listEquipmentFamilies` échoue | technicien l.149 | `.catch(() => undefined)` — fallback `equipmentFamily` brut l.511 |
| `getTechnicianProfile` échoue | technicien l.170 | `.catch(() => undefined)` — `kycVerified` reste `true` → bouton Accepter actif, refus 403 |
| Erreur de refetch (5 s) | client l.159, technicien l.227 | commentaire explicite « Erreur silencieuse en rafraîchissement périodique » |
| Sortie audio échouée | composants `useMediaUrl` | état `error` local |

**Différence client/technicien sur la gestion d'erreur** : le technicien sépare
`error` (page entière, l.101) et `actionError` (boutons, l.105, rendu l.607) ; le
client utilise **un seul** `error` rendu l.380, au-dessus des actions.

## Composants qui pourraient être sortis

| Bloc | Lignes | Taille |
|---|---|---|
| Card principale technicien | 487–746 | **260 lignes** |
| Devis (client) | 524–649 | 126 lignes |
| Diagnostic (client) | 454–521 | 68 lignes |
| Technicien assigné (client) | 718–759 | 42 lignes |
| Litige (client) | 406–448 | 43 lignes |
| Synthèse (client) | 687–715 | 29 lignes |

## Blocs « toujours affichés » alors qu'ils pourraient être conditionnels

| Bloc | Écran | Condition actuelle |
|---|---|---|
| Médias | client l.652 / technicien l.529 | **aucune** — rendu en `CANCELED` aussi |
| Chronologie | client l.661 | **aucune** — rendue en `CANCELED` |
| Synthèse | client l.687 | **aucune** |
| `TravelBanner` | client l.368 | **aucune** — rendu sans technicien |
| Avancement `DemandeProgress` | technicien l.535 | `!== 'CANCELED'` seulement, pas `technicianId` |
| Card « Diagnostic » | technicien l.795 | **aucune** — rendue en `CANCELED` |
| Card « Proposition tarifaire » | technicien l.860 | **aucune** — rendue en `CANCELED` |

---

# 6. COMPOSANTS UI RÉUTILISABLES DISPONIBLES

`frontend/src/components/ui/` — 39 fichiers `.tsx` + `index.ts`.

| Fichier | Lignes | Usage actuel | Présent sur les écrans mission |
|---|---|---|---|
| `alert.tsx` | 64 | bandeau 5 variantes, `role="alert"` si error | **les deux** (12 usages) |
| `avatar.tsx` | 63 | avatar photo/initiales, 5 tailles | **les deux** |
| `badge.tsx` | 29 | pastille 6 variantes | **les deux** |
| `breadcrumbs.tsx` | 49 | fil d'Ariane hiérarchique | ❌ — le client a un fil en dur l.344 |
| `button.tsx` | 67 | 5 variantes × 4 tailles, `isLoading`, haptique | **les deux** (14 usages) |
| `card.tsx` | 33 | `glass-card` + header/content | **technicien** (6 usages) ; **client : AUCUN** — le client écrit `<section>` brut l.455, 525, 652, 661, 687, 718 |
| `confirm-dialog.tsx` | 58 | confirmation sur `Modal` | **les deux** (3 instances) |
| `empty-state.tsx` | 33 | état vide icône+titre+action | ❌ **unused sur les deux** |
| `field.tsx` | 27 | label + hint + erreur | **les deux** |
| `input.tsx` | 18 | `<input>` `h-11` | technicien l.14 ; client via dialog litige |
| `modal.tsx` | 137 | modale/bottom-sheet + piège focus | socle de `ConfirmDialog` |
| `page-header.tsx` | 61 | `PageHeader` + `SectionHeader` | **les deux** ; `PageHeader` **technicien seulement** (l.485) |
| `rating-stars.tsx` | 45 | étoiles demi-remplissage | technicien l.595 + `rating-section` |
| `responsive-view.tsx` | 25 | une seule vue montée (≥1024 px) | technicien via `FreeDiagnosticSection` ; **client : AUCUN** |
| `select.tsx` | 26 | `<select>` stylé | client l.826 |
| `skeleton.tsx` | 54 | shimmer | **les deux** (loading l.281/l.327) |
| `status-badge.tsx` | 31 | `DemandeStatusBadge`, `QuoteStatusBadge` | **les deux** |
| `tabs.tsx` | 107 | 2 variantes × 2 modes | ❌ **unused sur les deux** |
| `textarea.tsx` | 18 | `<textarea>` | client l.850 ; `RatingForm` |
| `timeline.tsx` | 73 | `<ol>` étapes done/current/pending | socle de `MissionTimeline` + `DemandeProgress` |
| `toast.tsx` | 72 | portail des toasts | **les deux** via `useToast` |
| `icon.tsx` | 320 | bibliothèque Lucide inline, `ICON_NAMES` | **partout** |
| `push-notification-card.tsx` | 77 | opt-in push 5 états | technicien l.659 |
| `app-header.tsx` 30 · `bottom-nav.tsx` 194 · `gradient-hero-card.tsx` 72 · `label.tsx` 12 · `logo.tsx` 151 · `lottie-animation.tsx` 72 · `notification-item.tsx` 182 · `reward-progress-card.tsx` 75 · `scroll-to-top.tsx` 14 · `spinner.tsx` 25 · `stat-card.tsx` 70 · `switch.tsx` 44 · `user-avatar.tsx` 29 · `workspace-sidebar.tsx` 75 | — | layout/dashboard | ❌ |

**Incohérence d'import** : le technicien importe `Field`/`Input` via le barrel
`@/components/ui` (l.14) ; le client par chemin direct (l.13–15). `index.ts` ne
ré-exporte pas `Logo`, `LottieAnimation`, `NotificationItem`,
`PushNotificationCard`, `ResponsiveView`, `RewardProgressCard`,
`WorkspaceSidebar`.

---

# 7. AUTRES ÉCRANS SIMILAIRES

| Écran | Fichier | Lignes | Pattern |
|---|---|---|---|
| `/client/demandes` | `app/client/demandes/page.tsx` | **6** | simple redirection vers `<ClientDashboard variant="list" />` |
| `/technicien/demandes` | `app/technicien/demandes/page.tsx` | 382 | liste + tri pertinence **100 % côté client** (l'API ne pagine ni ne trie) |
| `/client/confirmation` | **ABSENT** | — | aucune route de ce nom dans `src/app/client/` |
| `/suivi` | `app/suivi/page.tsx` | 236 | page publique, `TrackingHero` + `MissionStepper` + `Celebration` |
| `/client/chronologies/[id]` | 14 | délègue à `components/chronologies/chronology-detail.tsx` (182) |
| `/technicien/chronologies/[id]` | 18 | idem |
| `chronologies-view.tsx` | 319 | vue liste + `DemandeProgress` |
| `tracking-preview.tsx` | 132 | — |
| `admin/missions` | — | supervision **par référence** (aucun endpoint de liste) |

**Comparaison de patterns**

| Pattern | client détail | technicien détail | `/suivi` | chronologies |
|---|---|---|---|---|
| En-tête | `<h1>` + `<nav>` maison l.344–364 | `PageHeader` l.485 | `TrackingHero` | `PageHeader` |
| Conteneur | `<section>` bruts | `Card` design system | — | `Card` |
| Avancement | `MissionTimeline` / `DemandeProgress` | `DemandeProgress` | `MissionStepper` | `DemandeProgress` |
| Événements | inline l.661–673 | lien externe l.748–762 | — | page dédiée |
| Responsive | **aucun** (`ResponsiveView` non importé) | via `FreeDiagnosticSection` | — | — |
| Erreur de chargement | 1 `error` | `error` + `actionError` + 3ᵉ return KYC | — | — |

**Constat** : `/client/demandes` délègue à `ClientDashboard` (8 fichiers, 743
lignes) ; `/technicien/demandes` n'a pas d'équivalent (page unique de 382
lignes). Le dashboard client a une paire `client-home-desktop-view` (191) /
`client-home-mobile-view` (188) — pattern `ResponsiveView` **absent des deux
écrans de détail**.

---

# 8. FICHIERS CLÉS POUR LA REFONTE

| # | Fichier | Lignes | Pourquoi |
|---|---|---|---|
| 1 | `frontend/src/app/technicien/demandes/[id]/page.tsx` | 1 085 | le plus gros, 21 sous-blocs, le plus de conditions |
| 2 | `frontend/src/app/client/demandes/[id]/page.tsx` | 931 | idem côté client |
| 3 | `frontend/src/lib/request-status.ts` | 77 | 3 dictionnaires statut, source du vocabulaire |
| 4 | `frontend/src/lib/technician-quote.ts` | 118 | règle de devis + garde anti-hook l.447–459 |
| 5 | `frontend/src/components/mission/conversation-section.tsx` | 216 | chat, seul composant partagé monté différemment |
| 6 | `frontend/src/components/mission/demande-media-section.tsx` | 350 | plus gros composant partagé |
| 7 | `frontend/src/components/mission/mission-summary.tsx` | 155 | récap, badge `context="client"` l.90 |
| 8 | `frontend/src/components/mission/travel-section.tsx` | 245 | GPS technicien, seul `TravelSection` |
| 9 | `frontend/src/lib/realtime/sse-context.tsx` | 68 | contrat `status` / `subscribe` des 2 pages |
| 10 | `frontend/src/components/mission/dispatch-sonar-widget.tsx` | 196 | seul usage de `partitionDispatchWaves` |
| 11 | `frontend/src/components/technician/diagnostic/use-free-diagnostic.ts` | 156 | séquence diagnostic → devis |
| 12 | `frontend/src/lib/dispute-status.ts` | 48 | litige, seul flux muté par le client |
| 13 | `frontend/src/components/mission/mission-info.tsx` | 64 | le client le réécrit en dur |
| 14 | `frontend/src/lib/ui-error-message.ts` | 211 | `toUserErrorMessage`, utilisé partout |
| 15 | `frontend/src/lib/technician-kyc-rules.ts` | 259 | `kycAcceptanceBanner` + 4 points de blocage |

---

# 9. SCHÉMA DE DONNÉES MISSION

Source : `backend/prisma/schema.prisma`.

## `Demande`

**Statuts** : `DemandeStatus` = `SUBMITTED | PENDING | ACCEPTED | SCHEDULED |
IN_PROGRESS | COMPLETED | CONFIRMED | CANCELED`.

| Champ | Type | Usage UI |
|---|---|---|
| `reference` | `String @unique` | titre `h1` client l.358, technicien l.490 |
| `status` | `DemandeStatus` | toutes conditions |
| `category` | `String` | `categoryLabel` (dérivé) l.362 / l.493 |
| `description` | `String?` | `MissionInfo` l.519 ; client l.697–703 |
| `equipmentType` | `String?` | technicien l.505–513 (fallback) |
| `equipmentFamily` | `String?` | technicien l.511 (libellé via `listEquipmentFamilies`) |
| `city` | `String` | l.692 / l.520 — snapshot jamais rendu optionnel |
| `cityId` / `zoneId` | `String?` | non affichés |
| `neighborhood`, `address`, `landmark` | `String?` | `locationLabel` client l.324–331 |
| `contactPhone` | `String?` | client l.712 |
| `latitude` / `longitude` | `Float?` | carte des deux côtés |
| `travelLatitude` / `travelLongitude` | `Float?` | `ownTravelPoint` technicien l.396–403 |
| `travelLocationUpdatedAt` | `DateTime?` | `recency` client l.888 |
| `technicianEnRouteAt` / `technicianArrivedAt` | `DateTime?` | derrière `travel.enRoute/arrived` l.371 |
| `scheduledAt` | `DateTime?` | technicien : `scheduledValue` → ISO l.286 |
| `requestedMode` | `RequestTiming` | `MissionInfo` l.522 |
| `requestedAt` | `DateTime?` | `MissionInfo` l.523 |
| `negotiationRequestedAt` | `DateTime?` | `negotiationUnlocked` l.311 / l.430 |
| `finalAmount` | `Int?` | **non lu par l'UI** |
| `domainId` / `brandId` / `modelId` / `problemId` | relations | `deviceLabel` client l.317–323, technicien l.474–481 |
| `createdAt` / `updatedAt` | `DateTime` | l.362 / `MissionInfo` |

Relations : `medias`, `messages`, `diagnostics`, `quotes`, `reviews`, `dispute`,
`events`, `notifications`, `dispatchWaves`, `financialTransactions`,
`fundsHolds`, `rewardFraudFlag`, `aiWarnings`, `conversationFlags`,
`classification`.

## `Quote`

| Champ | Usage UI |
|---|---|
| `amount`, `currency` | `formatAmount` technicien l.75, `formatQuoteAmount` client l.530 |
| `description` | l.533 / l.888 |
| `status` (`PENDING/ACCEPTED/REJECTED`) | `QuoteStatusBadge`, conditions l.592/625/631 et l.960/963/968 |
| `source` (`CATALOG/MANUAL`) | `catalogFlow` l.310 / l.429 |
| `travelAmount` | `latestQuote.travel` l.555 / l.912 |
| `initialReferencePrice` / `initialTravelFee` / `initialServiceFee` | **UI ne lit pas** — lisibles via `breakdown.*` l.551/555 |
| `diagnosticId`, `catalogDiagnosticId`, `catalogInterventionId` | l.537/543 et l.891/897 |
| **Champs calculés** (`commission`, `netTechnician`, `totalToDebit`, `repair`, `breakdown`) | **absents du schéma** — enrichis côté service. Repli `previewTechnicianQuote` technicien l.930/939 |

## `Diagnostic`

| Champ | Usage UI |
|---|---|
| `content` | l.468 / l.820 ; `slice(0,60)` pour le libellé audio l.474 / l.825 |
| `mode` (`CATALOG/MANUAL`) | badges l.460 / l.802–807 |
| `proposedIntervention` | l.480 / l.831 |
| `justification`, `notes`, `recommendation` | l.491–493 / l.836–849 |
| `audioStoragePath` | **jamais lu** → API `hasAudio` + `getDiagnosticAudioUrl` |

## `Message`

`content` (l.220 conversation), `senderId` + `sender` (rendu l.216), `createdAt`
(tri).

## `Review`

`rating`, `comment` (max 1000), `authorId`, `targetId` (déduit), `createdAt`.
`@@unique([demandeId, authorId])`.

## `DemandeMedia`

`kind` (`IMAGE/VIDEO/AUDIO`), `mimeType`, `sizeBytes`, `stored`, `fileName`.
**`storagePath` et `url` ne sont jamais exposés** — URLs signées à la demande.
`stored: false` → « Aperçu indisponible ».

## `DemandeEvent`

`type` (19 valeurs `DemandeEventType`), `fromStatus`, `toStatus`, `metadata`,
`createdAt`. `@@index([demandeId, createdAt])`.

## `DemandeDispute`

`category` (5), `description` (10–2000), `status` (4), `resolution`,
`decidedAt`. **`demandeId @unique`** — au plus un litige par mission.

---

# 10. RISQUES DE LA REFONTE

## Couverture de tests

**Aucun test de rendu React n'existe dans le dépôt** — pas de bibliothèque de
test de composants (`node --test` sur `.ts` purs uniquement, alias `@/` non
résolu).

**7 fichiers de tests statiques lisent les deux pages** (tous dans `src/lib/`) :

| Fichier | Tests concernés | Vérifie |
|---|---|---|
| `technician-quote.test.ts` | l.163, **233**, **253**, **272**, 310, 322 | aperçu devis, seuil, bouton bloqué, **3 garde-fous anti-hook**, `formatFCFA`, montants backend |
| `technician-kyc.test.ts` | l.414, 427 | bouton Accepter `disabled` (pas masqué), bandeau non bloquant, `kycRequired` |
| `dispute-status.test.ts` | l.48, 57, 75 | litige client (ouverture) / technicien (lecture seule, **jamais de POST**), pas de `window.innerWidth` |
| `demande-media.test.ts` | l.83, 101, 107, 115 | players lazy, ordre, permissions, hub catalogue supprimé, pré-acceptance |
| `diagnostic-libre.test.ts` | l.17, 32, 48, 70, 87 | flux unifié, validation, vues Desktop/Mobile isolées, audio lazy |
| `gps-helpers.test.ts` | l.74, 128, 146 | pas de tracking continu, carte Plan/Relief, pas de log coordonnées |
| `realtime/realtime-hooks.test.ts` | l.30, 37, 18 | chat SSE, rechargement piloté par événements, cleanup |
| `saspay-fees.test.ts` | l.143, 188, 211 | pas de frais SasPay côté client, mention technicien |

**Tests globaux qui parcourent ces fichiers** : `design-system.test.ts`
(l.20–33 balaie tous les `.tsx`, allowlist `mission-map-inner.tsx` l.42),
`navigation.test.ts` (l.106 sur `tracking-hero`), `demande-equipment.test.ts`
(l.44), `lottie.test.ts` (l.114).

## Ce qui n'est PAS verrouillé

- **L'ordre de rendu des sections** — aucun test ne le vérifie.
- **`EmptyState`, `Tabs`, `Card` côté client** — absents, donc introduire ces
  composants ne casse aucun test.
- **`mission-summary.tsx`, `mission-info.tsx`, `travel-banner.tsx`,
  `rating-section.tsx`, `rating-form.tsx`, `mission-timeline.tsx`,
  `demande-progress.tsx`, `mission-stepper.tsx`, `tracking-hero.tsx`,
  `celebration.tsx`, `mission-card.tsx`, `dispatch-sonar-widget.tsx`** —
  **aucun test direct**.
- Les `id` et ancrages.
- Le comportement responsive (aucun `ResponsiveView` sur le client aujourd'hui).
- La réactivité, l'agencement, le `aria` rendu.

## Ancrages HTML référencés ailleurs

**6 `id` statiques**, tous en `htmlFor` / `aria-describedby` :

| id | Fichier | Ligne | Référencé par |
|---|---|---|---|
| `disputeCategory` | client | 827 | `Field htmlFor` l.825 |
| `disputeDescription` | client | 851 | `Field htmlFor` l.840 |
| `scheduledAt` | technicien | 665 | `Field htmlFor` l.663 |
| `quoteAmount` | technicien | 989 | `Field htmlFor` l.983 |
| `quoteAmount-preview` | technicien | 1005 | `aria-describedby` l.996 |
| `quoteDescription` | technicien | 1044 | `Field htmlFor` l.1042 |

**1 `id` dynamique** : `dispatch-logs-${waves[0]?.id ?? 'all'}`
(`dispatch-sonar-widget.tsx` l.88), posé l.180, référencé par `aria-controls`
l.164.

**Aucun autre `id` dans `components/mission/*`.**

**Référencés ailleurs : NON.** Recherche exhaustive de `location.hash`,
`scrollIntoView`, `getElementById`, `querySelector` dans `src/` → 2 occurrences
hors périmètre : `admin/catalog/[domainId]/[problemId]/[diagnosticId]/page.tsx:89`
et `ui/modal.tsx:59` (piège de focus interne). **Aucun `location.hash` dans tout
`src/`.**

`ScrollToTop` (`ui/scroll-to-top.tsx`, monté au layout) replace en haut à chaque
route.

## Contenu référencé par email / push (URL avec hash)

**5 URLs push pointent vers les écrans de détail — aucune ne contient de
fragment `#`.**

| Source | URL | Événement |
|---|---|---|
| `technician.service.ts:1380` | `/client/demandes/${demandeId}` | technicien assigné |
| `technician.service.ts:1558` | `/client/demandes/${demandeId}` | en route / arrivé |
| `technician.service.ts:1742` | `/client/demandes/${demandeId}` | GPS |
| `collaboration.service.ts:506` | `/client/demandes/${demandeId}` | `quote_created` |
| `collaboration.service.ts:641` | `/technicien/demandes/${demandeId}` | `quote_accepted` / `quote_rejected` |

Propagées via `push.service.ts:136` (`data: { url, type }`). Aucun template
e-mail ne porte d'URL de détail de mission.

**Conclusion section 10** : il n'existe **aucun système d'ancrage** vers une
section d'un écran de détail — ni `#chat`, ni `#map`, ni `#devis`. Les URLs push
ouvrent la page par défaut, en haut. Les seules exceptions ci-dessus
(`admin/catalog` scroll) sont hors périmètre.

---

# Points d'attention

1. **Aucune recommandation n'a été formulée.** Les sections 1h, 2h et 5 listent
   des faits et des emplacements, pas des avis sur ce qu'il faudrait changer. Le
   tri des constats suit l'ordre demandé dans la commande, pas une gravité
   supposée.

2. **Les hauteurs sont des estimations** (px, basées sur la densité des
   sections), pas des mesures de rendu. Aucune maquette n'a été produite, donc
   le nombre exact d'écrans dépend de la hauteur de fenêtre, du zoom et du
   repliement des blocs conditionnels.

3. **Aucun test n'a été exécuté.** La section 10 décrit ce que les tests
   *vérifient d'après leur code source*, pas ce qu'ils *ont retourné*. La
   dernière exécution connue (chantier TRANSPARENCE-SASPAY) laissait 4 échecs
   front-end pré-existants.

4. **Lecture partielle de quelques fichiers volumineux**, à vérifier si un
   détail devient déterminant pour la refonte : `ui/bottom-nav.tsx` (194 l.),
   `ui/notification-item.tsx` (182 l.), `ui/logo.tsx` (151 l.), `ui/modal.tsx`
   (137 l.), `ui/tabs.tsx` (107 l.) — rôle établi par exports et props, détail
   interne non lu. Également
   `free-diagnostic-desktop-view.tsx` et `free-diagnostic-mobile-view.tsx`
   (lecture d'en-tête), et `sse-client.ts` l.85–154 (`subscribe`/`teardown`/
   `status`, pas la stratégie complète de reconnexion).

5. **Le schéma `Quote` ne contient pas `commission`, `netTechnician`,
   `totalToDebit`, `repair` ni `breakdown`** — tous deux sont injectés par la
   couche service. Le rapport le signale (section 9) car toute logique
   d'affichage de devis dans la refonte doit consommer les champs enrichis, et
   le repli `previewTechnicianQuote` du technicien (l.930/939) n'existe pas
   côté client.

6. **Point de vocabulaire à trancher avant refonte** :
   `MissionSummaryCard` n'est rendue que côté technicien mais affiche son badge
   avec `context="client"` (`mission-summary.tsx` l.90). Les deux dictionnaires
   de libellés coexistent (`request-status.ts` l.22–41) sans qu'un écran ne les
   expose tous les deux aujourd'hui.

7. **Un écart de style repéré pendant la lecture** (aucune incidence
   fonctionnelle) : `technicien/demandes/[id]/page.tsx` l.202 contient
   `let active = true;    const load = …` sur une seule ligne — probablement un
   effet d'un édition antérieure.