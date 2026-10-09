# RAPPORT — AUDIT : Menu mobile cassé + contenu obsolète landing

**Mode de la phase d'audit : lecture seule stricte.** Aucun fichier modifié,
aucun commit, aucun push pendant la phase d'analyse. Ce rapport et son push
sont un ajout documentaire, demandé explicitement.

Vérifié sur `frontend@d1dd9a0`, commit « chore(tech): Node 22 explicite et
alias @/ resolus par le runner de test ».

---

## 1. DIAGNOSTIC DU MENU MOBILE

### a) Fichier et composant

| Élément | Valeur |
|---|---|
| Chemin | `frontend/src/components/public/public-header.tsx` |
| Composant | `PublicHeader` (l.14–188), fichier de 188 lignes |
| Lignes du menu mobile | **104–132** (CTA + bouton burger) · **135–185** (drawer) |
| State d'ouverture | l.17 `const [isOpen, setIsOpen] = useState(false)` |
| `useEffect` menu | l.26–39 |

### b) Structure du menu

Monté **inline dans le JSX du `<header>`**, pas via portail, pas de portal
React, pas de composant séparé.

- **Conditionnel** : `{isOpen ? ( … ) : null}` (l.136 et l.185).
- **Classes du drawer** (l.137) :
  `fixed inset-0 z-50 bg-relio-bg/95 backdrop-blur-md lg:hidden`
  - `fixed inset-0` → couvre le viewport
  - `z-50`
  - `bg-relio-bg/95` → **opaque à 95 %**
  - `backdrop-blur-md`
  - `lg:hidden` → masqué à partir de 1024 px
- **Backdrop séparé ?** **Non.** Le fond est porté par la même `div` que les
  liens. Il n'y a pas de couche d'opacité séparée du contenu.
- Contenu (l.138–183) : `flex h-full flex-col`, logo + bouton X, `nav`
  (3 liens), puis CTA en `mt-auto`.

`bg-relio-bg` est bien un token défini : `--color-relio-bg: #0b0d12`
(`src/app/globals.css:64`), c'est-à-dire un **noir**. À 95 %, le fond est
pratiquement opaque.

### c) État d'ouverture

- **State** : `isOpen` (l.17), basculé par `onClick={() => setIsOpen(!isOpen)}`
  sur le burger (l.123).
- **Fermeture** : `closeMenu()` (l.41), câblé sur les 6 liens du drawer
  (l.151, 154, 157, 163, 170, 175) et sur le bouton X (l.143).
- **`useEffect` de scroll body : OUI** (l.27–39). Il mémorise
  `previousOverflow`, pose `document.body.style.overflow = 'hidden'`, et
  **restaure la valeur précédente** au nettoyage (l.37).
- Le même `useEffect` installe un listener `keydown` qui ferme sur `Escape`
  (l.31–33), également nettoyé.

### d) Cause du chevauchement

**Aucun des trois mécanismes évoqués dans la demande n'est en cause dans le
code lu.** Le drawer est `fixed inset-0`, `z-50`, avec un fond opaque à 95 % —
il *devrait* masquer le hero.

Ce qui explique le symptôme observé est un **autre** point, à **vérifier en
navigateur** :

Le `<header>` parent (l.44–51) porte la classe :

```
'sticky top-0 z-20 border-b backdrop-blur transition-all duration-300 safe-top'
```

Trois propriétés sont en présence, et deux d'entre elles sont connues pour
créer un **contexte de pile (stacking context)** et un **bloc conteneur pour
les descendants `position: fixed`** :

| Propriété | Effet |
|---|---|
| `backdrop-blur` | **`backdrop-filter` crée un bloc conteneur pour `position: fixed`** |
| `sticky` + `z-20` | crée un contexte de pile |
| `transition-all` (durée 300 ms) | crée un contexte de pile sur les navigateurs qui l'appliquent |

**Conséquence** : le `fixed inset-0` du drawer est résolu **par rapport au
`<header>`**, pas par rapport au viewport. Or le header est `sticky` d'une
hauteur de `h-14` / `h-12` (l.55, 48 ou 56 px). Un `inset-0` résolu contre un
parent de ~56 px de hauteur produit un drawer **d'environ 56 px de haut au
lieu de la hauteur de l'écran** — avec `h-full` (l.138) et `mt-auto` (l.161)
calculés sur cette base.

Le fond sombre ne couvrirait alors que la bande du header, tandis que les
 liens se flowed en dessous, par-dessus le hero, sur un fond transparent —
**ce qui correspond exactement aux symptômes décrits** : contenu du hero
visible derrière, items superposés, aucun backdrop.

> **À VÉRIFIER** : cette cause est déduite des propriétés CSS, pas observée
> dans un navigateur. La mission interdit toute vérification à
> l'infrastructure ; je n'ai pas de navigateur headless dans ce dépôt. La
> confirmation requiert l'inspecteur sur `www.relioo.space` en viewport mobile
> : relever la hauteur réelle de la `div` l.137.
>
> Le même effet s'applique à `transform`, `filter`, `perspective` et
> `will-change` — **aucun n'est présent** sur le header.

Point secondaire factuel : le `<header>` a `z-20`, le drawer `z-50`. Le
`z-50` est **local au contexte de pile du header** et ne s'élève donc pas
au-dessus d'un élément `z-30` placé ailleurs dans la page — dans le hero
actuel, aucun élément ne dépasse `z-10` (l.30).

### e) Accessibilité

| Point | État | Ligne |
|---|---|---|
| `role="dialog"` | ✅ présent | 137 |
| `aria-modal="true"` | ✅ présent | 137 |
| `aria-label` | ✅ « Menu de navigation » | 137 |
| Fermeture Escape | ✅ | 32 |
| Verrouillage du scroll | ✅ avec restauration | 29–38 |
| `aria-expanded` sur le burger | ✅ | 124 |
| `aria-label` dynamique du burger | ✅ (« Ouvrir » / « Fermer ») | 125 |
| **Focus piégé dans le menu** | ❌ **absent** — aucun `focus-trap`, aucune tabulation cyclique | — |
| **Focus déplacé à l'ouverture** | ❌ **absent** — aucun `ref` + `focus()` sur l'ouverture | — |
| **Focus restitué à la fermeture** | ❌ **absent** | — |
| `role="navigation"` / `nav` semantique | ✅ `<nav aria-label="Navigation mobile">` | 150 |
| Retour au focus / inert sur le fond | ❌ absent | — |

---

## 2. DIAGNOSTIC DU CONTENU OBSOLÈTE

### a) Fichier et ligne

**Le contenu obsolète est présent dans DEUX fichiers, pas un seul.**

| # | Fichier | Ligne | Texte exact |
|---|---|---|---|
| 1 | `frontend/src/components/landing/landing-sections.tsx` | **231** | `intro="Chaque dépannage confirmé compte. Le 5ᵉ vous offre votre prochaine intervention."` |
| 2 | `frontend/src/components/landing/faq.tsx` | **40** | `'Vos interventions confirmées sont comptabilisées. Dès le 5ᵉ dépannage, votre prochaine intervention vous est offerte, et des paliers plus généreux vous attendent jusqu'au smartphone.'` |

Note de méthode : la chaîne exacte de la demande (`"Le 5e vous offre…"`) est
**introuvable** telle quelle — le dépôt utilise le caractère **superscript `ᵉ`**
(U+1D49), pas `e`. Une recherche par « 5e intervention » ou « 5 missions » ne
renvoie **rien** ; la recherche par `5 dépannages` et par le caractère `ᵉ` est
celle qui aboutit.

### b) Autres incohérences

Le contenu obsolète ne se limite pas à la phrase d'intro. Toute la section est
construite sur l'ancien modèle.

**`landing-sections.tsx`, bloc `REWARDS_HIGHLIGHTS` (l.207–223)** — données
déclarées :

| Ligne | Champ `tier` | Champ `title` |
|---|---|---|
| 209 | `'5 dépannages'` | `'Dépannage 100 % offert + pack collector Relio'` |
| 214 | `'25 dépannages'` | `'Écran TV Smart LED'` |
| 219 | `'50 dépannages'` | `'Smartphone moderne'` |

**Écarts relevés :**

| # | Écart | Emplacement |
|---|---|---|
| 1 | Intro « Chaque dépannage confirmé **compte** » — le nouveau modèle est basé sur la **marge**, pas sur le décompte | `landing-sections.tsx:231` |
| 2 | « Le 5ᵉ vous offre votre prochaine intervention » | `landing-sections.tsx:231` |
| 3 | FAQ : « Dès le **5ᵉ dépannage** » | `faq.tsx:40` |
| 4 | FAQ : « vos interventions confirmées sont **comptabilisées** » | `faq.tsx:40` |
| 5 | Palier `'5 dépannages'` | `landing-sections.tsx:209` |
| 6 | Palier `'25 dépannages'` | `landing-sections.tsx:214` |
| 7 | Palier `'50 dépannages'` | `landing-sections.tsx:219` |
| 8 | Titre du palier 5 : « Dépannage 100 % offert » — récompense d'un **nombre** de missions | `landing-sections.tsx:210` |
| 9 | **Le commentaire de code le déclare lui-même** : l.200–206 dit « Paliers repris de `REWARDS_CATALOG` … 5 → intervention offerte + pack collector, 12 → fer, 25 → TV, 50 → smartphone, 100 → grand prix » — un catalogue à 5 paliers **qui n'existe plus** et n'est importé nulle part | `landing-sections.tsx:200–206` |
| 10 | Le libellé du lien l.262 « Voir tous les paliers » mène à `/client/recompenses`, page désormais structurée sur la marge | `landing-sections.tsx:262` |

**Images associées** (l.211, 216, 221) : `/recompense/tshirt_cap.png`,
`/recompense/tv.png`, `/recompense/smartphone.png`. Le dossier
`public/recompense/` contient 6 fichiers : `free_repair.png`, `iron.png`,
`mystery_box.png`, `smartphone.png`, `tshirt_cap.png`, `tv.png`.

> **À VÉRIFIER** : aucune image ne porte d'alt obsolète — les `alt` reprennent
> `reward.title` (l.242). Le rapprochement visuel image ↔ seuil actuel n'est pas
> évaluable sans jugement éditorial, hors périmètre factuel.

### c) Cohérence avec le nouveau système

**Les vraies informations existent, mais uniquement ailleurs.**

**Source de vérité côté client** : `frontend/src/lib/rewards-view.ts` (mirror
de `backend/src/rewards/rewards.config.ts`), dont l'en-tête (l.1–14) documente
explicitement le chantier 4-FONDATIONS-C.

Données réelles présentes dans le code :

| Élément | Valeur | Emplacement |
|---|---|---|
| Modèle | marge cumulée, pas nombre de missions | `rewards-view.ts:1–14` |
| Badges | `FIDELE` 10 000 · `OR` 50 000 · `PLATINE` 100 000 | `rewards-view.test.ts:38–40` |
| Nature | `ELECTROMENAGER_PETIT` 50 000 · `ELECTROMENAGER_MOYEN` 100 000 · `SMARTPHONE` 250 000 | `rewards-view.test.ts:42–44` |
| Tranche de crédit | `trancheXAF: 10_000`, `creditPerTrancheXAF: 500` | `rewards-view.test.ts:57–58` |
| Ratio | 5 % de la marge | `src/app/client/recompenses/page.tsx:45` |

**Où c'est affiché :**
- ✅ `src/app/client/recompenses/page.tsx` — hero « marge cumulée + palier
  courant » (l.160–173), section « Vos paliers » (l.324–329). **Page cohérente
  avec le nouveau modèle.**
- ❌ **La landing n'affiche nulle part** les seuils de marge, le ratio 5 %, la
  tranche 10 000/500, ni les paliers nature.

**Constat factuel** : la landing est le seul point du dépôt grand public
décrivant le programme, et c'est le seul qui soit obsolète. Elle ne contient
aucune mention du modèle LTV.

---

## 3. CONTENU DE LA LANDING (vue d'ensemble)

Ordre imposé en `src/app/page.tsx:15–18` (commentaire « ordre des blocs
imposé par la DC ») et rendu l.29–46.

| # | Section | Composant | Fichier | Lignes | Contenu | Type |
|---|---|---|---|---|---|---|
| — | En-tête | `PublicHeader` | `components/public/public-header.tsx` | 1–188 | Logo, nav desktop 3 liens, CTA, burger mobile + drawer | **dynamique** (useAuth) |
| 1 | Hero | `Hero` | `components/landing/hero.tsx` | 1–91 | Titre, promesse, 2 CTA, lien suivi par référence, image `/hero/hero_main.webp` | **dynamique** (useAuth) |
| — | Fond du hero | `HeroBackground` | `components/landing/hero-background.tsx` | 1–20 | 2 halos animés (`animate-glow-sweep`, `animate-breathe`) | statique |
| 2 | Comment ça marche | `HowItWorks` | `components/landing/landing-sections.tsx` | 62–96 | 4 étapes : décrire → technicien → devis → paiement. Données `STEPS` l.39–60 | statique |
| 3 | Pourquoi Relio | `WhyRelio` | `landing-sections.tsx` | 124–155 | 4 garanties : KYC, devis, paiement, suivi. Données `WHY` l.101–122 | statique |
| 4 | Services | `ServicesGrid` | `landing-sections.tsx` | 174–198 | 6 familles emoji → `/demande`. Données `SERVICES` l.165–172 | statique |
| 5 | Preuves sociales | `TrustStats` | `components/landing/trust-stats.tsx` | 1–93 | Chiffres villes. **S'auto-masque si `GET /cities` échoue** | **dynamique** (API) |
| 6 | FAQ | `Faq` | `components/landing/faq.tsx` | 1–74 | Questions / réponses | statique |
| 7 | Récompenses | `Rewards` | `landing-sections.tsx` | 225–268 | 3 paliers + lien `/client/recompenses` | statique |
| 8 | Devenir technicien | `BecomeTechnician` | `landing-sections.tsx` | 275–301 | Bloc minimal, 1 CTA | statique |
| 9 | CTA final | `CompactCta` | `landing-sections.tsx` | 304–324 | Bandeau sombre + CTA `/demande` | statique |
| — | Pied de page | `PublicFooter` | `components/public/public-footer.tsx` | 1–65 | Liens, mentions | statique |

**Sections statiques vs dynamiques** : 2 dynamiques sur 11 —
`PublicHeader` (useAuth), `Hero` (useAuth), `TrustStats` (API `GET /cities`).

**Sections dont le contenu est obsolète** : **1 sur 11** — `Rewards`
(`landing-sections.tsx:225–268`), plus la FAQ (`faq.tsx:40`) qui la complète.

Commentaire structurant à noter (`page.tsx` l.20–22) : le bloc témoignages a été
**supprimé** pour « faux avis », et le badge « 4.9/5 » du hero a été supprimé
pour « littéral sans source de données » (`hero.tsx:12–15`). La landing
applique une règle de **zéro chiffre inventé** — ce qui rend l'écart de la
section Récompenses d'une autre nature : il ne s'agit pas d'un chiffre inventé,
mais d'un **vrai chiffre devenu faux**.

---

## 4. ANIMATIONS EXISTANTES

### a) Inventaire

**Lottie** — `lottie-react`, 12 fichiers l'utilisent :

| Fichier | Animation |
|---|---|
| `components/public/public-header.tsx` | `MenuNavAnimation` (l.9, l.130) — **seule animation de la landing** |
| `components/ui/lottie-animation.tsx` | composant wrapper |
| `components/lottie/lottie-animations.tsx` | bibliothèque (fichier source dans `animatio json/`) |
| `app/client/confirmation/page.tsx` | `DemandeEnvoyeeAnimation` |
| `app/client/solde/recharger/page.tsx` | animation de recharge |
| `app/suivi/page.tsx` | `RechercheTechnicienAnimation` |
| `app/not-found.tsx` | `Erreur404Animation` |
| `components/auth/loading-screen.tsx` | `ConnexionAnimation` |
| `components/auth/verification-panel.tsx` | animation de vérification |

**framer-motion** — **2 fichiers seulement** :
- `components/client/client-auth-form.tsx`
- `components/notifications/notifications-center.tsx`

**CSS `@keyframes`** — **16** dans `src/app/globals.css` : `shimmer` (273),
`fade-in` (279), `slide-up` (288), `gradient-pan` (299), `marquee-x` (311),
`float-y` (320), `glow-sweep` (333), `breathe` (343), `floatWave` (367),
`word-rise` (377), `pulse-dot` (390), `sheen` (514), `stripe-move` (523),
`connector-grow` (540), `pop-in` (549), `confetti-fall` (560).

### b) Où elles sont utilisées

| Zone | Animations présentes |
|---|---|
| **Landing** | **2 classes CSS uniquement** — `animate-glow-sweep` et `animate-breathe`, toutes deux dans `hero-background.tsx:10–11`. Plus `MenuNavAnimation` dans le header. |
| Dashboards client | `animate-*` dans `client-home-blocks.tsx`, `demande-wizard.tsx`, `parrainage-overview.tsx`, `voice-recorder.tsx`, `reward-catalog.tsx` ; framer-motion dans `client-auth-form.tsx` |
| Dashboards technicien | `technicien/page.tsx`, `technicien/layout.tsx` |
| Dashboards admin | `admin/layout.tsx` |
| Composants partagés | `chronologies/tracking-preview.tsx`, `finance/withdrawal-panel.tsx`, `mission/celebration.tsx`, `mission/dispatch-sonar-widget.tsx`, `mission/mission-card.tsx`, `mission/mission-stepper.tsx`, `mission/demande-media-section.tsx` |
| Layouts | `app/loading.tsx` |

**Constat factuel** : la landing est la zone la moins animée du dépôt. 7 des
16 `@keyframes` et 0 framer-motion n'y apparaissent pas ; une seule animation
CSS y est active (`HeroBackground`).

---

## 5. PISTES D'ANIMATIONS POSSIBLES

**Liste descriptive, sans jugement ni proposition.**

### Zones déjà mobiles sur la landing

- `HeroBackground` — `animate-glow-sweep` (l.10), `animate-breathe` (l.11) :
  **les 2 seules animations CSS de la landing**
- `MenuNavAnimation` (Lottie) — joue au toggle du burger
  (`public-header.tsx:130`), `playKey={isOpen ? 'open' : 'closed'}`
- Transitions CSS de survol sur les liens du drawer :
  `transition-colors hover:bg-white/10` (l.151, 154, 157)
- `transition hover:border-orange-300 hover:shadow-md active:scale-95` sur les
  6 tuiles de services (`landing-sections.tsx:187`)
- `active:scale-[0.99]` sur le CTA final (`landing-sections.tsx:316`) et le CTA
  hero (`hero.tsx:45`)

### Zones statiques de la landing

- `HowItWorks` — 4 `<li>` sans aucune classe d'animation
- `WhyRelio` — 4 `<div>` sans animation
- `TrustStats` — sans animation
- `Faq` — sans animation (accordeil / repli non animés : **à vérifier**, le
  fichier a été lu en intégralité et ne contient aucune classe `animate-*`)
- `Rewards` — 3 cartes sans animation
- `BecomeTechnician`, `CompactCta` — sans animation

### Animations defined mais non utilisées sur la landing

16 `@keyframes` déclarés dans `globals.css` ; **14 ne sont référencés par aucun
des 5 fichiers de `components/landing/`**. À VÉRIFIER pour les 2 restants
s'ils sont utilisés ailleurs dans `src/`.

### Mécanismes déjà éprouvés ailleurs dans le dépôt

- **framer-motion** : 2 fichiers seulement (`client-auth-form.tsx`,
  `notifications-center.tsx`)
- **Lottie** : 9 emplacements, tous hors des corps de section de landing
- **`ResponsiveView`** : composant existant
  (`components/ui/responsive-view.tsx`) qui rend **une seule** vue, jamais deux
  arbres DOM — pattern déjà en usage admin
- **Réduction de mouvement** : **à vérifier** — aucune occurrence de
  `prefers-reduced-motion` relevée dans les fichiers lus

---

## 6. FICHIERS CLÉS

| # | Fichier | Rôle dans la landing / le menu |
|---|---|---|
| 1 | `frontend/src/components/public/public-header.tsx` | Header + **drawer mobile** (l.104–185). Cœur du problème 1. |
| 2 | `frontend/src/app/page.tsx` | **Ordre des 9 sections**, commentaire DC (l.15–22). Point d'entrée de la landing. |
| 3 | `frontend/src/components/landing/landing-sections.tsx` | **Section Récompenses obsolète** (l.200–268) + 4 autres sections. |
| 4 | `frontend/src/components/landing/faq.tsx` | **Second emplacement obsolète** (l.40). |
| 5 | `frontend/src/lib/rewards-view.ts` | **Source de vérité** du modèle LTV côté client. |
| 6 | `frontend/src/app/client/recompenses/page.tsx` | Rendu **cohérent** du nouveau modèle, à titre de comparaison. |
| 7 | `frontend/src/components/landing/hero.tsx` | Hero, 1re section, contient les 2 CTA. |
| 8 | `frontend/src/components/landing/hero-background.tsx` | Seules animations CSS de la landing. |
| 9 | `frontend/src/app/globals.css` | 16 `@keyframes`, tokens dont `--color-relio-bg` (l.64). |
| 10 | `frontend/src/lib/navigation.test.ts` | Test statique liant les ancres du header aux `id` de la landing. |

---

## 7. TESTS STATIQUES

### Tests lisant `public-header.tsx`

| Test | Ligne | Assertions |
|---|---|---|
| `src/lib/lottie.test.ts` | 129–132 | `assert.match(header, /MenuNavAnimation/)` · `assert.match(header, /aria-expanded/)` |
| `src/lib/navigation.test.ts` | 92–110 | Extrait tous les `href="/#…"` du header, exige que chacun existe comme `id="…"` dans `landing-sections.tsx` + `faq.tsx`. Exclut explicitement `/categories-title`. |
| `src/lib/demande-draft-routing.test.ts` | 119 | Le header doit contenir `/demande` |

### Tests lisant `landing-sections.tsx`

| Test | Ligne | Assertions |
|---|---|---|
| `src/lib/navigation.test.ts` | 92–110 | Ancres ci-dessus (`id="how-title"`, `id="services-title"` doivent rester) |
| `src/lib/demande-draft-routing.test.ts` | 117 | Doit contenir `/demande` |

### Tests lisant `faq.tsx`

`src/lib/navigation.test.ts:94` — le fichier est concaténé avec
`landing-sections.tsx` pour la vérification des ancres.

### Tests lisant `rewards-view.ts`

`src/lib/rewards-view.test.ts` — **36 tests** sur la logique pure. Ne lit aucun
`.tsx` : il importe `rewards-view.ts`. **Aucun test ne compare le contenu de la
landing au modèle réel.**

### Assertions susceptibles de casser lors d'une refonte

| # | Assertion | Fichier:ligne | Risque |
|---|---|---|---|
| 1 | `/MenuNavAnimation/` doit rester dans le header | `lottie.test.ts:130` | **élevé** — remplacer l'icône burger par une autre (ou un SVG inline) casse le test |
| 2 | `/aria-expanded/` doit rester dans le header | `lottie.test.ts:131` | **élevé** — supprimer l'attribut casse le test |
| 3 | Tout `href="/#x"` du header doit correspondre à un `id="x"` de la landing | `navigation.test.ts:95–107` | **élevé** — modifier un `id` de section ou un lien d'ancre casse le test |
| 4 | `/categories-title/` ne doit **pas** apparaître dans le header | `navigation.test.ts:108` | moyen |
| 5 | `landing-sections.tsx` doit contenir `/demande` | `demande-draft-routing.test.ts:117` | moyen |
| 6 | Le hero doit contenir `authenticated ? homePathForRole(user?.role) : '/demande'` (regex, l.139–142) | `demande-draft-routing.test.ts:138–142` | **élevé** — toute réécriture de l'expression ternaire casse le test |
| 7 | Le hero ne doit **pas** contenir `'/client/demande'` | `demande-draft-routing.test.ts:143` | moyen |

**Aucun test ne vérifie** : l'opacité du drawer, sa hauteur, la présence de
`role="dialog"`, le piégeage du focus, ni le contenu textuel de la section
Récompenses.

**Contrainte structurelle** (RÈGLE 3 du `AGENTS.md`) : ces tests sont
**statiques** — ils font `readFileSync` puis des `assert.match` sur le **texte
entier du fichier, commentaires et chaînes compris**. Un commentaire mentionnant
un mot surveillé peut donc casser une assertion. Les tests de
`rewards-view.test.ts` vérifient par ailleurs la **RÈGLE FCFA** : aucune
fonction ne doit produire de chaîne « 12 345 FCFA ».

---

## Points marqués « À VÉRIFIER »

1. **Hauteur réelle du drawer mobile** — la cause du chevauchement (§1d) est
   déduite de `backdrop-filter` créant un bloc conteneur pour `position: fixed`.
   **Non observée en navigateur** (aucun headless dans le dépôt, infrastructure
   interdite par la mission). À confirmer sur `www.relioo.space` via
   l'inspecteur.
2. **Images des paliers** — rapprochement image ↔ seuil actuel non évaluable
   factuellement (§ 2b).
3. **Répartition des 16 `@keyframes`** — l'affirmation « 14 non utilisés sur la
   landing » porte sur les 5 fichiers de `components/landing/` ; leur usage
   éventuel ailleurs dans `src/` n'a pas été inventorié (§ 5).
4. **`prefers-reduced-motion`** — aucune occurrence relevée dans les fichiers
   lus, mais la recherche n'a pas couvert l'intégralité de `src/` (§ 5).

---

**Fin du rapport. Aucun fichier de code modifié.**