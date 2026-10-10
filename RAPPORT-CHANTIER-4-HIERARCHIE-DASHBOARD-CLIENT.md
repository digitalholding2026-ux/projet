# RAPPORT — Chantier 4 : hiérarchie du dashboard client

> Chantier **livré et déployé**. Rapport factuel : chiffres réels, écarts
> notés, bugs rencontrés.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` — **backend non concerné** |
| Commit | `aae34c7` — *feat(client): l'intervention en cours passe en tete du dashboard* |
| Push | ✅ `899ae87..aae34c7` |
| Vercel | ✅ **vert et vérifié** (§ 6) |
| Migrations / dépendances | **aucune** |

---

## 2. Synthèse

Le client voyait ses interventions en cours dans une liste **triée par date**,
au même rang que les interventions terminées. Une demande déposée aujourd'hui
passait donc **devant** un dépannage commencé la veille : l'inverse de
l'urgence.

L'espace technicien traitait ce cas depuis le chantier 6B. Il n'avait jamais
été repris côté client. C'est le chantier 4.

---

## 3. Trois constats qui ont orienté le travail

### 3.1 La carte existait déjà, morte

`LiveMissionCard` n'était importé par **aucun fichier** du dépôt. Le
composant partagé qu'il enveloppe (`mission/mission-card.tsx`, 145 lignes) avait
été factorisé, le wrapper avait suivi — puis plus rien ne l'utilisait.

Il est **réactivé**, pas réécrit : un composant écrit pour exactement ce besoin
attendait d'être branché.

### 3.2 Le client n'avait aucune notion d'intervention en cours

Le technicien en a une depuis le chantier 6B. Côté client, `ACTIVE_STATUSES`
existait dans `client-dashboard.tsx` mais ne servait **qu'à un compteur de
badge** — jamais à la structure.

Elle est **dérivée** de `recent`, pas d'un appel supplémentaire : les données
sont déjà là, seule la lecture manquait. Pas de seconde source de vérité sur le
même écran.

### 3.3 Deux interventions en vol ne se départagent plus par date

La liste est triée par date de dépôt. Entre deux missions en vol, la plus
récente l'emportait — ce qui est **aléatoire**, pas un critère d'urgence.

Le départ se fait désormais par **progression réelle** :
`IN_PROGRESS` (3) > `SCHEDULED` (2) > `ACCEPTED` (1).

---

## 4. Ce qui change

### 4.1 Placement

| Vue | Avant | Après |
|---|---|---|
| Bureau | noyée dans « Dépannages récents » | **première carte**, avant la grille |
| Mobile | noyée dans « Dépannages récents » | **avant les actions rapides** |

Sur mobile, la mission passe avant les raccourcis : c'est une information avec
une échéance, pas un raccourci de navigation. Un premier jet l'avait placée
après, et le commentaire justifiait ce mauvais choix — **le code a été corrigé
pour que le commentaire dise vrai**.

La carte de solde reste au-dessus de tout : le solde est l'information la plus
consultée tous les jours, l'urgence une fois par intervention.

### 4.2 Convergence avec l'espace technicien

| Élément | Avant | Après |
|---|---|---|
| Noir mobile | `slate-950` (#020617) | `relio-bg` (#0b0d12) |
| Apparition | aucune | `AnimateOnScroll` |
| Délais | — | `0 / 80 / 160 ms` |

**La substitution du noir est invisible à l'œil** : les deux valeurs sont à
1,04:1 de contraste. Elle est faite pour une raison qui ne se voit pas —
`slate-950` n'appartient pas à l'échelle de marque. Deux valeurs pour le même
noir, c'est la promesse d'une divergence future.

### 4.3 Type de la carte

`LiveMissionCard` consomme désormais `RecentItem` — la forme dérivée par la
couche données — plutôt que la forme brute de l'API. Le composant n'en faisait
pas plus ; fabriquer un objet conforme à l'ancienne signature aurait été un
**mensonge de type**.

L'absence est normalisée à `null` plutôt que `undefined` : « absent » est une
absence de donnée, pas une absence de valeur.

---

## 5. Tests

**10 tests** ajoutés — **605 / 605** au total.

| Famille | Nb | Ce qui est verrouillé |
|---|---|---|
| Notion d'actif | 3 | statuts alignés sur le technicien · dérivation sans appel réseau · départ par progression |
| Placement | 3 | mission en tête sur les deux vues · carte **montée** et pas seulement importée · forme dérivée |
| Convergence | 2 | cascade démarrant à 0, délais croissants, sur les deux vues · noir de marque |
| Intégrité | 2 | vues séparées, grille bureau intacte, aucun fetch dans une vue · les 4 destinations préservées |

Non-régression : `HEAD` propre = **595 / 595**.

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | exit 0 |
| `npx oxlint src/` | **0 erreur** |
| `npm run test:unit` | **605 / 605** |

### Verrous existants respectés

`responsive.test.ts` impose plusieurs contraintes sur ces fichiers. Toutes
tenues, sans élargissement d'assertion :

- `xl:grid-cols-3` présent au bureau, **absent** du mobile
- `bg-slate-950` absent du bureau
- les 4 destinations présentes dans les **deux** vues
- aucun appel réseau dans une vue
- `bg-info-soft` et `bg-success-soft` préservés dans `client-home-blocks.tsx`
- `/client/demandes/historique` préservé dans `client-dashboard.tsx`

### Mutations

| Mutation | Résultat |
|---|---|
| Mission redescendue après les actions rapides | **2 échecs** |
| Cascade désordonnée | **1 échec** |
| Noir de marque revenu en dur | **1 échec** |

---

## 6. Déploiement — vérifié, pas supposé

`/client` **200**, `/technicien` **200**, `/` **200** (non-régression).

Le chunk de page ne fait que 210 o : le dashboard est rendu côté client. La
logique est dans le chunk partagé `1361-843388b1e2fa4a7f.js` (33 404 o).
Extrait du build servi :

```js
n.activeMission
  ? jsx(E.AnimateOnScroll, { delay: 0,
      children: jsx(A, { mission: n.activeMission }) })
  : null,
jsx("div", { className: "grid grid-cols-1 gap-6 xl:grid-cols-3", … })
```

La mission est **bien montée en premier**, avec `delay: 0`, avant la grille
bureau. `bg-relio-bg` : 3 occurrences. `IN_PROGRESS` : 4.

### Ce qui n'est pas vérifié

Le **rendu visuel**. `/client` exige une session : aucun écran n'a pu être
observé, par moi ni par aucun test.

---

## 7. Points d'attention

### 7.1 Le rendu n'est pas vérifié

Même limite que les chantiers 2, 3 et 4. Le dépôt n'a ni `jsdom` ni
bibliothèque de composants, et `next build` est interdit (RÈGLE 3).

À confirmer en navigateur : la carte d'intervention en cours, l'ordre sur
mobile, la fluidité de la cascade.

### 7.2 Une quatrième occurrence de la RÈGLE 3

Dans `live-mission-card.tsx`, j'ai cité dans un commentaire le nom exact du
type de l'API que le test surveillait pour empêcher son retour. **Le
commentaire a fait échouer son propre test.** Règle 3 — déjà rencontrée trois
fois sur ce dépôt ; c'est le piège récurrent de ce projet, pas une maladresse
isolée.

### 7.3 La substitution du noir est sans effet visible

Elle peut être annulée sans aucun risque visuel. Elle est conservée pour la
cohérence de la palette, pas pour une raison perceptible.

### 7.4 Du code mort subsiste dans le dossier

Signalé par l'audit, **hors périmètre** de ce chantier :

| Fichier | Lignes | État |
|---|---|---|
| `rewards-card.tsx` | 26 | **mort** — 0 référence |
| `activity-feed.tsx` | 83 | **composant mort** ; ses **types** sont utilisés par le dashboard technicien |

Le composant `activity-feed.tsx` ne peut pas être supprimé tel quel : le
technicien en importe les types `ActivityItem` / `ActivityTone`. Un chantier
de nettoyage déplacerait ces types hors du composant mort, puis le supprimerait.

### 7.5 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` absent, `.nvmrc` inopérant. Sur Node 20,
`npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` : fausse
alerte, pas une régression. Chiffres obtenus avec Node 22.20.0 installé **hors
dépôt** (`/tmp/opencode/n22`).

---

## 8. Suite

| # | Chantier | Statut |
|---|---|---|
| 1 | Socle de thème sombre | ✅ `e7f3407` |
| 2 | Dashboard technicien — socle | ✅ `cd12230` |
| 3 | Dashboard technicien — hiérarchie | ✅ `899ae87` |
| 4 | **Dashboard client — hiérarchie** | ✅ `aae34c7` |
| 5 | Nettoyage du code mort client | possible, faible risque |
| 6 | Pages restantes | page par page |

Le socle est en place et réutilisé tel quel sur les quatre chantiers. Les
pages restantes disposent maintenant des mêmes briques : tokens, variantes
sombres, `AnimateOnScroll`.