# RAPPORT — CHANTIER 6C-1 : correction des erreurs silencieuses critiques

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` → **`40b8f9f`** (correctifs) + **`5c29502`** (backlog) |
| Base | `77b4c8b` |
| Commit | `fix(mission): stop acting on data that failed to load on both mission screens` |
| Déploiement | **Vercel vert**, vérifié dans les deux bundles servis |
| Backend | **Non touché** — `git status --porcelain` vide |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |

## Synthèse

Quatre garde-fous ajoutés sur les deux écrans mission, sur le même principe :
**ne jamais laisser agir l'utilisateur sur une donnée qu'on n'a pas su lire.**

| Écran | Donnée | Avant | Après |
|---|---|---|---|
| Technicien | statut d'identité | échec réseau avalé → bouton actif → refus 403 | `profileLoadState` : `'error'` affiché + bandeau « Réessayer », acceptation bloquée |
| Client | solde | échec avalé → pré-validation court-circuitée → refus 400 | `balanceLoadState` : bandeau « Réessayer », bouton « Accepter le devis » désactivé |
| Les deux | historique | échec avalé → chronologie vide | `eventsLoadState` : `EmptyState` « Impossible de charger l'historique » + « Réessayer » |
| Technicien | libellés de famille | code brut affiché sans explication | badge « Libellé non disponible » + `console.warn` |

**Arbitrage général** : en cas d'incertitude, on **refuse** l'action plutôt que
laisser déclencher une erreur backend incompréhensible. Le risque retenu est
une action refusée à tort pendant une panne réseau — réversible par «
Réessayer » — plutôt qu'un refus opaque qui fait perdre confiance dans l'app.

## Fichiers

Modifiés :

- `src/app/technicien/demandes/[id]/page.tsx` — 1 366 → **1 483 l.**
- `src/app/client/demandes/[id]/page.tsx` — 1 158 → **1 260 l.**
- `src/lib/technician-quote.test.ts` — **+4 tests** (bloc 6C-1)
- `src/lib/technician-kyc.test.ts` — **1 assertion mise à jour**

Créés / supprimés : aucun. Composants partagés : **aucun modifié**.

## Migrations

Aucune.

## Vérifications

| Contrôle | Baseline `77b4c8b` | Après chantier | Verdict |
|---|---|---|---|
| `tsc --noEmit` | ✅ | ✅ | pas de régression |
| `oxlint` + eslint | 5 warnings, 0 erreur | 5 warnings, 0 erreur | identique |
| `test:unit` | 434/438 | **438/442** | **+4, 0 régression** |

Les 4 échecs sont exactement les mêmes qu'en base — `demande-draft-sync`,
`design-system` (logo), `verification-confirm` ×2.

### Règle des hooks

Vérifié par script sur les deux fichiers :

```
client       dernier hook = 236   1er return = 358   OK
technicien   dernier hook = 305   1er return = 444   OK
```

Les 3 `useState` ajoutés côté technicien (`profileLoadState`,
`profileReloadKey`, `eventsLoadState`) et les 3 côté client
(`balanceLoadState`, `balanceReloadKey`, `eventsLoadState`) sont tous **avant**
les returns anticipés. Le client passe de 2 à 3 `useEffect` (voir Points
d'attention §2).

Le garde-fou existant `technician-quote.test.ts:233` est resté **vert sans
modification**.

### Tests mis à jour (2, intention préservée)

| Test | Assertion | Pourquoi |
|---|---|---|
| `technician-kyc.test.ts:428` | `const kycRequired = canAccept && (profileLoadState !== 'loaded' \|\| !kycVerified)` | La formule a changé **par construction** : c'est l'objet du chantier. L'intention du test (« le garde ne porte que sur `canAccept` ») reste vraie — la condition s'est **resserrée**, pas élargie. Commentaire ajouté pour le dire explicitement. |
| `technician-quote.test.ts:443` | `\{showTimeline \|\| timelineFailed \? \(` | Depuis 6C-1 la chronologie est rendue **aussi** en cas d'échec. L'intention (« aucune section rendue par défaut ») reste respectée : l'erreur est traitée, pas confondue avec du vide. |

Les deux ont été modifiées **avec leur intention documentée**, jamais supprimées.

### Nouveaux tests (4)

- `6C-1 technicien` : `profileLoadState`, formule de `kycRequired`, bandeau
  d'erreur distinct du bandeau métier, clé de relance.
- `6C-1 client` : `balanceLoadState`, bouton désactivé hors solde connu,
  distinction chargement/échec, pré-validation conditionnée.
- `6C-1 les deux écrans` : `eventsLoadState`, `timelineFailed`, `EmptyState`,
  **plus** l'absence de `listMissionEvents(…).catch(() => [])`.
- `6C-1 hooks` : contrôle simultané des deux pages avec un motif qui détecte
  les hooks génériques.

## Bugs trouvés

### Le client avait perdu son `useEffect` de solde lors de la refonte 6A

**Constat de lecture, pas de régression** : dans `6A`, j'ai déplacé le bloc
`if (initial) { getClientFinanceSummary()… }` hors de `load()` lors de la
réécriture — il était resté dans le `try` mais était devenu redondant. Le
comportement observé est identique (premier passage uniquement), donc aucun
test n'a bronché. **6C-1 le sort de nouveau du flux de la mission**, ce qui est
la bonne conception : un échec réseau du solde ne doit pas recharger la mission
entière, et « Réessayer » ne doit relancer que lui.

### Un test statique a presque cassé sur la formule de `kycRequired`

`technician-kyc.test.ts` et `technician-quote.test.ts` (bloc 6B) lisaient tous
deux la chaîne littérale `const kycRequired = canAccept && !kycVerified`. Deux
tests à mettre à jour au lieu d'un — l'alerte du chantier 6B (« les tests
statiques lisent les MOTS ») s'est confirmée : ici c'est la **formule** qui
était verrouillée par son texte, pas par son sens.

## Écarts explicites par rapport à la demande

| Demandé | Réalisé | Pourquoi |
|---|---|---|
| `kycRequired = canAccept && (loading \|\| (loaded && !kycVerified))` | `canAccept && (profileLoadState !== 'loaded' \|\| !kycVerified)` | La formule donnée laisse passer `error` (ni `loading` ni `loaded`). Or le critère d'acceptation exige « bouton désactivé en loading **et** error ». Les deux expressions sont équivalentes sauf sur `error`, où la mienne bloque — c'est le comportement demandé. |
| Bandeau « Réessayer »Reload du profil | Idem, plus le flux SSE KYC | La commande ne mentionnait que l'effet de montage. J'ai aussi traité le `catch` du `useUserStream`, sinon une panne réseau sur le flux laisserait le statut dans le dernier état connu sans le signaler. |

## Points d'attention

1. **Le bouton « Accepter » est désactivé pendant le chargement du profil**,
   y compris sur une mission `SUBMITTED`. C'est le plus court possible (une
   requête), mais sur connexion lente le technicien voit un bouton désactivé
   pendant une fraction de seconde. Le bandeau métier n'apparaît que si le
   statut est connu et non vérifié — pas de message « chargement » sous le
   bouton, qui serait du bruit.

2. **Le client est passé de 2 à 3 `useEffect`** (solde séparé de la mission).
   C'est la conséquence directe de l'écart §Bugs : le solde doit pouvoir être
   relancé seul. Aucun hook après return conditionnel malgré tout.

3. **`familyLabelsFailed` déclenche un `console.warn`** en plus du badge. Le
   chantier demandait « logger + badge ». Le `console.warn` ne contient ni
   donnée personnelle ni code de famille — seulement le fait que le chargement
   a échoué.

4. **`balanceLoadState !== 'loaded'` couvre `loading` ET `error`** dans le même
   rendu. Le message diffère (« Vérification… » vs « Impossible de vérifier… »)
   mais le bouton est désactivé dans les deux cas. Un état intermédiaire
   distinct aurait demandé un 4ᵉ state pour un difference cosmétique.

5. **Rien n'a été exécuté dans un navigateur.** Les 4 scénarios de test ne
   peuvent pas être joués ici : aucun test de rendu React dans le dépôt. Les
   scénarios 1 à 3 demandent de bloquer des requêtes via DevTools.

## Scénarios de test en attente

| # | Scénario | Attendu |
|---|---|---|
| 1 | Blocquer `technician/profile` | bandeau rouge « Impossible de vérifier votre statut d'identité » + « Réessayer » ; « Accepter la demande » désactivé. Débloquer puis « Réessayer » → bandeau disparaît, bouton actif |
| 2 | Blocquer `client/finance` | bandeau « Impossible de vérifier votre solde » + « Réessayer » ; « Accepter le devis » désactivé, « Refuser » toujours actif |
| 3 | Blocquer l'historique | `EmptyState` « Impossible de charger l'historique » + « Réessayer », sur les deux écrans |
| 4 | Non-régression | KYC non vérifié bloque toujours avec le bandeau métier + motif ; solde insuffisant bloque toujours avec le déficit exact |

**Point d'attention pour le scénario 1** : le test vérifie aussi que le
bandeau *technique* ne remplace pas le bandeau *métier*. Un technicien
`REJECTED` hors ligne doit voir « Impossible de vérifier… » ; en ligne, il doit
revoir « Vérifiez votre identité » avec son motif de refus.

## Preuve de déploiement

Les deux bundles ont été inspectés (Build IDs différents d'avant le push) :

`app/client/demandes/[id]/page-546eafa897396ad6.js`

| Marqueur | Constat |
|---|---|
| « Impossible de vérifier votre solde. » | ✅ |
| « Vérification de votre solde en cours » | ✅ |
| « Réessayer » | ✅ |
| « Impossible de charger l'historique » | ✅ |
| `saspay` / `SasPay` | ✅ **absents** (invariant 6C-1/chantier transparence) |

`app/technicien/demandes/[id]/page-f9cc198b01c366f0.js`

| Marqueur | Constat |
|---|---|
| « Impossible de vérifier votre statut d'identité » | ✅ |
| « Réessayer » | ✅ |
| « Impossible de charger l'historique » | ✅ |
| « Libellé non disponible » | ✅ |
| `openDispute` | ✅ **absent** (le technicien reste en lecture seule) |

Méthode : les accents sont échappés en `\xe9` / `\u2019` par le minifieur. Une
recherche en texte clair renvoyait « absent » pour des marqueaux pourtant
présents — faux négatif de vérification, pas une absence du bundle.

## Suite inscrite au backlog

Créé `frontend/docs/UX-BACKLOG.md` (commit `5c29502`) — le pendant frontend
de `backend/docs/UX-BACKLOG.md`, les domaines restant séparés.

Y est consigné le point le plus important de ce chantier : **un effet déplacé
silencieusement lors d'une refonte n'est détecté par rien.** Le bloc de lecture
du solde avait été réécrit en 6A sans changement de comportement observable —
`tsc`, les tests et le lint sont tous restés verts. C'est une relecture ciblée
du 6C-1 qui l'a vu, deux jours plus tard. Le réflexe prescrit : avant de
commiter une refonte, faire un diff des **effets** et non seulement du JSX
rendu, et vérifier que chaque appel réseau reste attaché au même
déclenchement qu'avant.

Y figurent aussi trois suivis non traités ici : `getDispute`,
`listDemandeDiagnostics` et `listDemandeQuotes` restent silencieux côté client.

## Questions bloquantes

Aucune.