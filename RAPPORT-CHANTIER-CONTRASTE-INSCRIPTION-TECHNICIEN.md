# RAPPORT — Fix UI : contraste et hiérarchie `/technicien/inscription`

> Chantier **livré et déployé**. Rapport factuel : chiffres réels, écarts avec
> la demande notés, vérifications de production distinguées des vérifications
> statiques.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| Commit | `05757c3` — *fix(auth): contraste et hiérarchie du tunnel technicien* |
| Push | ✅ `4d4c517..05757c3`, branche `main` |
| Vercel | ✅ **vert et vérifié factuellement** (§ 7) |
| Migrations | **aucune** |
| Dépendances ajoutées | **aucune** |

**Aucun texte, aucun ordre de champ, aucune logique de validation, aucun appel
API n'a été modifié.** Uniquement des classes CSS. Le thème sombre est conservé.

---

## 2. Synthèse

Le thème sombre était cohérent mais **illisible**, sur deux points précis :

1. **Le bouton de soumission ne distinguait pas l'inactif de l'actif.** Le design
   system n'a qu'un seul état désactivé — le bouton reste orange, seulement
   atténué. Sur fond sombre, un orange atténué se lit comme une **couleur**, pas
   comme une action indisponible.
2. **Les champs ne montraient pas où saisir.** Le thème clair posait un fond de
   carte sur un fond nuit, sans relief.

---

## 3. Fichiers identifiés

| Fichier | Rôle | Partagé |
|---|---|---|
| `src/components/technician/technician-auth-form.tsx` | formulaire | non |
| `src/components/auth/auth-split.tsx` | coquille + colonne garanties | **oui — sert aussi à `/client/connexion`** |
| `src/components/auth/technician-auth-split.tsx` | lie les deux | non |

`AuthSplit` étant partagé, les corrections ont été faites **dans le composant
partagé** (option (a) de la consigne), sans duplication. Le tunnel client en
profite : sa colonne de garanties est elle aussi sur fond nuit.

---

## 4. Fichiers modifiés (3)

| Fichier | Nature | Volume |
|---|---|---|
| `technician-auth-form.tsx` | bouton, 6 champs, catégories, messages d'aide | +58 / −9 |
| `auth/auth-split.tsx` | garanties, progression, badge, surcharge des champs, halo | +22 / −8 |
| `lib/technician-auth.test.ts` | **+6 tests** | +73 / −0 |

**+147 / −23.**

---

## 5. Contrastes corrigés

| Élément | Avant | Après |
|---|---|---|
| **Bouton inactif** | `bg-primary` + `opacity-50` → orange terne | `bg-muted text-muted-foreground opacity-60` |
| **Bouton actif** | même orange terne | `bg-primary text-primary-foreground` + `hover:bg-primary-hover` |
| **Inputs / Select** | `bg-slate-800/80` quasi indistint | `bg-white/5` + `border-white/10` + `placeholder:text-white/40` |
| **Focus champ** | `ring-orange-500/30` | bordure orange + halo orange |
| **Œil mot de passe** | `text-muted-foreground` | `text-slate-400` → `hover:text-white` |
| **Catégorie sélectionnée** | `bg-orange-500/20` | `bg-orange-500/15` + `border-orange-500` |
| **Catégorie non sélectionnée** | `bg-slate-800/60` + bordure sombre | `bg-white/5` + `border-white/10` + hover `white/10` |
| **Icône catégorie (non sélectionnée)** | `text-slate-400` | `text-white/60` |
| **Carte de garantie** | `<li>` sans surface | `bg-white/5` + `border-white/10` + `backdrop-blur-sm` |
| **Titre garantie** | `text-sm text-white` | inchangé |
| **Description garantie** | `text-xs text-slate-400` | `text-sm text-white/70` |
| **Icône garantie** | `size-9`, `size="sm"` | `size-10`, `size="md"`, halo orange |
| **Libellé progression** | `text-xs text-slate-300` | `text-sm text-white/80` |
| **Pourcentage** | `font-semibold` | `font-bold` |
| **Badge** | `bg-orange-500/20` | `bg-orange-500/15` |
| **Messages d'aide** | `text-xs text-muted-foreground` | `text-white/60` |
| **Surcharge des champs (coquille)** | `bg-slate-800/80`, `shadow-inner` | `bg-white/5`, `border-white/10`, sans ombre interne |
| **Halo de page** | 2 halos | 3 halos (+1 discret en haut à droite) |

### 5.1 Cause racine du bouton

```css
/* button.tsx:22 */
'disabled:pointer-events-none disabled:opacity-50 select-none'
```

Le design system n'a **qu'un seul état désactivé** : `opacity-50` appliqué sur
`bg-primary`. Résultat sur fond sombre : orange à 50 % d'opacité, visuellement
proche d'une couleur d'accent — le visiteur ne pouvait pas distinguer
« pas encore valide » de « actif ».

Corrigé par un gris neutre explicite (`bg-muted text-muted-foreground`), avec
l'orange vif réservé au moment où l'action devient possible. Le `disabled` et le
`data-enabled` sont conservés : le gris n'est pas qu'un habillage d'un bouton
cliquable.

---

## 6. Tests

**6 tests** ajoutés dans `technician-auth.test.ts` — **552 / 552** au total.

| Test | Ce qui est verrouillé |
|---|---|
| bouton inactif ≠ actif | `canSubmit ?` + `bg-primary` + `bg-muted` + `disabled={!canSubmit}` |
| champs sur surface perceptible | `bg-white/5` + `border-white/10` + placeholder **≥ 6 surcharges** |
| catégorie sélectionnée distincte | voile orange **et** bordure orange |
| garanties lisibles | `bg-white/5` + `text-white/70` + icône accentuée |
| progression annoncée | libellé + dégradé + `aria-live="polite"` |
| aucun ton clair résiduel | surcharge `[&_input]`, `[&_select]`, `[&_.text-muted-foreground]` |

Le test des champs compte les surcharges (≥ 6) : une surcharge oubliée laisserait
un champ invisible **au milieu** du formulaire, sans qu'aucun autre test ne le
signale.

Validés **par mutation** :

| Mutation | Résultat |
|---|---|
| Carte de garantie-retour à `<li>` sans surface | **2 échecs** |
| Bouton inactif repassé en `bg-primary opacity-50` | **2 échecs** |

### Non-régression

| État | Résultat |
|---|---|
| `HEAD` propre (`git stash -u`) | **546 / 546**, 0 échec |
| Après chantier | **552 / 552**, 0 échec |

`technician-auth.test.ts` (24 tests) et le tunnel client passent : aucun
contrat de soumission n'a bougé.

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npx oxlint src/` | **0 erreur** |
| `npm run test:unit` | **552 / 552** |

---

## 7. Déploiement Vercel — vérifié, pas supposé

`GET /technicien/inscription` → **200**.

**7.1 — La feuille de style du chantier est bien servie**

La page référence `020d7146360edef2.css` et `b47836220ff4339b.css` (128 617 o).
La seconde contient les classes du chantier, compilées :

```css
.bg-white\/5{background-color:#ffffff0d}
.bg-white\/5{background-color:color-mix(in oklab,var(--color-white) 5%,transparent)}
.text-white\/70
```

`#ffffff0d` = blanc à **5,3 %** en repli, et la version `color-mix` pour les
navigateurs qui le supportent. L'ancienne feuille ne contient aucune de ces
classes → c'est bien le build `05757c3`.

**7.2 — Le HTML servi est un squelette**

La page rend le formulaire **côté client** : le HTML initial (21 412 o) ne
contient pas encore les classes du formulaire. C'est attendu — les contrôles se
font donc sur la feuille de style, seule à contenir les classes compilées.

---

## 8. Bugs et erreurs rencontrés

### 8.1 Un commentaire JSX multiligne a cassé la compilation

En réécrivant le commentaire de la surcharge des champs, une ligne a commencé
par `*`. En JSX, `*/` termine le commentaire sur **n'importe quelle ligne** :
le `tsc` a rejeté le fichier (`TS1005`). Corrigé en indendant la suite du
commentaire.

### 8.2 Un `border-white/10` sans `border`

La surcharge posait `[&_input]:border-white/10` sans `border` : la couleur seule
ne produit **aucune** bordure si l'épaisseur est nulle. Le design system pose
bien `border` par défaut, mais la surcharge `[&_label]…` du conteneur ne le
force pas. `border` explicite ajouté — sans lui, la bordure serait invisible
pour les champs du client, qui ne portent pas la surcharge du formulaire
technicien.

---

## 9. Points d'attention

### 9.1 Le rendu n'est pas vérifié — c'est le cœur de la demande

Ni `jsdom` ni bibliothèque de composants, et `next build` interdit (RÈGLE 3).
Les tests prouvent que les **classes sont présentes**, pas que le **contraste
ressenti** est bon. Or la demande porte exactement sur le ressenti.

À confirmer manuellement sur `/technicien/inscription` :

| Point | Risque |
|---|---|
| Bouton inactif | le gris se distingue-t-il assez de l'orange actif ? |
| Champs à blanc 5 % | perceptibles sur le téléphone, en plein soleil ? |
| Bordures à blanc 10 % | visibles, ou trop discrètes ? |
| Description des garanties | `white/70` sur `white/5` — contraste suffisant ? |
| Placeholder à blanc 40 % | lisible sans dominer ? |

### 9.2 Le tunnel client a changé d'apparence

`/client/connexion` et `/client/inscription` partagent la coquille : garanties,
progression et champs reçoivent les mêmes corrections. C'est cohérent (le fond
est nuit des deux côtés), mais **c'est un changement visible sur une page qui ne
figuraient pas dans la demande**.

### 9.3 Les catégories restent sur `bg-orange-500/15`

La consigne proposait `bg-primary/15`. Conservé à 20 % → **15 %** pour rester
sur la teinte du design system déjà employée par les garanties et le badge.
Écart mineur, sans effet de contraste : le voile est doublé d'une bordure.

### 9.4 La grille des catégories reste `sm:grid-cols-2`

Déjà en 2 colonnes sur desktop (C.2). Inchangé.

### 9.5 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` n'est pas présent, `.nvmrc` est
inopérant. Sur Node 20, `npm run test:unit` échoue **39/39** en
`ERR_UNKNOWN_FILE_EXTENSION` : fausse alerte, pas une régression. Chiffres
obtenus avec Node 22.20.0 installé **hors dépôt** (`/tmp/opencode/n22`).

### 9.6 `rewards-hero.png` (432 Ko) toujours sans usage — reporté.

---

## 10. Questions bloquantes

**Aucune.** La demande était entièrement tranchée.

Points ouverts, sans blocage :

- Confirmation navigateur des cinq points du § 9.1 ;
- Validation du changement d'apparence du tunnel client (§ 9.2).