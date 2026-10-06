# RAPPORT — COMMIT ORPHELIN `verification-loop-fix.test.ts`

## Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-06 |
| Sujet | Committer la modification orpheline du test de boucle de vérification, en commit dédié |
| Dépôt | `Repairdom-frontend` uniquement |
| Commit | `e920495` `test: harden verification panel loop fix` |
| Parent | `b378d7d` (le commit qui avait rendu ces tests rouges) |
| Push | `b378d7d..e920495  main -> main` — `main` aligné sur `origin/main` |
| Fichiers | 1 modifié, 0 créé, 0 supprimé |

## Synthèse

Modification orpheline committée en commit isolé, sans aucun autre changement.
**Aucun code de production touché** : le diff ne porte que sur
`src/lib/verification-loop-fix.test.ts`.

### Ce que l'investigation a révélé (et qui justifie le commit)

Cette modification n'est pas un simple ajustement cosmétique : elle **répare un
test cassé**. Le commit `b378d7d` a remplacé la navigation cliente du panneau de
vérification par une navigation dure (`window.location.assign(destination)`,
`verification-panel.tsx:202`), mais avait laissé les deux tests asserter
l'ancien mécanisme `router.replace('/client/demandes')`.

Vérifié par rejeu du fichier **dans son état `HEAD`**, hors dépôt (RÈGLE 4) :

| Version | Résultat | Détail |
|---|---|---|
| `HEAD` (`b378d7d`) | **15 / 17** | `not ok 16` et `not ok 17` |
| Avec la modification | **17 / 17** | aucun échec |

L'arbre de travail frontend était donc **rouge depuis `b378d7d`**, et la
modification orpheline le repasse au vert. Elle était réellement utile, pas
cosmétique.

### Les assertions sont plus fortes, pas affaiblies

C'est le point important du « harden » : la modification **durcit** le test au
lieu de l'assouplir pour le faire passer.

- **Avant** — le test n'affirmait que la présence d'une garde
  `if (!user?.emailVerified) return;` devant un `router.replace`. Cette garde
  laissait la boucle *structurellement possible* : elle ne faisait que la rendre
  dépendante d'un timing (`emailVerified` encore périmé dans le `AuthProvider`).
- **Après** — le test **interdit toute navigation cliente** dans le composant :
  `assert.doesNotMatch(panel, /router\.(push|replace)\(/)`. La boucle ne peut
  donc plus être réintroduite par une modification ultérieure, quelle qu'elle soit.

Les commentaires ajoutés documentent le mécanisme de la boucle
(`RoleGuard` → `/client/verification` sur contexte `AuthProvider` périmé) et la
raison pour laquelle la navigation complète la rend impossible.

## Fichiers

Modifié :
- `frontend/src/lib/verification-loop-fix.test.ts` — 2 tests réécrits
  (le panneau ne fait plus de navigation cliente ; le délai résiduel ne force pas
  de navigation cliente), +23 / -12 lignes.

Créé / supprimé : aucun.

**Vérifié par RÈGLE 4** : aucun `.js` parasite dans `frontend/src`
(`find src -name '*.js'` → vide). La compilation a été faite sur une **copie** de
`src/` sous `/tmp`, jamais dans le dépôt.

## Migrations

Sans objet (aucun changement de schéma).

## Vérifications

Node local : **v20.20.2** → `node --test` refuse les `.ts`
(`ERR_UNKNOWN_FILE_EXTENSION`). Application de la RÈGLE 4 : copie de `src/` hors
dépôt, compilation `tsc` dans la copie, exécution du test compilé. La copie
conserve l'arborescence, ce qui permet aux assertions `readFileSync(new URL(...,
import.meta.url))` de résoudre leurs cibles.

| Vérification | Résultat |
|---|---|
| `npx oxlint src/` | ✅ **0 avertissement** |
| `npx tsc --noEmit` | ✅ **0 erreur** |
| `verification-loop-fix.test.ts` — version modifiée | ✅ **17 / 17** |
| `verification-loop-fix.test.ts` — version `HEAD` (rejeu) | 15 / 17 — **2 échecs**, proving du caractère correctif |
| Assertions du diff confrontées au composant réel | ✅ **4 / 4** |
| `find src -name '*.js'` (garde RÈGLE 4) | ✅ vide |

Non exécuté : `npm run test:unit` complet (limite Node 20). Les 33 autres
fichiers de test frontend ne sont pas concernés par ce commit et n'ont pas été
rejoués.

## Bugs trouvés

1. **Test cassé non détecté depuis `b378d7d`.** `b378d7d` a modifié le composant
   sans mettre à jour ses tests contractuels, laissant `main` en rouge. Corrigé
   par ce commit. **Cause racine possible à surveiller** : rien dans la chaîne
   backend (`tsc`, `vitest`) ni dans les commandes frontend exécutables en Node 20
   (`oxlint`, `tsc`) ne couvrait `npm run test:unit`. Le gap est donc **structurel**,
   pas ponctuel — voir Points d'attention.

## Points d'attention

- **Je n'affirme pas que Vercel est vert.** Le push déclenchera un redéploiement,
  mais je n'ai pas de moyen factuel de le constater ici. Seule la modification est
  un fichier de test : **aucun impact runtime attendu en production**, mais la
  confirmation reste à faire.
- `npm run test:unit` **reste inexécutable en Node 20**. Le contrôle réel des
  tests frontend dépend d'un environnement Node 22+. Tant que ce n'est pas le cas,
  ce type de dérive (test becoming rouge) peut se reproduire sans être vu.
- Les 3 dettes pré-existantes (`demandes.service.ts:96`, alias `@/` non résolu
  par `node --test`, `package-lock.json` désynchronisé) sont **volontairement
  laissées en l'état**, conformément à votre décision et à leur consignation
  dans `backend/docs/UX-BACKLOG.md`.

## Questions bloquantes

Aucune.

Décision en attente pour la suite, hors de ce rapport : le prochain chantier
(#4B — parrainage) attend la validation de #4A en production.
