# RAPPORT — AUDIT 404 technicien : violation de la règle des Hooks (régression 4-A)

> Audit d'investigation **lecture seule**. Correctif **non appliqué** — en attente
> de validation. Supersede et corrige `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md`,
> qui exonérait à tort la page concernée.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Nature | **Audit lecture seule stricte** — aucun fichier de `backend/` ni `frontend/` modifié, aucun commit applicatif, aucun push applicatif |
| Dépôts concernés | `Repairdom-backend` (`09cc75f`), `Repairdom-frontend` (`7dc1818`) — **inchangés** |
| Dépôt racine | `projet` — ce rapport seul |
| Verdict | **Cause racine identifiée et prouvée : régression introduite par le chantier 4-A (commit frontend `7dc1818`).** Un `useMemo` est appelé après trois returns anticipés → « Rendered more hooks than during the previous render » → `src/app/error.tsx`. |
| Statut du correctif | **non appliqué**, en attente de validation (5 questions ouvertes) |

## Synthèse

La page `/technicien/demandes/[id]` plante au rendu pour **toute** mission technicien,
assignée ou non. Le render 1 retourne tôt sur `if (loading)` et n'appelle pas le
`useMemo` ; le render 2, après réception des données, l'appelle → le nombre de
hooks passe de 14 à 15 → React lève une exception → la frontière d'erreur de route
affiche « Une erreur est survenue ».

Les **404** sur `diagnostics` / `quotes` / `events` et le **403** SSE ne sont **pas**
la cause : ils sont le comportement normal d'une mission **disponible (non assignée)**,
et le frontend les absorbe volontairement.

Preuve : sur `e920495` (avant 4-A), il n'y a **aucun** hook après les returns. Sur
`7dc1818` (4-A), il y en a **un**, à la ligne 447.

## Fichiers

| Type | Chemin |
|---|---|
| Créé (racine) | `RAPPORT-AUDIT-404-HOOKS-TECHNICIEN.md` |
| Modifié (code) | **aucun** |
| Supprimé | **aucun** |

Fichier incriminé (non modifié) :
`frontend/src/app/technicien/demandes/[id]/page.tsx`

Fichiers inspectés (lecture seule) :
`frontend/src/app/technicien/demandes/page.tsx`,
`frontend/src/app/technicien/demandes/[id]/page.tsx`,
`frontend/src/app/error.tsx`, `frontend/oxlint.json`,
`frontend/src/lib/api/technician-service.ts`,
`frontend/src/lib/realtime/use-mission-stream.ts`,
les 9 autres fichiers modifiés par 4-A (contrôle de périmètre),
`backend/src/collaboration/collaboration.service.ts`,
`backend/src/technician/technician.service.ts`,
`backend/src/realtime/realtime.controller.ts`.

## Migrations

**Aucune.** Audit sans modification.

## Vérifications

| Commande | Résultat |
|---|---|
| `git status` (racine, backend, frontend) | **aucun fichier de code modifié** par cet audit |
| `tsc` / `oxlint` / tests | **non exécutés** — audit sans modification ; rien de neuf à tester |
| Comparaison `git show e920495:…` vs `7dc1818:…` | **preuve de la régression** (voir Bugs #1) |
| Contrôle de périmètre sur les 9 autres fichiers de 4-A | **aucune autre violation** |
| Analyse sémantique des codes HTTP (backend) | requireAccess, getDemandeDetail, realtime.controller lus ligne à ligne |

> Aucune commande exigeant une infrastructure n'a été lancée.

## Bugs trouvés

| # | Constat | Emplacement | Impact | Suite |
|---|---|---|---|---|
| **1** | **Violation de la règle des Hooks — régression 4-A.** `useMemo` appelé **après** trois returns anticipés (`:326`, `:340`, `:388`), donc au `:447`. Render 1 → 14 hooks, render 2 → 15 hooks → `Rendered more hooks than during the previous render`. | `frontend/src/app/technicien/demandes/[id]/page.tsx:447` | **Bloquant.** `/technicien/demandes/[id]` est **inutilisable pour toute mission technicien**, assignée ou non. Introduit par le commit `7dc1818`. | **Correctif proposé, non appliqué** (voir Recommandation) |
| **2** | **Le rapport précédent est faux sur ce point.** `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` conclut « aucun décompilateur trouvé, aucune exception imputable à 4-A ». L hadn't vérifié que la **position** du `useMemo` par rapport aux returns — seulement son import et sa déclaration avant usage dans le JSX. | `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` | Conclusion erronée ayant fait perdre un cycle d'investigation | **Ce rapport supersede le précédent** |
| **3** | **Aucun outillage ne peut voir ce bug.** `tsc` ne détecte pas l'ordre de hooks ; `oxlint` n'a **aucun plugin react-hooks** (`frontend/oxlint.json` ne désactive que `no-explicit-any`) ; mes 19 tests `technician-quote.test.ts` sont des **assertions statiques par regex** qui vérifient la présence des appels, **jamais leur position**. Test à vide. | `frontend/oxlint.json`, `frontend/src/lib/technician-quote.test.ts` | Récidive garantie tant que ces trois filets restent muets | **Signalé** |
| 4 | **Incohérence de masquage SSE/API.** `realtime.controller.ts:72` renvoie `403 ForbiddenException` là où `collaboration.service.ts:92/94` masquent volontairement en 404. | `backend/src/realtime/realtime.controller.ts:70-73` | Fuite mineure d'information (existence d'une mission révélée). Aucun impact UI : le frontend ignore ces 403. | Signalé, hors périmètre |
| 5 | **`error.tsx` n'expose ni message ni `digest`.** L'exception est journalisée (`:16`) mais invisible à l'écran. | `frontend/src/app/error.tsx:15-17, 23` | Diagnostic très lent en production (2 cycles d'audit perdus) | Signalé, hors périmètre |

## Points d'attention

1. **Le périmètre du bug 1 est borné à un seul fichier.** Contrôle effectué sur les
   9 autres fichiers modifiés par 4-A : `technicien/revenus/page.tsx` (hooks 50-54,
   returns 87+), `admin/finances/page.tsx` (hooks 49-60, returns 126+),
   `relio-funds-section.tsx` (hooks 30-54, returns 82+),
   `use-free-diagnostic.ts` (hooks 44-49, return 135), vues diagnostic libre
   (aucun hook). **Tous propres.**

2. **Le bug ne dépend PAS de l'état assigné / disponible.** Toute mission technicien
   ouverte en détail plante. Le trio 404 + 403 a seulement été nécessaire pour
   *documenter* le symptôme ; la mission « disponible » n'est pas la cause.

3. **Les 404 ne sont pas anormaux.** `requireAccess`
   (`collaboration.service.ts:79-97`) produit « Demande introuvable » **dans les deux
   cas** : Demande inexistante **et** Demande existante mais non assignée. C'est un
   masquage d'existence volontaire, et le frontend l'absorbe
   (`page.tsx:211-213`, commentaire `:203-208`).

4. **Le comportement attendu est bien « afficher la preview publique »** (option (a)
   de la piste 3). Le design est correct : c'est `GET /technician/demandes/:id`
   (`technician.service.ts:1101-1110`) qui répond 200 pour une mission éligible non
   assignée, en vue publique + médias. **L'implémentation du frontend plante, pas le
   backend.**

5. **Piste 4 invalidée.** La page utilise `useParams<{ id: string }>()`
   (`page.tsx:~84`), pas `params: Promise<…>`. Aucun problème Next 15 de conversion
   asynchrone.

6. **Piste 5 — À VÉRIFIER.** Payload RSC `["","technicien","demandes"]` sans UUID :
   très probablement une **conséquence** de l'exception (quand `error.tsx` prend le
   relais, Next réémet un payload de segmenterreur). À confirmer une fois le bug 1
   corrigé ; si le payload est toujours sans UUID après correction, une autre cause
   existe.

7. **L'UUID du second signalement est `f97117cc-99d7-…`** (préfixe `f`) alors que
   le premier était `97117cc-99d7-…`. Même mission avec une coquille de recopie, ou
   mission différente ? **Sans effet sur le diagnostic** (le bug est indépendant de
   l'identifiant), mais à confirmer.

## Questions bloquantes

1. **Valider le correctif 1** (suppression du `useMemo`, 1 bloc de 4 lignes) ?
   Le dépôt frontend est autonome : livrable et poussable immédiatement, ou à
   grouper avec d'autres correctifs.
2. **Aligner le SSE sur le masquage 404** (`realtime.controller.ts:72`) ? Aucun impact
   UI, mais ferme une fuite d'information mineure.
3. **Activer `eslint-plugin-react-hooks`** pour empêcher la récurrence ? Implique une
   dépendance et un changement du script `lint` — non fait sans accord.
4. **Corriger `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md`** (conclusion fausse, bug #2) ?
   Non modifié dans le cadre de cet audit lecture seule.
5. **UUID `f97117cc-…` vs `97117cc-…`** : même mission ou mission différente ?

## Recommandation (non appliquée)

**Correctif 1 — obligatoire, minimal.** Supprimer le `useMemo` (`page.tsx:447-450`) et
appeler directement la fonction pure :

```ts
const quotePreview =
  parsedQuote.amount === null ? null : previewTechnicianQuote(parsedQuote.amount);
```

`previewTechnicianQuote` est un calcul trivial sur un nombre : la mémoïsation
n'apporte rien et supprime définitivement le risque d'ordre de hooks.

**Correctif 2 — alternative, plus fragile.** Déplacer le bloc dérivé
(`parsedQuote`, `quotePreview`, `canSubmitQuote`, `:445-456`) **au-dessus** de la
ligne 326. Toute insertion future avant un return réintroduirait le bug.

**Correctif 3 — outillage, le vraiFAIL.** Activer `eslint-plugin-react-hooks`
(`rules-of-hooks`) : seule détection fiable de cette classe de bug.

**Correctif 4 — robustesse.** Exposer le `digest` dans `error.tsx` (au moins en
développement) pour qu'une exception de rendu ne soit plus un écran muet.

**Correctif 5 — test de non-régression.** Ajouter à `technician-quote.test.ts` une
assertion **structurelle** garantissant qu'aucun hook n'apparaît après le premier
`return` du composant (et non plus seulement que les appels existent).

## Scénario de reproduction (à exécuter par le commanditaire)

1. Se connecter comme technicien KYC validé.
2. Depuis `/technicien/demandes` (missions disponibles) **ou** depuis le dashboard,
   ouvrir **n'importe quelle** mission.
3. **Attendu après correctif 1** : la page s'affiche intégralement (aucune erreur).
   `GET /technician/demandes/:id` → 200 ; `diagnostics|quotes|events` → 404 sur une
   mission disponible (normal, non visible) ; `/realtime/missions/:id` → 403 sur une
   mission disponible (normal, non visible).
4. **Contrôle** : la console ne doit plus afficher
   `Rendered more hooks than during the previous render`.