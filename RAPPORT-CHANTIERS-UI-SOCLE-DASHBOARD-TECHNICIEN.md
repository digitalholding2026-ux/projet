# RAPPORT — Refonte UI : socle du dashboard technicien

> Chantiers 1 et 2 **livrés et déployés** dans un même push logique.
> Rapport factuel : chiffres réels, écarts notés, bugs rencontrés.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôts | `Repairdom-frontend` — **backend non concerné** |
| Commits | `e7f3407` — socle de thème · `cd12230` — dashboard technicien |
| Push | ✅ `e4d881b..e7f3407` puis `e7f3407..cd12230` |
| Vercel | ✅ **vert et vérifié** (§ 6) |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |

**Périmètre validé et respecté** : le socle visuel seulement. Ni l'ordre des
blocs, ni leur structure, ni aucun texte n'ont changé.

---

## 2. Décisions prises

Trois arbitrages validés avant écriture :

1. **Variantes sombres sur les composants partagés** — plutôt qu'un jeu de
   tokens parallèle.
2. **Socle visuel seul pour le dashboard** — la réorganisation des blocs reste
   un chantier séparé.
3. **`relio-card` #151922 adopté** pour les surfaces.

Un quatrième arbitrage est intervenu en cours de route : **le dashboard client
reste en thème clair** (option « a »). Les deux espaces auraont deux grammaires
cohérentes chacune dans sa palette — choix assumé, à revisiter.

---

## 3. Chantier 1 — Socle de thème sombre

### 3.1 Tokens

Trois rôles ajoutés, nommés par usage et non par couleur :

| Token | Rôle |
|---|---|
| `--color-relio-surface` | surface translucide |
| `--color-relio-border` | bordure standard (blanc 8 %) |
| `--color-relio-elevated` | état survolé |

`relio-text` et `relio-muted` **existaient déjà et n'étaient utilisés nulle
part** (0 occurrence). Ils sont activés, pas créés.

### 3.2 Un échec d'accessibilité mesuré

| Encre | Sur `relio-card` | Verdict |
|---|---|---|
| `slate-500` (avant) | **3,70:1** | ❌ sous AA (4,5:1) |
| `relio-muted` (après) | **6,86:1** | ✅ AA |

Le token jamais utilisé est précisément celui qui corrigeait le défaut.
**Deux** occurrences corrigées (sous-titres de KPI, horodatages d'activité).

Deux autres occurrences de `slate-500` en zone sombre sont **une pastille et une
icône** : elles plafonnent à 3,70:1, donc au-dessus du seuil de 3:1 propre aux
éléments non textuels. **Laissées intactes** — les modifier eût été un changement
visuel sans motif de conformité.

### 3.3 Variantes sombres, strictement opt-in

`Badge` est rendu dans **42 fichiers**, `Card` dans 43. Prop `tone` ajoutée,
**défaut `light` inchangé** : aucun usage existant n'est touché.

---

## 4. Chantier 2 — Dashboard technicien

### 4.1 Le flash de fond à l'ouverture

Le fond sombre était posé sur le div racine du rendu **final** seulement. Les
deux sorties anticipées — chargement et erreur — ne le portaient pas. Résultat :
l'écran s'affichait clair, puis virait au noir à l'arrivée des données, **à
chaque ouverture**.

Les trois états passent désormais par une coque partagée (`TechShell`), ce qui
rend l'oubli impossible par refactor.

### 4.2 Deux notations d'une même valeur, sur le même écran

`slate-900/70` (cartes du dashboard) et `#0F172A` (carte de progression) sont
**la même couleur**, affichées côte à côte. Les deux passent par `relio-card`.
`reward-progress-card` sort aussi de l'allowlist de couleurs en dur : cette
liste est un registre de dérogations, et y faire figurer un fichier conforme
finit par l'immuniser contre la règle qui l'a fait entrer.

### 4.3 La hiérarchie visuelle était inversée

Un badge du thème clair posé sur une carte sombre affiche un fond à
**15–17:1** de contraste avec elle : il devient l'élément le plus **lumineux**
de l'écran. Ce n'est pas une étiquette, c'est un bloc lumineux.

Les **6 badges** du dashboard demandent la variante sombre (voile teinte 15 %,
encre 300).

### 4.4 Cibles tactiles

| Avant | Après |
|---|---|
| 36 px (`size="sm"`) | **48 px** (seuil Android) |

Boutons d'action du dashboard et bandeau KYC. Le technicien manipule son
téléphone d'une main, en mouvement, sans toujours le regarder.

### 4.5 Mouvement, mesuré et non décoratif

| Effet | Détail |
|---|---|
| Cascade d'apparition | 80 ms d'écart, 4 blocs — composant **déjà livré**, seulement câblé |
| Compteur animé | revenus du jour, `requestAnimationFrame` natif, 600 ms, ease-out |

Le compteur : chiffres **tabulaires** (aucun saut de mise en page), annulé si
l'utilisateur a refusé le mouvement, `aria-hidden` sur la valeur animée (un
lecteur d'écran ne doit pas entendre trente fois le même chiffre), valeur
finale exposée via `aria-label`.

**Les deux autres KPI ne sont pas animés** : « 3 / 10 » et « – » afficheraient
« 0 / 10 » une demi-seconde — une information fausse à l'écran.

### 4.6 `min-h-screen` → `min-h-dvh`

Sur mobile, les barres d'URL rétractables réduisent la hauteur utile. `screen`
vaut la hauteur la plus haute : d'où un espace mort au bas.

---

## 5. Tests

**21 tests** ajoutés (13 + 8) — **594 / 594** au total.

### 5.1 Le contraste est calculé, pas asserted

`relio-theme.test.ts` extrait les tokens de `globals.css` au moment de
l'exécution et **calcule** luminance relative puis rapport de contraste. Une
assertion textuelle y aurait répondu « oui » tant que les deux valeurs
existent dans le fichier — y compris après les avoir rapprochées à 2:1.

### 5.2 Non-régression

| État | Résultat |
|---|---|
| `HEAD` propre | **581 / 581** |
| Après les deux chantiers | **594 / 594** |

`tsc` exit 0 · `oxlint` **0 erreur** · aucune dépendance ajoutée.

### 5.3 Mutations vérifiées

| Mutation | Résultat |
|---|---|
| Un état sorti de la coque | **2 échecs** |
| Badge sans variante sombre | **1 échec** |
| `min-h-dvh` → `min-h-screen` | **1 échec** |
| Compteur sans respect du mouvement réduit | **1 échec** |
| Délais de cascade désordonnés | **1 échec** |
| Basculer le mode sombre par défaut (chantier 1) | **1 échec** |
| `relio-muted` assombri sous AA | **2 échecs** |
| Suppression du rôle `relio-border` | **2 échecs** |

---

## 6. Déploiement — vérifié, pas supposé

Un HTTP 200 ne prouve pas qu'un nouveau build est servi. Le dashboard étant
protégé par `RoleGuard`, la vérification porte sur le chunk
`app/technicien/page-7c0332ab9335b48a.js` (43 037 o).

| Marqueur | Occurrences | Confirme |
|---|---|---|
| `Espace technicien` | 1 | bon chunk, bonne page |
| `min-h-dvh` | 1 | hauteur dynamique |
| `bg-relio-bg` | 1 | fond sur la coque |
| `tabular-nums` | 1 | compteur sans saut de mise de page |
| `prefers-reduced-motion` | 1 | mouvement refusé respecté |
| `tone` / `dark` | 1 / 1 | variante sombre des badges |

Extrait de la coque servie :

```js
"flex min-h-dvh flex-col gap-6 rounded-3xl bg-relio-bg p-4 text-slate-100 sm:p-6"
```

Extrait du compteur servi :

```js
{"aria-label":n(t),children:{"aria-hidden":!0,className:"tabular-nums",children:n(i)}}
```

Non-régression HTTP : `/` **200**, `/client` **200**, `/technicien` **200**.

---

## 7. Bugs et erreurs rencontrés

### 7.1 Une faille de test que j'ai refermée

Essayer de basculer `dark` par défaut : **aucun test ne l'a détecté**. Cette
régression repeindrait 42 fichiers sans la moindre alerte — exactement la
famille de régression silencieuse que ce dépôt a déjà subie deux fois. Deux
tests verrouillent désormais le défaut lui-même.

### 7.2 Une mesure qui a invalidé mon propre test

J'avais posé l'invariant « la carte doit contraster avec le fond ». Mesure :
`relio-card` sur `relio-bg` ne fait que **1,11:1**. Les deux surfaces sont
quasi identiques — **la carte ne se distingue que par sa bordure**.

Une carte est un conteneur décoratif, exempté du WCAG 1.4.11. Mais si la
bordure n'apportait rien, le token `--color-relio-border` ne servirait à rien.
Invariant remplacé : **la bordure doit apporter plus de séparation que le
remplissage** — mesurable, et il protège contre sa suppression silencieuse.

### 7.3 Trois défauts de code trouvés par mes propres tests

J'avais écrit les tests, puis laissé le code ne pas les satisfaire :

- la carte de statut gardait sa surface **en dur** (j'avais converti `TechCard`,
  oublié qu'elle s'applique par-dessus) ;
- le bouton du bandeau KYC est resté à 36 px ;
- deux blocs partageaient `delay={0}` — une cascade dont les délais ne sont pas
  croissants n'est pas une cascade.

### 7.4 Un commentaire a fait échouer un test

Dans `tech-overview.tsx`, j'ai écrit « le blanc translucide remplace
`bg-slate-900/70` ». Le test surveillait cette chaîne pour empêcher son retour —
et mon commentaire la réintroduisait. Réécrit en langage métier (RÈGLE 3,
rappelée à l'usage).

### 7.5 Une erreur de calcul dans un test

Composition du blanc translucide : j'ai écrit `alpha * 255` là où il fallait
l'opacité du **blanc**. Produisait `NaN`. C'est le test qui l'a attrapé.

---

## 8. Points d'attention

### 8.1 Le rendu n'est pas vérifié

Ni `jsdom` ni bibliothèque de composants, `next build` interdit (RÈGLE 3). Les
tests prouvent que les **tokens, classes et branches** sont là, pas que
l'**agencement** est bon. Le dashboard étant protégé par `RoleGuard`, il n'a
même pas pu être inspecté sans session.

À confirmer manuellement sur `/technicien` : la hiérarchie des badges, le
décompte des revenus, les 48 px au doigt, la fluidité de la cascade.

### 8.2 Les cartes ne se détachent que par leur bordure (1,11:1)

Si le dashboard paraît trop plat, il faudra monter `--color-relio-elevated` ou
la bordure. Ce n'est plus un problème de contraste de texte.

### 8.3 Deux grammaires coexistent désormais

Dashboard technicien sombre, dashboard client clair. Choix assumé (option « a »),
mais c'est la seule chose qui mérite vraiment le mot « harmoniser ».

### 8.4 Réorganisation des blocs : non faite

Hors périmètre. Exigerait d'élargir les assertions d'ordre de
`onboarding-components.test.ts`, qui verrouillent la position des blocs par
`indexOf`.

### 8.5 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` absent, `.nvmrc` inopérant. Sur Node 20,
`npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` : fausse
alerte, pas une régression. Chiffres obtenus avec Node 22.20.0 installé **hors
dépôt** (`/tmp/opencode/n22`).

### 8.6 Reportés des chantiers précédents

- `rewards-hero.png` (432 Ko) toujours sans usage.
- Illustration du parcours `/devenir-technicien` à 574 Ko, au-dessus du seuil
  de 500 Ko retenu.

---

## 9. Suite

| # | Chantier | Statut |
|---|---|---|
| 1 | Socle de thème sombre | ✅ livré |
| 2 | Dashboard technicien (socle) | ✅ livré |
| 3 | **Dashboard technicien (structure et hiérarchie)** | possible |
| 4 | Dashboard client | socle prêt |
| 5 | Pages restantes | page par page |

Le chantier 3 réutilise le même socle : il n'y a plus de dette de thème à
combler avant de réorganiser.