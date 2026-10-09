# RAPPORT — Refonte UI `/devenir-technicien` (mobile-first)

> Chantier **livré et déployé**. Rapport factuel : chiffres réels, écarts avec
> la demande notés, vérifications de production distinguées des vérifications
> statiques.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| Commit | `4d4c517` — *feat(technicien): refonte UI de /devenir-technicien (visuel seul)* |
| Push | ✅ `5cc5776..4d4c517`, branche `main` |
| Vercel | ✅ **vert et vérifié factuellement** (§ 8) |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |

**Aucun texte, aucun contenu, aucun ordre de section, aucune logique n'a été
modifié.** Uniquement des classes CSS et une illustration.

Deux arbitrages demandés avant push, appliqués :

1. **Assertion élargie** sur `devenir-technicien.test.ts`, puis nouveau rythme
   appliqué aux deux blocs du barème (§ 5.1).
2. **`bg-muted/30` → `bg-muted/60`**, pour un rythme perceptible au soleil (§ 4.2).

L'illustration à **574 Ko** est poussée telle quelle, sur arbitrage explicite.

---

## 2. Synthèse

La page s'ouvre sur du **profond** au lieu d'un aplat orange, et les dix
sections alternent leurs fonds au lieu de former une suite de blocs
identiques. Les cartes se soulèvent au survol, l'illustration du parcours est
intégrée, les chiffres tiennent la taille qui porte leur section.

---

## 3. Fichiers (8)

| Fichier | Nature |
|---|---|
| `src/app/devenir-technicien/page.tsx` | hero, rythme, illustration, cartes, FAQ, CTA — +153 / −92 |
| `src/app/devenir-technicien/recrutement-stats.tsx` | chiffres XXL, sans carte — +45 / −24 |
| `src/app/globals.css` | `.recruit-halo`, `.card-premium` — +43 |
| `src/lib/technician-recruitment-ui.test.ts` | **créé** — 12 tests, 174 l. |
| `src/lib/devenir-technicien.test.ts` | assertion élargie — +14 / −6 |
| `src/lib/technician-recruitment-ui.test.ts` | enregistré dans `package.json` et `tsconfig.json` |
| `public/technicien/parcours-illustration.png` | **déplacé** depuis `public/` (renommage suivi par git) |
| `package.json` · `tsconfig.json` | enregistrement du nouveau fichier de test |

**+426 / −130.**

---

## 4. Sections

| # | Section | Avant | Après |
|---|---|---|---|
| 1 | Hero | aplat orange, `brand-gradient` | **noir profond** + 2 halos |
| 2 | Relio en chiffres | carte blanche, chiffres `text-2xl` | chiffres **`text-5xl md:text-6xl`**, sans carte |
| 3 | Votre parcours | blanc | **gris clair** + **illustration** + halo |
| 4 | Une intervention | blanc | blanc |
| 5 | Pourquoi rejoindre | blanc | **gris clair** + halo |
| 6 | Combien gagner | blanc | blanc |
| 7 | Ce qu'on attend | blanc | **gris clair** |
| 8 | Notre engagement | blanc | blanc |
| 9 | FAQ | blanc | **gris clair**, ombres, chevron orange |
| 10 | CTA final | carte blanche | **noir profond** + halo centré |

**10 halos décoratifs** au total (2 par bandeau, 1 par section alternée),
tous `pointer-events: none` : un halo intercepterait les clics sur les CTA
qu'il recouvre.

### 4.1 Typographie et espacements

| Élément | Avant | Après |
|---|---|---|
| H2 | `text-xl sm:text-2xl` | `text-3xl md:text-4xl` |
| Sous-titres | `text-sm` | `text-base md:text-lg` |
| Respiration | `mt-12` + cartes `mt-6` | `py-12 md:py-16` sur les 7 sections de contenu |
| Grilles | `gap-3` | `gap-4 md:gap-6` |
| Avant première carte | `mt-6` | `mt-8 md:mt-12` |
| Cartes | `rounded-xl p-5 shadow-card` | `rounded-2xl p-6` + élévation |

### 4.2 Le gris alterné est à 60 %, pas 30 %

`--muted` vaut `#eef0f3`. À 30 %, le composite donne `#f9fafb` sur fond blanc :
le rythme était **presque invisible**. Passé à **60 %** (≈ `#f4f5f7`), l'alternance
se perçoit.

La valeur est **fixée par un test** (`bg-muted\/60`, comptage exact) : elle ne
peut pas redescendre en silence à la prochaine refonte.

Le dégradé de fond sous l'illustration a suivi (`from-muted/30` → `from-muted/60`) :
à 30 %, il fondait dans un fond plus clair que lui.

---

## 5. Ce qui a été demandé avant push

### 5.1 Assertion élargie, puis rythme appliqué

`devenir-technicien.test.ts:331` **verrouillait deux chaînes littérales** :

```js
assert.match(page, /hidden w-full border-collapse[\s\S]*?sm:table/);
assert.match(page, /<ul className="mt-6 grid gap-3 sm:hidden">/);
```

La seconde interdisait **toute** classe supplémentaire. Conséquence constatée au
chantier : après avoir appliqué le nouveau rythme à ces deux blocs, le test a
cassé et les classes ont dû être **restaurées à l'identique**. Le tableau et
les cartes mobiles du barème restaient alors les **deux seuls blocs de la page
hors du rythme**.

L'assertion vérifie désormais l'**intention** :

| Avant (littéral) | Après (intention) |
|---|---|
| chaîne `<ul>` complète | `/<ul[^>]*\bsm:hidden\b[^>]*>/` |
| `hidden … sm:table` en ordre fixe | `/<table[^>]*\bhidden\b[^>]*\bborder-collapse\b[^>]*\bsm:table\b[^>]*>/` |

Invariants réellement protégés : la bascule mobile, la grille, et la source
partagée (`QUOTE_AMOUNTS_XAF.map` apparaît exactement 2 fois).

**Les deux blocs ont ensuite reçu le nouveau rythme** : `mt-6` → `mt-8 md:mt-12`,
`rounded-xl` → `rounded-2xl`, `gap-3` → `gap-4`, et `card-premium` sur les
cartes mobiles. **Aucun bloc de la page n'est désormais hors rythme.**

Assertions élargies **validées par mutation** : retirer la bascule du tableau →
1 échec ; retirer celle des cartes → 1 échec.

### 5.2 Illustration à 574 Ko

Poussée telle quelle, sur arbitrage explicite. Elle dépasse de **74 Kio** le
seuil de 500 Kio retenu pour les visuels du programme de récompenses. Point
ouvert, non bloquant (§ 9.1).

---

## 6. Tests

**12 tests** créés — **546 / 546** au total.

| Famille | Nb | Ce qui est verrouillé |
|---|---|---|
| Bandeau | 2 | fond profond · absence d'aplat orange · halos · `pointer-events: none` |
| Rythme | 3 | ≥3 fonds alternés · **taux à 60 %** · respiration des 7 sections de contenu |
| Illustration | 2 | chemin branché **+ fichier présent sur disque** · `width`/`height`/`alt` |
| Chiffres | 2 | `text-5xl` et `md:text-6xl` · filet vertical unique |
| Cartes | 2 | classe unique partagée · `translateY(-2px)` · neutralisée si moins de mouvements |
| Intégrité | 1 | la page reste un composant serveur |

Validés **par mutation** :

| Mutation | Résultat |
|---|---|
| Hero repassé en `bg-orange-500` | **3 échecs** |
| Chiffres repassés en `text-2xl` | **1 échec** |
| Bascule mobile retirée du tableau | **1 échec** |
| Bascule mobile retirée des cartes | **1 échec** |

### Non-régression

| État | Résultat |
|---|---|
| `HEAD` propre (`git stash -u`) | **534 / 534**, 0 échec |
| Après chantier | **546 / 546**, 0 échec |

Les 546 incluent les 534 d'origine — aucun test perdu, aucun échec nouveau.

---

## 7. Vérifications

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npx oxlint src/` | **0 erreur**, 2 warnings préexistants hors périmètre |
| `npm run test:unit` | **546 / 546** |

Les 2 warnings (`client/parrainage/page.tsx`, imports inutilisés) sont présents
sur `HEAD` propre, sans rapport avec ce chantier.

---

## 8. Déploiement Vercel — vérifié, pas supposé

Un HTTP 200 ne prouve pas qu'un nouveau build est servi. Cinq contrôles.

**8.1 — La page servie contient la refonte**

`GET /devenir-technicien` → 200, 115 435 o.

```
bg-muted/60   ×8      bg-relio-bg  ×4
card-premium  ×52     recruit-halo ×10
```

**8.2 — L'illustration est servie à sa taille exacte**

```
/technicien/parcours-illustration.png   200   574 122 o
```

**8.3 — L'ancien chemin est bien mort**

```
/parcours-illustration.png   404
```

**8.4 — L'aplat orange a disparu du bandeau**

`grep -c "brand-gradient"` sur la page servie → **0**.

**8.5 — La feuille de style du chantier est servie**

Deux CSS référencés. `8bf40afc34d4668d.css` contient `.card-premium` ;
l'autre (`020d714…`, inchangée) ne le contient pas. Le build est bien `4d4c517`.

---

## 9. Points d'attention

### 9.1 L'illustration dépasse le seuil de poids (74 Kio au-dessus)

Arbitré explicitement par le demandeur. Le premier affichage de la section 3
coûte 574 Ko sur mobile. À recompresser si la page est visée sur réseau lent.

### 9.2 Le rendu visuel n'est pas vérifié

Ni `jsdom` ni bibliothèque de composants, et `next build` interdit (RÈGLE 3).
Les contrôles § 8 prouvent que les **bonnes classes** sont livrées et servies,
pas que l'**agencement** est bon à l'écran.

Restent à confirmer manuellement :

| Point | Risque |
|---|---|
| Rythme à 60 % | perceptible sur mobile ? trop marqué sur desktop ? |
| Halo du CTA final | `size-72`/`sm:size-96` centré en `-top-24` : à 375 px il déborde franchement, l'`overflow-hidden` le tronque — effet faible possible |
| Illustration sur 375 px | hauteur réelle, lisibilité du texte interne |
| Chiffres XXL | `text-6xl` sur deux colonnes de 375 px ne déborde pas ? |

### 9.3 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` n'est pas présent, `.nvmrc` est
inopérant. Sur Node 20, `npm run test:unit` échoue **39/39** en
`ERR_UNKNOWN_FILE_EXTENSION` : fausse alerte, pas une régression. Chiffres
obtenus avec Node 22.20.0 installé **hors dépôt** (`/tmp/opencode/n22`).

### 9.4 Trois erreurs commises pendant ce chantier, corrigées

Consignées parce qu'elles sont instructives, pas pour excuser :

1. **Chemin d'asset erroné dans un test** — `../public/…` depuis `src/lib/`
   résout vers `src/public/`, inexistant ; le test concluait que l'image
   manquait alors qu'elle était là. Corrigé en `../../public/…`.
   → **Deuxième fois sur ce dépôt**, après le même cas dans
   `landing-animations.test.ts`. La cause est identifiée : je raisonne sur la
   racine du dépôt au lieu du répertoire du fichier de test.

2. **Media query ciblée à tort** — `lastIndexOf('@media (prefers-reduced-motion: reduce)')`
   trouvait la règle **globale** (qui ne traite que les durées) au lieu de celle
   de `.card-premium`. Le test aurait validé la mauvaise règle. Corrigé en
   cherchant la media query située **après** la déclaration de la classe.

3. **Comptage de sections faux** — la fenêtre de recherche ne captait pas les
   `className` posées sur la ligne suivante, soit les **quatre sections
   longues**. Le seuil était également faux (les deux bandeaux n'ont pas de
   respiration verticale). Corrigé en assertion exacte : 9 sections,
   7 avec padding.

Aucun de ces défauts n'a atteint le code produit : dans les trois cas,
`tsc` et la suite ont suffi à les révéler — jamais le lint.

### 9.5 `rewards-hero.png` (432 Ko) toujours sans usage — reporté.

---

## 10. Questions bloquantes

**Aucune.** Les deux arbitrages ont été appliqués, l'illustration validée en
l'état.

Points ouverts, sans blocage :

- Compression de l'illustration (§ 9.1) ;
- Confirmation navigateur des quatre points du § 9.2 ;
- Node 22 sur la machine de développement (§ 9.3).