# RAPPORT — CHANTIER D2 : WIZARD PUBLIC + MODALE AUTH

**Date** : 2026-10-06
**Dépôt du code** : `Repairdom-frontend` (`https://github.com/digitalholding2026-ux/Repairdom-frontend.git`)
**Statut du code** : ⚠️ **implémenté et vérifié en local, NON commité et NON poussé** — en attente de validation
**Dépend de** : chantier D1 (backend brouillon, commit `8177aa6`, déployé sur Railway)

---

## 1. SYNTHÈSE

Le tunnel de création de demande devient **public** : un visiteur sans compte
peut remplir les 4 étapes sur `/demande`, sa progression est sauvegardée côté
serveur via le brouillon D1, puis il s'inscrit (ou se connecte) dans une modale
au clic sur « Envoyer ». La Demande naît au `convert`, une fois le cookie posé.

Le composant `DemandeWizard` reste **unique** et gère deux modes :

| Mode | Condition | Comportement |
|---|---|---|
| Anonyme | `!authLoading && !authenticated` | Sauvegarde progressive vers `/demandes/drafts` (debounce 800 ms) |
| Authentifié | `authenticated` | **Inchangé** : upload des médias puis `POST /demandes` |

Aucun endpoint backend n'a été touché (interdit par le cahier des charges).
Aucune dépendance ajoutée.

---

## 2. 🔴 BLOCANT FONCTIONNEL IDENTIFIÉ

### 2.1 L'inscription d'un CLIENT ne connecte pas l'utilisateur

**Constat factuel** (code backend, hors périmètre de D2) :

| Fait | Emplacement |
|---|---|
| Le cookie n'est posé que si `user.emailVerified` est vrai | `backend/src/auth/auth.controller.ts:43` |
| Un compte CLIENT est créé avec `emailVerified: false` | `backend/src/auth/auth.service.ts:235` |

Conséquence directe : après `signUp`, **aucun cookie n'est posé**, donc
`POST /demandes/drafts/:token/convert` répond **401**.

### 2.2 Traitement retenu dans D2

`handleAuthSuccess` (`demande-wizard.tsx`) détecte `emailVerified === false` et :

1. **ne ferme pas** la modale ;
2. affiche « Votre compte est créé. Vérifiez votre boîte mail pour confirmer
   votre adresse, puis revenez vous connecter : votre demande est conservée. » ;
3. **conserve le brouillon** (7 jours) — rien n'est perdu ;
4. laisse le visiteur repasser par l'onglet connexion.

### 2.3 Conséquence produit

Le tunnel « inscription → envoi » **ne peut pas se terminer en un seul
passage**. Chemin réel : inscription → vérification e-mail → retour →
connexion → envoi. Mais à ce retour, le wizard ne relit pas le brouillon
(voir § 8.3) : le visiteur repart d'un formulaire vide alors que sa demande
attend toujours côté serveur.

### 2.4 Options pour un chantier ultérieur — aucune appliquée ici

| Option | Effet |
|---|---|
| Poser le cookie à l'inscription même non vérifiée | Tunnel fluide, mais accès granted avant vérification e-mail |
| Autoriser `convert` avec un compte non vérifié | Idem, avec la même réserve sécurité |
| Vérifier l'e-mail avant d'autoriser `convert` | Plus strict : le tunnel reste en deux passages |

**Décision requise.** C'est le seul élément qui empêche le tunnel « demande
d'abord, inscription à la fin » de fonctionner d'un bout à l'autre.

---

## 3. FICHIERS CRÉÉS

| Chemin | Lignes | Rôle |
|---|---|---|
| `frontend/src/app/demande/page.tsx` | 39 | Route **publique** `/demande` |
| `frontend/src/components/client/demande-auth-modal.tsx` | 359 | Modale d'authentification 2 onglets |
| `frontend/src/lib/demande-draft-storage.ts` | 50 | Persistance du token (localStorage) |
| `frontend/src/lib/demande-draft-sync.ts` | 154 | Logique **pure** : projection, seuil, diff, restauration |
| `frontend/src/lib/demande-draft-sync.test.ts` | 289 | 25 tests unitaires |
| `frontend/src/lib/demande-draft-routing.test.ts` | 269 | 19 tests statiques (liens, redirect, contrat) |

**Total : 1160 lignes de code + tests.**

---

## 4. FICHIERS MODIFIÉS

| Chemin | Nature |
|---|---|
| `frontend/src/app/client/demande/page.tsx` | Remplacé par `redirect('/demande')` (33 → 14 l.) |
| `frontend/src/components/client/demande-wizard.tsx` | **+502 l.** — voir § 5 |
| `frontend/src/lib/api/request-service.ts` | +136 l. — 4 fonctions brouillon |
| `frontend/src/lib/demande-media.test.ts` | Ancre de test re-pointée (16 l.) |
| `frontend/package.json` | Enregistrement des 2 tests dans `test:unit` |
| `frontend/tsconfig.json` | Exclusion des 2 tests du typecheck Next |

### 4.1 Les 16 liens migrés vers `/demande`

| Fichier | Occurrences |
|---|---|
| `components/landing/hero.tsx` | 1 (conditionnel `authenticated`) |
| `components/landing/landing-sections.tsx` | 2 (6 tuiles + CTA final) |
| `components/chronologies/tracking-preview.tsx` | 1 |
| `components/public/public-header.tsx` | 1 |
| `components/client/dashboard/client-home-desktop-view.tsx` | 3 |
| `components/client/dashboard/client-home-mobile-view.tsx` | 2 |
| `components/client/dashboard/client-home-blocks.tsx` | 1 |
| `components/client/client-dashboard.tsx` | 1 |
| `components/client/recompenses/reward-catalog.tsx` | 1 |
| `app/client/layout.tsx` | 1 (sidebar) |
| `app/client/confirmation/page.tsx` | 2 (`backHref` + bouton) |

**Vérification exhaustive par programme** (pas un `grep` manuel) : le test
`demande-draft-routing.test.ts` parcourt tous les `.ts`/`.tsx` et échoue si un
seul lien vers `'/client/demande'` subsiste hors page de redirect.

---

## 5. MODIFICATIONS DU WIZARD (détaillées)

### 5.1 Détection du mode

```ts
const { authenticated, loading: authLoading, refresh } = useAuth();
const isAnonymousMode = !authLoading && !authenticated;
```

`isAnonymousMode` vaut **faux pendant `authLoading`** : sans cette garde, un
client déjà connecté verrait brièvement le bandeau « progression sauvegardée »
et déclencherait un PATCH inutile avant la réponse de `GET /auth/me`.

### 5.2 Restauration (refresh / onglet fermé)

Lecture du token en `localStorage` → `GET /demandes/drafts/:token` → projection
vers l'état du wizard. En cas d'erreur (404/410/reseau), le token est oublié et
on repart d'une page vierge : mieux vaut une saisie recommencée qu'un état
restauré à moitié qui ment sur le backend.

Deux points techniques :
- `requestedAt` est stocké en ISO par le backend, `datetime-local` attend du
  local → fonction inverse `toLocalDatetimeValue` ajoutée ;
- le brouillon ne mémorise pas la **précision** GPS → `coords` restauré avec
  `accuracy: null` (l'UI affiche « Position enregistrée » sans chiffre inventé).

### 5.3 Synchronisation progressive (debounce 800 ms)

| Situation | Action |
|---|---|
| Pas de token + saisie sous le minimum DTO | Rien (on ne crée **jamais** de brouillon vide) |
| Pas de token + `description` ≥ 10 car. et `city` non vide | `POST /demandes/drafts` avec l'intégralité des champs |
| Token existant + champs modifiés | `PATCH /demandes/drafts/:token` (diff uniquement) |
| Token existant + rien de modifié | Aucun appel |
| `410 Gone` | Token effacé, nouveau brouillon au prochain tick |
| Autre erreur | `console.warn` **sans le token**, le wizard reste utilisable |

### 5.4 Soumission bifurquée

`handleSubmit` ne fait plus que **arborter** :

- gardes de validation (inchangées) ;
- si anonyme → ouverture de la modale, **aucun appel réseau** ;
- si authentifié → `submitAsAuthenticatedClient()` (comportement historique).

`submitAsAuthenticatedClient` a été **factorisé** car il sert aussi de repli
quand le brouillon est introuvable ou expiré au moment de convertir.

### 5.5 Flux post-authentification

```
onSuccess(user)
  ├─ emailVerified === false → message « vérifiez votre e-mail » (modale ouverte)
  └─ sinon
       ├─ await refresh()                    ← D.1
       ├─ upload des médias (report, D.2)
       │    └─ échec(s) → « Réessayer » / « Continuer sans »
       └─ convertDemandeDraft(token, medias) ← D.3
            ├─ succès → clearDemandeDraftToken() → /client/confirmation
            ├─ 404/410 → repli POST /demandes avec l'état local
            └─ autre → message + « Réessayer »
```

**Échec d'un média** : l'utilisateur choisit. `scope` borne à la fois l'upload
et le corps de la conversion — « Continuer sans » exclut donc réellement le
fichier de la Demande, au lieu de l'envoyer sans `storagePath` (ce qui créerait
une ligne média vide). Les médias refusés restent dans la grille.

---

## 6. MODALE D'AUTHENTIFICATION

**Autonome et non réutilisée** : `ClientAuthForm` vit sous `/client`, suppose un
formulaire en 2 étapes avec ville du référentiel, et gère ses propres
redirections. Le mutualiser aurait recouplé le tunnel public au layout authentifié.

| Onglet | Champs |
|---|---|
| `Créer mon compte` (défaut) | Prénom, Nom, Téléphone, E-mail, Mot de passe, CGU |
| `J'ai déjà un compte` | E-mail, Mot de passe, lien « Mot de passe oublié ? » |

**Règles du backend reproduites côté UI** (sinon 400 à la volée) :

| Règle | Origine |
|---|---|
| Mot de passe ≥ 8 car. + 1 majuscule + 1 minuscule + 1 chiffre | `assertPasswordStrong` |
| Ville et adresse **exigées** pour un CLIENT | `auth.service.ts:177-182` |

La ville et l'adresse sont **reportées depuis le wizard** (props `city` /
`address`) : le wizard les a déjà collectées à l'étape 3, les redemander serait
redondant.

**409 → bascule automatique** : `switchToSignIn(...)` bascule sur l'onglet
connexion, préremplit l'email et affiche « Cet email est déjà utilisé.
Connectez-vous pour continuer. »

**401 `EMAIL_VERIFICATION_REQUIRED`** → message dédié invitant à vérifier l'e-mail.

---

## 7. SÉCURITÉ

| Exigence | Mise en œuvre |
|---|---|
| Token jamais journalisé | Test automatique : aucune instruction `console.*` ne contient `draftToken`, `readDemandeDraftToken` ou `created.token` dans 4 fichiers |
| Token non prédictible | Généré par `crypto.randomUUID()` côté backend (chantier D1) |
| Token pas dans une URL de tracking | Jamais passé en query param |
| Token effacé après conversion | `clearDemandeDraftToken()` + état remis à `null` — sinon le wizard suivant repartirait d'un brouillon converti (409) |
| Accès au brouillon | Le token EST l'autorisation ; `id` / `convertedToDemandeId` / `convertedByUserId` ne sont jamais exposés |
| Aucune dépendance ajoutée | — |
| Aucune route protégée oubliée | `/client/demande` redirige **sans** `RoleGuard` |

---

## 8. POINTS D'ATTENTION

### 8.1 🔴 Le bloquant d'inscription (§ 2)

Seul élément empêchant le tunnel de fonctionner d'un bout à l'autre.

### 8.2 `PublicHeader` : cohérence du tunnel

Le bouton `Déposer une demande` n'est rendu que si `user?.role === 'CLIENT'`
(`public-header.tsx:70`). Un visiteur anonyme y voit donc toujours
`J'ai besoin d'un dépannage` → `/client/inscription` (flux conservé, non touché
comme demandé par le cahier des charges). Le chemin principal est la CTA de la
landing, qui est corrigée.

### 8.3 Reprise après vérification e-mail

Un visiteur qui revient connecté sur `/demande` **ne relit pas** son brouillon
(le mode authentifié n'appelle pas le backend brouillon, conformément à la
consigne B.4). Il repart donc d'un formulaire vide alors que sa demande
attend toujours côté serveur.

### 8.4 Le bouton « Envoyer la demande » n'a pas de `disabled`

Comportement historique conservé : la désactivation reste gérée par
`canContinue` sur les étapes 0-2, et par les gardes de `handleSubmit` à l'étape 3.

### 8.5 Préremplissage profil réservé au mode authentifié

`getMe()` n'est appelé que si `authenticated` — inutile (et 401) pour un anonyme.

---

## 9. VÉRIFICATIONS EFFECTUÉES

| Contrôle | Résultat |
|---|---|
| `tsc --noEmit` | **OK** (0 erreur) |
| `oxlint src/` | **propre** (0 avertissement) |
| `demande-draft-sync.test.ts` | **25/25 verts** |
| `demande-draft-routing.test.ts` | **19/19 verts** |
| Tests existants du wizard | **5 échecs avant, 5 après, noms identiques** → 0 régression |

La non-régression a été établie **factuellement** par `git stash` : exécution du
même harnais sur `HEAD` propre puis sur l'arbre de travail, puis comparaison
des noms d'échecs.

### 9.1 Note sur l'exécution des tests en local

Node v20 est installé sur cette machine alors que le projet exige Node 22+ pour
le runner natif `.ts`. Les tests frontend ont donc été exécutés via un harnais
local qui compile les fichiers purs (`tsc`) puis lance `node --test` **en dehors
du dépôt** — aucune pollution du dépôt de travail, aucune infrastructure.
`npm run test:unit` s'exécutera normalement sur Vercel / en CI.

### 9.2 Deux bugs réels trouvés par les tests

| Bug | Impact | Correction |
|---|---|---|
| `toDraftPayload` **oubliait `brandId`** | Un visiteur reprenant son brouillon aurait eu `canContinue` bloqué à l'étape 0 → **demande inatteignable** | `brandId` projeté explicitement |
| Une assertion que j'avais écrite faux (`> 0` sur une tranche commençant à l'ancre) | Faux négatif de test | Corrigée |

Le harnais a aussi **écrit des `.js` dans le dépôt** via des liens symboliques
(`rm` à travers un lien vers `src/lib`). Détecté par `git status`, nettoyé,
puis le harnais a été corrigé pour ne plus écrire dans le dépôt. **Aucun fichier
parasite ne subsiste** (vérifié par `find src -name "*.js"` → vide).

---

## 10. CRITÈRES D'ACCEPTATION

| Critère | État |
|---|---|
| Route `/demande` créée et publique | ✅ |
| Wizard détecte le mode (anonyme / authentifié) | ✅ |
| En mode anonyme, brouillon créé et mis à jour (debounce 800 ms) | ✅ |
| Token persisté en `localStorage` | ✅ |
| Modale auth créée (2 onglets) | ✅ |
| 409 au signup → bascule sur onglet login | ✅ |
| Après auth → upload médias + convert + redirect | ✅ (sauf § 2) |
| 16 liens mis à jour vers `/demande` | ✅ |
| `/client/demande` redirige vers `/demande` | ✅ |
| Mode authentifié inchangé | ✅ |
| Token `localStorage` supprimé après conversion | ✅ |
| Tests frontend verts + nouveaux | ✅ 44/44 |
| Lint + tsc propres | ✅ |
| Push → Vercel vert | ❌ **non poussé** |

---

## 11. SCÉNARIO DE TEST PRODUCTION

> ⚠️ Non exécutable tant que le code n'est pas poussé, et le parcours
> d'inscription reste bloqué par § 2.

| # | Test | Attendu |
|---|---|---|
| 1 | Navigation privée → `/demande`, remplir les 4 étapes | PATCH `/demandes/drafts/:token` debouncés (Network) |
| 2 | localStorage | clé `relio_demande_draft_token` présente |
| 3 | Clic « Envoyer » | modale ouverte, **aucun** `POST /demandes` |
| 4 | S'inscrire avec un e-mail neuf | § 2 s'applique : message « vérifiez votre e-mail » |
| 5 | E-mail déjà utilisé (409) | bascule onglet « J'ai déjà un compte » + email prérempli |
| 6 | F5 après 2 étapes | champs préremplis depuis le brouillon |
| 7 | Utilisateur déjà connecté sur `/demande` | **aucun** brouillon créé, modale jamais ouverte |
| 8 | Landing → « Décrire ma panne » | arrive sur `/demande` (pas `/client/connexion`) |
| 9 | `/client/demande` | redirect immédiat vers `/demande` |
| 10 | Après convert réussi | `relio_demande_draft_token` **absente** ; mission en SUBMITTED dans `/client/demandes` |

---

## 12. PROCHAINE ÉTAPE

1. **Répondre au point § 2** (blocant inscription).
2. Valider puis **committer et pousser** le frontend D2 → Vercel.
3. Exécuter le scénario de production ci-dessus.
4. Chantier D3 probable : reprise du brouillon pour un utilisateur revenu
   connecté (§ 8.3), et arbitrage du bloquant d'inscription.