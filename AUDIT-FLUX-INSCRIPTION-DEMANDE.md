# AUDIT — Flux inscription + création de demande (Relio)

**Date** : 2026-10-06 · **Périmètre** : `/config/projet/frontend` + `/config/projet/backend` · **Mode** : lecture seule stricte (aucun test, aucun build, aucun serveur, aucune modification, aucun commit)
**Version auditée** : backend `e6037c7` · frontend `48defd8`

---

## 1. FLUX INSCRIPTION CLIENT ACTUEL

### a) Landing → CTA « Décrire ma panne »

Fichier : `frontend/src/components/landing/hero.tsx`

| Libellé exact | Condition | URL cible | Ligne |
|---|---|---|---|
| `Décrire ma panne` | non connecté | `/client/demande` | libellé l. 24 · href l. 23 · `<Link>` l. 43-49 |
| `Continuer ma mission` | connecté | `homePathForRole(user?.role)` | l. 23-24 |

Expression exacte (l. 22-24) :

```
const { user, authenticated } = useAuth();
const primaryHref = authenticated ? homePathForRole(user?.role) : '/client/demande';
const primaryLabel = authenticated ? 'Continuer ma mission' : 'Décrire ma panne';
```

Autres CTA vers `/client/demande` : 6 tuiles services `landing-sections.tsx` l. 186 ; CTA `Commencer maintenant` l. 315.

CTA vers `/client/inscription` : `public-header.tsx` l. 96 (`J'ai besoin d'un dépannage`), l. 115 (`Dépannage`), l. 170.

**Comportement effectif si non connecté** : le CTA est un `<Link>` client sans logique conditionnelle → navigation vers `/client/demande` → le `RoleGuard` du layout `/client` (`client/layout.tsx` l. 87-90) redirige vers `/client/connexion?redirect=%2Fclient%2Fdemande` (`guard-decision.ts` l. 63-66).

Pendant `loading`, `hero.tsx` ne déstructure pas `loading` → `authenticated` vaut `user !== null` → le Hero affiche transitoirement `Décrire ma panne`.

### b) Écran d'inscription `/client/inscription`

**Chaîne de composants** :

- `frontend/src/app/client/inscription/page.tsx` (12 l., serveur) l. 2 → `RegisterSplit`
- `frontend/src/components/auth/register-split.tsx` (35 l.) l. 5-6 → `ClientAuthForm` + `AuthSplit`
- `frontend/src/components/client/client-auth-form.tsx` (438 l., `'use client'`) — composant porteur du formulaire
- `frontend/src/components/ui/{field,input,select,button,alert}.tsx`
- `frontend/src/lib/api/auth-service.ts` (service réseau)
- `frontend/src/lib/api/cities-service.ts` (`listCities`)

**Champs — formulaire 2 étapes, état local `step` (`useState<1|2>(1)` l. 50)**

Étape 1 — libellés de progression `1. Identité & Contact` (l. 166-172) :

| Libellé | `id` | Contrôle | required | placeholder | Ligne |
|---|---|---|---|---|---|
| `Prénom` | `client-firstName` | Input text | oui | `Votre prénom` | 194-202 |
| `Nom` | `client-lastName` | Input text | oui | `Votre nom` | 203-211 |
| `Téléphone` (hint `WhatsApp de préférence`) | `client-phone` | Input text `inputMode="tel"` | oui | `6XX XXX XXX` | 217-226 |
| `WhatsApp (facultatif)` | `client-whatsapp` | Input text `inputMode="tel"` | non | `6XX XXX XXX` | 230-250 |
| `Adresse e-mail` | `client-email` | Input email | oui | `vous@exemple.cm` | 252-263 |

Étape 2 — `2. Localisation & Sécurité` :

| Libellé | `id` | Contrôle | required | placeholder | Ligne |
|---|---|---|---|---|---|
| `Ville` | `client-city` | Select (`value={c.name}`) | oui | `Sélectionnez votre ville` | 266-275 |
| `Adresse précise` (hint `Quartier, rue, lieu-dit, numéro de maison…`) | `client-address` | Input text | oui | idem hint | 278-293 |
| `Mot de passe` | `client-password` | Input password + bouton œil | oui | `Au moins 8 caractères` | 296-345 |
| `Confirmer le mot de passe` | `client-confirmPassword` | Input password | oui | `Confirmez votre mot de passe` | 349-361 |
| `J'accepte les Conditions d'utilisation de Relio.` | — | checkbox | oui (`required` HTML l. 371) | — | 365-380 |

Aucun `maxLength`, `pattern`, `minLength` sur aucun champ. Aucun attribut `name` (contrôles identifiés par `id`).

**Validation frontend** — expressions booléennes recalculées au rendu, déclenchées **uniquement sur `onSubmit`** (l. 162) :

| Règle | Expression | Ligne |
|---|---|---|
| Password ≥ 8 | `password.length >= 8` | 57 |
| Concordance | `password === confirmPassword` | 58 |
| Email | `/\S+@\S+\.\S+/.test(email.trim())` | 60 |
| Passage étape 2 | `firstName.trim() !== '' && lastName.trim() !== '' && emailValid` | 63-64 |
| Soumission | `email && passwordValid && passwordsMatch && (firstName && lastName && cityValue && address && acceptTerms)` | 69-73 |

Aucun `onBlur` ni validation bloquante au `onChange`.

**Endpoint appelé** : `signUp()` → `POST ${siteConfig.apiBaseUrl}/auth/register` (`auth-service.ts` l. 98-102), en-tête `Content-Type: application/json`, `credentials: 'include'` (l. 63). **Aucun header `Authorization`.**

**Payload exact** (`auth-service.ts` l. 72-102, assemblage l. 113-121) :

```json
{
  "firstName": "<trim>",
  "lastName": "<trim>|absent si vide",
  "phone": "<trim>|absent si vide",
  "whatsapp": "<trim>|absent si vide",
  "email": "<trim>",
  "password": "<non trimé>",
  "city": "<trim>|absent si vide",
  "address": "<trim>|absent si vide"
}
```

`role` n'est **jamais** envoyé pour un client : `payload.role` n'est ajouté que si `input.role === 'TECHNICIAN'` (l. 81-89). `cityId` / `categories` également réservés au technicien (l. 87-88). Retour attendu `{ user: AuthUser; mode: 'real' }`.

**Redirection après succès** (`client-auth-form.tsx` l. 123-131) :

- `emailVerified === false` → `router.push('/client/verification?email=<encodé>')` (l. 126)
- sinon → `await refresh()` puis `router.push(safeRedirect(window.location.search, '/client', homePathForRole(session.user.role)))` (l. 128-129)
- `safeRedirect` (l. 127-142) n'accepte un `?redirect=` que s'il commence par `/client`, ne commence pas par `//` et ne contient pas `\`

Cas d'erreur notables : 409 sur register → redirection vers `/client/verification` **sans message d'erreur** (l. 144-148) ; `EMAIL_VERIFICATION_REQUIRED` en login → redirection sans message (l. 151-155).

### c) Vérification email `/client/verification`

- `frontend/src/app/client/verification/page.tsx` (15 l.) l. 3 → `VerificationPanel`
- `frontend/src/components/auth/verification-panel.tsx` (210 l.) — lit `?token` (l. 45) et `?email` (l. 46)

| Action | Endpoint | Corps |
|---|---|---|
| Vérifier | `POST /auth/verify-email` (`auth-service.ts` l. 156-162) | `{"token": …}` |
| Renvoyer | `POST /auth/resend-verification` (l. 164-170) | `{"email": …}` |
| Purge session | `POST /auth/logout` (l. 144-146) | — |
| Resync | `GET /auth/me` | — |

Séquence (l. 59-95) : `verifyEmail` → `logout()` → `refresh()` → `setVerified(true)`.

**Blocage avant dashboard** : implémenté dans `guard-decision.ts` l. 44-46 :

```
if (role === 'CLIENT' && emailVerified === false && pathname !== '/client/verification') {
  return { action: 'redirect', to: '/client/verification' };
}
```

Un CLIENT authentifié non vérifié est bloqué sur **toutes** les routes `/client/*` sauf `/client/verification`.

**Note factuelle** : côté backend, `verifyEmail` **pose le cookie** (`auth.controller.ts` l. 53) ; côté frontend, le panneau appelle `logout()` immédiatement après (l. 69) puis `refresh()`. L'utilisateur doit ensuite cliquer sur `Aller à la connexion` (l. 129-133).

### d) Première connexion `/client`

- `frontend/src/app/client/layout.tsx` (115 l., `'use client'`) — `CLIENT_NAV` l. 17-25, `CLIENT_PUBLIC_PATHS` l. 27-31, `<RoleGuard expectedRole="CLIENT" publicPaths={CLIENT_PUBLIC_PATHS}>` l. 87-90 enveloppant `{children}`
- **Aucun provider d'auth dans ce layout** — `AuthProvider` est monté dans le layout racine `src/app/layout.tsx` l. 69
- `frontend/src/app/client/page.tsx` (10 l.) → `ClientDashboard`
- `use-client-dashboard-data.ts` l. 58-64 : `user=null`, `demandes=[]`, `historique=[]`, `balance=null`, `showBalance=false`, `loading=true`, `error=null`
- Effet l. 66-102 : `getMe()` l. 71 → si `me.role !== 'CLIENT'` → `router.replace(homePathForRole(me.role))` l. 74 ; puis `Promise.all([listMyDemandes(), listMyDemandeHistory()])` l. 77-80 ; puis `getClientFinanceSummary()` l. 86-90 (échec silencieux l. 90)

Avant ce chargement, l'écran affiché est `LoadingScreen` du `RoleGuard` (`role-guard.tsx` l. 43-45), pas le squelette `ClientDashboardSkeleton`.

---

## 2. FLUX CRÉATION DE DEMANDE ACTUEL

### a) Accès au wizard `/client/demande`

`frontend/src/app/client/demande/page.tsx` (20 l.) — `<PageHeader title="Nouvelle demande" backHref="/client/demandes" />` + `<DemandeWizard />` (l. 17, sans prop).

**16 points d'entrée** dans l'UI (tous des `<Link>`) :

| Fichier | Ligne(s) | Libellé |
|---|---|---|
| `app/client/layout.tsx` | 82 (sidebar desktop) | `Nouvelle demande` |
| `dashboard/client-home-desktop-view.tsx` | 53-58 / 83-88 / 167-172 | `Nouvelle demande` / `Créer une demande` ×2 |
| `dashboard/client-home-mobile-view.tsx` | 105-112 / 150-156 | `Créer une demande` ×2 |
| `dashboard/client-home-blocks.tsx` | 94-102 | `Demander un dépannage` |
| `client-dashboard.tsx` | 118-121 | `Déposer une panne` |
| `landing/hero.tsx` | 23 | `Décrire ma panne` |
| `landing/landing-sections.tsx` | 186 (×6 tuiles) | `Électricité` `Plomberie` `Climatisation` `Appareils` `Serrurerie` `Autre` |
| `landing/landing-sections.tsx` | 315 | `Commencer maintenant` |
| `public/public-header.tsx` | 71-75 | `Déposer une demande` |
| `chronologies/tracking-preview.tsx` | 123-128 | `Lancer un dépannage` |
| `recompenses/reward-catalog.tsx` | 232-237 | `Lancer un dépannage` |
| `app/client/confirmation/page.tsx` | 38 / 104-108 | `Retour` / `Déposer une nouvelle demande` |

Absents : bottom nav mobile, footer, pages `/client/demandes*`.

**Guard appliqué** : `RoleGuard expectedRole="CLIENT"` du layout `/client` l. 87-90. **Aucun guard propre dans la page.**

**Comportement si non connecté** : redirection vers `/client/connexion?redirect=%2Fclient%2Fdemande` (`guard-decision.ts` l. 61-66) + `LoadingScreen` pendant la redirection (`role-guard.tsx` l. 43-44).

### b) Structure du wizard

Fichier : `frontend/src/components/client/demande-wizard.tsx` — **1216 lignes**, `'use client'` l. 1, `export function DemandeWizard()` l. 86. **Aucune prop.**

**27 `useState`** :

| L. | Variable | Initial | | L. | Variable | Initial |
|---|---|---|---|---|---|---|
| 89 | `step` | `0` | | 117 | `families` | `[]` |
| 90 | `categoryId` | `''` | | 118 | `equipmentFamily` | `''` |
| 91 | `city` | `''` | | 122 | `description` | `''` |
| 92 | `neighborhood` | `''` | | 123 | `domains` | `[]` |
| 93 | `address` | `''` | | 124 | `catalogLoading` | `true` |
| 94 | `landmark` | `''` | | 125 | `domainId` | `''` |
| 95 | `contactPhone` | `''` | | 126 | `domainName` | `''` |
| 96 | `requestedMode` | `'ASAP'` | | 127 | `brandId` | `''` |
| 97 | `requestedAt` | `''` | | 128 | `brands` | `[]` |
| 98 | `medias` | `[]` | | 102 | `voiceKey` | `0` |
| 99 | `mediaError` | `null` | | 103 | `isSubmitting` | `false` |
| 106 | `uploadStatus` | `null` | | 104 | `error` | `null` |
| 110 | `coords` | `null` | | 111-112 | `geoLoading`/`geoError` | `false`/`null` |

Constantes : `STEPS` l. 34, `MAX_MEDIAS = 5` l. 42, `MAX_MEDIA_BYTES = 25*1024*1024` l. 43, `DESCRIPTION_MIN_LENGTH = 10` l. 121, `OTHER_DOMAIN = '__other__'` l. 36, `CATEGORY_ICONS` l. 60-68.

**Hooks** : `useRouter` l. 87 (utilisé **uniquement** l. 427-431) · `useState` ×27 l. 89-128 · `useRef` `mediasRef` l. 239 · `useMemo` ×6 l. 170-212 · `useEffect` ×4 l. 130-151, 156-168, 243-250, 459-466 · `memo(StepProgress)` l. 1074.

**Absents** : `useCallback`, `useReducer`, `useSearchParams`, `useParams`, `useAuth`, `useStableIdempotencyKey`, `framer-motion`.

**4 `useEffect`** :

| # | Lignes | Deps | Effet |
|---|---|---|---|
| 1 | 130-151 | `[]` | charge `listCatalogDomains()` + `listEquipmentFamilies()` ; échecs silencieux (`.catch(() => setDomains([]))` l. 136, `.catch(() => undefined)` l. 143) |
| 2 | 156-168 | `[]` | `getMe()` → préremplit `city` (l. 161) et `contactPhone` (l. 162) **seulement si vides** ; `.catch(() => undefined)` l. 163 |
| 3 | 243-250 | `[]` | cleanup `URL.revokeObjectURL` au démontage |
| 4 | 459-466 | `[hasStarted, isSubmitting]` | `beforeunload` → `event.preventDefault()` l. 462 |

**État initial** : `step=0`, tous champs texte vides, `requestedMode='ASAP'`, `catalogLoading=true`. Chargements asynchrones au montage : domaines, familles, profil.

**Nombre d'étapes : 4** — `const STEPS = ['Votre appareil', 'Votre panne', 'Où et quand ?', 'Vérifiez et envoyez'];` l. 34

#### Étape 0 — « Votre appareil » (l. 491-590)

Titre `Quel appareil avez-vous ?` (l. 494).

| Contrôle | Libellé | Type | required | Lignes |
|---|---|---|---|---|
| Grille domaines | `domain.name` + description | `DomainTile` = `<button type="button" aria-pressed>` | oui | 519-529 |
| Tuile hors catalogue | `Autre appareil` / `Non présent dans la liste` | `DomainTile` | oui | 530-536 |
| Marque | `Marque` | `SelectChip` via `DeviceChips` | **oui si domaine catalogue** | 540-554 |
| Famille d'équipement (parcours Autre) | `Quel type d'appareil souhaitez-vous faire réparer ?` | `SelectChip`, `role="group"` | **oui dans ce parcours** | 556-588 |
| État vide catalogue | `Catalogue indisponible` + `Décrire sans catégorie` | `EmptyState` | — | 507-517 |
| État vide marques | `Aucune marque disponible` / `Cette marque n'est pas encore disponible sur Relio.` | — | — | 547-551 |
| État vide familles | `Types d'appareil indisponibles` / `Rechargez la page…` | — | — | 566-570 |

#### Étape 1 — « Votre panne » (l. 592-740)

Titre `Décrivez votre panne` (l. 595). Sous-titre : `…max 5 fichiers, 25 Mo chacun.`

| Libellé | `id` | Contrôle | required | placeholder / maxLength | Lignes |
|---|---|---|---|---|---|
| `Description du problème` | `demande-description` | Textarea | **oui** | `Ex. : Mon téléphone ne charge plus depuis hier, même avec un autre câble.` · `maxLength={1000}` l. 613 · `rows={4}` | 603-618 |
| `Message vocal` | — | `VoiceRecorder` | non | hint `Décrivez oralement la panne (max 3 min). Rien n'est envoyé sans validation.` | 621-632 |
| `Vidéo (facultatif)` | `demande-video` | `<input type="file" accept="video/*" multiple>` | non | `Filmez la panne ou choisissez une vidéo (25 Mo max).` | 635-663 |
| `Photos (facultatif)` | `demande-photos` | `<input type="file" accept="image/*" multiple>` | non | `Jusqu'à 5 fichiers au total · 25 Mo maximum chacun.` | 666-694 |

Hint dynamique l. 607 : `10 caractères minimum ({description.trim().length}/10).`

#### Étape 2 — « Où et quand ? » (l. 742-898)

| Libellé | `id` | Contrôle | required | placeholder | Lignes |
|---|---|---|---|---|---|
| `Ville *` | `demande-city` | Input text | **oui** | `Ex. : Yaoundé` | 744-752 |
| `Adresse précise (facultatif)` | `demande-address` | Input text | non | `Ex. : 12 rue des Lilas, 3e étage` | 754-762 |
| `Quartier / secteur (facultatif)` | `demande-neighborhood` | Input text | non | `Ex. : Mbankomo, quartier centre` | 764-771 |
| `Point de repère (facultatif)` | `demande-landmark` | Input text | non | `Ex. : à côté de la pharmacie du quartier` | 773-780 |
| `Téléphone pour l'intervention (facultatif)` | `demande-contact-phone` | Input tel | non | `Ex. : +237 6 00 00 00 00` | 782-795 |
| `Position GPS (facultatif)` | — | `<Button>Utiliser ma position` | non | — | 797-835 |
| `Quand souhaitez-vous être dépanné ? *` | — | radiogroup : `Dès que possible` / `À une date précise` | oui (défaut ASAP) | — | 837-883 |
| `Date et heure souhaitées *` | `demande-datetime` | Input `datetime-local` (`min={minRequestedAt}` l. 891) | oui si `SCHEDULED` | — | 885-895 |

**Aucun `Select`** dans le wizard : la ville est une saisie texte libre.

#### Étape 3 — « Vérifiez et envoyez » (l. 900-948)

Aucun champ de saisie. `SummaryRow` : `Appareil` (l. 903-915), `Panne` (l. 916-923), `Localisation` (l. 924-929), `Moment souhaité` (l. 930-935), chacune avec un bouton `Modifier` → `jumpTo(...)`.

`Alert variant="info"` l. 938-941 : `Vous recevrez un devis à valider avant toute intervention. Aucun paiement n'est demandé ici.`

`Alert` pour `uploadStatus` l. 942-946.

#### Validations par étape

**Aucun `onBlur`, aucun message d'erreur par champ, aucun `<form onSubmit>`.** Seule la règle du bouton :

```
200: const canContinue = useMemo(() => {
203:   if (step === 0) {
204:     if (domainId === '') return false;
205:     if (domainId === OTHER_DOMAIN) return equipmentFamily.trim() !== '';
206:     return brandId !== '';
207:   }
209:   if (step === 1) return description.trim().length >= DESCRIPTION_MIN_LENGTH;
210:   if (step === 2) return city.trim() !== '' && (requestedMode === 'ASAP' || requestedAt !== '');
211:   return true;
212: }, [step, domainId, equipmentFamily, brandId, description, city, requestedMode, requestedAt]);
```

Gardes de `handleSubmit` (l. 370-383), à la soumission uniquement :

| Condition | Message |
|---|---|
| `description.trim().length < 10` | `Décrivez votre problème en quelques mots (10 caractères minimum).` |
| `OTHER_DOMAIN && equipmentFamily === ''` | `Sélectionnez le type d'appareil qui correspond le mieux à votre situation.` |
| `domainId !== '' && !== OTHER_DOMAIN && brandId === ''` | `Sélectionnez une marque disponible pour cette catégorie.` |

**`city`, `requestedMode`, `requestedAt`, `contactPhone` et les médias ne sont pas revalidés** dans `handleSubmit`.

#### Navigation

| Handler | Lignes | Code |
|---|---|---|
| `goNext` | 331-334 | `setError(null); if (step < STEPS.length - 1) setStep(s => s + 1)` — **ne revalide rien** |
| `goBack` | 336-339 | `setError(null); if (step > 0) setStep(s => s - 1)` |
| `jumpTo` | 341-344 | `setError(null); setStep(targetStep)` — cible non bornée |

- Boutons l. 951-967 : `Retour` (`disabled={isSubmitting}` l. 953) · `Continuer` (`disabled={!canContinue}` l. 959) · `Envoyer la demande` (`isLoading={isSubmitting}` l. 963, **aucun `disabled` explicite**)
- Animation : `<div key={step} className="animate-pop-in">` l. 490 (CSS `globals.css:584-586`) — **pas de framer-motion**
- **Aucun `?step=` en URL, aucun `useSearchParams`, aucun `pushState`, aucun `popstate`**
- **Aucun `scrollTo`/`scrollIntoView`** ; le `ScrollToTop` du layout ne réagit qu'aux changements de `pathname`

#### Persistance intermédiaire

**Aucune.** Recherche exhaustive : ni `localStorage`, ni `sessionStorage`, ni cookie, ni appel backend de brouillon dans `demande-wizard.tsx`. Les seules occurrences de `localStorage` dans `src/` sont `push/push-client.ts:59,69` et `technician/use-onboarding-banner-visibility.ts:34,46` — aucun des deux fichiers n'est importé par le wizard.

Seule persistance de l'effort : l'avertissement `beforeunload` (l. 448-466), condition `hasStarted` (l. 450-458). **Il ne couvre pas la navigation interne** (ex. le lien `Retour` du `PageHeader`).

**Écran de reprise de brouillon : ABSENT.**

### c) Soumission finale

Fonction `handleSubmit` l. 365-446. Ordre exact :

| Ordre | Lignes | Action |
|---|---|---|
| 1 | 366-367 | `setError(null)` ; `setUploadStatus(null)` |
| 2 | 370-383 | gardes sync (3) |
| 3 | 384 | `setIsSubmitting(true)` |
| 4 | 388-399 | boucle **séquentielle** `await uploadDemandeMedia(media.file, media.kind)` ; `setUploadStatus('Envoi des fichiers i/n…')` |
| 5 | 400 | `setUploadStatus(null)` |
| 6 | 401-426 | `await createDemande({...})` |
| 7 | 427-431 | `router.push('/client/confirmation?…')` |
| 8 | 432-445 | catch : message + **cleanup best-effort** `deleteUploadedDemandeMedia` sur chaque `storagePath` déjà uploadé (l. 437-443, erreurs silencieuses l. 440-442) |

**Upload médias — AVANT la création** (ordre vérifié par test : `demande-media.test.ts` l. 41-47).

| | Valeur | Ligne |
|---|---|---|
| Nombre max (vocal+vidéo+photos) | `5` | wizard l. 42 · DTO `@ArrayMaxSize(5)` |
| Taille max | `25 * 1024 * 1024` | wizard l. 43 |
| `accept` vidéo | `video/*` | l. 652 |
| `accept` photos | `image/*` | l. 683 |
| Endpoint | `POST /demandes/medias/upload` | `request-service.ts` l. 235-243 |
| Corps upload | `FormData` : `file` + `kind` | l. 237-238 |

**Payload exact de `createDemande`** — couche wizard (l. 401-426) puis couche service (`request-service.ts` l. 154-188) :

```json
{
  "categoryId": "…",
  "description": "…",
  "equipmentFamily": "…",
  "city": "…",
  "neighborhood": "…", "address": "…", "landmark": "…", "contactPhone": "…",
  "latitude": 0.0, "longitude": 0.0,
  "requestedMode": "ASAP|SCHEDULED",
  "requestedAt": "ISO",
  "domainId": "…", "brandId": "…",
  "modelId": undefined, "problemId": undefined,
  "medias": [{ "kind": "IMAGE|VIDEO|AUDIO", "name": "…", "mimeType": "…", "sizeBytes": 0, "storagePath": "…" }]
}
```

`modelId` et `problemId` sont **toujours `undefined`** (jamais produits par le wizard).

**Idempotence : ABSENTE.** Ni `Idempotency-Key`, ni champ `idempotencyKey`. `useStableIdempotencyKey` existe (`frontend/src/lib/use-stable-idempotency-key.ts:15-34`) mais n'est importé que par `app/client/solde/recharger/page.tsx:20,68` et `components/finance/withdrawal-panel.tsx:24,116`.

**Redirection** (l. 427-431) :

```
router.push(`/client/confirmation?ref=${encodeURIComponent(result.reference)}&id=${encodeURIComponent(result.id)}&mode=${encodeURIComponent(requestedMode)}&req=${encodeURIComponent(requestedAtIso ?? '')}`);
```

**`/client/confirmation`** — `app/client/confirmation/page.tsx` (117 l.), Server Component `async` l. 26. Lit `searchParams` `ref`, `id`, `mode`, `req` (l. 27-31). **Aucun appel réseau.** Affiche référence + bouton copie, bloc `Et maintenant ?`, `PushNotificationCard`, boutons `Suivre ma demande` (l. 94-98), `Suivre ma mission` → `/suivi?reference=` (l. 99-103), `Déposer une nouvelle demande` (l. 104-108), `Retour à l'accueil` (l. 109-113).

### d) Composant `DemandeWizard` — synthèse

| Aspect | Constat |
|---|---|
| Props | Aucune (l. 86) |
| State | 27 `useState` — détail § 2.b |
| Hooks | `useRouter`, `useState`×27, `useRef`, `useMemo`×6, `useEffect`×4, `memo` |
| Effets de bord | uploads (l. 388-399) · suppression d'uploads orphelins à l'échec (l. 437-443) · préremplissage profil (l. 156-168) · création/révocation d'`URL.createObjectURL` (l. 271 / l. 326 / l. 243-250) · géolocalisation ponctuelle (l. 797-835) · `beforeunload` (l. 459-466) · navigation l. 427-431 |

**Sous-catégorie** : ABSENTE comme notion distincte. Les deux notions de second niveau sont la **marque** (`brandId`, obligatoire si domaine catalogue) et la **famille d'équipement** (`equipmentFamily`, uniquement dans le parcours `Autre appareil`). `categoryId` est dérivé de `domain.category` l. 227, forcé `'autre'` l. 222, borné `|| 'autre'` l. 402.

---

## 3. BACKEND — CRÉATION DE DEMANDE

Préfixe global `app.setGlobalPrefix('api')` (`backend/src/main.ts:16`) → toutes les routes `/api/...`.

### a) `POST /api/demandes`

| Élément | Valeur | Ligne |
|---|---|---|
| Controller | `@Controller('demandes')` | `demandes.controller.ts:15` |
| Guards (classe) | `@UseGuards(JwtAuthGuard, RolesGuard)` | l. 16 |
| Rôle (classe) | `@Roles('CLIENT')` | l. 17 |
| Route | `@Post()` | l. 64 |
| Méthode | `create(@CurrentUser() user, @Body() dto: CreateDemandeDto)` → `this.demandesService.create(user.id, dto)` | l. 65-67 |
| Status | **201** (aucun `@HttpCode`) | — |

**DTO** : `backend/src/demandes/dto/create-demande.dto.ts` (187 l.)

| Champ | Validateurs (ordre) | L. |
|---|---|---|
| `categoryId` | `@IsIn(ALLOWED_CATEGORIES)` — `electricite, plomberie, climatisation, electromenager, serrurerie, informatique, autre` (`categories.ts:3-11`) | 58-59 |
| `domainId?` | `@IsOptional @IsString @IsNotEmpty @MaxLength(80)` | 67-71 |
| `brandId?` | idem | 73-77 |
| `modelId?` | idem | 79-83 |
| `problemId?` | idem | 85-89 |
| `description` | `@IsString @IsNotEmpty @MinLength(10) @MaxLength(1000) @Matches(/\S/)` | 106-111 |
| `equipmentFamily?` | `@IsOptional @IsString @MaxLength(40)` | 118-121 |
| `city` | `@IsString @IsNotEmpty @MaxLength(120)` | 123-126 |
| `zoneId?` | `@IsOptional @IsString @IsNotEmpty @MaxLength(80)` | 131-135 |
| `neighborhood?` | `@IsOptional @IsString @MaxLength(120)` | 137-140 |
| `address?` | `@IsOptional @IsString @MaxLength(200)` | 142-145 |
| `landmark?` | idem | 147-150 |
| `contactPhone?` | `@IsOptional @IsString @MaxLength(30)` | 152-155 |
| `latitude?` | `@IsOptional @IsNumber({allowNaN:false,allowInfinity:false}) @Min(-90) @Max(90)` | 161-165 |
| `longitude?` | idem `@Min(-180) @Max(180)` | 167-171 |
| `medias?` | `@IsOptional @IsArray @ArrayMaxSize(5) @ValidateNested({each:true}) @Type(() => RequestMediaDto)` | 173-178 |
| `requestedMode?` | `@IsOptional @IsIn(['ASAP','SCHEDULED'])` | 180-182 |
| `requestedAt?` | `@IsOptional @IsDateString` | 184-186 |

`RequestMediaDto` (l. 27-55) : `kind` `@IsIn(['IMAGE','VIDEO','AUDIO'])` · `name` `@IsString @IsNotEmpty @MaxLength(255)` · `mimeType` `@IsString @IsNotEmpty @MaxLength(120)` · `sizeBytes` `@IsInt @Min(1) @Max(26214400)` · `storagePath?` `@IsOptional @IsString @IsNotEmpty @MaxLength(500)`.

**Service** : `DemandesService.create(clientId, dto)` — `demandes.service.ts:86-215`

Opérations **avant** transaction : `medias` l. 87 · trim description l. 96 · **400 si pas de description ET pas de média** l. 97-101 · `requestedMode` défaut `'ASAP'` l. 102 · `resolveRequestedAt` l. 103 (400 si `SCHEDULED` sans date valide ou date passée) · `resolveDevice` l. 104 · gestion `equipmentFamily` l. 110-129 (400 si famille + domaine l. 112-116 ; 400 si `autre` sans famille l. 117-122 ; famille doit exister et être active l. 123-126) · `resolveDemandeGeo` l. 132.

**Transaction : OUI** — `this.prisma.$transaction(async (tx) => {…})` l. 137-191, dans une boucle `for (let attempt = 0; attempt < 5; attempt += 1)` l. 134.

Étapes dans la transaction :

1. `generateReference()` l. 58-63 (alphabet l. 53, longueur `REFERENCE_LENGTH = 6` l. 54)
2. `tx.demande.create({ data: {…}, include: clientInclude() })` l. 138-180 — champs l. 140-161 ; `equipmentType: null` l. 143 ; `clientId` l. 155 ; `medias: { create: … }` l. 162-177 **seulement si `medias.length > 0`** (sinon `undefined` l. 177) ; `include` l. 179 (défini l. 551-570)
3. `recordEvent(tx, { type: 'CREATED', actorUserId: clientId, toStatus: 'SUBMITTED' })` l. 183-188 (`mission-events.ts:95-106`)
4. `return created` l. 190

Après transaction : `P2002` → retry l. 207-211 ; 5 échecs → `Error` brute l. 214 (→ 500 via filtre global).

**Dispatch** : **synchrone, après commit, encapsulé dans try/catch** — l. 196-204 :

```
await this.dispatch.dispatchWave1(demande.id);   // l.197
catch { this.logger.error(...) }                  // l.198-204
```

`dispatchWave1` → `runWave(demandeId, DISPATCH_WAVE_1, new Date())` (`dispatch.service.ts:234-236`). **Pas de fire-and-forget ni `setTimeout`.** La vague 2 tourne via `DispatchScheduler` : `onModuleInit()` l. 26-31 → `setInterval(…, 60_000)` l. 27 (`DISPATCH_SWEEP_INTERVAL_MS`, l. 16).

**Réponse** : `toApiDemande(this.withTechnician(demande))` l. 206 (`demande-helpers.ts:110-170`) — 31 champs dans l'ordre :

`id, reference, status, categoryId, categoryLabel, description, equipmentType, equipmentFamily, city, cityId, zoneId, zone, cityRef, neighborhood, address, landmark, contactPhone, latitude, longitude, technicianId, technician, scheduledAt, requestedMode, requestedAt, domain, brand, model, problem, negotiationRequestedAt, finalAmount, medias[], mediaPersisted, storageStatus, createdAt`

- `medias[]` = `{ id, kind, name, mimeType, sizeBytes, stored }` — **`storagePath` jamais exposé** (l. 163-164)
- `mediaPersisted: false` l. 166 et `storageStatus: 'metadata-only'` l. 167 — **valeurs constantes, non calculées**

### b) Upload médias

| Route | Ligne | Status |
|---|---|---|
| `POST /api/demandes/medias/upload` | 28-41 | 201 (`@HttpCode(CREATED)` l. 29) |
| `DELETE /api/demandes/medias/upload` | 44-49 | 200 |
| `GET /api/demandes/:id/medias/:mediaId/file` | 53-62 | 200 |

Guards de classe identiques : `JwtAuthGuard, RolesGuard` + `@Roles('CLIENT')` (l. 16-17).

`@UseInterceptors(FileInterceptor('file', { limits: { files: 1, fileSize: DEMANDE_MEDIA_MAX_BYTES } }))` l. 30-34. Paramètres : `@UploadedFile() file` l. 37 · `@Body('kind') kind?: string` l. 38 — **champ texte libre, aucun DTO, aucun `@IsIn` au niveau HTTP**.

**Stockage** : Supabase Storage via **`fetch` REST** (`technician/supabase-storage.service.ts`), pas de SDK.

- `SUPABASE_URL` l. 40 · `SUPABASE_SERVICE_ROLE_KEY` l. 41
- `POST ${baseUrl}/storage/v1/object/${bucket}/${path}` l. 280, en-tête `x-upsert: 'false'` l. 284, timeout `UPLOAD_TIMEOUT_MS = 30_000` l. 25
- Bucket : `DEMANDE_BUCKET = 'relio-demande-medias'` l. 15 (**privé**)
- **Chemin** : `` `demandes/${userId}/${randomUUID()}-${name}` `` (`demande-media.service.ts:100`), `name` = `sanitizeFileName(file.originalname)` l. 56-60
- URL signées : TTL `DEMANDE_SIGNED_URL_TTL_SECONDS = 15 * 60` l. 20

| Contrainte | Valeur | Ligne |
|---|---|---|
| Taille max | `25 * 1024 * 1024` | `demande-media.service.ts:26` (appliqué l. 32 controller, l. 87-89 service, l. 43 DTO) |
| Nb max fichiers | `5` | l. 25 (service) · l. 24 (DTO) |
| `IMAGE_MIMES` | `image/jpeg`, `image/png`, `image/webp` | l. 31 |
| `VIDEO_MIMES` | `video/mp4`, `video/webm`, `video/quicktime` | l. 32 |
| `AUDIO_MIMES` | `audio/webm`, `audio/mp4`, `audio/mpeg`, `audio/ogg`, `audio/wav`, `audio/x-m4a` | l. 33-40 |

Validations service (l. 79-103) : fichier absent/vide → 400 · > 25 Mo → 400 · mime hors liste → 400 · `kind` ≠ mime → 400.

**Suppression d'orphelin** : `DemandeMediaService.deleteUploadedMedia(userId, storagePath)` l. 107-112 — vérifie le préfixe `demandes/${userId}/` l. 108-110, puis supprime. **Aucune ligne en base créée ou supprimée.**

**Association média ↔ demande** : le média est uploadé **avant** la création de la demande, donc stocké **sans `demandeId`** (`demande-media.service.ts:13-23, 77-78`). L'association se fait dans la transaction (`medias.create` l. 162-177). `DemandeMedia.demandeId` est **requis, `onDelete: Cascade`** (`schema.prisma:799-800`).

### c) Modèle `Demande` — `backend/prisma/schema.prisma:484-600`

| Champ | Type | L. | Nullabilité / défaut |
|---|---|---|---|
| `id` | `String` | 485 | `@id @default(dbgenerated("gen_random_uuid()"))` |
| `reference` | `String` | 486 | **`@unique`** |
| `status` | `DemandeStatus` | 487 | `@default(SUBMITTED)` |
| `category` | `String` | 488 | requis |
| `description` | `String?` | 493 | nullable |
| `equipmentType` | `String?` | 500 | nullable (toujours `null` en création) |
| `equipmentFamily` | `String?` | 506 | nullable |
| `city` | `String` | 511 | **requis** (snapshot texte) |
| `cityId` | `String?` | 512 | nullable |
| `zoneId` | `String?` | 519 | nullable |
| `neighborhood` `address` `landmark` `contactPhone` | `String?` | 521-524 | nullable |
| `latitude` `longitude` | `Float?` | 528-529 | nullable |
| `travelLatitude` `travelLongitude` `travelLocationUpdatedAt` `technicianEnRouteAt` `technicianArrivedAt` | `Float?`/`DateTime?` | 538-542 | nullable |
| **`clientId`** | `String` | **544** | **requis, non nullable** |
| `technicianId` | `String?` | 546 | nullable |
| `scheduledAt` | `DateTime?` | 547 | nullable |
| `requestedMode` | `RequestTiming` | 548 | `@default(ASAP)` |
| `requestedAt` | `DateTime?` | 549 | nullable |
| `domainId` `brandId` `modelId` `problemId` | `String?` | 553-559 | nullable, `onDelete: SetNull` |
| `negotiationRequestedAt` | `DateTime?` | 562 | nullable |
| `finalAmount` | `Int?` | 564 | nullable |
| `createdAt` | `DateTime` | 588 | `@default(now())` |
| `updatedAt` | `DateTime` | 589 | `@updatedAt` |

**Relations** (l. 543-587) : `client` `@relation("ClientDemandes", fields:[clientId], references:[id], onDelete: **Cascade**)` l. 543 · `technician` `onDelete: SetNull` l. 545 · `cityRef` `SetNull` l. 513 · `zoneRef` `SetNull` l. 520 · `domain/brand/model/problem` `SetNull` · `medias[]`, `aiWarnings[]`, `conversationFlags[]`, `classification?`, `messages[]`, `diagnostics[]`, `quotes[]`, `reviews[]`, `dispute?`, `events[]`, `notifications[]`, `dispatchWaves[]`, `financialTransactions[]`, `fundsHolds[]`, `rewardFraudFlag?`

**Index** : `@@index([clientId])` l. 591 · `[technicianId]` 592 · `[status]` 593 · `[zoneId]` 594 · `[domainId]` 595 · `[brandId]` 596 · `[modelId]` 597 · `[problemId]` 598 · `[equipmentFamily]` 599.

---

## 4. AUTHENTIFICATION — ÉTAT ACTUEL

### a) `AuthProvider`

`frontend/src/components/auth/auth-provider.tsx` — 60 lignes, `'use client'`.

- Chargement de `user` : `getMe()` → `GET ${NEXT_PUBLIC_API_URL}/auth/me` (`auth-service.ts:117-119`), `credentials: 'include'` l. 63. Appel unique au montage (`useEffect` l. 39-41) puis à chaque `refresh()` explicite (l. 27-37).
- États exposés (l. 49) : `{ user, authenticated: user !== null, loading, refresh, logout }`
- **Aucun état `error`.** Le `catch` l. 32-33 absorbe toute erreur et positionne `user = null` — le status HTTP (401/403/500) n'est ni conservé ni exposé.
- **Aucune gestion du 401** : pas de `router.replace`, pas de `logout()` automatique. La redirection provient du `RoleGuard`, à partir de `authenticated === false` — donc pour **toute** erreur de `GET /auth/me`.
- Le token n'est **jamais lu** par le frontend. Envoi : uniquement `credentials: 'include'` (l. 63), SSE l. 179 / `EventSource(…, {withCredentials:true})` l. 94, upload l. 604. **Aucun header `Authorization` dans tout `src/`.**
- `logout()` l. 43-45 → `logoutAndGoHome()` (`auth-service.ts:148-154`) : `POST /auth/logout` l. 145 puis `window.location.href = '/'` l. 152 dans un `finally`.
- `useAuth()` l. 56-60 ; lève une Error si hors provider l. 58. `AuthContext` **non exporté** (l. 21). 8 consommateurs : `app/client/layout.tsx:35`, `auth/role-guard.tsx:23`, `public/public-header.tsx:15`, `landing/hero.tsx:22`, `auth/verification-panel.tsx:47`, `client/client-auth-form.tsx:31`, `app/technicien/layout.tsx:38`, `app/admin/layout.tsx:58`.

### b) `RoleGuard`

`frontend/src/components/auth/role-guard.tsx` (48 l.) — délègue la décision à `frontend/src/lib/guard-decision.ts` (`decideGuard` l. 37-67).

| Cas | Comportement | Ligne |
|---|---|---|
| `loading === true` | `{action:'loading'}` → `<LoadingScreen />`, **aucune redirection** | `guard-decision.ts:39` · `role-guard.tsx:43-44` |
| Non authentifié + route publique | `{action:'show'}` | `guard-decision.ts:63` |
| Non authentifié + route protégée | redirect `${publicPaths[0]}?redirect=<pathname encodé>` via `router.replace` | l. 64-66 · `role-guard.tsx:37-41` |
| Authentifié, mauvais rôle | redirect `roleHomePath(role)` | l. 57-59 |
| Authentifié CLIENT `emailVerified === false` | redirect `/client/verification` (sauf si déjà sur cette route) | l. 44-46 |
| Authentifié + pathname public | redirect vers l'habitat du rôle | l. 57 |

**Utilisé uniquement par les 3 layouts** : `app/client/layout.tsx:87-90`, `app/technicien/layout.tsx:83-85`, `app/admin/layout.tsx:123-125`. **Aucune page `page.tsx` n'instancie son propre `RoleGuard`.**

`/admin` a `publicPaths={[]}` → non authentifié ⇒ `fallback = '/'` **sans** paramètre `redirect` (l. 64-65).

### c) Cookie d'auth

Posé **uniquement par le backend** : `backend/src/auth/auth.service.ts:561-570`, appel `res.cookie(...)` **l. 563**.

| Attribut | Valeur | Ligne |
|---|---|---|
| name | `repairdom_token` | `auth.types.ts:28` |
| `httpOnly` | `true` | 564 |
| `secure` | `this.isProduction` | 565 (défini l. 562, l. 96) |
| `sameSite` | `secure ? 'none' : 'lax'` | 566 |
| `path` | `'/'` | 567 |
| `maxAge` | `COOKIE_MAX_AGE_MS` = 7 j | 568 (`auth.types.ts:29`) |
| `domain` | **NON DÉFINI** | — |
| `expires` | **NON DÉFINI** | — |

**Posé** : `auth.controller.ts:44` (register TECHNICIAN) · **l. 53** (verify-email) · **l. 90** (login). **Non posé** au register d'un CLIENT (condition `if (user.emailVerified)` l. 43 ; `auth.service.ts:235` crée le CLIENT avec `emailVerified: false`).

**Lu** : `jwt-auth.guard.ts:18` — `request.cookies?.[COOKIE_NAME]` — **seul point de lecture dans tout le backend**. **Aucun header `Authorization` n'est lu.**

**Supprimé** : `auth.controller.ts:132` → `clearAuthCookie` l. 574 · `jwt-auth.guard.ts:29-34` (best-effort, `catch {}` silencieux).

`cookieParser()` monté l. 25. CORS `credentials: true` l. 44 ; en prod `CORS_ORIGINS` obligatoire sinon exception au démarrage l. 36-38.

### d) Middleware Next.js

**ABSENCE CONFIRMÉE.** Aucun `middleware.ts` ni `middleware.js` dans `/config/projet/frontend`. Vérifié par glob `**/middleware.{ts,js}`, `find -name "*middleware*"`, et `git ls-files | grep -i middleware` → seul résultat `src/lib/no-middleware.test.ts`. `next.config.ts` (11 l.) ne déclare ni `middleware`, ni `rewrites`, ni `redirects`, ni `headers`. `vercel.json` = `{"framework":"nextjs"}`.

Historique : `7bca936` l'introduit (« feat(auth): password reset pages and anti-flash middleware ») → `55bc2b0` le supprime (« Remove cross-domain middleware, harden RoleGuard with neutral loading screen ») → `0f9e9da` documente la règle + le test. Motif documenté `frontend/ARCHITECTURE.md:26-29`. Règle d'interdiction `ARCHITECTURE.md:33-38`.

`no-middleware.test.ts` (70 l.) : test 1 l. 20-32 (aucun `middleware.ts`/`.js` à la racine de `src/`) ; test 2 l. 34-70 (aucun fichier de `src/` — hors `*.test.ts` — ne contient `document.cookie`, `from 'next/headers'`, `require('next/headers')`).

---

## 5. ROUTES PUBLIQUES VS PROTÉGÉES

**Mécanisme : `RoleGuard` unique dans le layout `/client`.** 22 fichiers (21 `page.tsx` + 1 `layout.tsx`), aucun sous-layout.

| Route | Fichier | Statut | Mécanisme |
|---|---|---|---|
| `/client` | `page.tsx` | Protégée | layout l. 87 + `getMe()`+redir dans `use-client-dashboard-data.ts:71-74` |
| `/client/connexion` | `page.tsx` | **Publique** | `CLIENT_PUBLIC_PATHS` l. 28 |
| `/client/inscription` | `page.tsx` | **Publique** | l. 29 |
| `/client/verification` | `page.tsx` | **Publique** | l. 30 |
| `/client/demande` | `page.tsx` | Protégée | layout l. 87 seul (aucun garde dans la page ni dans le wizard) |
| `/client/demandes` | `page.tsx` | Protégée | layout + `use-client-dashboard-data.ts:71-74` |
| `/client/demandes/historique` | `page.tsx` | Protégée | idem |
| `/client/demandes/[id]` | `[id]/page.tsx` (931 l.) | Protégée | layout l. 87 seul |
| `/client/chronologies` | `page.tsx` | Protégée | layout |
| `/client/chronologies/[id]` | `[id]/page.tsx` | Protégée | layout |
| `/client/notifications` | `page.tsx` | Protégée | layout |
| `/client/parrainage` | `page.tsx` | Protégée | layout **+** garde propre l. 27-36 |
| `/client/profil` | `page.tsx` | Protégée | layout **+** garde propre l. 45-64 |
| `/client/recompenses` | `page.tsx` | Protégée | layout seul (commentaire l. 41-44) |
| `/client/solde` | `page.tsx` | Protégée | layout seul |
| `/client/solde/recharger` | `page.tsx` | Protégée | layout seul |
| `/client/solde/recharge/result` | `page.tsx` | Protégée | layout seul |
| `/client/confirmation` | `page.tsx` (117 l., serveur) | Protégée | layout seul (aucun appel réseau, aucune vérif. de session) |
| `/client/technicien/[id]` | `[id]/page.tsx` | Protégée | layout seul |

Comparaison : `/technicien` — 15 routes, `TECHNICIAN_PUBLIC_PATHS` l. 30. `/admin` — 24 routes, `publicPaths={[]}` l. 123. Hors ces 3 segments, 11 routes publiques sous le seul layout racine.

---

## 6. PATTERNS EXISTANTS DE STOCKAGE TEMPORAIRE

### Cookies

**Aucun usage frontend.** Zéro occurrence de `document.cookie`, `cookies()`, `next/headers`, `set-cookie` dans `frontend/src` (hors `no-middleware.test.ts` qui les interdit l. 56-58). **Aucun `sessionStorage`** dans tout `frontend/src`.

### localStorage — 2 emplacements, centralisés

| Fichier | Clé | Opérations | Usage |
|---|---|---|---|
| `src/lib/push/push-client.ts:56-77` | `relio-push-endpoint` | `getItem` l. 60 · `setItem` l. 72 · `removeItem` l. 73 | endpoint Web Push ; wrapper avec `storage?: Storage \| null` injectable l. 59, 69 |
| `src/lib/technician/use-onboarding-banner-visibility.ts:18,34,46` | `ONBOARDING_BANNER_STORAGE_KEY` (**valeur littérale À VÉRIFIER** — constante déclarée dans `onboarding-banner.tsx`) | `getItem` l. 34 · `setItem(…, 'true')` l. 46 | dismiss bannière onboarding technicien |

**Aucun helper centralisé** de type `src/lib/storage.ts` ; aucun accès inline dans un composant.

### Tokens temporaires backend

| Ressource | Champs | Lignes | Pattern de cleanup |
|---|---|---|---|
| `User.emailVerificationToken` / `…ExpiresAt` | `String?` / `DateTime?` | 266-267 | **consommé et mis à `null`** à la vérification (`auth.service.ts:319-322`) ; **aucune purge par expiration** |
| `User.passwordResetToken` / `…ExpiresAt` | `String?` / `DateTime?` | 273-274 | consommé à `null` (`auth.service.ts:445-453`) ; aucune purge |
| `User.tokenVersion` | `Int @default(0)` | 275 | révocation de session |
| `TopupIntent.idempotencyKey` | `String @unique` | **1501** | **recherche par clé AVANT création** (`financial.service.ts:1777-1781`, re-lecture l. 1815) ; **aucune purge** |
| `WithdrawalRequest.idempotencyKey` | `String @unique` | **1547** | identique (`financial.service.ts:2240-2244`, 2255, 2305) ; aucune purge |
| `RelioWithdrawal.idempotencyKey` | `String? @unique` | **1470** | aucune purge |
| `DispatchWave` | `@@unique([demandeId, wave, userId, channel])` | 671 | idempotence par contrainte unique ; aucune purge |
| `FundsHold.reference` / `FinancialTransaction.reference` | `@unique` | 1592 / 1419 | idempotence ; aucune purge |

Champs `otp`, `attempts`, `consumed`, `usedAt`, `expiresAt` (nom exact) : **AUCUN** dans `schema.prisma`. Le seul `code` est `EquipmentFamily.code @unique` l. 1194 (pas un token).

### Cleanup paresseux — `PasswordResetAttempt` (pattern de référence)

`schema.prisma:775-783` : `id`, `email String`, `ip String?`, `createdAt DateTime @default(now())` · `@@index([email, createdAt])` l. 781 · `@@index([ip, createdAt])` l. 782. Commentaire l. 774 : « aucune donnée sensible (ni token, ni mot de passe) ».

Pattern **LAZY CLEANUP à chaque demande, pas de cron** (`auth.service.ts:360-366`) :

```
await this.prisma.passwordResetAttempt.deleteMany({
  where: { createdAt: { lt: new Date(Date.now() - PASSWORD_RESET_ATTEMPT_RETENTION_MS) } },
}).catch(() => undefined);
```

Rétention `24 h` l. 61 · seuils `3/email/h` l. 56 et `10/IP/h` l. 57 → 429 l. 381-384.

**Aucun `@Cron` / `@nestjs/schedule` dans le dépôt.** Seul `setInterval` : `dispatch.scheduler.ts:27` (60 000 ms).

---

## 7. SCHÉMA PRISMA — MODÈLES PERTINENTS

### `User` — l. 250-329

| Champ | Type | L. | Attributs |
|---|---|---|---|
| `id` | `String` | 251 | `@id @default(dbgenerated("gen_random_uuid()"))` |
| `role` | `Role` | 252 | non null, **`@default(CLIENT)`** |
| `firstName` | `String` | 253 | **requis** |
| `lastName` `phone` | `String?` | 254-255 | nullable |
| `city` | `String?` | 258 | nullable (texte libre) |
| `cityId` | `String?` | 259 | nullable |
| `address` `whatsapp` `avatarUrl` | `String?` | 261-263 | nullable |
| `email` | `String` | 264 | **`@unique`** |
| `emailVerified` | `Boolean` | 265 | **`@default(true)`** |
| `emailVerificationToken` / `…ExpiresAt` | `String?` / `DateTime?` | 266-267 | nullable |
| `passwordHash` | `String` | 268 | **requis** |
| `passwordResetToken` / `…ExpiresAt` | `String?` / `DateTime?` | 273-274 | nullable |
| `tokenVersion` | `Int` | 275 | `@default(0)` |
| `isActive` | `Boolean` | 280 | `@default(true)` |
| `createdAt` / `updatedAt` | `DateTime` | 281-282 | `@default(now())` / `@updatedAt` |

Relations l. 260, 283-328 : `cityRef` `ServiceCity?` `@relation("CityClients", onDelete: SetNull)` · `demandes Demande[] @relation("ClientDemandes")` · `technicianDemandes Demande[] @relation("TechnicianDemandes")` · `technicianProfile TechnicianProfile?` · `kycDocuments` · `kycReviewsMade` / `kycReviews` · `messages` · `diagnostics` · `quotes` · `aiWarnings` ×2 · `conversationFlagsSent` / `conversationFlagReviews` · `reviewsAuthored` / `reviewsReceived` · `disputesOpened` / `disputesDecided` · `pricingHistories` · `demandeEvents` · `notifications` · `dispatchWaves` · `financialTransactions` ×2 · `relioWithdrawals` · `topupIntents` ×2 · `withdrawalRequests` ×2 · `fundsHolds` ×2 · `pushSubscriptions` · `clientRewardProgress ClientRewardProgress?` · `rewardFraudFlags` ×2

**Index** : `@unique` sur `email` (264), `@id` sur `id` (251). **Aucun `@@index`, aucun `@@unique` composé.**

### `Demande` — l. 484-600

Voir le tableau complet en § 3.c.

### `DemandeMedia` — l. 785-804

| Champ | Type | L. | Attributs |
|---|---|---|---|
| `id` | `String` | 786 | `@id @default(dbgenerated("gen_random_uuid()"))` |
| `kind` | `MediaKind` | 787 | requis (`IMAGE, VIDEO, AUDIO` l. 121-125) |
| `fileName` | `String` | 788 | requis |
| `mimeType` | `String` | 789 | requis |
| `sizeBytes` | `Int` | 790 | requis |
| `stored` | `Boolean` | 791 | `@default(false)` |
| `url` | `String?` | 792 | nullable (jamais utilisé par le flux actuel) |
| `storagePath` | `String?` | 798 | nullable (bucket privé) |
| **`demandeId`** | `String` | **800** | **requis, non nullable**, `onDelete: Cascade` l. 799 |
| `createdAt` | `DateTime` | 801 | `@default(now())` |

Index : `@@index([demandeId])` l. 803. Aucun `@unique`, aucun index sur `storagePath`.

### `ServiceCity` — l. 422-438

`id` `String` 423 `@id @default(dbgenerated("gen_random_uuid()"))` · `name String` 424 requis · `slug String` 425 **`@unique`** · `isActive Boolean` 426 `@default(true)` · `sortOrder Int` 427 `@default(0)` · `createdAt`/`updatedAt` 428-429 · Relations l. 434-437 : `zones Zone[] @relation("CityZones")`, `clients User[] @relation("CityClients")`, `technicians TechnicianProfile[] @relation("CityTechnicians")`, `demandes Demande[] @relation("CityDemandes")`. **Aucun `@@index`.**

### `Zone` — l. 444-464

`id` 445 `@id @default(dbgenerated("gen_random_uuid()"))` · **`cityId String` 446 requis** · `name String` 447 requis · `slug String` 448 requis (pas `@unique` seul) · `isActive Boolean` 449 `@default(true)` · `sortOrder Int` 450 `@default(0)` · `createdAt`/`updatedAt` 451-452 · `city ServiceCity` 454 `@relation("CityZones", onDelete: **Cascade**)` · `coverages TechnicianZoneCoverage[]` 455 · `demandes Demande[] @relation("ZoneDemandes")` 459.

Index : **`@@unique([cityId, slug])` l. 461** · `@@index([cityId])` l. 462 · `@@index([isActive])` l. 463.

### `PasswordResetAttempt` — l. 775-783

`id` 776 `@id @default(dbgenerated("gen_random_uuid()"))` · `email String` 777 requis · `ip String?` 778 nullable · `createdAt DateTime` 779 `@default(now())` · `@@index([email, createdAt])` l. 781 · `@@index([ip, createdAt])` l. 782. Aucun `@unique`, **aucun champ `token`**.

---

## 8. TESTS EXISTENTS

### Inscription client

| Fichier | Ce qui est testé | Statut |
|---|---|---|
| `backend/src/auth/auth.spec.ts` | register l. 102-145 · login l. 147-190 · verifyEmail/resend l. 192 · JWT + cookie l. 221-247 · guards l. 250-283 | Existe |
| `backend/src/auth/register-technician.spec.ts` | register TECHNICIAN l. 104, l. 192 ; **register CLIENT — comportement inchangé** l. 223 | Existe |
| `backend/src/auth/password-reset-flow.spec.ts` | flow HTTP login → reset → login → dashboard l. 151 | Existe |
| `backend/src/auth/password-reset.spec.ts` | `requestPasswordReset` l. 109 · `resetPassword` l. 163 · `validateResetToken` + invalidation JWT l. 225 | Existe |
| `backend/src/auth/email-templates.spec.ts` | `buildVerificationEmail` l. 22 · `buildMissionAvailableEmail` l. 58 | Existe |
| `backend/test/auth-demandes.e2e-spec.ts` | register + cookie l. 44-63 · doublon 409 · `me` · `me` sans cookie 401 · login+logout l. 122-132 | Existe — **contredit `auth.controller.ts:43`, non exécuté** |

### Création de demande

| Fichier | Ce qui est testé | Statut |
|---|---|---|
| `backend/src/demandes/demande-multimedia.spec.ts` | DTO description 10-1000 l. 38 · DTO médias l. 88 · `kindForMimeType` l. 118 · **`create` — multimédia sans texte** l. 165 · `toApiDemandePublic` l. 205 · **`uploadMedia` validation** l. 254 · **`getMediaFileUrl` permissions** l. 293 | Existe |
| `backend/src/demandes/demande-equipment.spec.ts` | DTO `equipmentFamily` l. 90 · **`create` — indice structuré** l. 101 · `toApiDemande` l. 182 | Existe |
| `backend/src/demandes/demande-gps.spec.ts` | DTO GPS l. 58 · sérialiseurs l. 84 | Existe |
| `backend/src/demandes/demandes-priority.spec.ts` | `compareDemandePriority` l. 13 | Existe |
| `backend/src/demandes/demandes-rewards-wiring.spec.ts` | `updateStatus` — câblage récompenses l. 109 | Existe |
| `backend/test/auth-demandes.e2e-spec.ts` | POST/GET `/demandes` l. 74-99 · 401 sans auth l. 107 · DTO invalide 400 l. 114 | Existe |

### Wizard (frontend)

**Aucun test ne rend ni ne monte le composant.** Tous lisent le source par `readFileSync` + `assert.match`. Aucune lib de rendu dans `package.json` (devDeps : tailwind, oxlint, typescript, @types/*).

| Fichier | Ce qui est testé | Statut |
|---|---|---|
| `frontend/src/lib/demande-equipment.test.ts` | l. 17-27 parcours Autre (chaînes attendues) · l. 29-34 liste sans « Toutes les marques » · l. 36-42 service | Statique |
| `frontend/src/lib/demande-media.test.ts` | l. 17-24 description ≥ 10 · l. 26-39 parcours simplifié · **l. 41-47 upload AVANT création + cleanup à l'échec** (seul contrôle d'ordre) · l. 49-57 limites 5/25 Mo/180 s | Statique |
| `frontend/src/lib/gps-helpers.test.ts` | l. 24-72 **exécution réelle** de `travel-location.ts` · l. 74-89 absence de tracking · l. 95-109 helper unique · l. 111-126 `formatTravelAccuracyShort` · l. 128-158 absence de logs | Mixte |
| `frontend/src/lib/navigation.test.ts` | l. 97-104 présence de `/client/confirmation` dans le wizard · l. 123-136 `RoleGuard` dans les 3 layouts privés | Statique |
| `frontend/src/lib/responsive.test.ts` | l. 66-92 parité des destinations · l. 83-85 `/client/demande` dans les 2 vues | Statique |

**Non couvert** : `canContinue` / `disabled` du bouton, messages des 3 gardes de `handleSubmit`, payloads de `createDemande`, redirection + ses 4 query params, `goNext`/`goBack`/`jumpTo`, l'étape 2 entière, l'étape 3, les 4 `useEffect`, `beforeunload`, l'exactitude de `STEPS`.

### RoleGuard / guards

| Fichier | Ce qui est testé | Statut |
|---|---|---|
| `frontend/src/lib/role-guard.test.ts` | `decideGuard` : loading l. 49 · non authentifié l. 64 · mauvais rôle l. 79 · `LoadingScreen` sans texte ni lien l. 94-106 | Existe (unitaire pur) |
| `frontend/src/lib/no-middleware.test.ts` | absence de `middleware.ts` l. 20-32 · `repairdom_token` jamais lu l. 34-70 | Existe |
| `frontend/src/lib/password-reset.test.ts` | mentions `RoleGuard` l. 70, l. 85 | Existe |
| `backend/src/auth/auth.spec.ts` | `JwtAuthGuard` + `RolesGuard` + métadonnées `@Roles` par controller + présence des handlers l. 250-283 ; `signToken`/`verifyToken`, falsification, compte désactivé, `setAuthCookie`/`clearAuthCookie` l. 221-247 | Existe |
| `backend/src/disputes/disputes.spec.ts` | routes — guards et rôles l. 403 | Existe |
| `backend/src/push/push.spec.ts` | auth et validation du controller l. 172 | Existe |
| `backend/test/push.module.e2e-spec.ts` | `POST /api/push/subscribe` sans cookie → 401 l. 38 | Existe |
| `backend/test/realtime.module.e2e-spec.ts` | `GET /api/realtime/user` sans cookie → 401 l. 39 | Existe |

**Absences** : aucun `*.spec.ts` pour `demandes.controller.ts` isolément · `demande-media.service.ts` (tests dans `demande-multimedia.spec.ts`) · `email.service.ts` · `env.validation.ts` · `main.ts` · `prisma.service.ts` · `GET /api/cities` · aucun test de composant pour `RoleGuard`, `AuthProvider`, `ClientAuthForm`, `VerificationPanel`, `DemandeWizard`.

---

## 9. FRICTIONS IDENTIFIÉES (FACTUELLES)

### Champs obligatoires dans le wizard (ordre et nombre)

| Ordre | Étape | Champ | Règle | Ligne |
|---|---|---|---|---|
| 1 | 0 | domaine | `domainId !== ''` | 204 |
| 2a | 0 | **marque** (si domaine catalogue) | `brandId !== ''` | 206 |
| 2b | 0 | **famille** (si `OTHER_DOMAIN`) | `equipmentFamily.trim() !== ''` | 205 |
| 3 | 1 | description | `trim().length >= 10` · `maxLength 1000` | 209 · 613 |
| 4 | 2 | ville | `city.trim() !== ''` | 210 |
| 5 | 2 | date | `requestedMode === 'ASAP' \|\| requestedAt !== ''` | 210 |

**5 champs bloquants, dont 2 alternatifs (marque OU famille).** Non requis : quartier, adresse précise, point de repère, téléphone, GPS, médias (0 à 5).

Côté inscription, 6 champs bloquants : `firstName`, `lastName`, `email`, `password` (+ `city`, `address` exigés côté service `auth.service.ts:177-182` bien que `city`/`address` soient `@IsOptional` dans le DTO l. 73-76 / 37-40).

### Points d'abandon

| Point | Constat | Ligne |
|---|---|---|
| **CTA landing → wizard** | Non connecté ⇒ clic sur `Décrire ma panne` ⇒ redirection immédiate vers `/client/connexion` | `guard-decision.ts:61-66` |
| **`LoadingScreen` sans sortie** | Aucune redirection pendant `loading` ; aucun texte, bouton ni lien pendant le chargement de `GET /auth/me` | `role-guard.tsx:43-44` · `loading-screen.tsx:10-15` |
| **Redirection possible pendant le wizard** | Un `getMe()` en échec (réseau, expiration de cookie) ⇒ `authenticated=false` ⇒ `RoleGuard` redirige vers `/client/connexion?redirect=%2Fclient%2Fdemande`, **state détruit** | `demande-wizard.tsx:163` + `guard-decision.ts:61-66` |
| **Erreur de catalogue silencieuse** | `listCatalogDomains` en échec ⇒ `setDomains([])` → écran `Catalogue indisponible` ; `listEquipmentFamilies` en échec ⇒ `.catch(() => undefined)` **sans message** | l. 136, l. 143 |
| **Erreur `getMe` silencieuse** | `.catch(() => undefined)` — aucun feedback | l. 163 |
| **Erreur `listCatalogBrands` silencieuse** | `.catch(() => setBrands([]))` — aucun message | l. 229 |
| **Erreur suppression d'upload** | `deleteUploadedDemandeMedia` en échec ⇒ `.catch` vide l. 440-442 — fichiers orphelins non signalés | l. 437-443 |
| **Erreur `/auth/me` dans l'AuthProvider** | `catch` l. 32-33 sans état `error` — ni code HTTP ni message exposés | `auth-provider.tsx:32-33` |
| **Double-clic sur `Envoyer la demande`** | `isSubmitting` n'est **pas** réinitialisé après succès (uniquement dans le `catch` l. 444) ; pas d'idempotence serveur | l. 444 · `request-service.ts:152` |
| **`beforeunload` ne couvre pas la navigation interne** | Le lien `Retour` du `PageHeader` (`backHref="/client/demandes"`) quitte le wizard sans avertissement | `demande/page.tsx:15` |

### État non sauvegardé (perte au refresh)

| Ressource | Persisté ? |
|---|---|
| `step`, `domainId`, `brandId`, `equipmentFamily`, `description`, `city`, `neighborhood`, `address`, `landmark`, `contactPhone`, `requestedMode`, `requestedAt`, `medias`, `coords` | **NON** — `useState` uniquement, aucune écriture `localStorage`/`sessionStorage`/cookie/backend |
| URL d'étape | **NON** — aucun `?step=`, aucun `pushState`, aucun `popstate` |
| Scroll entre étapes | **NON** — aucun `scrollTo`/`scrollIntoView` |
| Préremplissage ville/téléphone | **CONDITIONNEL** — `getMe()` n'est appelé qu'au montage ; si le wizard est démonté/remonté, le préremplissage est recalculé | l. 156-168 |
| Octets des médias | **NON** — objets `File` en mémoire ; URLs révocatées au démontage l. 243-250 |
| Écran de reprise | **ABSENT** |

### Points exigeant un cookie valide

| Point | Ligne |
|---|---|
| `JwtAuthGuard` lit **exclusivement** `request.cookies?.['repairdom_token']` — aucun header `Authorization` n'est lu → impossible de créer une demande sans cookie | `jwt-auth.guard.ts:18` |
| `POST /demandes` exige `JwtAuthGuard` + `@Roles('CLIENT')` au niveau **classe** | `demandes.controller.ts:16-17` |
| `POST /demandes/medias/upload` et `DELETE /demandes/medias/upload` — mêmes guards de classe | l. 16-17 |
| Le `storagePath` retourné est rangé sous `demandes/${userId}/…` → la suppression d'orphelin vérifie le préfixe `demandes/${userId}/` : un média uploadé par un compte ne peut pas être nettoyé par un autre | `demande-media.service.ts:100, 108-110` |
| Cookie posé au register CLIENT : **conditionnel** `if (user.emailVerified)` → **jamais** pour un CLIENT | `auth.controller.ts:43` + `auth.service.ts:235` |

### Erreurs silencieuses

| Emplacement | Comportement | Ligne |
|---|---|---|
| `AuthProvider.refresh` | `catch { setUser(null) }` — statut HTTP perdu | `auth-provider.tsx:32-33` |
| `handleSubmit` guards de ville/date | Non revérifiés | `demande-wizard.tsx:370-383` |
| `listEquipmentFamilies` | `.catch(() => undefined)` | l. 143 |
| `getMe` (wizard) | `.catch(() => undefined)` | l. 163 |
| `listCities` (inscription) | `.catch(() => {})` — le `Select` ville reste vide sans message | `client-auth-form.tsx:54` |
| `logout` après verify-email | échec ignoré | `verification-panel.tsx:70-72` |
| `getClientFinanceSummary` | échec silencieux | `use-client-dashboard-data.ts:90` |
| `clearAuthCookie` dans le guard | `catch {}` | `jwt-auth.guard.ts:32-34` |
| Nettoyage d'uploads à l'échec | `catch {}` | `demande-wizard.tsx:440-442` |
| 409 au register | redirection **sans message d'erreur** | `client-auth-form.tsx:144-148` |
| `EMAIL_VERIFICATION_REQUIRED` au login | redirection **sans message d'erreur** | `client-auth-form.tsx:151-155` |
| Erreur upload après `setError` | `setUploadStatus(null)` l. 434 — le message d'erreur HTTP de l'upload est écrasé par le message générique l. 433 | l. 433-434 |

### Points factuels supplémentaires

| Constat | Emplacement |
|---|---|
| `mediaPersisted: false` et `storageStatus: 'metadata-only'` sont des **valeurs constantes**, pas un état calculé | `demande-helpers.ts:166-167` |
| `jumpTo` n'borne pas la valeur cible | `demande-wizard.tsx:341-344` |
| `goNext` ne revalide rien | l. 331-334 |
| `Envoyer la demande` n'a pas de `disabled` explicite | l. 963 |
| `minRequestedAt` calculé **une seule fois** au montage (`useMemo` deps `[]`) | l. 170 |
| Aucun `abort`/`cancel` sur `getMe` — un appel lent prolonge le préremplissage | l. 156-168 |
| Les 6 tuiles services de la landing mènent toutes à `/client/demande` sans transmettre la catégorie | `landing-sections.tsx:186` |
| `/admin` : `publicPaths={[]}` ⇒ redirection vers `/` **sans** paramètre `redirect` | `guard-decision.ts:64-65` |

---

## 10. FICHIERS CLÉS

| # | Fichier | Rôle |
|---|---|---|
| 1 | `frontend/src/app/client/layout.tsx` (115 l.) | Pose `RoleGuard CLIENT` + `CLIENT_PUBLIC_PATHS` sur **toutes** les routes `/client/*` |
| 2 | `frontend/src/lib/guard-decision.ts` (67 l.) | Logique pure de décision : loading, public, non authentifié, mauvais rôle, email non vérifié |
| 3 | `frontend/src/components/auth/role-guard.tsx` (48 l.) | Wrapper client + `LoadingScreen` + `router.replace` |
| 4 | `frontend/src/components/client/demande-wizard.tsx` (1216 l.) | Wizard complet : 4 étapes, 27 `useState`, upload, soumission, redirection |
| 5 | `backend/src/demandes/demandes.controller.ts` | `POST /demandes` l. 64-67 + 3 routes médias ; guards de classe l. 16-17 |
| 6 | `backend/src/demandes/demandes.service.ts` | `create()` l. 86-215 : validations, transaction l. 137-191, dispatch l. 196-204 ; `updateStatus` l. 410-549 |
| 7 | `frontend/src/components/client/client-auth-form.tsx` (438 l.) | Formulaire d'inscription 2 étapes + connexion ; payload l. 112-122 ; redirections l. 123-158 |
| 8 | `frontend/src/lib/api/request-service.ts` | `createDemande` l. 150-192 · `uploadDemandeMedia` l. 235-243 · `deleteUploadedDemandeMedia` l. 246-252 |
| 9 | `backend/src/demandes/dto/create-demande.dto.ts` (187 l.) | Contrat d'entrée complet + `RequestMediaDto` |
| 10 | `frontend/src/components/auth/auth-provider.tsx` (60 l.) | `user`/`authenticated`/`loading`/`refresh`/`logout` ; `GET /auth/me` |
| 11 | `backend/src/auth/auth.controller.ts` | `register` l. 36-47 (cookie conditionnel l. 43) · `login` l. 86-91 · `verify-email` l. 49-55 · `me` l. 94-97 |
| 12 | `backend/src/auth/auth.service.ts` | `register` l. 153-271 · `setAuthCookie` l. 561-570 · `clearAuthCookie` l. 572-580 · `verifyToken` l. 536-559 |
| 13 | `backend/src/auth/jwt-auth.guard.ts` (38 l.) | **Lecture du cookie uniquement** l. 18 ; 401 l. 19 et l. 35 |
| 14 | `backend/src/demandes/demande-media.service.ts` | Validations upload l. 79-103 · chemin `demandes/${userId}/…` l. 100 · suppression orphelin l. 107-112 |
| 15 | `frontend/src/lib/api/auth-service.ts` | `apiFetch` (`credentials:'include'`) l. 61-69 · `signUp` l. 71-105 · `signIn` l. 107-115 · `getMe` l. 117-119 · `safeRedirect` l. 127-142 |

---

## 11. ENDPOINTS ACTUELS DU FLUX

Préfixe `/api` (`main.ts:16`). « Auth » = `JwtAuthGuard` (cookie `repairdom_token` uniquement).

| Méthode | Chemin | Auth | Rôle | DTO | Retour |
|---|---|---|---|---|---|
| POST | `/api/auth/register` | Non | — (public) | `RegisterDto` | 201 `{ user, mode:'real' }` — cookie **seulement si** `emailVerified` |
| POST | `/api/auth/login` | Non | — (public) | `LoginDto` | 200 `{ user, mode:'real' }` + cookie |
| POST | `/api/auth/logout` | Non | — (public) | — | 200 `{ success:true, mode:'real' }` + clearCookie |
| GET | `/api/auth/me` | **Oui** | tout | — | `AuthUser` (13 champs) ; 401 / 403 si `isActive=false` |
| PATCH | `/api/auth/me` | **Oui** | tout | `UpdateMeDto` | 200 `AuthUser` |
| POST | `/api/auth/me/avatar` | **Oui** | tout | multipart | 200 (avatar) |
| POST | `/api/auth/verify-email` | Non | — (public) | `VerifyEmailDto{token}` | 200 `{user,mode}` + **cookie** ; 400 `EMAIL_INVALID_OR_EXPIRED` / `EMAIL_ALREADY_VERIFIED` |
| POST | `/api/auth/resend-verification` | Non | — (public) | `ResendVerificationDto{email}` | 200 `{ ok:true }` (réponse constante) |
| GET | `/api/auth/reset-password/:token/validate` | Non | — (public) | — | `{ valid }` |
| POST | `/api/auth/forgot-password` | Non | — (public) | — | 200 / 429 (3/h email, 10/h IP) |
| POST | `/api/auth/reset-password` | Non | — (public) | token + newPassword | 200 |
| GET | `/api/cities` | **Non** | — (**public**) | — | `[{ id, name, zones:[{id,name}] }]` |
| GET | `/api/catalog/domains` | **Oui** | CLIENT, TECHNICIAN, ADMIN | — | domaines |
| GET | `/api/catalog/domains/:id/brands` | **Oui** | CLIENT, TECHNICIAN, ADMIN | — | marques |
| GET | `/api/catalog/families` | **Oui** | CLIENT, TECHNICIAN, ADMIN | — | familles |
| **POST** | **`/api/demandes`** | **Oui** | **CLIENT** | **`CreateDemandeDto`** | **201** `toApiDemande` (31 champs) |
| POST | `/api/demandes/medias/upload` | **Oui** | CLIENT | multipart `file` + `kind?` (**aucun DTO**) | 201 `{ storagePath, kind, name, mimeType, sizeBytes }` ; 400 / 413 |
| DELETE | `/api/demandes/medias/upload` | **Oui** | CLIENT | body `{ storagePath }` | 200 ; 400 si préfixe non propriétaire |
| GET | `/api/demandes` | **Oui** | CLIENT | — | `Demande[]` (`clientId` filtré, `notIn [CONFIRMED,CANCELED]`, take 100) |
| GET | `/api/demandes/my/history` | **Oui** | CLIENT | — | `Demande[]` (`in [CONFIRMED,CANCELED]`) |
| GET | `/api/demandes/:id` | **Oui** | CLIENT | — | `Demande` ; **404** si `clientId ≠ user.id` |
| GET | `/api/demandes/:id/medias/:mediaId/file` | **Oui** | CLIENT | — | `{ url }` signée (TTL 15 min) ; 404 |
| PATCH | `/api/demandes/:id/status` | **Oui** | CLIENT | `UpdateDemandeStatusDto{status}` | 200 `Demande` ; 404 / 400 transition / 409 concurrence ou règlement bloqué |
| GET | `/api/demandes/:id/dispute` | **Oui** | CLIENT | — | litige ; 404 si non partie |
| GET | `/api/demandes/chronologies/mine` | **Oui** | CLIENT, TECHNICIAN | — | chronologies |
| — | **`DELETE /api/demandes/:id`** | — | — | — | **ABSENT** |
| — | **`PATCH /api/demandes/:id`** (édition du contenu) | — | — | — | **ABSENT** |

**Annulation** : uniquement via `PATCH /api/demandes/:id/status` avec `status:'CANCELED'`. Transitions client autorisées (`demandes-lifecycle.ts:16-22`) : `SUBMITTED→CANCELED`, `PENDING→CANCELED`, `ACCEPTED→CANCELED`, `SCHEDULED→CANCELED`, `COMPLETED→CONFIRMED`.

---

## 12. QUESTIONS OUVERTES POUR LA REFONTE

### 12.1 Guards backend

| Guard | Emplacement | Effet sur un brouillon non authentifié |
|---|---|---|
| `JwtAuthGuard` | `demandes.controller.ts:16` (classe) · définit l. 18-19 | `POST /demandes` → **401** sans cookie |
| `JwtAuthGuard` | `demandes.controller.ts:16` (classe) · définit l. 18-19 | `POST /demandes/medias/upload` → **401** |
| `JwtAuthGuard` | `demandes.controller.ts:16` (classe) | `DELETE /demandes/medias/upload` → **401** |
| `RolesGuard` + `@Roles('CLIENT')` | `demandes.controller.ts:16-17` (classe) | s'applique à **toutes** les routes du controller, pas seulement à `create` |
| `JwtAuthGuard` | `auth.controller.ts:95` | `GET /auth/me` → 401 ⇒ `AuthProvider.user = null` sans message |
| `JwtAuthGuard` | `collaboration.controller.ts:17-18`, `reviews.controller.ts:11` | routes `/demandes/:id/…` → 401 |

**Aucun mécanisme « route publique conditionnelle » n'existe** : il n'existe **aucun décorateur `@Public`** dans tout `src/**/*.controller.ts` (0 occurrence). Le caractère public d'une route n'est obtenu que par l'absence de `@UseGuards` au niveau classe/méthode.

### 12.2 Guards frontend

| Point | Emplacement | Effet |
|---|---|---|
| `RoleGuard expectedRole="CLIENT"` | `client/layout.tsx:87-90` | enveloppe **toutes** les routes `/client/*`, y compris `/client/demande` |
| `decideGuard` — non authentifié + non publique | `guard-decision.ts:61-66` | `redirect → /client/connexion?redirect=%2Fclient%2Fdemande` |
| `decideGuard` — `loading` | `guard-decision.ts:39` | `LoadingScreen` (aucune sortie) |
| `decideGuard` — CLIENT `emailVerified === false` | `guard-decision.ts:44-46` | bloque **toutes** les routes `/client/*` sauf `/client/verification` |
| `decideGuard` — mauvais rôle | `guard-decision.ts:57-59` | `redirect → roleHomePath(role)` |
| `decideGuard` — chemin public + authentifié | `guard-decision.ts:57` | redirige vers l'habitat du rôle |
| `AuthProvider.refresh` catch | `auth-provider.tsx:32-33` | toute erreur ⇒ `authenticated = false` ⇒ triggering la redirection ci-dessus |

**Le seul mécanisme de dérogation existant est `CLIENT_PUBLIC_PATHS`** (`layout.tsx:27-31`), une liste de 3 chaînes exactes comparées par `publicPaths.includes(pathname)` (`guard-decision.ts:41`). Aucun motif/glob, aucune liste par rôle, aucune route « semi-privée ».

### 12.3 Redirections dans le wizard et la page

| Emplacement | Comportement |
|---|---|
| `demande-wizard.tsx:156-168` | `getMe()` au montage ; `.catch(() => undefined)` l. 163 — **ne redirige pas** mais laisse le préremplissage vide |
| `demande-wizard.tsx:427-431` | unique `router.push` (succès) |
| `use-client-dashboard-data.ts:71-74` | `getMe()` puis `router.replace(homePathForRole(me.role))` si rôle ≠ CLIENT |

### 12.4 Structure du wizard

| Élément | État actuel |
|---|---|
| `step` | `useState(0)` l. 89 — non persisté, absent de l'URL |
| Champs | 13 `useState` non persistés |
| Médias | objets `File` en mémoire ; `URL.createObjectURL` révocatés au démontage l. 243-250 |
| Preuve d'effort | `hasStarted` l. 450-458 (volatil) + `beforeunload` l. 459-466 |
| Idempotence | **ABSENTE** — ni header, ni champ, ni `useStableIdempotencyKey` |
| Nettoyage | `deleteUploadedDemandeMedia` best-effort l. 437-443, `catch {}` l. 440-442 |
| Erreur unique | un seul `Alert` global l. 488 ; `mediaError` propagé aux 3 champs médias |

### 12.5 Modèle `Demande` — contraintes exigeant un `clientId`

| Contrainte | Emplacement | Conséquence factuelle |
|---|---|---|
| **`clientId String` — requis, non nullable** | `schema.prisma:544` | `tx.demande.create` ne peut pas être exécuté sans `clientId` (`demandes.service.ts:155`, alimenté par `user.id` via `@CurrentUser()` l. 66) |
| **`client User @relation(..., onDelete: Cascade)`** | `schema.prisma:543` | cascade destructive sur suppression de compte |
| **`@@index([clientId])`** | `schema.prisma:591` | index non-null |
| **Filtrage par `clientId` dans la requête** | `demandes.service.ts:368-371`, `381-384`, `393-394` | toutes les lectures client exigent `clientId` dans le `where` |
| **`userId` dans le chemin de stockage** | `demande-media.service.ts:100` | `demandes/${userId}/…` — un média uploadé par un compte ne peut être supprimé que par ce compte l. 108-110 |
| **`@CurrentUser() user` en paramètre obligatoire** | `demandes.controller.ts:65` | signature du handler |
| `reference @unique` | `schema.prisma:486` | génération avec retry l. 134-208 |
| **`DemandeMedia.demandeId` requis** | `schema.prisma:800` | aucun média ne peut exister sans demande |

**Contraintes jointes au modèle `User`** pour un rattachement ultérieur : `email @unique` l. 264 · `role @default(CLIENT)` l. 252 · `emailVerified @default(true)` l. 265 · `emailVerificationToken`/`ExpiresAt` l. 266-267.

---

### Points marqués « À VÉRIFIER »

1. `test/auth-demandes.e2e-spec.ts:52-55` exige un cookie après `POST /api/auth/register` pour un CLIENT, ce qui contredit `auth.controller.ts:43`. Test non exécuté (audit lecture seule).
2. `ONBOARDING_BANNER_STORAGE_KEY` — valeur littérale non relevée (déclarée dans `onboarding-banner.tsx`).
3. Attributs `domain` / `expires` du cookie : confirmés **absents** de `auth.service.ts:561-580`. La compatibilité cross-site effective de `SameSite=None` **sans `Domain`** n'est pas vérifiable depuis le code seul.
4. `mediaPersisted: false` et `storageStatus: 'metadata-only'` (`demande-helpers.ts:166-167`) : valeurs constantes, aucun chemin de code ne les recalcule.
5. La liste des « autres guards customs » repose sur `**/*.guard.ts` + grep `@UseGuards` : seuls `jwt-auth.guard.ts`, `roles.guard.ts`, `tracking-throttle.guard.ts` existent.
6. Aucun composant `MediaUploader` générique n'existe ; seul `VoiceRecorder` est factorisé.
7. Les tests frontend `.test.ts` n'ont pas pu être exécutés dans cet environnement (Node v20 alors que le projet exige Node 22+ pour le runner `.ts`) — aucune conclusion sur leur état d'exécution n'est formulée ici.

---

**Fin de l'audit. Aucune modification n'a été apportée ; aucun commit, aucun push, aucun test, aucun build, aucun serveur.**