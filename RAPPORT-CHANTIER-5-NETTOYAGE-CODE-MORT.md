# RAPPORT — Chantier 5 : nettoyage du code mort client

> Chantier **livré et déployé**. Rapport factuel : usage vérifié avant
> suppression, garde-fou posé pour qu'il ne revienne pas.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` — **backend non concerné** |
| Commit | `01291bc` — *chore(client): retire 110 lignes de composants jamais montes* |
| Push | ✅ `aae34c7..01291bc` |
| Vercel | ✅ `/`, `/client`, `/technicien` — **200** |
| Migrations / dépendances | **aucune** |

---

## 2. Synthèse

Deux composants occupaient le dossier du dashboard client **sans jamais avoir
été affichés** : le flux d'activité (84 lignes) et la carte de récompenses
(26 lignes).

Ils ne cassaient rien, ne ralentissaient rien. Ils étaient là, à grossir, et à
donner l'illusion d'une fonctionnalité existante.

**Bilan : −119 lignes / +11 lignes.**

---

## 3. Usage vérifié AVANT suppression

Le mot « mort » mérite une preuve, pas une impression.

| Composant | Imports réels | Verdict |
|---|---|---|
| `ActivityFeed` | **0** | mort |
| `RewardsCard` | **0** (1 mention dans un commentaire) | mort |

Le cas de `activity-feed.tsx` était le subtil : **ses types étaient vivants**.
`ActivityItem` et `ActivityTone` sont importés par le dashboard technicien
(`tech-overview.tsx:13` et `technicien/page.tsx:22-24`). Un fichier dont la
moitié des exports sert et l'autre non — d'où l'impossibilité de le supprimer
tel quel.

---

## 4. Ce qui a été fait

### 4.1 Déplacement, pas suppression

Les deux types sont **déplacés** vers `src/lib/activity-types.ts`, les deux
imports sont **repointés**. `src/lib/` plutôt que le dossier client, parce que
le flux d'activité **ne lui appartient pas** : les deux espaces connectés
décrivent la même information, avec la même forme.

Pas de `export * from` : deux chemins vers la même définition de type, c'est
deux réponses à « lequel est le bon ? ».

### 4.2 Un commentaire qui pointait vers du vide

`reward-progress-card.tsx` mentionnait les deux cartes qu'il avait absorbées.
L'une n'existe plus depuis ce chantier. Réécrit en langage métier plutôt que
de laisser un pointeur vers du vide — une mention historique n'a pas à rester
exacte, mais un renvoi cassé ment sur l'état du dépôt.

### 4.3 Un garde-fou, sinon il revient

**Un composant mort ne casse rien** : ni `tsc`, ni le lint, ni aucun test.
Rien ne le sanctionne, donc rien ne l'empêche de revenir.

`dead-code-guard.test.ts` verrouille trois choses :

1. les fichiers retirés ne reviennent pas ;
2. plus aucune source ne les référence ;
3. **aucun fichier du dossier `client/dashboard/` n'est importé ailleurs**.

Le troisième est le plus utile : il attrape **n'importe quel futur orphelin**,
pas seulement ceux déjà connus.

Volontairement limité à ce qui a été constaté. Un garde « aucun export sans
usage » exigerait de résoudre les imports dynamiques, les barrels et les points
d'entrée — et deviendrait flou.

---

## 5. Tests

**4 tests** ajoutés — **609 / 609**.

Non-régression : `HEAD` propre = **605 / 605**.

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | exit 0 |
| `npx oxlint src/` | **0 erreur** |
| `npm run test:unit` | **609 / 609** |

### Mutations

| Mutation | Résultat |
|---|---|
| Composant mort réintroduit dans le dossier | **1 échec** |
| Référence résiduelle vers un composant retiré | **1 échec** |

---

## 6. Un point de méthode

Le test « aucune source ne référence ces composants » a d'abord échoué **sur
lui-même** : il contenait forcément les noms qu'il surveillait. Deux corrections
ont été nécessaires — exclure le fichier de test de sa propre recherche, et
déplacer les motifs sensibles dans le test d'export.

Puis le même piège s'est reproduit une cinquième fois sur ce dépôt : un
commentaire dans `activity-types.ts` citait la syntaxe de ré-export que le test
interdisait. **Règle 3 — cinquième occurrence.**

Je le consigne parce que le motif est stable et que le traiter en cas isolé
manquerait le fond : sur un dépôt où les tests lisent le texte entier des
fichiers, **un commentaire est du code qui s'exécute**.

---

## 7. Points d'attention

### 7.1 Le garde-fou a une limite assumée

Il détecte un composant **mort**, pas un composant **mort à moitié** — un
fichier importé uniquement pour ses types, comme l'était `activity-feed.tsx`,
passe le contrôle n°3. C'est précisément le cas que ce chantier a traité, et il
a été trouvé par l'audit, pas par le test.

Un contrôle de ce type demanderait d'analyser quels imports sont des
`import type` et lesquels apportent une valeur — faisable, mais au prix d'un
analyseur de plus. À faire si le motif se répète.

### 7.2 Le rendu n'est pas vérifié

Même limite que les chantiers 2 à 4 : ni `jsdom` ni bibliothèque de
composants, `next build` interdit (RÈGLE 3).

Ce chantier est le **premier des cinq dont le risque de rendu est faible** :
aucun écran affiché ne change. Les pages répondent 200 et `tsc` garantit que
le code supprimé n'était appelé nulle part.

### 7.3 Node 22 est requis et absent de la machine

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
| 4 | Dashboard client — hiérarchie | ✅ `aae34c7` |
| 5 | **Nettoyage du code mort client** | ✅ `01291bc` |
| 6 | Pages restantes | page par page |

Les deux dashboards sont traités. Le socle — tokens, variantes sombres,
`AnimateOnScroll`, garde-fou de code mort — est réutilisable tel quel sur les
pages restantes.

## 9. Arbitrage suivant

Les deux dashboards sont traités. Restent les écrans de mission :

| Écran | Lignes |
|---|---|
| `technicien/demandes/[id]` | **1 482** |
| `client/demandes/[id]` | **1 259** |

Ce sont les deux plus gros fichiers non traités. Ils montrent **la même
mission des deux côtés**, et rien ne garantit qu'ils se ressemblent.

C'est le meilleur candidat pour la suite, et le plus risqué : les deux sont
verrouillés par plusieurs tests statiques (`dispute-status`,
`technician-quote`, `demande-equipment`, `diagnostic-libre`).

Un chantier de **convergence des deux écrans de mission** aurait plus de sens
que de traiter l'un puis l'autre : les traiter séparément produirait deux
refontes qui divergeraient à nouveau.
