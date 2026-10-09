# RAPPORT — Refonte UI `/devenir-technicien` (mobile-first)

> Chantier **écrit et vérifié, non poussé**. Rapport de demande de validation.
> Aucun commit applicatif, aucun push Vercel.
> **Aucun texte, contenu, ordre de section ni logique n'a été modifié.**

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| HEAD | `5cc5776` |
| Commits produits | **aucun** — en attente de validation |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |

---

## 2. Synthèse

La page s'ouvre sur du **profond** au lieu d'un aplat orange, et les dix
sections alternent désormais leurs fonds au lieu de former une suite de blocs
identiques. Les cartes se soulèvent au survol, l'illustration du parcours est
intégrée, les chiffres tiennent la taille qui porte leur section.

Ordre des sections, textes, ancres et composants fonctionnels : **inchangés**.

---

## 3. Fichiers

| Fichier | Nature |
|---|---|
| `src/app/devenir-technicien/page.tsx` | hero, rythme, illustration, cartes, FAQ, CTA — +243 / −124 |
| `src/app/devenir-technicien/recrutement-stats.tsx` | chiffres XXL, sans carte — +69 / −24 |
| `src/app/globals.css` | `.recruit-halo`, `.card-premium` — +31 |
| `src/lib/technician-recruitment-ui.test.ts` | **créé** — 12 tests |
| `src/lib/technician-recruitment-ui.test.ts` | enregistré dans `package.json` et `tsconfig.json` |
| `public/technicien/parcours-illustration.png` | **déplacé** depuis `public/` (renommage suivi par git) |

**8 fichiers** au total (création, déplacement et modifications confondus).

---

## 4. Sections — le tableau demandé est respecté à la lettre

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

**5 halos décoratifs**, tous `pointer-events: none` (un halo intercepterait les
clics sur les CTA qu'il recouvre).

### Typographie (Partie G)

- **H2** : `text-3xl md:text-4xl font-bold` — avant `text-xl sm:text-2xl`
- **Sous-titres** : `text-base md:text-lg` — avant `text-sm`
- **Respiration** : `py-12 md:py-16` sur les sept sections de contenu
- **Grilles** : `gap-4 md:gap-6` — avant `gap-3`
- **Avant première carte** : `mt-8 md:mt-12` — avant `mt-6`

Les deux bandeaux (hero, CTA final) gardent leur padding par contenu : ce sont
des blocs pleins, pas des sections de contenu.

---

## 5. Illustration du parcours

Intégrée en tête de section 3, au-dessus des quatre étapes.

- `width={1600}` / `height={900}` : **obligatoires**, sans quoi le navigateur
  réserve zéro hauteur et la page saute au chargement
- `sizes` responsive, `alt` renseigné
- Dégradé de fond pour fondre l'image dans la section plutôt que la poser
  comme une carte de plus
- **Poids : 574 122 o** — voir § 9.1

---

## 6. Vérifications

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npx oxlint src/` | **0 erreur**, 2 warnings préexistants hors périmètre |
| `npm run test:unit` | **546 / 546** |

**Non-régression prouvée** : `HEAD` propre (`git stash -u`) = **534 / 534**.
Les 546 incluent les 534 d'origine — aucun test perdu, aucun échec nouveau.

Les 2 warnings (`client/parrainage/page.tsx`, imports inutilisés) sont
présents sur `HEAD` propre, sans rapport avec ce chantier.

### Les 12 nouveaux tests

| Famille | Nb | Ce qui est verrouillé |
|---|---|---|
| Bandeau | 2 | fond profond · absence d'aplat orange · halos · `pointer-events: none` |
| Rythme | 3 | ≥3 fonds alternés · 2 fonds profonds · respiration des 7 sections de contenu |
| Illustration | 2 | chemin branché **+ fichier présent sur disque** · `width`/`height`/`alt` |
| Chiffres | 2 | `text-5xl` et `md:text-6xl` · filet vertical unique |
| Cartes | 2 | classe unique partagée · élévation `translateY(-2px)` · neutralisée si moins de mouvements |
| Intégrité | 1 | la page reste un composant serveur |

Validés **par mutation** :

| Mutation | Résultat |
|---|---|
| Hero repassé en `bg-orange-500` | **3 échecs** |
| Chiffres repassés en `text-2xl` | **1 échec** |

### Tests existants : ordre des sections préservé

`devenir-technicien.test.ts` (23 tests) vérifie l'**ordre des dix sections**
par `indexOf` sur les ancres `aria-labelledby`. Ordre préservé, test vert.

---

## 7. Bugs trouvés

### 7.1 Un test existant plus strict que la demande

`devenir-technicien.test.ts:331` verrouille deux chaînes **littérales** :

```
/<ul className="mt-6 grid gap-3 sm:hidden">/
/hidden w-full border-collapse[\s\S]*?sm:table/
```

J'avais appliqué le nouveau rythme (`mt-8`, `rounded-2xl`) à ces deux blocs :
**le test a cassé**. Les classes ont été **restaurées à l'identique**.

**Conséquence assumée** : l'espacement du tableau et des cartes mobiles du
barème (section 6) reste sur l'ancien `mt-6` et `rounded-xl`. **Ces deux blocs
n'ont pas reçu le nouveau rythme**, alors que leurs voisins l'ont.

À trancher : élargir l'assertion, ou laisser ces deux blocs en écart.

### 7.2 Trois de mes propres tests étaient faux

Le code produit était correct dans les trois cas ; ce sont les tests qui
étaient mal écrits.

1. **Chemin d'asset faux** — `../public/…` depuis `src/lib/` résout vers
   `src/public/`, inexistant. Le test concluait que l'image manquait alors
   qu'elle était là. Corrigé en `../../public/…`.
   → **C'est la deuxième fois que je commets cette erreur** sur ce dépôt, après
   le même cas dans `landing-animations.test.ts`. Signalée, pas passée sous
   silence : elle se reproduit parce que je raisonne sur le dépôt racine au
   lieu du répertoire du fichier de test.

2. **Mauvaise media query ciblée** — `lastIndexOf('@media (prefers-reduced-motion: reduce)')`
   trouvait la règle **globale** du dépôt (qui ne traite que les durées), pas
   celle de `.card-premium`. Le test aurait validé la mauvaise règle. Corrigé
   en cherchant la media query **située après** la déclaration de la classe.

3. **Comptage de sections faux** — la fenêtre de recherche ne captait pas les
   `className` posées sur la ligne suivante, soit **les quatre sections
   longues** : celles-là échappaient au comptage. Le seuil de 9 était également
   faux (les deux bandeaux n'ont pas de respiration verticale). Corrigé avec
   une assertion exacte : 9 sections, 7 avec padding.

### 7.3 Un accident de lecture pendant la vérification

En ajoutant les classes CSS, un bloc a été **remplacé** au lieu d'être inséré
avant : l'en-tête de commentaires du bloc « apparitions au scroll » a disparu.
Repéré immédiatement par `tsc` + la suite de tests, restauré. Sans ce double
contrôle, la page d'accueil perdait 12 lignes de commentaire.

---

## 8. Ce qui n'a pas été fait

- **Aucun test de rendu** — ni `jsdom` ni bibliothèque de composants, et
  `next build` est interdit (RÈGLE 3). Les tests prouvent que les classes sont
  **présentes et cohérentes**, pas que la page est **belle**.
- **Aucun test mobile** — les scénarios 375 px et desktop sont à confirmer à la main.
- **`shadow-card` conservé** sur le tableau et les cartes mobiles du barème
  (cf. 7.1).

---

## 9. Points d'attention

### 9.1 L'illustration dépasse le seuil de poids que vous aviez fixé

| Fichier | Poids | Seuil 500 Kio |
|---|---|---|
| `parcours-illustration.png` | **574 122 o** | **dépassé de 74 Kio** |

Les visuels du programme de récompenses avaient été recompressés pour passer
sous 500 Kio. Celui-ci échappe à ce traitement. Il n'a pas été recompressé :
l'environnement n'a pas d'outil de compression d'image, et la mission
interdit de redimensionner côté code.

### 9.2 Le fond profond du CTA final perd son halo sur petits écrans

Le halo central (`size-72`/`sm:size-96`) est positionné en
`-top-24 left-1/2 -translate-x-1/2`. À 375 px, une section `p-6` est plus
étroite que le halo translaté, qui déborde donc franchement — le `overflow-hidden`
de la section le tronque proprement, mais l'effet est faible. À confirmer.

### 9.3 `bg-muted/30` est un gris très clair, pas un gris « très clair »

La spec disait `bg-muted/30`. `--muted` vaut `#eef0f3` en clair, donc `30 %`
de ce gris sur fond blanc donne **#f9fafb** — presque blanc. L'alternance
risque d'être **trop subtile pour être perçue** sur un écran de téléphone en
plein soleil. Si le rythme reste plat au test, il faudra monter à
`bg-muted/60` ou `bg-muted`.

### 9.4 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` n'est pas présent, `.nvmrc` est
inopérant. Sur Node 20, `npm run test:unit` échoue **39/39** en
`ERR_UNKNOWN_FILE_EXTENSION` : fausse alerte, pas une régression. Chiffres
obtenus avec Node 22.20.0 installé **hors dépôt** (`/tmp/opencode/n22`).

### 9.5 `rewards-hero.png` (432 Ko) toujours sans usage — reporté.

---

## 10. Questions bloquantes

**Aucune pour le push**, mais deux arbitrages sont recommandés avant :

1. **§ 9.1 — compressor l'illustration ?** Elle dépasse de 74 Kio le seuil que
   vous aviez fixé pour les autres visuels. Fournir un fichier recompressé
   (comme pour les récompenses), ou accepter le poids en l'état ?
2. **§ 7.1 — élargir `devenir-technicien.test.ts` ?** Sinon le tableau et les
   cartes mobiles du barème restent les deux seuls blocs de la page sans le
   nouveau rythme.

---

## 11. Décision attendue

```
☐ Valider — pousser le commit refonte UI
☐ Compresser d'abord l'illustration (574 Ko → < 500 Ko)
☐ Élargir l'assertion de devenir-technicien.test.ts, puis pousser
☐ Corriger avant push (préciser)
```