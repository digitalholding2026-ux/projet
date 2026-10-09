# RAPPORT — Chantier 3 : hiérarchie du dashboard technicien

> Chantier **livré et déployé**. Un seul changement de comportement, rien
> d'ajouté ni de retiré.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` — **backend non concerné** |
| Commit | `899ae87` — *refactor(technicien): l'intervention en cours passe en tete du dashboard* |
| Push | ✅ `cd12230..899ae87` |
| Vercel | ✅ **vert et vérifié** (§ 5) |
| Migrations / dépendances | **aucune** |

---

## 2. Le changement

**L'intervention en cours passe du 6ᵉ bloc au 1ᵉ.**

Elle était commentators « bandeau prioritaire » dans le code, et se trouvait
sixième : sous l'en-tête, deux bandeaux de paperwork, une grille de trois
cartes de chiffres et le statut de disponibilité. Sur un téléphone, **hors du
premier écran**. Un technicien dont l'intervention commence à 14 h devait faire
défiler pour la retrouver.

C'est le seul changement de comportement. Aucun bloc ajouté, aucun retiré,
aucun texte modifié, aucune donnée touchée.

### Nouvel ordre de lecture

Du plus temporel au plus différé :

| # | Bloc | Pourquoi à cette place |
|---|---|---|
| 1 | **Intervention en cours** | elle se joue à une heure donnée |
| 2 | Onboarding / dossier | conditionne le paiement |
| 3 | Bandeau KYC | urgence administrative |
| 4 | Statut + KPI | utile, jamais urgent |
| 5 | Radar, activité, liste, compte | consultation |

---

## 3. Deux détails qui comptent

**Les délais de la cascade ont été réalignés** sur le nouvel ordre :
`0 → 80 → 120 → 180 → 240 ms`. Sans cela, la composition aurait animé le
statut *avant* la mission : le technicien verrait son statut apparaître, puis
la mission en dessous — l'impression que le compte change d'avis.

**Le bandeau gagne une cible tactile de 48 px.** C'est un `<Link>` : il
occupait 40 px, sous le seuil Android.

---

## 4. Un test dont le nom devenait faux

`onboarding-components.test.ts` affirmait : « la checklist est placée après
l'en-tête ». Après le déplacement, elle est après l'en-tête **et** après la
mission. L'affirmation n'était plus vraie.

Plutôt que de déplacer le code pour satisfaire une phrase, le test exprime
désormais **l'invariant réel** — l'ordre de priorité — et conserve
l'assertion d'origine sur le bandeau KYC, inchangée depuis le chantier 5A.

```
mission < checklist        (nouveau)
checklist < bandeau KYC     (invariant 5A, conservé)
```

Un test qui exige une position absolue bloquerait cette amélioration, ou pire
donnerait **l'impression de la garantir alors qu'elle n'est plus vraie** — le
même défaut que les tests de libellé croisés, rencontré sur un chantier
précédent.

---

## 5. Déploiement — ce qui est vérifié, ce qui ne l'est pas

`/technicien` → **200**. Nouveau hash de chunk :
`page-9342ea0d71c3a926.js` (43 096 o), contre `page-7c0332ab…` avant.

**Vérifié dans le build servi** :

```js
eo ? jsx(p.AnimateOnScroll, {delay: 0,
      children: jsx(i(), {href: "/technicien/demandes/" + eo.id,
        className: "block min-h-13 rounded-2xl border border-orange-500/30 …"
```

`delay: 0` est bien porté par le bandeau de mission, `min-h-13` est bien sur
le lien. `min-h-13` : 1 · `min-h-12` : 6 · `tabular-nums` : 1 · `bg-relio-bg` : 1.

### Ce que je n'ai PAS pu vérifier

**L'ordre de rendu, lui.** Dans un bundle minifié, les composants sont émis
dans l'ordre de leurs *dépendances*, pas de leur *position dans l'arbre*. Les
positions relevées dans le fichier (`Reprendre` à 34 020, `Revenus du jour` à
35 360) sont cohérentes avec le nouvel ordre, mais **elles ne le prouvent
pas** : ce serait un raisonnement circulaire.

L'ordre réel ne peut être établi que dans le navigateur. C'est le point qui
reste à confirmer.

---

## 6. Tests

**595 / 595** — 2 tests ajoutés, non-régression prouvée.

| Vérification | Résultat |
|---|---|
| `npx tsc --noEmit` | exit 0 |
| `npm run test:unit` | **595 / 595** |
| `npm run lint` | **0 erreur** |
| `HEAD` propre | **594 / 594** |

### Mutations

| Mutation | Résultat |
|---|---|
| Mission redescendue sous les KPI | **2 échecs** |
| Cascade désordonnée | **1 échec** |
| Cible tactile du bandeau retirée | **1 échec** |

---

## 7. Points d'attention

### 7.1 Le dashboard exige une session

`RoleGuard` protège `/technicien`. Toutes les vérifications de ce chantier
portent sur le code compilé, **jamais sur un rendu réel** — ni moi ni aucun
test ne l'ont vu. Le comportement observable demande une validation en
navigateur avec un compte technicien.

### 7.2 La grille KPI reste sous le paperwork

C'est deliberé : les chiffres n'ont pas d'échéance. Mais un technicien qui
cherche son revenu du jour doit maintenant faire défiler. Le KPI pourrait
devenir un encart compact dans l'en-tête si la lecture du revenu s'avère
fréquente — à mesurer, pas à deviner.

### 7.3 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` absent, `.nvmrc` inopérant. Sur Node 20,
`npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` : fausse
alerte, pas une régression. Chiffres obtenus avec Node 22.20.0 installé **hors
dépôt** (`/tmp/opencode/n22`).

---

## 8. Suite

| # | Chantier | Statut |
|---|---|---|
| 1 | Socle de thème sombre | ✅ livré (`e7f3407`) |
| 2 | Dashboard technicien — socle visuel | ✅ livré (`cd12230`) |
| 3 | Dashboard technicien — hiérarchie | ✅ livré (`899ae87`) |
| 4 | **Dashboard client** | socle prêt |
| 5 | Pages restantes | page par page |

Le chantier 4 est le suivant. Il réutilise `relio-theme`, les variantes
sombres et `AnimateOnScroll` — aucune dette de thème ne reste à combler.