# RAPPORT — Animations de la landing (modérée, CSS pur, micro-interactions)

> Chantier **livré et déployé**. Rapport factuel : chiffres réels, écarts avec
> la demande explicitement notés, vérifications de production distinguées des
> vérifications statiques.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| Commit | `0e598c1` — *feat(landing): apparitions au scroll et micro-interactions, sans bundle* |
| Push | ✅ `55771f0..0e598c1`, branche `main` |
| Vercel | ✅ **vert et vérifié factuellement** (§ 7) |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |
| Bundle | **inchangé** — 0 Ko ajouté (décision produit : CSS pur) |

Décisions produit appliquées : intensité modérée · stylesheet pur ·
micro-interactions complètes. Les trois écarts constatés (§ 5) ont été
**acceptés explicitement** par le demandeur avant push.

---

## 2. Synthèse

La landing — zone la moins animée du dépôt avant ce chantier — reçoit une
apparition au scroll et des réactions au survol, **sans un octet de bundle
supplémentaire**.

Un composant client minimal ne décide que *quand* un bloc devient visible et
remet la main au navigateur : durée, courbe et amplitude vivent dans la feuille
de style. **28 blocs animés** sur les 8 sections demandées, en cascade bornée.

Hero, bandeau de navigation et pied de page sont restés intacts.

---

## 3. Fichiers

### Créés (2)

| Fichier | Lignes | Rôle |
|---|---|---|
| `src/components/landing/animate-on-scroll.tsx` | 75 | bascule l'état visible au premier croisement |
| `src/lib/landing-animations.test.ts` | 207 | 19 tests |

### Modifiés (6)

| Fichier | Nature |
|---|---|
| `src/app/globals.css` | +87 l. — section « apparitions au scroll » |
| `src/components/landing/landing-sections.tsx` | 5 sections enveloppées |
| `src/components/landing/faq.tsx` | 7 accordéons enveloppés |
| `src/components/landing/trust-stats.tsx` | 2 indicateurs enveloppés |
| `package.json` | enregistrement dans `test:unit` |
| `tsconfig.json` | ajout à la liste d'exclusion |

Total : **8 fichiers**, +500 / −117.

---

## 4. Sections animées

| Section | Fichier | Nb | Délai |
|---|---|---|---|
| Comment ça marche | `landing-sections.tsx` | 4 | `index * 100` |
| Pourquoi Relio | `landing-sections.tsx` | 4 | `index * 100` |
| Services | `landing-sections.tsx` | 6 | `Math.min(index * 60, 300)` |
| Récompenses | `landing-sections.tsx` | 3 | `index * 100` |
| Devenir technicien | `landing-sections.tsx` | 1 | — |
| CTA final | `landing-sections.tsx` | 1 | — |
| Preuves sociales | `trust-stats.tsx` | 2 | `index * 150` |
| FAQ | `faq.tsx` | 7 | `Math.min(index * 50, 250)` |

**Total : 28 blocs.**

Deux régimes de cascade, volontairement distincts :

- **grilles bornées** (3 à 7 entrées) → pas fixe de 50 à 150 ms ; le dernier
  élément attend au plus quelques centaines de millisecondes ;
- **cascades longues** (6 tuiles, 7 questions) → pas plafonné par `Math.min`,
  sinon le dernier élément d'une liste de vingt entrées apparaîtrait bien après
  le défilement.

**Non animés, verrouillé par test** : Hero, HeroBackground, PublicHeader,
PublicFooter. Ils ont déjà leur propre traitement, et le menu mobile vient
d'être corrigé au chantier précédent.

---

## 5. Écarts avec la demande

Trois écarts, dont **deux corrections de la spec elle-même**. Tous acceptés par
le demandeur avant push.

### 5.1 Le délai de cascade était inopérant — *correction de la spec*

La spec posait `style={{ animationDelay }}` en ligne, alors que la classe CSS
décrit une **`transition`** (état de départ → état visible), pas une animation.
`animation-delay` ne s'applique pas à une transition.

**Conséquence si rien n'avait été corrigé** : les 28 blocs seraient apparus
simultanément, sans cascade.

La transition a été conservée — elle permet l'état de départ en CSS pur, donc un
rendu serveur sans contenu masqué par du JavaScript — et le délai corrigé.

*Le demandeur a reconnu l'erreur dans sa spec et demandé qu'elle serve de
référence : quand du CSS est proposé dans une consigne, ses propriétés doivent
être confrontées au mécanisme réellement utilisé.*

### 5.2 Filet `@media (scripting: none)` — *ajout non demandé*

La classe masque le contenu (`opacity: 0`) et attend que le composant ajoute
`is-visible`. **Sans JavaScript, ce basculement n'arrive jamais** : la landing
resterait invisible sur toute sa longueur. Ce n'est pas un défaut d'apparence,
c'est un contenu perdu — pour un visiteur à faible bande passante, un ancien
navigateur, ou un bloqueur de scripts.

Support plus récent que les apparitions elles-mêmes, coût nul. **Confirmé en
production** (§ 7.4).

### 5.3 Deux anneaux de focus concurrentiels — *ServicesGrid*

`.card-hover:focus-visible` pose un `outline` ; les tuiles portaient aussi
`focus-visible:outline-none` + `focus-visible:ring-2`. Les deux se
concurrencent. `outline-none` retiré : les deux anneaux restent superposés, mais
la règle explicite gagne. Aucun test ne le vérifiait — à confirmer au clavier.

---

## 6. Tests

**19 tests** ajoutés — **534 / 534** au total (515 avant).

| Famille | Nb | Ce qui est verrouillé |
|---|---|---|
| Composant | 5 | directive client · `IntersectionObserver` · déconnexion · `prefers-reduced-motion` · absence de bibliothèque d'animation |
| Feuille de style | 7 | état masqué puis révélé · amplitude 28 px / 600 ms · survol · focus clavier · réduction sur les **deux** états · réduction du survol · filet sans script |
| Sections | 4 | import · ≥5 usages · cascades plafonnées · micro-interactions |
| Intégrité | 3 | hero/header/footer intacts · accordéon non animé · chaque fichier animé l'est bien |

### Pourquoi ces tests existent

Une landing animée casse de deux façons qu'aucun outil statique ne voit :

1. **Le contenu reste invisible** — un bloc masqué dont l'état visible
   n'arrive jamais. Invisible au `tsc`, invisible au lint, invisible à la
   lecture du source ; visible *à l'écran*, ou plutôt invisible.
2. **Le mouvement passe outre le choix du visiteur** — l'option « moins de
   mouvements » ne neutralise les durées que si l'on surcharge aussi `opacity`
   et `transform`.

La règle globale du dépôt (`globals.css:238`) ramène les durées à
`0.01ms !important` **sans toucher `opacity` ni `transform`** : sans surcharge,
les blocs resteraient figés à mi-course (opacité 0, décalés de 28 px) au lieu
d'afficher leur état final. La règle ajoutée cible les deux états, et un test le
vérifie explicitement.

### Vérification par mutation

| Mutation | Résultat |
|---|---|
| Retirer `.is-visible` de la règle de réduction de mouvement | **1 échec** |
| Débrancher `TrustStats` de l'animation | **1 échec** (après correction, ci-dessous) |

### Faille trouvée dans mes propres tests

Le test « plusieurs sections animées » comptait les occurrences dans
**`landing-sections.tsx` seul**. Débrancher `TrustStats` — ou la FAQ — laissait
le compte global intact : **la mutation passait sans aucun échec**, test vert
ne prouvant rien.

Corrigé par un test supplémentaire vérifiant, fichier par fichier, que chacun est
animé. Mutation rejouée : détectée.

### Tests existants

Vérifiés verts, aucun modifié : `landing-rewards-content.test.ts` (11),
`navigation.test.ts` (ancres), `lottie.test.ts`, `demande-draft-routing.test.ts`,
`design-system.test.ts`, et les 33 autres.

Les ancres `id="how-title"`, `id="faq"`, `id="preuves-title"`… sont
**inchangées** : l'enveloppe porte l'id, pas la section.

---

## 7. Déploiement Vercel — vérifié, pas supposé

Un HTTP 200 ne prouve pas qu'un nouveau build est servi. Quatre contrôles sur
la production après le push.

**7.1 — La feuille de style du chantier est bien servie**

La page référence deux feuilles CSS. `c55cbc28cbb340c0.css` (124 775 o) contient
les classes du chantier ; l'ancienne (`020d714…`, 7 438 o) ne les contient pas.
Le build est bien `0e598c1`.

**7.2 — Les classes attendues sont présentes et completas**

```
animate-on-scroll            animate-on-scroll.is-visible
.card-hover                 .card-hover:hover
.card-hover:active          .card-hover:focus-visible
.icon-hover                 scripting:none
```

**7.3 — Le HTML servi contient les 28 blocs animés**

```
26 animate-on-scroll     36 card-hover     28 icon-hover
```

**7.4 — Le filet sans script est compilé, et la règle de réduction aussi**

```css
@media (scripting: none) { .animate-on-scroll { … } }
@media (prefers-reduced-motion: reduce) {
  .animate-on-scroll, .animate-on-scroll.is-visible { opacity: 1; transform: none }
  .card-hover, .card-hover:active, .card-hover:focus-visible .icon-hover, …
}
```

La règle globale du dépôt (`transition-duration: .01ms !important`) est
présente en amont, comme attendu. **Les deux mécanismes se cumulent
correctement** : l'option du système neutralise durée et état.

**7.5 — Non-régression HTTP**

| Contrôle | Résultat |
|---|---|
| `GET /` | 200, 105 491 o (contre 100 908 o avant : +4 583 o de classes et d'attributs) |

---

## 8. Vérifications

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npx oxlint src/` | **0 erreur**, 2 warnings préexistants hors périmètre |
| `npm run test:unit` | **534 / 534** |
| `.js` parasites dans `src/` | **aucun** |

**Non-régression prouvée** : `HEAD` propre (`git stash -u`) = **515 / 515**.
Les 534 incluent les 515 d'origine — aucun test perdu, aucun échec nouveau.

Les 2 warnings (`client/parrainage/page.tsx`, imports inutilisés) sont présents
sur `HEAD` propre, sans rapport avec ce chantier.

---

## 9. Ce qui n'a PAS été vérifié

**Le rendu visuel et le ressenti du scroll.** Le dépôt n'a ni `jsdom` ni
bibliothèque de test de composants, et `next build` est interdit en local
(RÈGLE 3). Les tests prouvent que le code est **présent et cohérent**, pas que
l'animation **s'est bien jouée**.

Les contrôles § 7 sont des vérifications HTTP : ils prouvent que les **bonnes
classes** sont livrées et servies, pas que l'**agencement** est correct à
l'écran.

Restent à confirmer manuellement sur mobile :

| Point | Risque |
|---|---|
| **Flash initial** | Blocs à `opacity: 0` entre le rendu serveur et le premier `IntersectionObserver` |
| **SEO au premier paint** | Le HTML est complet (donc indexable), mais invisible sans exécution de script |
| **Cascade sur 375 px** | `rootMargin: -50px` peut retarder l'apparition des premiers blocs |
| **Anneau de focus clavier** | § 5.3 — deux anneaux superposés, à confirmer au Tab |

---

## 10. Points d'attention

1. **`will-change` non posé.** La spec le mentionnait comme optionnel. Avec 28
   blocs, la propriété ne rend pas durablement et consomme de la mémoire GPU. À
   surveiller si le scroll saccade sur mobile bas de gamme.

2. **28 observateurs simultanés.** Acceptable jusqu'à ~50 éléments. Au-delà
   (nouvelle section longue), mutualiser un observateur unique.

3. **28 `useEffect` / `useState` client.** `landing-sections.tsx` reste un Server
   Component : importer un composant client imbriqué est pris en charge par
   React 19 sans transformer le parent. Vérifié par `tsc` et par l'absence de
   `'use client'` dans le fichier.

4. **Le retour en haut de page ne rejoue rien.** Comportement voulu
   (l'observateur se débranche) : un visiteur qui remonte voit les blocs déjà
   apparus, sans mouvement.

5. **Node 22 est requis et absent de la machine** — seul Node 20.20.2 est
   installé, et `nvm` n'est pas présent, donc `.nvmrc` est inopérant. Sur Node
   20, `npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` :
   fausse alerte, pas une régression. Chiffres obtenus avec Node 22.20.0
   installé **hors dépôt** (`/tmp/opencode/n22`).

6. **Les 16 keyframes CSS signalés inutilisés par l'audit sont inchangés.**
   Certains servent ailleurs (mission, célébration) ; le tri des morts est un
   chantier séparé.

7. **`rewards-hero.png` (432 Ko) toujours sans usage** — reporté du chantier
   précédent.

---

## 11. Questions bloquantes

**Aucune.** Les décisions produit et les trois écarts étaient tranchés avant
push.

Points ouverts, sans blocage :

- Confirmation navigateur des quatre points du § 9 ;
- Sort de `rewards-hero.png` (point 10.7) ;
- Node 22 sur la machine de développement (point 10.5).