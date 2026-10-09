# RAPPORT — Remplacement des visuels du programme de récompenses

> Chantier livré et déployé. Rapport factuel : chiffres réels, échecs
> rencontrés distingués des régressions, écarts avec la demande notés.

---

## 1. Statut

| Élément | Valeur |
|---|---|
| Date | 2026-10-09 |
| Dépôt concerné | `Repairdom-frontend` (`frontend/`) — **backend non concerné** |
| Commit | `55771f0` — *fix(landing): visuels du programme alignes sur la marge + suppression du code mort* |
| Push | ✅ `ddaf86d..55771f0`, branche `main` |
| Vercel | ✅ **vert et vérifié factuellement** (§ 7) |
| Migrations | **aucune** — ni Prisma, ni base de données |
| Dépendances ajoutées | **aucune** |

Décisions produit validées et appliquées : voie A (suppression du composant
mort), `rewards-hero.png` conservé sans référence, images recompressées par le
demandeur.

---

## 2. Synthèse

Les 3 visuels de la section Récompenses ont été remplacés par des images
représentant réellement la récompense annoncée — les anciennes portaient
l'ancien programme « nombre de missions » **gravé dans l'image**, ce qu'aucun
texte de la landing ne pouvait rattraper.

Le composant `reward-catalog.tsx` (240 lignes) a été supprimé : il n'était
**monté nulle part** et son contenu était intégralement sur le modèle aboli.
Sa disparition ne change aucun écran rendu. Les 5 PNG orphelins (~9,5 Mo)
ont suivi.

`public/recompense/` passe de **~12,6 Mo à 1,9 Mo**.

---

## 3. Fichiers

### Modifiés (4)

| Fichier | Nature |
|---|---|
| `src/components/landing/landing-sections.tsx` | 3 références d'images + commentaire de cadrage l.212-215 |
| `src/lib/landing-rewards-content.test.ts` | +71 l. — 4 tests ajoutés |
| `src/lib/demande-draft-routing.test.ts` | entrée retirée + libellé de test corrigé |
| `src/lib/design-system.test.ts` | sortie de `HEX_ALLOWLIST` |

### Supprimés (6)

| Fichier | Taille |
|---|---|
| `src/components/client/recompenses/reward-catalog.tsx` | 240 lignes |
| `public/recompense/free_repair.png` | 1 914 796 o |
| `public/recompense/iron.png` | 1 932 242 o |
| `public/recompense/mystery_box.png` | 1 910 350 o |
| `public/recompense/tshirt_cap.png` | 1 874 017 o |
| `public/recompense/tv.png` | 1 896 232 o |

**Total libéré : ~9,5 Mo** (9 527 612 o exactement).

### Détenus dans `public/recompense/` (4)

| Fichier | Poids | Référencé par |
|---|---|---|
| `petit-electromenager.png` | 485 140 o | `landing-sections.tsx:221` |
| `electromenager-moyen.png` | 506 982 o | `landing-sections.tsx:226` |
| `smartphone.png` | 465 629 o | `landing-sections.tsx:231` |
| `rewards-hero.png` | 432 079 o | **rien** (décision produit) |

---

## 4. Contenu remplacé

### 4.1 Landing — `landing-sections.tsx`

| Ligne | Avant | Après |
|---|---|---|
| 221 | `/recompense/tshirt_cap.png` | `/recompense/petit-electromenager.png` |
| 226 | `/recompense/tv.png` | `/recompense/electromenager-moyen.png` |
| 231 | `/recompense/smartphone.png` | inchangé (fichier remplacé en amont) |

Le décalage **image ↔ récompense** est désormais nul : le t-shirt illustrait un
« Petit électroménager », la TV un « Électroménager moyen ».

Les `alt` s'appuient sur `reward.title` — cohérents automatiquement, aucune
retouche. Le commentaire l.212-215, qui documentait ce décalage comme « point
connu, non traité dans ce chantier », a été réécrit.

### 4.2 Preuve de suppression des références

```
grep -rnE "free_repair|iron\.|mystery_box|tshirt_cap|tv\.png" src/  → exit 1 (0 résultat)
grep -rn  "reward-catalog|RewardCatalog"               src/  → exit 1 (0 résultat)
```

Vérifié **avant** suppression des PNG, comme l'exige la mission.

---

## 5. Bugs trouvés

### 5.1 Un composant mort depuis longtemps, invisible pour les tests

`reward-catalog.tsx` n'était importé par **aucun fichier** de `src/`. Il ne
subsistait que parce que deux tests statiques le lisaient :

- `design-system.test.ts:47` — seule raison de sa présence : il contredit la
  règle « aucun hex arbitraire », il était donc dans `HEX_ALLOWLIST`.
- `demande-draft-routing.test.ts:124` — vérifiait la présence de `/demande`.

**Son contenu était entièrement sur l'ancien modèle** : `PALIER 5/12/25/50/100
DÉPANNAGES`, et un champ `requiredServices` qui pilotait un badge « Verrouillé
(3/12) » fondé sur un nombre de missions. Le programme réel est basé sur la
marge cumulée (`NATURE_THRESHOLDS` = 50 000 / 100 000 / 250 000 FCFA,
`backend/src/rewards/rewards.config.ts:113-116`). Le champ n'a plus de sens
métier.

**Pourquoi il avait survécu au chantier précédent** (`bf3fd5b`) : celui-ci avait
traité `landing-sections.tsx`, `faq.tsx` et `public-header.tsx`.
`reward-catalog.tsx` est **hors du périmètre de la page** — la page
`/client/recompenses` tire ses paliers de `rewards-view.ts` piloté par l'API.
L'écart est resté invisible, et aucun test ne pouvait le signaler : les deux
tests le lisant ne portaient que sur une couleur et un lien.

**Impact de la suppression : aucun.** Aucun écran rendu ne change.

### 5.2 Bug dans mon propre test, détecté à l'exécution

La première version du test « les visuels sont bien présents sur disque »
résolvait `../public/recompense/…` depuis `src/lib/`, soit `src/public/` — un
répertoire inexistant. Le test échouait à juste titre :

```
visuel petit-electromenager.png référencé mais absent de public/recompense/
```

**Ni `tsc` ni le lint ne l'avaient vu** — l'erreur ne se manifeste qu'à
l'exécution. Corrigé en `../../public/recompense/…`. Signalé ici parce que
c'est exactement le type de défaut qu'un `tsc` vert ne prouve pas.

### 5.3 Libellé de test devenu faux

`demande-draft-routing.test.ts` s'appelait « D2 : les 16 appels à l'ancien
chemin ont bien été migrés ». La liste ne contenait que 11 entrées, et elle en
compte 10 après suppression — le « 16 » ne correspondait à rien. Le test et son
commentaire ont été réécrits en langage qualitatif (« tous les appels »), le
commentaire expliquant que la liste est un instantané de l'audit, pas un
compte à maintenir.

---

## 6. Vérifications

Toutes exécutées sur **Node 22.20.0** (voir § 8, point 3).

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit` | **exit 0** |
| `npm run lint` (`oxlint`) | **0 erreur**, 2 warnings préexistants hors périmètre |
| `npm run test:unit` | **515 / 515 verts** |

### Preuve de non-régression

| État | Résultat |
|---|---|
| `HEAD` propre (`git stash -u`) | **511 / 511**, 0 échec |
| Après chantier | **515 / 515**, 0 échec |

Les 515 incluent les 511 d'origine : **aucun test perdu, aucun échec nouveau**.

Comparaison par les **noms** des tests, pas seulement par le compte : les deux
fichiers modifiés (`demande-draft-routing.test.ts`,
`design-system.test.ts`) passent tous leurs assertions restantes.

### Les 2 warnings de lint sont préexistants

`src/app/client/parrainage/page.tsx:15` et `:19` — imports
`REFERRAL_MAX_REFERRALS` et `isReferralExpired` inutilisés. Vérifiés présents
sur `HEAD` propre, sans rapport avec ce chantier.

### Les nouveaux tests ne sont pas verts à vide

4 tests ajoutés, validés **par mutation** :

| Mutation | Résultat |
|---|---|
| Réinjecter `tshirt_cap.png` dans la landing | **2 échecs** |
| Retirer `smartphone.png` du disque | **1 échec** (« absent de public/recompense/ ») |

Les tests échouent bien quand la régression est réintroduite.

### Conformité RÈGLE 3

Les tests de ce dépôt lisent le **texte entier** des fichiers, commentaires
compris. Les 4 nouveaux tests construisent donc leurs noms de fichiers
**par morceaux** (`'petit' + '-electromenager.png'`), pour ne pas se
auto-déclencher via leurs propres commentaires. Aucun mot-clé surveillé par une
assertion négative n'apparaît dans un commentaire lu par un test.

---

## 7. Déploiement Vercel — vérifié, pas supposé

Un HTTP 200 sur la racine ne prouve pas qu'un nouveau build est servi. Quatre
contrôles ont été faits sur la production après le push.

**7.1 — Les 4 visuels sont servis, aux tailles exactes des fichiers commités**

```
petit-electromenager.png  200  485140
electromenager-moyen.png  200  506982
smartphone.png            200  465629
rewards-hero.png          200  432079
```

Les tailles correspondent octet pour octet : c'est bien le build `55771f0`.

**7.2 — Les 5 PNG supprimés sont bien absents**

```
free_repair.png  404      iron.png         404
mystery_box.png  404      tshirt_cap.png    404      tv.png  404
```

**7.3 — La landing servie référence les nouveaux visuels**

`GET https://www.relioo.space/` (100 908 o) contient
`petit-electromenager`, `electromenager-moyen`, `smartphone` — et **aucune**
occurrence de `tshirt_cap`, `tv.png`, `free_repair`, `mystery_box`.

**7.4 — Non-régression HTTP**

| Contrôle | Résultat |
|---|---|
| `/` | 200 |
| `GET /recompense/rewards-hero.png` | 200 (asset servi, sans référence — conforme à la décision produit) |

---

## 8. Points d'attention

1. **`public/recompense/` ne contient plus que 4 fichiers.** Toute référence
   future à `tv.png` ou `tshirt_cap.png` produira une image cassée à l'écran,
   invisible en `tsc` et au lint. Les 4 tests ajoutés verrouillent désormais ce
   cas de figure.

2. **`rewards-hero.png` (432 Ko) reste sans usage.** Ce n'est pas un asset à
   optimiser : il est **inerte**. Il pèse sur le dépôt et sur le bundle de
   déploiement Vercel pour rien tant qu'aucun écran ne l'expose. À trancher
   lors du prochain aménagement de `/client/recompenses`.

3. **Node 22 est requis et absent de la machine** — seul Node 20.20.2 est
   installé, et `nvm` n'est pas présent, donc `.nvmrc` est inopérant. Sur Node
   20, `npm run test:unit` échoue **39/39** en `ERR_UNKNOWN_FILE_EXTENSION` :
   fausse alerte, pas une régression. Les chiffres de ce rapport ont été
   obtenus avec Node 22.20.0 installé **hors dépôt** (`/tmp/opencode/n22`).
   → Chantier à ouvrir pour l'outillage d'environnement.

4. **`saspay-fees.test.ts:267`** lit `../../backend/src/financial/saspay-fees.ts`
   pour vérifier la symétrie des taux avec le frontend. Le test échoue si le
   dépôt backend n'est pas présent à côté — couplage entre les deux dépôts à
   garder en tête.

5. **La suppression de `reward-catalog.tsx` n'est pas couverte par un test de
   rendu.** Son absence est vérifiée (`existsSync`), mais l'éventuelle
   réapparition d'un composant portant `requiredServices` ne serait pas
   détectée : la règle « aucun composant de lots sur le modèle nombre de
   missions » n'est verrouillée par aucun test.

6. **`DesignSystem` allowlist** : la sortie de `reward-catalog.tsx` de
   `HEX_ALLOWLIST` est un ajustement textile, pas une rupture. Le fichier
   supprimé était le seul à violer la règle des couleurs hex, donc la
   liste rétrécit legitimately.

---

## 9. Écarts avec la demande

| Demande | Réalisé | Écart |
|---|---|---|
| 9 tests à ajouter dans `landing-rewards-content.test.ts` | **4 tests** | Les 5 autres étaient **déjà couverts** par les 7 tests existants du fichier (absence des chaînes obsolètes `5/25/50 dépannages`, du « 5ᵉ vous offre », seuils et libellés du backend). Les aouter aurait été du doublon. |
| `smartphone.png` « inchangé » (l. 230) | inchangé | Aucun écart : le fichier avait été remplacé en amont, la référence ne changeait pas. |
| 2 tests « mis à jour, ce n'est pas une casse » | mis à jour, **+ libellé et commentaire corrigés** dans `demande-draft-routing.test.ts` | Écart mineur : le test annonçait « les 16 appels », un compte devenu faux. La mission n'en parlait pas ; c'est une correction de fond, pas cosmétique. |
| Aucun fichier backend modifié | respecté | — |

---

## 10. Vérifications non faites

Par construction de la mission, **aucun test nécessitant une infrastructure** :

- Pas de `next build`, pas de `npm run dev` (interdit par RÈGLE 3).
- **Aucun rendu React** : le dépôt n'a ni jsdom ni testing-library. Les
  scénarios de production 1 à 4 (scroll, affichage visuel, DevTools) **n'ont
  pas été joués en local** et restent à confirmer à la main sur mobile.
- Pas de base de données : le chantier n'en touche pas.

Les contrôles de la § 7 sont des vérifications HTTP sur la production, pas des
tests de rendu : ils prouvent que les **bons fichiers** sont servis, pas que
l'**agencement visuel** est correct à l'écran.

---

## 11. Questions bloquantes

**Aucune.** Les trois décisions produit ont été rendues avant le chantier et
appliquées telles quelles.

Points ouverts, sans blocage, pour la suite :

- Que faire de `rewards-hero.png`, aujourd'hui sans usage (point 8.2) ?
- La machine de développement n'a pas Node 22 (point 8.3).