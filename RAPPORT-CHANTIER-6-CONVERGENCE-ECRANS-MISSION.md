# RAPPORT — Chantier 6 : convergence des écrans de mission

> Chantier **livré et déployé**. Rapport factuel : la divergence découverte
> est exposée, pas corrigée — et c'est délibéré.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt | `Repairdom-frontend` — **backend non concerné** |
| Commit | `4874951` — *refactor(mission): convergence du rendu du devis et de l'historique* |
| Push | ✅ `01291bc..4874951` |
| Vercel | ✅ **vert et vérifié** (§ 7) |
| Migrations / dépendances | **aucune** |
| Bilan | **−65 lignes net** (hors nouveaux composants et tests) |

---

## 2. Synthèse

Le récapitulatif chiffré d'un devis existait en **quatre copies** : deux côté
client, deux côté technicien. Chacune pouvait diverger sans qu'aucune réaction
n'atteigne qui que ce soit.

Le détail d'une mission est pourtant l'écran où une divergence fait le plus de
mal : c'est là que deux personnes regardent les mêmes chiffres.

---

## 3. Constat majeur — une divergence NON corrigée

### 3.1 Ce que le chantier a trouvé

Quand le devis est **ancien**, donc privé du champ `totalToDebit`, les deux
écrans ne retombent pas sur la même formule :

| Écran | Formule de repli |
|---|---|
| client | `amount + déplacement` |
| technicien | `(réparation ?? amount) + déplacement` |

### 3.2 Ce que cela produit, chiffré

Sur un devis `montant 100 000 / réparation 80 000 / déplacement 20 000` :

| Écran | Affiche |
|---|---|
| client | **120 000 FCFA** |
| technicien | **100 000 FCFA** |

**Écart de 20 000 FCFA entre le client et le technicien, sur le même devis.**
Le cas n'est pas théorique : le commentaire du code technicien lui-même indique
que le repli sert précisément aux devis émis avant l'introduction du champ.

### 3.3 Pourquoi ce n'est pas corrigé ici

Choisir la formule, c'est **décider quel montant est juste**. Le backend
n'expose pas `totalToDebit` sur ces devis : on ne sait donc pas, à la lecture
du frontend, lequel des deux montants est le bon. C'est une décision produit,
pas un calcul.

Corriger l'un des deux aurait fait changer un montant affiché sans que la
décision ait été prise.

### 3.4 Comment la divergence est rendue impossible à ignorer

Le composant partagé porte la **présentation**, jamais le **montant** : le
calcul reste à l'appel de chaque écran, là où il se produit. Un test échoue
si les deux formules se rapprochent — pour que la question soit reprise devant
un arbitrage explicite, plutôt que résolue par accident.

---

## 4. Convergence 1 — le récapitulatif (`QuoteLines`)

| | Avant | Après |
|---|---|---|
| Copies | **4** | **1** |
| Calcul du total | dans chaque copie | **à l'appelant** |
| Libellé du total | figé par écran | **prop** |
| Balise | `div/span` (technicien), `dl/dt/dd` (client) | **polymorphe** |

**Les libellés diffèrent légitimement** : « Total à payer » parle au client de
ce qu'il doit, « Total client (brut) » dit au technicien ce que son client va
débiter. Ce n'est pas le même mot, donc pas le même composant figé.

**Sémantique préservée** : le client rend une liste de définitions
(`dl`/`dt`/`dd`), le technicien une grille (`div`/`span`). Perdre le `dt`/`dd`
serait une régression d'accessibilité **invisible à l'œil** — d'où une
assertion dédiée.

---

## 5. Convergence 2 — l'historique injoignable (`TimelineUnavailable`)

Même écran des deux côtés, recopié, à une phrase près. Chacun disait en effet
que le reste restait consultable, mais le client « le reste de la page » et le
technicien « les autres informations de cet onglet » — parce que les deux
écrans n'ont pas la même forme.

La **description reste un paramètre** ; icône, titre et bouton sont identiques
et n'ont pas lieu de l'être deux fois.

Le doublon `formatPrice` — simple enveloppe de `formatFCFA`, défini dans les
deux fichiers — est supprimé. Aucun test ne le verrouillait.

---

## 6. Tests

**6 tests** ajoutés — **615 / 615**.

Non-régression : `HEAD` propre = **609 / 609**.

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | exit 0 |
| `npx oxlint src/` | **0 erreur** |
| `npm run test:unit` | **615 / 615** |

### Mutations

| Mutation | Résultat |
|---|---|
| Copie dupliquée réintroduite côté client | **1 échec** |
| Divergence de formule effacée côté technicien | **2 échecs** |
| Sémantique `dt`/`dd` perdue côté client | **1 échec** |

### Un test existant élargi

`technician-quote.test.ts` exigeait le titre « Impossible de charger
l'historique » en **littéral dans chaque page**. La factorisation l'a rendu
absent des pages — la vérification portait alors sur une chaîne déplacée, plus
sur le comportement.

L'assertion porte désormais sur l'intention : les deux écrans montent le
composant, et **le composant** porte toujours le titre et l'action de réessai.
Deux assertions ajoutées, dont une qui vérifie que le composant reste
paramétrable — un composant figé sur la description d'un seul écran serait une
régression de justesse.

Exiger la chaîne dans chaque page interdisait la factorisation, et donnait
l'impression de garantir un libellé qui n'y est plus.

---

## 7. Déploiement — vérifié, pas supposé

`/client` **200**, `/technicien` **200**, `/` **200**.

Chunk `app/client/demandes/[id]/page-c5079567e41b4bcf.js` (52 994 o). Les
chaînes y sont échappées en unicode par la minification ; sous cette forme :

| Marqueur | Occurrences |
|---|---|
| `Total à payer` | 2 |
| `labelTag:"dt"` | 2 |
| `valueTag:"dd"` | 2 |
| `totalToDebit` (avec formule de repli au point d'appel) | 1 |

Le composant partagé est monté des deux côtés, la sémantique `dt`/`dd` est
présente, et le calcul du total reste bien au point d'appel.

**Non-régression fonctionnelle** : aucun montant affiché n'a changé. Les
lignes déplacées produisent les mêmes valeurs qu'avant.

---

## 8. Ce qui a été volontairement écarté

### 8.1 `TravelSection` et `TravelBanner`

L'audit les présentait comme deux implémentations d'une même chose. Ils ne le
sont pas :

| | `TravelSection` (technicien) | `TravelBanner` (client) |
|---|---|---|
| Rôle | **agit** — GPS, rafraîchissement, actions métier | **affiche** |
| Lignes | 245 | 60 |

Les factoriser aurait produit une abstraction qui ne simplifie rien : un
composant affichant, un composant agissant, et une couche de traduction entre
les deux. Écarté.

### 8.2 La convergence structurelle des deux écrans

Elle est **verrouillée** par des tests qui encodent des décisions produit :

| Verrou | Fichier de test |
|---|---|
| Onglets du technicien : 4 libellés, onglet par défaut, non-resynchronisation | `technician-quote.test.ts` l.529-572 |
| Ordre strict des 8 sections du client | `technician-quote.test.ts` l.393-451 |
| `useMemo`/`useCallback` interdits côté technicien | `technician-quote.test.ts` l.259-260 |
| `openDispute` exigé côté client, interdit côté technicien | `dispute-status.test.ts` |

Ces verrous protègent des corrections passées. Les défier aurait produit deux
refontes qui divergent à nouveau. **Le chantier porte donc sur `components/mission/`,
là où aucun verrou n'existe.**

---

## 9. Points d'attention

### 9.1 La divergence de formule attend une décision

C'est le point ouvert de ce chantier. Deux questions :

1. Sur un devis sans `totalToDebit`, quel montant est le bon ?
2. Faut-il **migrer** ces devis anciens pour leur calculer le champ manquant
   côté backend ? Ce serait plus robuste que de laisser chaque écran se
   débrouiller — et cela supprimerait la divergence à la source.

Tant que la question n'est pas tranchée, le test échouera si quelqu'un
rapproche les deux formules par inadvertance.

### 9.2 Trois de mes tests ont d'abord été faux

Avant d'être justes, ils :

- comptaient le **mot** « Diagnostic » — qui apparaît 33 fois dans la page
  client pour des raisons sans rapport avec le récapitulatif ;
- construisaient une regex en remplaçant la balise ouvrante et **oubliant la
  fermeture** ;
- citaient dans un commentaire un libellé qu'un autre test surveillait
  (**sixième occurrence de la règle 3**).

C'est la proportion normale quand on écrit des tests statiques sur du texte,
et la raison pour laquelle chaque test doit être validé par mutation.

### 9.3 Le rendu n'est pas vérifié

Ni `jsdom` ni bibliothèque de composants, `next build` interdit (RÈGLE 3).
Les contrôles § 7 prouvent que le bon code est servi, pas que l'affichage est
correct. Les deux écrans exigent une session.

### 9.4 Node 22 est requis et absent de la machine

Seul Node 20.20.2 est installé, `nvm` absent, `.nvmrc` inopérant. Sur Node 20,
`npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` : fausse
alerte, pas une régression. Chiffres obtenus avec Node 22.20.0 installé **hors
dépôt** (`/tmp/opencode/n22`).

---

## 10. Suite

| # | Chantier | Statut |
|---|---|---|
| 1 | Socle de thème sombre | ✅ `e7f3407` |
| 2 | Dashboard technicien — socle | ✅ `cd12230` |
| 3 | Dashboard technicien — hiérarchie | ✅ `899ae87` |
| 4 | Dashboard client — hiérarchie | ✅ `aae34c7` |
| 5 | Nettoyage du code mort client | ✅ `01291bc` |
| 6 | **Écrans de mission — convergence** | ✅ `4874951` |

**Suite naturelle** : trancher la divergence de formule (§ 9.1). C'est la
seule chose que ce chantier laisse en suspens, et elle est visible par les
utilisateurs.