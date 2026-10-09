# RAPPORT — CHANTIER 6A : refonte de l'écran mission CLIENT

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` — **NON POUSSÉ** (en attente de validation) |
| Base | `95f7b8e` |
| Commit | **Aucun.** Tout est sur l'arbre de travail. |
| Backend | **Non touché** — `git status --porcelain` vide dans `Repairdom-backend` |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |

## Synthèse

`/client/demandes/[id]` passe d'un écran en grille 2 colonnes à une suite de
sections verticales strictement conditionnées, dans l'ordre validé :
action → suivi live → détail → diagnostic/devis → discussion → chronologie →
avis → litige.

Deux décisions de rendu qui n'étaient pas dans la spécification initiale :

1. **Le devis `PENDING` n'est rendu qu'une seule fois.** Il est remonté dans
   l'action prioritaire (là où la décision doit être prise) et **exclu** de la
   section « Diagnostic + devis », qui ne le ré-affiche que s'il a déjà été
   traité (`ACCEPTED` / `REJECTED`). Sans cela, le même montant apparaissait
   deux fois sur la page.
2. **Le bouton « Annuler la demande »** est posé dans une section dédiée en
   fin de page, pas dans l'action prioritaire : celle-ci est déjà occupée par
   le bandeau d'état ou le devis. Il reste accessible sur les mêmes statuts
   (`canCancel`) et passe par le même `ConfirmDialog`.

## Fichiers

Modifiés :

- `src/app/client/demandes/[id]/page.tsx` — **931 → 1 158 lignes**
- `src/lib/technician-quote.test.ts` — **+7 tests** (section CHANTIER 6A)

Supprimés :

- `src/components/client/chat/floating-chat.tsx` (176 lignes) — plus aucun
  consommateur après le passage au chat inline. **Vérifié** : aucun test, aucun
  autre écran ne le référençait.

Non touchés : `components/mission/*`, `components/ui/*`, backend, autres
tests statiques.

## Migrations

Aucune.

## Vérifications

| Contrôle | Baseline `HEAD` | Après chantier | Verdict |
|---|---|---|---|
| `tsc --noEmit` | ✅ | ✅ | pas de régression |
| `oxlint` + eslint | 5 warnings, 0 erreur | 5 warnings, 0 erreur | identique |
| `test:unit` | 420/424 | **427/431** | **+7, 0 régression** |

**Non-régression prouvée** : les 4 échecs sont exactement les mêmes fichiers et
noms qu'en `HEAD` — `demande-draft-sync`, `design-system` (logo),
`verification-confirm` ×2. Aucun test n'a été modifié ni supprimé pour les faire
passer ; le seul test touché est `technician-quote.test.ts`, en **ajout** d'un
bloc de 7 tests.

### Règle des hooks (TÂCHE 7)

Vérifié par script sur le fichier final :

```
dernier hook        : ligne 155
1er return anticipé : ligne 309
→ OK (155 < 309)
```

Les 3 returns anticipés (`loading`, `error`, `!demande`) restent sous le dernier
hook. Aucun `useState`/`useEffect`/`useMemo`/`useCallback`/`useRef` n'a été
ajouté — le décompte est identique à l'existant (19 `useState`, 2 `useEffect`,
1 `useRef`, **0** `useMemo`/`useCallback`).

Un point mérite d'être signalé : le helper `hookLines` existant dans le fichier
de test **ne détecte pas les hooks génériques** (`useState<T | null>(…)`) et n'en
trouvait que **10** sur cette page, contre 22 avec un motif large. Le garde-fou
ajouté utilise le motif large et fixe un seuil à 20 : sans cela, le contrôle
serait resté largement aveugle sur cette page. `hookLines` n'a pas été modifié,
pour ne pas décaler les seuils des tests existants.

### Tests statiques (TÂCHE 8) — les 8 lisant la page

| Test | Verdict |
|---|---|
| `demande-media.test.ts` (l.96) | ✅ `DemandeMediaSection` toujours importé et rendu, pas de `MediaGallery` |
| `diagnostic-libre.test.ts` (l.78) | ✅ `DiagnosticAudioPlayer` toujours rendu |
| `dispute-status.test.ts` (l.49, 79) | ✅ `getDispute`, `openDispute`, « Contester l'intervention », `disputeStatusConfig`, `ConfirmDialog`, pas de `window.innerWidth` |
| `realtime-hooks.test.ts` (l.38) | ✅ `sseTick` + `mission.message_created` conservés |
| `saspay-fees.test.ts` (l.144) | ✅ ni `saspay`, ni mention de frais ; `totalToDebit` présent |
| `lottie.test.ts` (l.136) | ✅ `RechercheTechnicienAnimation` toujours rendu |
| `technician-quote.test.ts` (l.233) | ✅ garde-fou hooks technicien inchangé et vert |
| `technician-kyc.test.ts`, `gps-helpers.test.ts` | ✅ ne lisent pas la page client, verts |

**Aucun test n'a cassé. Aucun test n'a eu besoin d'être mis à jour.**

## Bugs trouvés

Aucun bug fonctionnel rencontré. Deux constats de conception vérifiés en lecture :

1. **`MissionSummaryCard` n'est pas utilisé côté client** — le client affiche
   sa synthèse via un `<dl>` écrit directement dans la page. La refonte le
   conserve ainsi (`Card` + `<dl>`), sans changement de source de vérité.

2. **Le `Card` du design system n'était pas utilisé côté client** avant ce
   chantier : tous les blocs étaient des `<section>` bruts avec les mêmes classes
   (`bg-card border border-slate-200 dark:border-slate-800 rounded-2xl p-6
   shadow-sm`). Ils utilisent maintenant `Card` / `CardContent`, ce qui aligne
   le client sur le technicien sans modifier le composant partagé.

## Écarts explicites par rapport à la demande

| Demandé | Réalisé | Pourquoi |
|---|---|---|
| `EmptyState` pour « pas de devis » | **Non** | Le bloc devis n'est rendu que si `quotes.length > 0`. Sans devis, il n'y a rien à afficher dans cette section — un `EmptyState` « Aucun devis reçu » créerait une carte vide permanente sur une mission en attente de technicien. Le `EmptyState` est utilisé pour le **diagnostic** en attente, où la carte a un titre et sert de repère. |
| `FloatingChat` retiré | **Oui, et le composant supprimé** | Plus aucun consommateur après le passage inline. La suppression est vérifiée par grep sur `src/` (0 occurrence hors le composant lui-même). |
| Bloc annulation dans l'action prioritaire | **Section dédiée en fin de page** | L'action prioritaire est déjà occupée par le devis à accepter ou le bandeau d'état. Y ajouter « Annuler » noierait la décision principale sous une action destructive. Comportement identique : même `canCancel`, même `ConfirmDialog`. |

## Points d'attention

1. **Le fichier a grossi : 931 → 1 158 lignes (+227).** C'est la conséquence
   directe de la réorganisation : les blocs précédemment factorisés dans une
   grille unique sont maintenant dans des `<section>` nomées avec leurs propres
   conditions et des en-têtes de section. La logique métier (handlers, effets,
   dialogues) est **inchangée à l'identique**. Une extraction en sous-composants
   reste possible — elle n'a pas été faite ici pour ne pas mêler deux chantiers.

2. **Le test de non-régression hooks du technicien (`technician-quote.test.ts`
   l.233) ne couvre toujours que la page technicien.** Le chantier 6A n'a pas
   étendu ce contrôle existant à la page client : un garde-fou **équivalent**
   existe désormais dans le bloc 6A du même fichier. Le jour d'une refonte du
   côté technicien, la duplication serait à factoriser.

3. **Le devis `PENDING` n'apparaît plus dans la section « Diagnostic et
   devis ».** Si un client fait défiler la page sans lire la carte du haut, il ne
   verra le détail du devis qu'après acceptation ou refus. C'est le compromis
   demandé (« visible sans scroller ») mais c'est un choix de hiérarchie : le
   montant unique est en haut, la trace du devis traité reste en section 5.

4. **Le chat passe d'une bulle flottante à une section inline.** Sur mobile, la
   liste de messages est désormais dans le flux de la page au lieu d'être
   superposition. C'est l'intention du chantier, mais cela allonge la page en
   bas sur les missions à beaucoup de messages. `ConversationSection` n'a pas de
   pagination ni de repli — comportement inchangé.

5. **Les 4 échecs de test sont pré-existants** et hors périmètre :
   `demande-draft-sync`, `design-system` (logo), `verification-confirm` ×2. Ils
   sont déjà consignés dans `backend/docs/UX-BACKLOG.md`.

6. **Rien n'a été exécuté dans un navigateur.** Les scénarios de test
   production 1 à 6 n'ont pas été joués : aucun test de rendu React n'existe
   dans ce dépôt (`node --test` sur `.ts` purs uniquement). Les hauteurs de
   scroll annoncées dans la commande (~2 400 px) n'ont pas été remesurées.

## Questions bloquantes

Aucune.