# RAPPORT — FIX URGENT : BOUCLE INFINIE SUR `/client/verification`

**Date** : 2026-10-06
**Dépôt concerné** : `Repairdom-frontend` (`3c33b63`)
**Nature** : correctif d'urgence — le parcours inscription → vérification ne
surchargeait pas en production

---

## 1. SYNTHÈSE

Le parcours complet échouait en production sur deux points indépendants. La
boucle principale avait **une cause différente de celle que le brief
soupçonnait** — et le diagnostic correct a été vérifié avant d'écrire le
correctif.

| Problème | Cause réelle | Hypothèse du brief |
|---|---|---|
| Boucle infinie | Deux règles du garde se renvoient la balle | « Query string » — **infirmé** |
| Dialogue « Quitter la page ? » | `beforeunload` armé sur le mauvais état | **confirmé**, avec une nuance |

---

## 2. CAUSE EXACTE — LA BOUCLE

### 2.1 L'hypothèse « query string » est fausse

`role-guard.tsx:22` utilise `usePathname()` de Next.js, qui **retire déjà la
query string et le fragment**. Le `pathname` reçu vaut donc
`/client/verification`, jamais `/client/verification?from=demande`.

La comparaison stricte de `guard-decision.ts:44` n'échouait donc **jamais** de
ce fait. Le.split('?') ajouté n'est qu'une défense supplémentaire : la
décision pure est testable sans React, et un appelant futur pourrait passer
une URL complète.

### 2.2 La vraie cause : la ligne 57 se renvoyait l'utilisateur à lui-même

`guard-decision.ts` enchaînait deux règles incompatibles :

```
Règle 1 (l. 44) : CLIENT + emailVerified === false + pathname ≠ '/client/verification'
                  → redirect '/client/verification'

Règle 2 (l. 57) : isPublicPath (donc pathname === '/client/verification')
                  → redirect roleHomePath(CLIENT) = '/client'
```

Un utilisateur non vérifié **déjà sur** sa page de vérification ne satisfait
pas la règle 1 (donc pas de redirection), mais satisfait la règle 2 →
`/client`. Là, la règle 1 s'applique → retour à `/client/verification` → règle 2
→ …

**Le cycle était donc : `/client/verification` → `/client` → `/client/verification` → ∞**

Confirmé par exécution de la logique avant correctif :

| Cas (avant) | Verdict |
|---|---|
| CLIENT non vérifié sur `/client/verification` | `redirect → /client` ← **l'amorce** |
| CLIENT non vérifié sur `/client/demandes` | `redirect → /client/verification` |
| TECHNICIAN non vérifié sur `/technicien/verification` | `redirect → /technicien` ← même boucle |
| CLIENT **vérifié** sur `/client/verification` | `redirect → /client` |

### 2.3 La même boucle existait côté technicien

Règle symétrique (l. 50-56) + même règle 2 : un technicien non vérifié était
pingpongé entre `/technicien/verification` et `/technicien`. Sans effet tant que
le backend vérifie les techniciens à la création, mais corrigé par principe.

### 2.4 Le correctif

```ts
const verificationPath = verificationPathFor(role);
if (verificationPath && emailVerified === false) {
  if (path !== verificationPath) return { action: 'redirect', to: verificationPath };
  return { action: 'show' };   // ← la sortie de boucle
}
```

Un utilisateur non vérifié qui est **déjà** là est `show`, jamais redirigé. La
redirection ne sert qu'à l'**amener** depuis le reste de l'espace.

**Comportements préservés** (vérifiés par test) : un CLIENT **vérifié** sur
`/client/verification` est toujours renvoyé vers `/client` — c'est ce qui
rendait l'ancienne redirection « raisonnable » à première vue.

---

## 3. CAUSE EXACTE — LE DIALOGUE « QUITTER LA PAGE ? »

Le listener `beforeunload` (l. 923-929) était armé sur :

```ts
if (!hasStarted || isSubmitting) return;
```

Or **`isSubmitting` n'est jamais mis à `true` pendant la conversion** : le
parcours anonyme passe par `runDraftConversion`, qui pilote l'état `converting`
(l. 802). Le listener restait donc armé pendant l'upload et la conversion, et la
redirection finale déclenchait le dialogue natif — alors que la demande était
partie.

Correction : `if (!hasStarted || isSubmitting || converting) return;`, avec
`converting` ajouté au tableau de dépendances.

---

## 4. PROBLÈME 3 — LE PANNEAU DE VÉRIFICATION

L'attente de `user?.emailVerified` **était déjà en place** (chantier D2.5) et
correcte. Il manquait un filet si le contexte ne se synchronise jamais.

**Écart assumé avec le brief** : le « timeout de sécurité de 5 s » demandé
**n'est pas implémenté comme redirection forcée**, et voici pourquoi : pousser
vers `/client/demandes` avec un `emailVerified` périmé ferait reboucler le
`RoleGuard` vers `/client/verification` — **c'est exactement la boucle qu'on
corrige**. Un filet qui recrée le bug est pire que pas de filet.

À la place, le filet **réactualise** : si l'état n'a pas suivi après 1,5 s,
`refresh()` est rappelé. Si la synchronisation n'aboutit jamais, l'utilisateur
reste sur la page et y trouve le bouton « Voir mes demandes » — dégradation
gracieuse, pas de boucle.

---

## 5. FICHIERS MODIFIÉS

| Chemin | Nature |
|---|---|
| `src/lib/guard-decision.ts` | Règle « e-mail non vérifié » réécrite (+ sortie de boucle), comparaison sur chemin nu, helper `verificationPathFor` |
| `src/components/client/demande-wizard.tsx` | `beforeunload` désarmé pendant `converting` (+ dépendance) |
| `src/components/auth/verification-panel.tsx` | Filet de **réactualisation** au lieu d'une redirection forcée (+24 l.) |
| `src/lib/verification-loop-fix.test.ts` | **Nouveau** — 17 tests |
| `package.json` / `tsconfig.json` | Enregistrement du test |

---

## 6. TESTS — PREUVE QU'ILS ÉCHOUAIENT AVANT

**17/17 verts** après correctif. Les 6 tests qui documentent les bugs ont été
écrits **en premier** et **échouent sur le code d'origine** :

```
not ok 1 - CLIENT non vérifié SUR sa page de vérification → show
not ok 2 - le cas précédent NE BOUCLE PAS : /client/verification n'est plus renvoyé vers /client
not ok 3 - TECHNICIAN non vérifié SUR sa page de vérification → show (symétrie)
not ok 4 - query string ignorée : /client/verification?from=demande → show
not ok 14 - le wizard retire le beforeunload pendant la conversion
not ok 17 - le panneau NE force PAS la redirection (une boucle se recréerait)

# tests 17 | pass 11 | fail 6     ← avant correctif
# tests 17 | pass 17 | fail 0     ← après correctif
```

Vérification faite par `git stash` des trois fichiers de production, puis
comparaison.

Le test n° 2 est le plus parlant : il **joue la chaîne complète** et vérifie
qu'elle converge (`['/client/verification']`) au lieu d'osciller.

### Non-régression

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit` | **OK** |
| `oxlint src/` | **propre** |
| Nouveau fichier | **17/17** |
| Suites existantes | **60/60** (16 D2.5 + 25 sync + 19 routage) |
| Fichiers `.js` parasites dans `src/` | **aucun** (vérifié, un nettoyage effectué) |

---

## 7. POINTS D'ATTENTION

1. **Non testé en production par moi** : le parcours dépend d'un e-mail réel
   que je ne peux pas recevoir. Le scénario ci-dessous vous revient.
2. **`/client/verification` répond 200 sans cookie** : c'est normal, la route
   est publique par design. Ce qui compte est le comportement **avec** une
   session non vérifiée, vérifié par test unitaire.
3. Le filet de 1,5 s est court. Si l'API `GET /auth/me` est lente, une seule
   réactualisation peut ne pas suffire ; l'utilisateur reste alors sur la page
   avec le bouton. Acceptable, mais un retry plus robuste serait possible.
4. **Le même type de boucle reste possible ailleurs** : toute règle
   `isPublicPath → roleHomePath` combinée à une redirection par état peut créer
   un cycle. `technicien/verification` était le seul autre cas, corrigé.
5. Le bruit `console.warn` du catalogue (FIX précédent) reste en place.

---

## 8. QUESTIONS BLOQUANTES

**Aucune.**

---

## 9. SCÉNARIO DE TEST PRODUCTION

| # | Étape | Attendu |
|---|---|---|
| 1 | Navigation privée → `/demande`, remplir les 4 étapes | Le catalogue s'affiche |
| 2 | « Envoyer » → s'inscrire avec un e-mail neuf | **Pas** de dialogue « Quitter la page ? » |
| 3 | Après l'inscription | `/client/verification` **s'affiche**, pas de boucle |
| 4 | Ouvrir l'e-mail, cliquer le lien | `/client/demandes`, la demande est visible |

L'étape **3** valide la sortie de boucle ; l'étape **2** valide le `beforeunload`.