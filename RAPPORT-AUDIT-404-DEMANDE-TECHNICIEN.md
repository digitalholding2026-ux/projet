# RAPPORT — AUDIT 404 « Demande introuvable » sur `/technicien/demandes/[id]`

> Audit d'investigation, sans aucune modification de code. Suite du chantier
> 4-FONDATIONS-A, signalement production d'un écran d'erreur sur la page de
> détail mission côté technicien.

---

## Statut

| | |
|---|---|
| Date | 2026-10-08 |
| Nature | **Audit lecture seule stricte** — aucun fichier de `backend/` ni `frontend/` modifié, aucun commit applicatif, aucun push applicatif |
| Dépôts concernés | `Repairdom-backend` (`09cc75f`), `Repairdom-frontend` (`7dc1818`) — **inchangés par cet audit** |
| Dépôt racine | `projet` — ce rapport seul |
| Verdict | **Cause des 404 / 403 identifiée et sans rapport avec 4-A.** L'écran « Une erreur est survenue » **reste non expliqué** : donnée manquante dans le relevé fourni (statut de l'appel principal). |

## Synthèse

Le signal reported — un 404 « Demande introuvable » sur
`/api/demandes/{id}/diagnostics|quotes|events` et un **403** sur le flux SSE —
**ne correspond pas à un UUID invalide**. Le contrôleur SSE renvoie 404 quand la
Demande n'existe pas et 403 seulement quand elle existe mais n'appartient pas à
l'utilisateur : **le 403 prouve que la Demande existe**. Les 404 sont le
**masquage volontaire** des routes collaboratives pour un technicien non assigné,
et le frontend les tolère explicitement. Aucune des quatre pistes proposées
(draft token utilisé comme id, redirection post-convert, push, localStorage) ne
tient : les 15 sites du dépôt qui construisent une URL `/technicien/demandes/{id}`
utilisent tous un vrai `Demande.id`.

Reste inexpliqué : l'écran « Une erreur est survenue » n'est pas un message
d'API mais le titre de la frontière d'erreur de route `src/app/error.tsx:23`,
donc une **exception au rendu**. Aucun décompilateur n'a été trouvé par lecture
statique, y compris dans les blocs modifiés par 4-A.

## Fichiers

| Type | Chemin |
|---|---|
| Créé (racine) | `RAPPORT-AUDIT-404-DEMANDE-TECHNICIEN.md` |
| Modifié (code) | **aucun** |
| Supprimé | **aucun** |

Fichiers **inspectés** (lecture seule) :
`frontend/src/app/technicien/demandes/[id]/page.tsx`,
`frontend/src/app/technicien/{page,demandes/page,historique/page,revenus/page}.tsx`,
`frontend/src/components/technician/dashboard/tech-overview.tsx`,
`frontend/src/components/technician/revenus/top-missions.tsx`,
`frontend/src/components/technician/diagnostic/use-free-diagnostic.ts`,
`frontend/src/components/client/demande-wizard.tsx`,
`frontend/src/components/mission/{mission-summary,mission-info,demande-media-section,mission-map-inner,travel-section,conversation-section}.tsx`,
`frontend/src/lib/{ui-error-message,format-fcfa,demande-draft-storage,request-timing}.ts`,
`frontend/src/lib/api/{request-service,technician-service}.ts`,
`frontend/src/app/error.tsx`,
`backend/src/collaboration/{collaboration.service,collaboration.controller}.ts`,
`backend/src/technician/technician.service.ts`,
`backend/src/realtime/realtime.controller.ts`,
`backend/src/demandes/{demande-draft.service,demande-helpers}.ts`,
`backend/src/dispatch/dispatch.service.ts`, `backend/src/admin/admin.service.ts`.

## Migrations

**Aucune.** Audit sans modification.

## Vérifications

| Commande | Résultat |
|---|---|
| `git status` (racine, backend, frontend) | **aucun fichier de code modifié** par cet audit |
| `tsc` / `oxlint` / tests | **non exécutés** — audit sans modification de code ; rien à vérifier |
| Recherche exhaustive des URLs `/technicien/demandes/` | 7 sites frontend, 15 sites backend (URL + code), **tous sur `Demande.id`** |
| Analyse desemantic des codes HTTP | backend + frontend lus ligne à ligne |

> Aucune commande exigeant une infrastructure n'a été lancée. Aucun test n'a
> été exécuté : il n'y a, par construction, rien de neuf à tester.

## Bugs trouvés

| # | Constat | Emplacement | Impact | Suite |
|---|---|---|---|---|
| 1 | **Les 404 de la page technicien sont un comportement normal, pas un bug.** `requireAccess` masque en `NotFoundException` toute mission non assignée, pour ne pas révéler son existence. Le frontend les absorbe (`.catch(() => [])`). | `backend/src/collaboration/collaboration.service.ts:90-95` · `frontend/src/app/technicien/demandes/[id]/page.tsx:209-214` | aucun — conceptionnel | aucun correctif |
| 2 | **Incohérence de masquage entre SSE et API** : `realtime.controller.ts:72` renvoie `403 ForbiddenException` là où l'API masque en 404. | `backend/src/realtime/realtime.controller.ts:70-73` | **Faible fuite d'information** : l'existence d'une mission est révélée à un utilisateur non concerné (réponse différente de 404). Pas d'impact UI (le frontend ignore ces 403). | **Signalé, non corrigé** (hors périmètre de l'audit) |
| 3 | **Le `draftToken` est un `randomUUID()` v4, donc indiscernable d'un `Demande.id`.** Toute URL saisie manuellement ou Issue d'un ancien bookmark est impossible à distinguer à l'œil. | `backend/src/demandes/demande-draft.service.ts` (`randomUUID()`) vs `prisma/schema.prisma` (`gen_random_uuid()`) | Risque de diagnostic trompeur, pas de défaut fonctionnel | **Signalé** |
| 4 | **`src/app/error.tsx` n'affiche ni code ni `digest`** (elle journalise à la console mais n'expose rien). Toute exception de rendu donne un écran générique sans piste. | `frontend/src/app/error.tsx:15-17, 23` | Diagnostic très lent en production | **Signalé, non corrigé** |

## Points d'attention

1. **Une donnée manque dans le relevé fourni** : le statut de
   `GET /api/technician/demandes/97117cc-99d7-42fd-b718-a8a4857d9fc9`. S'il vaut
   200, la page a bien reçu la mission et l'exception est **au rendu** (il faut le
   `digest` de `console.error`, `error.tsx:16`). S'il vaut 404, l'écran attendu
   serait un `Alert` de page (`page.tsx:225`) et non `error.tsx`, ce qui
   orienterait le diagnostic ailleurs. **À VÉRIFIER.**

2. **Aucun décompilateur trouvé par lecture statique.** Balayage effectué sur les
   blocs modifiés par 4-A (`page.tsx:906-929` commission/net, `:959-1010`
   formulaire + aperçu) et sur les composants rendus. Tous les accès sont gardés
   (`?.`, `??`). Exemples vérifiés et **sains** : `mission-summary.tsx:108`
   (`summary.technician` gardé), `page.tsx:396-414` (`demande.travel?.`),
   `page.tsx:462-465` (`demande.domain?.name`), `mission-info.tsx:10`
   (`description: string | null`). **Aucun correctif deviné n'a été appliqué.**

3. **Les 4 pistes de la mission sont réfutées**, avec preuves :
   - `demande-wizard.tsx:828-832` pousse vers `/client/confirmation?...&id=${result.id}`,
     **jamais** vers `/technicien/demandes/…` ; `result.id` est l'id de la Demande
     (`demande-draft.service.ts:306` renvoie `DemandesService.create()`).
   - `demande-wizard.tsx:824` appelle `clearDemandeDraftToken()` dès la conversion
     réussie : le token ne survit pas en localStorage.
   - `readDemandeDraftToken()` n'est lu que dans le wizard (`:320`, `:793`).
   - Push : `collaboration.service.ts:641` utilise `demandeId` ; `dispatch.service.ts:420`,
     `admin.service.ts:360/486` utilisent des URLs **sans** identifiant.

4. **Le 403 est le discriminant** : `realtime.controller.ts:70` (404 si la Demande
   n'existe pas) vs `:71-73` (403 si elle existe mais n'appartient pas à
   l'utilisateur). Le 403 observé **établit que la Demande existe**.

5. **La prémisse de la mission est infirmée** : l'UUID `97117cc-99d7-42fd-b718-a8a4857d9fc9`
   n'est pas « un UUID qui ne correspond à aucune Demande », mais l'id d'une
   **Demande existante non assignée** à ce technicien.

## Questions bloquantes

1. **Statut HTTP de `GET /api/technician/demandes/97117cc-99d7-42fd-b718-a8a4857d9fc9` ?**
   Absent du relevé — c'est l'appel qui détermine l'orientation du diagnostic.
2. **Quel endpoint SSE a renvoyé 403 ?** `/realtime/missions/97117cc…`,
   `/realtime/technician/stream` ou `/realtime/user` ?
3. **Quel `digest` / message dans la console navigateur ?** (`error.tsx:16`)
4. **La mission avait-elle été acceptée par un autre technicien** au moment du clic ?
5. **La page fonctionnait-elle avant 4-A pour cette même mission ?** — **À VÉRIFIER** :
   aucune exception de rendu imputable au chantier n'a été identifiée.

## Recommandation (non appliquée)

1. **Ne rien corriger côté URL** : le UUID est légitime.
2. **Cadrer la cause réelle** avant toute écriture : relever l'appel principal et
   le `digest` console.
3. **Correctif de robustesse recommandé** (quel que soit le verdict) : `error.tsx`
   devrait exposer un `digest` technique sous forme repliable, pour qu'une
   exception de rendu ne soit plus un écran muet.
4. **Correctif de sécurité recommandé** : aligner `realtime.controller.ts:72` sur le
   masquage 404 de `collaboration.service.ts:92/94`.
5. **Trancher la distinction 404 / 403 côté SSE** : aujourd'hui le frontend ignore
   les deux, donc l'alignement est sans risque.

## Scénario de reproduction (à exécuter par le commanditaire)

1. Se connecter comme technicien KYC validé et disponible.
2. Depuis `/technicien/demandes`, ouvrir une mission **non encore acceptée**.
3. Attendu si le comportement est conforme : page rendue, `GET /technician/demandes/:id` → 200,
   `diagnostics|quotes|events` → 404, `/realtime/missions/:id` → 403, **pas** d'écran d'erreur.
4. Consigner le `digest` de `console.error` si l'écran d'erreur apparaît malgré tout.