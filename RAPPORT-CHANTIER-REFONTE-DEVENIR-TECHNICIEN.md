# RAPPORT — CHANTIER : refonte de `/devenir-technicien`

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt applicatif | `Repairdom-frontend` — commit `644f037`, poussé sur `main` |
| Dépôt | `projet` (racine) — ce rapport |
| Node | 22.20.0 (`/tmp/opencode/node-v22.20.0-linux-x64`) |
| Préalable | audit de la même page, supprimé du dépôt racine entre-temps (voir §Historique) |
| Vercel | **VERT** — vérifié par requête, voir §Déploiement |

## Synthèse

La page de recrutement est refondue en 10 sections. Les trois défauts de fond
relevés par l'audit sont corrigés : le parcours affiché est désormais le
parcours réel, le paiement est la section que le technicien candidat cherche en
premier, et le barème réel est affiché — calculé, pas recopié.

Aucune donnée n'est inventée. Les deux seuls chiffres de la page sont les
villes et les zones couvertes, lus sur l'endpoint public des villes et masqués
si l'appel échoue.

## Fichiers

| | Fichier | Lignes | Nature |
|---|---|---|---|
| M | `src/app/devenir-technicien/page.tsx` | 214 → **558** | réécrit, 10 sections |
| A | `src/app/devenir-technicien/recrutement-stats.tsx` | **97** | unique fragment client |
| A | `src/lib/devenir-technicien.test.ts` | **368** | 23 tests statiques |
| M | `package.json` | — | test ajouté à `test:unit` |
| M | `tsconfig.json` | — | test ajouté à `exclude` |

**Aucun composant partagé modifié. Aucune dépendance ajoutée.** Vérifié par
`git status` : les répertoires `components/ui`, `components/public`,
`components/landing` et `components/auth` sont intacts.

## Les trois corrections de fond

### 1. Le parcours affiché n'était pas le parcours réel

L'ancienne page listait **8 étapes linéaires**, dont le contrôle d'identité en
3ᵉ position. Le code réel (`ONBOARDING_STEP_DEFS`) définit **4 étapes**, dans un
autre ordre (profil → identité → zones → disponibilité), et **aucune n'est
bloquante** : l'ordre vient d'un hook d'affichage, pas d'une machine à états.

La page importe désormais `ONBOARDING_STEP_DEFS` et le mappe, description et
route comprises. Un changement du parcours d'onboarding se répercute ici sans
intervention. C'est la seule garantie qu'un test d'ordre statique ne donnerait
pas : il vérifierait la position des titres, pas leur origine.

### 2. Le paiement, absent de l'ancienne page

C'est la question n° 1 d'un technicien candidat, et l'information existait déjà
dans le dépôt (`auth-split.tsx:78`) sans être reprise.

Elle apparaît désormais à trois endroits : étape 4 de la section « Une
intervention, concrètement », argument n° 3 de la section « Pourquoi rejoindre »,
et réponse 3 de la FAQ. L'étape 4 est **isolée visuellement** — badge « La clé »,
fond `bg-primary/5`, `sm:col-span-2` — car un simple item dans une liste n'aurait
pas la force d'un argument.

### 3. Le barème, calculé et non recopié

L'ancienne page ne contenait aucun montant. Le barème réel existe dans le code :
`TECHNICIAN_FEE_LABEL = 'Commission Relio (500 FCFA + 4 %)'`
(`technician-quote.ts:46`).

Le tableau de la section 6 appelle `previewTechnicianQuote(quote)` et affiche
`preview.net`. Les 4 devis d'exemple sont des littéraux ; **aucun net ne l'est**.
Les 4 montants ont été vérifiés contre la formule `devis + 2 000 − commission` :

| Devis | Commission | Net |
|---|---|---|
| 5 000 | 700 | 6 300 |
| 10 000 | 900 | 11 100 |
| 15 000 | 1 100 | 15 900 |
| 25 000 | 1 500 | 25 500 |

Si le barème change côté serveur, le tableau suit et **c'est le test qui
échoue**, pas la page en silence.

## Migrations

**ABSENT.** Aucun changement de schéma, aucune API, aucune migration.

## Vérifications

| Contrôle | Avant | Après | Verdict |
|---|---|---|---|
| `tsc --noEmit` | ✅ | ✅ | pas de régression |
| `npm run lint` | 5 warnings, 0 erreur | 5 warnings, 0 erreur | **identique** |
| `node --test src/lib/devenir-technicien.test.ts` | — | **23/23** | nouveau fichier |
| `npm run test:unit` | 453/457 | **476/480** | **+23, 0 régression** |

Les 4 échecs restants sont les **mêmes, par nom** : `demande-draft-sync`,
`design-system` (logo), `verification-confirm` ×2. Aucun test existant modifié
ou supprimé.

Les 5 warnings de lint sont **préexistants et hors périmètre** : 2 dans
`src/app/client/parrainage/page.tsx`, 3 dans `src/app/admin/catalog/...`.
Vérifié par `npm run lint` : aucun warning ne pointe un fichier de ce chantier.

Aucun test n'a nécessité d'infrastructure : pas de base, pas de serveur, pas de
build, pas de rendu navigateur.

## Bugs trouvés

Aucun bug de produit. En revanche, **quatre de mes propres tests ont d'abord
échoué à tort** — ils testaient la faute de frappe du test, pas la règle. Tous
corrigés, et les corrections valent d'être consignées.

### 1. « information » contient « formation »

Le garde-fou « la page ne promet pas de formation » échouait sur la phrase
`l'information existait déjà dans le dépôt`, dans un commentaire. Comparaison
par sous-chaîne : c'est `includes('formation')`, pas le mot. Corrigé en
comparaison par **mot entier** (`/\bformation\b/i`), et le libellé n'a pas été
écarté.

### 2. L'ordre des sections mesurait les titres, pas le rendu

Les 10 sections sont dans le bon ordre dans le JSX. Le test échouait quand même :
« Une intervention, concrètement » et « Votre parcours pour rejoindre Relio »
sont **cités dans le commentaire d'en-tête**, en haut du fichier. `indexOf`
trouvait donc la section 4 avant la section 3.

Corrigé en ancrant la mesure sur `aria-labelledby` et la balise de montage, qui
sont **uniques** et reflètent l'ordre réellement rendu. C'est la correction la
plus instructive du lot : un test d'ordre par titre est un test qui mesure le
commentaire d'en-tête.

### 3. Le référentiel d'icônes en lisait 9 au lieu de 46

Le test extrayait les noms du dictionnaire `paths` (`pin: (`). Or `icon.tsx` a
deux structures : le **tableau `ICON_NAMES`** (46 entrées, entre apostrophes) qui
alimente le type `IconName`, et le dictionnaire des tracés, dont les clés ne sont
pas entre apostrophes. Résultat : 11 iconnes déclarées invalides, toutes en réalité
valides.

Le test avait raison de les contester ; c'est son référentiel qui était faux.
Corrigé sur `ICON_NAMES`, avec une garde `size >= 40` pour détecter une
dégradation future du parseur.

### 4. La `metadata` recopiait le barème

`description` contenait le littéral « 500 FCFA + 4 % ». Un point de vérité de
plus, exactement le défaut que la mission voulait éviter, et que le test « le
libellé vient de `TECHNICIAN_FEE_LABEL` » a attrapé. Interpolé depuis la même
constante que le tableau.

## Écarts explicites par rapport à la demande

| Demandé | Réalisé | Pourquoi |
|---|---|---|
| Hero : CTA secondaire « Déjà technicien ? Se connecter » | idem, **partagé avec le CTA final** | La structure cible demandait « Se connecter » seul en hero, mais la variante conditionnelle du tunnel appelle déjà cette formule. Deux libellés pour un même lien auraient rendu la factorisation demandée au point 6 incohérente. Une constante, deux usages. |
| FAQ : « Vérification d'identité sous 48 h ouvrées » | **Délai retiré** | Aucun délai de traitement n'existe dans le code. La mission demandait explicitement de ne pas inventer de chiffres ni de délais ; un test verrouille l'absence de `sous N h` et `sous N jours`. Inscription et profil sont annoncés (2 min / 15 min) parce que ce sont des faits d'usage, pas des engagements de traitement. |
| « Prérequis : inscription en 2 minutes » | conservé | Idem : durée de saisie, pas de traitement par un tiers. |
| ~400 lignes | **558** | 10 sections, dont un tableau doublé pour le mobile (version cartes) et 6 questions de FAQ. Le budget était indicatif (« c'est acceptable »). |

## Points d'attention

1. **La section des chiffres ne s'affiche qu'après le premier rendu.** C'est le
   prix du pattern `trust-stats` exigé : un Server Component ne peut pas
   `fetch` côté client. Conséquence concrète : **Test 4 du scénario production ne
   verrra PAS `GET /cities` dans le panneau Network au rechargement** — il faut
   l'onglet puis filtrer sur XHR. Ce n'est pas un défaut ; c'est le comportement
   de l'implémentation existante de la landing, reproduit à l'identique.

2. **La FAQ utilise `<details>` natif.** Aucune dépendance JS, comme demandé, mais
   le composant ne s'ouvre pas au clavier comme un `button` le ferait sur
   certains navigateurs. Le composant `landing/faq.tsx` utilise la même balise :
   c'est le standard du dépôt, pas une divergence.

3. **`aria-labelledby` au lieu d'`id` seul.** Plus long, mais c'est ce qui permet
   au test d'ordre de s'ancrer sur une chaîne unique dans le fichier.

4. **La page n'affiche toujours aucun avis.** Aucun n'existe dans le produit, et
   la landing a supprimé les siens pour cette raison exacte (`app/page.tsx:22`).
   Un test interdit explicitement d'en introduire.

5. **Aucune promesse n'est faite sur la qualité du réseau.** La page dit que les
   techniciens sont vérifiés — c'est un fait produit, le KYC est bloquant pour
   recevoir des missions (`onboarding-steps.ts:100`). Elle ne dit pas combien ils
   sont, ni combien gagnent : aucun point d'accès public ne l'expose.

6. **Les 4 devis d'exemple sont un choix éditorial**, pas une donnée. Un test
   vérifie qu'ils restent au-dessus du minimum de 5 000 FCFA autorisé.

## Scénarios de test production

Les 5 scénarios de la mission sont applicables tels quels. Deux précisions :

- **Test 4 (chiffres réels)** : voir point d'attention 1 sur la visibilité de
  l'appel dans Network.
- **Test 5 (non-régression)** : `technician-auth-split.tsx:43` pointe toujours
  vers `/devenir-technicien`, vérifié par le nouveau test **et** par
  `technician-auth.test.ts:192`, qui reste vert.

## Questions bloquantes

Aucune. Le travail est indexé, vérifié et **n'a pas été poussé**, conformément à
la consigne « NE PAS PUSH AVANT VALIDATION ».
