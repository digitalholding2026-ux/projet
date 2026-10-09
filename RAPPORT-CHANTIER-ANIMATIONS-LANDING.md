# RAPPORT — Animations de la landing (modérée, CSS pur, micro-interactions)

> Chantier **écrit et vérifié, non poussé** : ce rapport est la demande de
> validation. Aucun commit applicatif, aucun push Vercel.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| HEAD au moment du chantier | `55771f0` |
| Commits produits | **aucun** — en attente de validation |
| Migratations | **aucune** |
| Dépendances ajoutées | **aucune** (décision produit : CSS pur, sans bibliothèque d'animation) |

Décisions produit appliquées : intensité modérée · stylesheet pur · micro-interactions complètes.

---

## 2. Synthèse

La landing gagne une apparition au scroll et des micro-interactions de survol,
**sans un octet de bundle supplémentaire**. Un composant client minimal
(`AnimateOnScroll`) ne décide que *quand* un bloc devient visible et remet la
main au navigateur ; toute la Durée, la courbe et l'amplitude vivent dans la
feuille de style.

Les **8 sections** demandées sont animées. Hero, bandeau de navigation et pied
de page sont laissés intacts — ils ont déjà leur propre traitement, et le menu
mobile vient d'être corrigé.

**Le rendu visuel n'a pas été vérifié** (§ 8). Aucun outil du dépôt ne permet
de le faire.

---

## 3. Fichiers

### Créés (2)

| Fichier | Lignes | Rôle |
|---|---|---|
| `src/components/landing/animate-on-scroll.tsx` | 74 | bascule l'état visible au premier croisement |
| `src/lib/landing-animations.test.ts` | 207 | 19 tests |

### Modifiés (6)

| Fichier | Nature |
|---|---|
| `src/app/globals.css` | +87 l. — section « apparitions au scroll » |
| `src/components/landing/landing-sections.tsx` | 5 sections enveloppées |
| `src/components/landing/faq.tsx` | accordéons enveloppés |
| `src/components/landing/trust-stats.tsx` | 2 indicateurs enveloppés |
| `package.json` | enregistrement dans `test:unit` |
| `tsconfig.json` | ajout à la liste d'exclusion |

---

## 4. Sections animées

| Section | Fichier | Nombre | Délai |
|---|---|---|---|
| Comment ça marche | `landing-sections.tsx` | 4 | `index * 100` |
| Pourquoi Relio | `landing-sections.tsx` | 4 | `index * 100` |
| Services (6 tuiles) | `landing-sections.tsx` | 6 | `Math.min(index * 60, 300)` |
| Récompenses (3 cartes) | `landing-sections.tsx` | 3 | `index * 100` |
| Devenir technicien | `landing-sections.tsx` | 1 | — |
| CTA final | `landing-sections.tsx` | 1 | — |
| Preuves sociales | `trust-stats.tsx` | 2 | `index * 150` |
| FAQ | `faq.tsx` | 7 | `Math.min(index * 50, 250)` |

**Total : 28 blocs animés.**

Deux régimes de cascade, volontairement distincts :

- **grille bornée** (3 à 7 entrées) → pas fixe de 50 à 150 ms, le dernier
  élément attend au plus quelques centaines de millisecondes ;
- **cascades longues** (6 tuiles, 7 questions) → pas plafonné par `Math.min`,
  sinon le dernier élément d'une liste de vingt entrées apparaîtrait bien après
  le défilement.

**Non animés, volontairement** : Hero, HeroBackground, PublicHeader,
PublicFooter. Un test verrouille cette liste — voir § 6.

---

## 5. Écarts avec la demande (3, dont 2 corrections de la spec)

### 5.1 Le délai de cascade était inopérant dans l'implémentation fournie

L'implémentation proposée posait `style={{ animationDelay }}` en ligne, alors
que la classe CSS utilise une **`transition`** (état de départ → état visible),
et non une animation. `animation-delay` ne s'applique pas à une transition :
**le décalage de cascade n'aurait produit aucun effet**, et les 28 blocs
apparaîtraient tous ensemble.

La spec indiquait `animationDelay`, la CSS qu'elle proposait indiquait
`transition` : les deux ne sont pas compatibles. Conservé la transition (elle
permet l'état de départ en CSS pur, donc un rendu serveur sans contenu masqué
« figé » par du JS), corrigé le délai.

### 5.2 Filet de sécurité sans JavaScript — **ajout non demandé**

La classe masque le contenu (`opacity: 0`) et attend que le composant ajoute
`is-visible`. **Sans JavaScript, ce basculement n'arrive jamais** : la landing
resterait invisible sur toute sa longueur. Ce n'est pas un défaut d'apparence,
c'est un contenu perdu — pour un visiteur à faible bande passante, un ancien
navigateur, ou un bloqueur de scripts.

Ajout d'une règle :

```css
@media (scripting: none) {
  .animate-on-scroll { opacity: 1; transform: none; }
}
```

Support navigateur plus récent que les animations d'apparition elles-mêmes. Elle
ne coûte rien et ferme le scénario.

### 5.3 Anneau de focus des tuiles ServicesGrid

La classe `.card-hover:focus-visible` pose un `outline`. Les tuiles portaient
aussi `focus-visible:outline-none` + `focus-visible:ring-2` (anneau Tailwind).
Les deux se concurrencent.

L'`outline-none` a été retiré : les deux anneaux restent superposés, mais
c'est la règle explicite qui gagne. Aucun test ne le vérifiait — c'est un point
à confirmer au clavier.

---

## 6. Tests

**19 tests** dans `landing-animations.test.ts`, soit **534/534** au total
(515 avant).

| Famille | Nb | Ce qui est verrouillé |
|---|---|---|
| Composant | 5 | directive client · `IntersectionObserver` · déconnexion · `prefers-reduced-motion` · absence de bibliothèque d'animation |
| Feuille de style | 7 | état masqué puis révélé · amplitude 28 px / 600 ms · survol · focus clavier · réduction sur les **deux** états · réduction du survol · filet sans script |
| Sections | 4 | import · au moins 5 usages sur la landing · cascades plafonnées · micro-interactions |
| Intégrité | 3 | hero/header/footer non enveloppés · accordéon non animé · chaque fichier d'une section animée l'est bien |

### Pourquoi ces tests existent

Une landing animée casse de deux façons qu'aucun outil statique ne voit :

1. **Le contenu reste invisible** — un bloc masqué dont l'état visible
   n'arrive jamais. Invisible au `tsc`, invisible au lint, invisible à la
   lecture du source. Visible à l'écran, ou plutôt invisible.
2. **Le mouvement passe outre le choix du visiteur** — l'option « moins de
   mouvements » ne neutralise que les durées si on ne surcharge pas aussi
   `opacity` et `transform`. Les blocs restent alors figés **à mi-course**
   (opacité 0, décalés de 28 px) au lieu d'afficher leur état final.

La règle globale du dépôt (`globals.css:238`) ramène les durées à
`0.01ms !important`, mais elle ne touche ni `opacity` ni `transform`. C'est
pourquoi le bloc `@media (prefers-reduced-motion: reduce)` ajouté cible les
**deux** états, et qu'un test le vérifie explicitement.

### Vérification par mutation

| Mutation | Résultat |
|---|---|
| Retirer `.is-visible` de la règle de réduction de mouvement | **1 échec** |
| Débrancher `TrustStats` de l'animation | **1 échec** (après correction d'une faille, § 6) |

Les tests échouent bien quand la régression est réintroduite.

### Faille trouvée dans mes propres tests

Le test « plusieurs sections animées » comptait les occurrences dans
**`landing-sections.tsx` seul**. Débrancher `TrustStats` — ou la FAQ — laissait
le compte global intact : **la mutation passait sans aucun échec**. Un test
vert qui ne prouve rien.

Corrigé par un test supplémentaire qui vérifie, fichier par fichier, que chacun
est animé. Mutation rejouée : détectée.

### Tests existants

Vérifiés verts, aucun modifié : `landing-rewards-content.test.ts` (11),
`navigation.test.ts` (ancres), `lottie.test.ts`, `demande-draft-routing.test.ts`,
`design-system.test.ts`, et les 33 autres.

Les ancres `id="how-title"`, `id="faq"`, `id="preuves-title"`… sont
**inchangées** : les enveloppements portent l'id, pas les sections.

---

## 7. Vérifications

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npx oxlint src/` | **0 erreur**, 2 warnings préexistants hors périmètre |
| `npm run test:unit` | **534 / 534** |
| Fichiers `.js` parasites dans `src/` | **aucun** |

**Non-régression prouvée** : `HEAD` propre (`git stash -u`) = **515 / 515**.
Les 534 incluent les 515 d'origine — aucun test perdu, aucun échec nouveau.

Les 2 warnings (`client/parrainage/page.tsx`, imports inutilisés) sont
présents sur `HEAD` propre, sans rapport avec ce chantier.

---

## 8. Ce qui n'a PAS été vérifié

**Le rendu visuel.** Le dépôt n'a ni `jsdom` ni bibliothèque de test de
composants, et `next build` est interdit en local (RÈGLE 3). Les tests
prouvent que le code est **présent et cohérent**, pas que l'animation **s'est
bien jouée**.

Trois points que seul un navigateur peut trancher :

| Risque | Description |
|---|---|
| **Flash initial** | Blocs à `opacity: 0` entre le rendu serveur et le premier `IntersectionObserver`. La spec le jugeait acceptable ; c'est à confirmer à l'œil. |
| **SEO** | La landing est un Server Component : le HTML est complet, mais **invisible au premier paint**. Le contenu reste indexable (le HTML est présent), mais un crawler sans exécution de scripts peut ne pas le voir. À vérifier sur le rendu servi. |
| **Mobile 375 px** | Le `rootMargin: -50px` peut retarder l'apparition des premiers blocs sur petit écran. |

Le menu mobile (corrigé au chantier précédent) n'a pas été touché : aucun
enveloppe sur `PublicHeader`.

---

## 9. Points d'attention

1. **`will-change` n'est pas posé.** La spec le mentionnait comme optionnel.
   Non utilisé ici : avec 28 blocs, une telle propriété ne rend durablement et
   consomme de la mémoire GPU. À surveiller si le scroll saccade sur mobile bas
   de gamme.

2. **28 observateurs simultanés.** Acceptable jusqu'à ~50 éléments. Au-delà
   (nouvelle section longue), mutualiser un observateur unique.

3. **28 `useEffect` / `useState` client.** Le fichier `landing-sections.tsx`
   reste un Server Component ; l'import d'un composant client imbriqué est pris
   en charge par React 19 sans transformer le fichier parent. Vérifié par
   `tsc` et par l'absence de `'use client'` dans le fichier.

4. **Le retour en haut de page ne rejoue rien.** Comportement voulu
   (l'observateur se débranche). Un visiteur qui remonte voit les blocs déjà
   apparus, sans mouvement.

5. **Node 22 est requis et absent de la machine** — seul Node 20.20.2 est
   installé, et `nvm` n'est pas présent, donc `.nvmrc` est inopérant. Sur Node
   20, `npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` :
   fausse alerte, pas une régression. Les chiffres de ce rapport ont été
   obtenus avec Node 22.20.0 installé **hors dépôt** (`/tmp/opencode/n22`).

6. **Les 16 keyframes CSS inutilisés** signalés par l'audit sont **inchangés**.
   Le chantier ne les touche pas : certains servent ailleurs (mission,
   célébration), et le tri des morts est un chantier séparé.

---

## 10. Questions bloquantes

**Aucune pour l'écriture du code.** Les trois décisions produit étaient
rendues avant le chantier.

Points ouverts, sans blocage :

- **Le rendu visuel reste à confirmer** (§ 8). C'est la seule réserve sérieuse :
  la validation navigateur est nécessaire avant de conclure que le chantier est
  terminé.
- Faut-il conserver `rewards-hero.png` (432 Ko, toujours sans usage) ?

---

## 11. Décision attendue

```
☐ Valider — pousser 55771f0 + le nouveau commit animations
☐ Ne pas valider — corriger avant push (préciser)
```