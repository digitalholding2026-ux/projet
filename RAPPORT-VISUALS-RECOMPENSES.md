# RAPPORT — Remplacement des visuels du programme de récompenses

> Chantier **non démarré** : bloqué avant toute modification de code. Ce rapport
> est un document de validation, à faire relire avant d'engager le chantier.
> Aucun fichier source modifié, aucun commit applicatif, aucun push Vercel.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) |
| HEAD au moment de l'audit | `cc5e7a8` (*Add files via upload*) |
| Commits produits par ce chantier | **aucun** |
| Fichiers source modifiés | **aucun** |
| État push / déploiement | inchangé — les visuels sont déjà en ligne sur Vercel |

Les 4 visuels ont été **récupérés depuis GitHub** (`69c4fd6` + `cc5e7a8`) :
la suppression de l'ancien `smartphone.png` et l'ajout des 4 nouveaux fichiers
étaient déjà poussés par le demandeur.

---

## 2. Synthèse

La partie **landing** est faisable immédiatement : les 3 références d'images de
`landing-sections.tsx` se remplacent 1:1 par les 3 visuels reçus.

La partie **page client** est bloquée. `reward-catalog.tsx` consomme **6 visuels**
alors que 3 seulement ont un remplaçant, et ledit composant est **mort** — il
n'est monté nulle part et son contenu intégral est encore sur l'ancien modèle
« nombre de missions », que la page `/client/recompenses` a déjà abandonné.

---

## 3. Fichiers déposés par le demandeur (récupérés)

`frontend/public/recompense/` — poids réels :

| Fichier | Poids | Attendu pour |
|---|---|---|
| `petit-electromenager.png` | 1 875 151 o | Palier 50 000 FCFA de marge |
| `electromenager-moyen.png` | 1 964 606 o | Palier 100 000 FCFA de marge |
| `smartphone.png` | 1 651 127 o | Palier 250 000 FCFA de marge (écrase l'ancien) |
| `rewards-hero.png` | 1 729 778 o | Hero de `/client/recompenses` (**aucun emplacement en code**) |

---

## 4. Audit des usages (exhaustif, avant modification)

Commandes : `grep -rn "recompense/" src/` et
`grep -rniE "tshirt_cap|free_repair|iron\.|mystery_box|tv\.png" src/`.

**2 fichiers seulement.** `/client/recompenses/page.tsx` et
`/client/parrainage/page.tsx` ne référencent **aucun** visuel : la page est
pilotée par les API backend (marge cumulée, paliers, crédits).

### 4.1 `src/components/landing/landing-sections.tsx`

| Ligne | Actuel | Cible | Statut |
|---|---|---|---|
| 220 | `/recompense/tshirt_cap.png` | `/recompense/petit-electromenager.png` | faisable |
| 225 | `/recompense/tv.png` | `/recompense/electromenager-moyen.png` | faisable |
| 230 | `/recompense/smartphone.png` | inchangé (nouveau fichier) | fait |

Les `alt` s'appuient sur `reward.title` (`Petit électroménager`,
`Électroménager moyen`, `Smartphone`) — cohérents automatiquement, aucune
retouche nécessaire. Commentaire de cadrage l.212-214 à réécrire : il documente
explicitement le décalage image ↔ récompense comme « point connu, non traité »,
ce décalage étant désormais corrigé.

### 4.2 `src/components/client/recompenses/reward-catalog.tsx`

| Ligne | Actuel | Remplaçant reçu |
|---|---|---|
| 31 | `/recompense/free_repair.png` | **aucun** |
| 39 | `/recompense/tshirt_cap.png` | `petit-electromenager.png` ? |
| 47 | `/recompense/iron.png` | **aucun** |
| 55 | `/recompense/tv.png` | `electromenager-moyen.png` ? |
| 63 | `/recompense/smartphone.png` | `smartphone.png` |
| 71 | `/recompense/mystery_box.png` | **aucun** |

---

## 5. Constat bloquant : `reward-catalog.tsx` est du code mort

### 5.1 Le composant n'est monté nulle part

`RewardCatalog` n'est importé par **aucun fichier** de `src/`. Le composant et
ses 240 lignes sont inatteignables à l'exécution.

Il subsiste parce que **deux tests statiques le lisent** :

- `src/lib/design-system.test.ts:47` — le fichier est dans `HEX_ALLOWLIST`
  (seule raison de sa présence : il contredit la règle « aucun hex arbitraire »).
- `src/lib/demande-draft-routing.test.ts:124` — vérifie la présence de `/demande`.

### 5.2 Son contenu est intégralement obsolète

Ce n'est pas seulement les visuels : **les textes** sont sur l'ancien modèle.

| Champ | Valeurs actuelles |
|---|---|
| `tier` | `PALIER 5 DÉPANNAGES` (×2), `12`, `25`, `50`, `100 DÉPANNAGES` |
| `requiredServices` | 5, 5, 12, 25, 50, 100 |
| Commentaire l.23-24 | « le compteur de dépannages confirmés, lui, est réel » |

Le programme réel est fondé sur la **marge cumulée** chez Relio :
`NATURE_THRESHOLDS` = 50 000 / 100 000 / 250 000 FCFA
(`backend/src/rewards/rewards.config.ts:113-116`), awardés en
`CLIENT_REWARD_CREDIT` sur demande. `requiredServices` n'a plus de sens
métier : le badge « Verrouillé (3/12) » affiche un compteur de missions.

### 5.3 Pourquoi ce fichier n'a pas été nettoyé au chantier précédent

Le chantier de la landing (`bf3fd5b`) a traité `landing-sections.tsx`,
`faq.tsx` et `public-header.tsx`. `reward-catalog.tsx` est **hors du périmètre
de la page** : il est invisible sur `/client/recompenses`, dont le contenu vient
de `rewards-view.ts` piloté par l'API. L'écart est donc resté invisible — et
aucun test ne le signale, les deux tests le lisant ne portant que sur la couleur
et un lien.

---

## 6. Arbitrage demandé

### Voie A — supprimer `reward-catalog.tsx` (**recommandée**)

- Supprimer 240 lignes de code mort, sur un modèle de programme aboli.
- Retirer `reward-catalog.tsx` de `design-system.test.ts:47` (allowlist) et de
  `demande-draft-routing.test.ts:124` (liste).
- Aucun visuel n'est orphelin : les 5 PNG restants seraient alors eux aussi
  supprimables (`free_repair`, `iron`, `mystery_box`, `tshirt_cap`, `tv`).
- **Aucune modification visible** sur `/client/recompenses` : la page ne l'a
  jamais utilisé.

### Voie B — migrer `reward-catalog.tsx` sur le modèle marge

- Aligner `tier` et le déverrouillage sur la marge cumulée, en lieu et place de
  `requiredServices`.
- Nécessite **6 visuels**, ou 3 visuels + 3 entrées nature/alimentation.
- Réactive un composant inutilisé : le travail ne sert aucun écran.

### `rewards-hero.png`, cas à part

Le fichier est fourni mais **aucun code ne référence de hero** sur
`/client/recompenses`. L'ajouter = création d'un emplacement, donc hors du
strict « remplacement de références » demandé.

---

## 7. Nettoyage des anciens fichiers

`frontend/public/recompense/` contient encore 5 fichiers non remplacés :

| Fichier | Poids | Référencé par |
|---|---|---|
| `free_repair.png` | 1 914 796 o | `reward-catalog.tsx:31` |
| `iron.png` | 1 932 242 o | `reward-catalog.tsx:47` |
| `mystery_box.png` | 1 910 350 o | `reward-catalog.tsx:71` |
| `tshirt_cap.png` | 1 874 017 o | `reward-catalog.tsx:39` + `landing-sections.tsx:220` |
| `tv.png` | 1 896 232 o | `reward-catalog.tsx:55` + `landing-sections.tsx:225` |

**Aucun ne pourra être supprimé tant que `reward-catalog.tsx` existe** : le
`grep` de vérification doit retourner 0 avant suppression (RÈGLE imposée par
la mission). Le nettoyage est donc **subordonné à l'arbitrage § 6**.

---

## 8. Performance

Les **7 PNG** de `public/recompense/` pèsent **~12,6 Mo** au total ; chaque
fichier est entre 1,65 et 1,96 Mo. Le critère « chaque image < 500 Ko » du
scénario de production n'est atteint par aucun d'entre eux, anciens compris.

La mission interdit de redimensionner côté code et l'environnement n'a pas
d'outil de compression d'image. **L'optimisation doit venir des fichiers
fournis**, ou d'un outillage CI à décider.

---

## 9. Tests

### À ajouter (une fois l'arbitrage arbitré)

Sur `landing-sections.tsx` :
- contient `petit-electromenager.png`, `electromenager-moyen.png`, `smartphone.png`
- ne contient plus `tshirt_cap.png`, `tv.png`, `free_repair.png`, `iron.png`, `mystery_box.png`
- ne contient plus `'5 dépannages'`, `'25 dépannages'`, `'50 dépannages'`

Un test existe déjà et couvre la 3ᵉ ligne :
`src/lib/landing-rewards-content.test.ts` (7 tests, ajouté en `bf3fd5b`).
Il reste à l'étendre aux deux premières lignes.

### Tests existants à ne pas casser

| Test | Lien avec le chantier |
|---|---|
| `design-system.test.ts:47` | `reward-catalog.tsx` en allowlist hex |
| `demande-draft-routing.test.ts:124` | `reward-catalog.tsx` doit contenir `/demande` |
| `landing-rewards-content.test.ts` | seuils + absence d'obsolète (landing) |
| `navigation.test.ts:94-107` | ancres `id="…-title"` de la landing |
| `lottie.test.ts:130-131` | `MenuNavAnimation`, `aria-expanded` du header |

En **voie A**, les deux premiers tests doivent être modifiés : ce n'est pas une
cassure, mais un ajustement Textile à faire dans le même commit.

---

## 10. Vérifications déjà passées

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | exit 0 |
| `npx oxlint src/` | 0 erreur, 2 warnings préexistants (`client/parrainage/page.tsx`, imports inutilisés, sans rapport) |
| `npm run test:unit` | **511 / 511** verts |
| `git status` | propre — aucun fichier modifié par ce chantier |

> ⚠️ Node 22 requis et absent de la machine (seul Node 20.20.2 est installé).
> Sur Node 20, `npm run test:unit` échoue **39/39** en
> `ERR_UNKNOWN_FILE_EXTENSION` : fausse alerte, pas une régression. Node
> 22.20.0 a été installé **hors dépôt** (`/tmp/opencode/n22`) pour obtenir les
> chiffres réels ci-dessus.

---

## 11. Points d'attention

1. **Node 22 absent de la machine** — `nvm` n'est pas installé, `.nvmrc` est
   inopérant. À traiter dans un chantier dédié.
2. **`saspay-fees.test.ts:267`** lit `../../backend/src/financial/saspay-fees.ts` :
   le test frontend échoue si le dépôt backend n'est pas présent à côté.
3. **Test frontend `landing-rewards-content.test.ts` ≠ source de vérité** :
   comparer les seuils affichés au backend est une garantie de cohérence, pas
   de correction. La source reste `rewards.config.ts`.
4. Les tests frontend sont **statiques** (`readFileSync` + regex sur le texte
   intégral, commentaires compris). Un commentaire contenant un nom surveillé
   par une assertion négative peut casser un test — cf. RÈGLE 3.

---

## 12. Questions bloquantes

1. **`reward-catalog.tsx` : suppression (voie A) ou migration (voie B) ?**
   Recommandation : **voie A**.
2. **`rewards-hero.png` : où le placer ?** Aucun emplacement n'existe. Créer un
   hero sur `/client/recompenses`, ou garder le fichier inutilisé ?
3. **Optimisation des PNG** : les 7 fichiers pèsent ~12,6 Mo. Les fournir
   recompressés, ou accepter le poids en l'état ?

---

## 13. Décision attendue

```
☐ Voie A — supprimer reward-catalog.tsx (recommandé)
☐ Voie B — migrer reward-catalog.tsx sur le modèle marge
☐ Landing immédiatement, reward-catalog arbitré plus tard
☐ rewards-hero.png : créer un hero sur /client/recompenses
☐ rewards-hero.png : ne pas l'utiliser
```