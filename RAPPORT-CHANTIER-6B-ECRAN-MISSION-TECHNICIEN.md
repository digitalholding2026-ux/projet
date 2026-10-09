# RAPPORT — CHANTIER 6B : refonte de l'écran mission TECHNICIEN

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` → **`77b4c8b`** |
| Base | `1b65884` |
| Commit | `refactor(mission): restructure technician mission screen into four contextual tabs` |
| Déploiement | **Vercel vert**, vérifié dans le bundle JS servi |
| Backend | **Non touché** — `git status --porcelain` vide dans `Repairdom-backend` |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |
| Validation | Utilisateur — les 3 décisions de conception acceptées avant push |

### Preuve de déploiement

Contrôlé dans
`/_next/static/chunks/app/technicien/demandes/%5Bid%5D/page-3fe8aebf810529d1.js`
(Build ID différent d'avant le push).

Les 4 onglets, dans l'ordre du bundle :

| Onglet | Offset |
|---|---|
| `Aperçu` | 33 464 |
| `Diagnostic & Devis` | 33 490 |
| `Discussion` | 33 525 |
| `Détails` | 33 552 |

Ordre **strictement croissant**. Les 4 `role="tabpanel"` sont présents, dans le
même ordre (Aperçu 34 106 → Diagnostic 37 736 → Discussion 44 722 → Détails
51 482). `variant:"segmented"` et `label:"Sections de la mission"` confirmés :
le composant `Tabs` du design system est bien celui rendu.

**Absent du bundle** : `openDispute`, `saspay` — les deux invariants de sécurité
tiennent en production.

**Note de méthode** : les libellés accentués sont échappés en `\xe7` / `\xe9`
par le minifieur. Une recherche en texte clair renvoyait « absent » pour « Aperçu »
et « Détails » : c'était un faux négatif de ma vérification, pas une absence
dans le bundle. Les recherches ci-dessus utilisent les séquences échappées.

## Synthèse

`/technicien/demandes/[id]` passe d'un empilement de 21 blocs — dont une card
principale de 260 lignes — à un **header permanent + 4 onglets contextuels**,
via le composant `Tabs` du design system jusqu'ici inutilisé.

L'action principale monte dans le header : avant, un technicien devait
défiler toute la fiche pour trouver « Accepter la demande » ou « Démarrer
l'intervention ».

Trois décisions de conception, avec leur raison :

1. **`defaultTabFor` n'est appelé qu'à la première arrivée des données.**
   La commande demandait « une fois un onglet choisi, on ne le change plus ».
   Appliqué littéralement, le technicien qui vient d'accepter une mission
   resterait sur « Aperçu » et devrait retrouver l'onglet à la main au moment
   précis où il va travailler. `handleAccept` fait donc `setActiveTab` — c'est
   la suite de **son** clic, pas un effet de bord d'un événement entrant.
   Conséquence : un événement SSE ne fait **jamais** sauter l'onglet.
2. **Le switch sur `tabInitialized` est un render conditionnel**, pas un hook :
   il est placé après les 3 returns anticipés, donc le nombre de hooks ne
   varie pas. Le test l.233 reste vert.
3. **`canSubmitQuote` est calculé dans le composant parent**, pas dans
   `QuoteForm`. L'original le calculait au milieu de la card ; le garder dans
   le parent préserve la lecture par `technician-quote.test.ts` (`disabled={!canSubmitQuote}`,
   `quoteAmountError(amountValue)`), et évite que le sous-composant porte sa
   propre logique de soumission.

## Fichiers

Modifiés :

- `src/app/technicien/demandes/[id]/page.tsx` — **1 085 → 1 366 lignes**
- `src/lib/technician-quote.test.ts` — **+7 tests** (bloc CHANTIER 6B)

Créés / supprimés : aucun fichier, aucun composant partagé modifié.

## Migrations

Aucune.

## Vérifications

| Contrôle | Baseline `1b65884` | Après chantier | Verdict |
|---|---|---|---|
| `tsc --noEmit` | ✅ | ✅ | pas de régression |
| `oxlint` + eslint | 5 warnings, 0 erreur | 5 warnings, 0 erreur | identique |
| `test:unit` | 427/431 | **434/438** | **+7, 0 régression** |

**Non-régression prouvée** : les 4 échecs sont exactement les mêmes fichiers et
noms qu'en base — `demande-draft-sync`, `design-system` (logo),
`verification-confirm` ×2. Aucun test existant modifié ou supprimé.

### Règle des hooks (TÂCHE 6)

```
dernier hook        : ligne 260
1er return anticipé : ligne 393
→ OK (260 < 393)
```

Aucun `useMemo` ni `useCallback` introduit. Les 3 returns anticipés
(`loading`, `error && !demande` avec le cas KYC imbriqué et le squelette
`!profileLoaded`, `!demande`) restent dans le composant parent, au-dessus de
tout hook.

Le garde-fou existant `technician-quote.test.ts:233` est resté **vert sans
modification** — il vérifie `lastHook < earlyReturnIndex` avec un seuil de
10 hooks détectés ; le page en compte 22 au motif large.

### Tests statiques (TÂCHE 7) — les 5 qui lisent cette page

| Test | Verdict |
|---|---|
| `technician-quote.test.ts` (l.163, 233, 253, 272, 310, 322) | ✅ 6 tests verts, aucun fail |
| `technician-kyc.test.ts` (l.414, 427) | ✅ bouton `disabled` exact, bandeau, `kycRequired` |
| `demande-media.test.ts` (l.94) | ✅ `DemandeMediaSection` + `getTechnicianDemandeMediaFileUrl` |
| `diagnostic-libre.test.ts` (l.78) | ✅ `FreeDiagnosticSection` toujours monté |
| `dispute-status.test.ts` (l.57) | ✅ litige lecture seule, **jamais de POST** |

**Un test a cassé puis été corrigé** : `dispute-status.test.ts` échouait parce
que le commentaire du sous-composant `DetailsTab` citait `openDispute`. Le test
fait `assert.doesNotMatch(page, /openDispute/)` sur le fichier entier — il
interdit le mot, pas seulement l'appel. Le commentaire a été réécrit en
langage métier. **C'était un vrai piège** : un commentaire anodin peut casser
un garde-fou statique.

`technician-quote.test.ts` a également exigé que la prop du sous-composant
s'appelle `canSubmitQuote` (et non `canSubmit`) et que le bouton porte
`disabled={!canSubmitQuote}` : le test lit cette chaîne exacte. La prop a été
renommée plutôt que le test assoupli.

## Sous-composants extraits

| Composant | Lignes | Rôle |
|---|---|---|
| `QuoteForm` | 990–1111 (**115**) | formulaire de proposition tarifaire (montant, aperçu live, description) |
| `TechnicianHeader` | 1112–1288 (**160**) | card identité + bandeau KYC + action principale du statut |
| `DetailsTab` | 1289–1366 (**69**) | litige, push, chronologie, récapitulatif, avis |

Tous < 250 lignes comme demandé. **Le composant principal fait 1 219 lignes**
(l.149 → fin), soit au-dessus du seuil de 1 200 — voir Points d'attention §1.

Les 3 restent dans le **même fichier**, ce qui était nécessaire : les tests
statiques lisent ce fichier par chemin (`read('../app/technicien/demandes/[id]/page.tsx')`)
et, dans un fichier séparé, aucun composant ne verrait plus `FreeDiagnosticSection`,
`DemandeMediaSection`, `<Button disabled …>` ou `previewTechnicianQuote`.

## Répartition dans les onglets

**Header permanent** : `PageHeader` + card (référence, badge, catégorie, ville,
client) + bandeau KYC + une seule action principale selon le statut.

**Onglet Aperçu** : bandeaux appareil, `MissionInfo`, médias, `DemandeProgress`,
`TravelSection`, carte mission, fiche client + réputation.

**Onglet Diagnostic & Devis** : `FreeDiagnosticSection`, diagnostic publié
(badges CATALOG/MANUAL, audio, 4 champs), card devis (commission, net, mention
Option A), formulaire de devis.

**Onglet Discussion** : `ConversationSection` en pleine largeur.

**Onglet Détails** : litige lecture seule, `PushNotificationCard`, lien
chronologie, `MissionSummaryCard`, `RatingSection`.

## Écarts explicites par rapport à la demande

| Demandé | Réalisé | Pourquoi |
|---|---|---|
| Badge non-lus sur l'onglet Discussion | **Non** | `ConversationSection` n'expose aucun compteur d'unread vers l'extérieur : il gère son état en interne et n'a pas de callback. Le añadir demanderait de modifier un composant partagé, explicitement interdit par le périmètre. |
| `defaultTab` sur `status === 'ACCEPTED'` seul | **+ `!hasAcceptedQuote`** | Tel quel, un technicien dont le devis est déjà accepté (donc qui doit planifier) atterrissait dans l'onglet Diagnostic au lieu de l'Aperçu où se trouve le bouton « Planifier ». La condition de la commande + ce correctif couvrent les deux cas. |
| Extraction en 3 sous-composants si > 1 200 l. | **Fait, mais le total reste 1 366** | Les 3 sous-composants sont dans le même fichier (contrainte des tests statiques). L'extraction sert la lisibilité, pas le volume total. |

## Points d'attention

1. **Le fichier total fait 1 366 lignes**, soit +281 par rapport à l'original.
   Le composant principal est à 1 219 lignes (seuil demandé : 1 200). L'écart
   de 19 lignes vient des commentaires de cadrage ajoutés en tête de section.
   Pour repasser sous le seuil il faudrait soit alléger ces commentaires, soit
   sortir `DetailsTab` dans un fichier séparé — ce qui casserait
   `dispute-status.test.ts` puisqu'il vérifie `getDispute` / l'absence
   d'`openDispute` sur le fichier de la page.

2. **La nouvelle tabulation change le comportement de défilement.** Un
   technicien sur une mission `SCHEDULED` voit maintenant l'onglet Aperçu
   seul (~4 blocs) au lieu de 21 blocs empilés. La profondeur de contenu est
   inchangée : on la atteint par onglet.

3. **`MissionSummaryCard` reste dans l'onglet Détails**, comme demandé. Il
   porte un badge `context="client"` (constat de l'audit, non corrigé ici :
   hors périmètre).

4. **Le `switch` sur `tabInitialized` est un render conditionnel.** React le
   tolère (setState pendant le rendu du même composant), mais c'est le
   mécanisme le moins courant du fichier. L'alternative — initialiser
   `activeTab` via un `useEffect` sur `[demande, tabInitialized]` — aurait
   ajouté un 5ᵉ `useEffect`, ce que la commande interdit (« garder les 4 »).
   Un test verrouille le comportement dans les deux cas de toute façon.

5. **Rien n'a été exécuté dans un navigateur.** Les 11 scénarios de test ne
   peuvent pas être joués ici : aucun test de rendu React dans le dépôt. Les
   hauteurs de scroll (~3 800 px) n'ont pas été remesurées.

## Règle à mémoriser

L'utilisateur a demandé d'inscrire durablement la leçon du chantier : **les
commentaires de code ne doivent jamais contenir les mots-clés surveillés par
les tests statiques** (`openDispute`, `window.innerWidth`, `saspay`,
`MediaGallery`, `useMemo`…).

Elle est maintenant écrite dans `AGENTS.md` (section RÈGLE 3), avec le tableau
des mots interdits et le test associé à chacun. **Ce n'est pas une évidence
cosmétique** : le chantier 6B a lui-même cassé `dispute-status.test.ts` pour
cette raison, et le test avait raison — c'est le commentaire qui était fautif.

## Questions bloquantes

Aucune.