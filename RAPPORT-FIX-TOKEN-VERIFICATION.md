# RAPPORT — FIX : TOKEN DE VÉRIFICATION CONSOMMÉ AVANT LE CLIC

**Date** : 2026-10-06
**Dépôts concernés** : `Repairdom-backend` (`00d8090`), `Repairdom-frontend` (`b3498bf`)
**Nature** : correctif — un lien e-mail à usage unique était brûlé par un tiers

---

## 1. SYNTHÈSE

Cliquer sur le lien de vérification affichait « Ce lien de vérification est
invalide ou a expiré » alors que tout venait de réussir. **Deux causes
indépendantes** ont été traitées, dont une que le brief n'avait pas identifiée et
qui était la plus vicieuse.

---

## 2. CAUSE EXACTE — CONFIRMÉE

### 2.1 Cause 1 — préchargement des lecteurs e-mail

`VerificationPanel` déclenchait `POST /auth/verify-email` **dans un `useEffect`
au montage**. Gmail, Outlook SafeLinks et les scanners antivirus visitent les
liens des e-mails ; **certains exécutent le JavaScript**. Le token était donc
consommé à la place de l'utilisateur.

**Confirmé par le code** : le POST était déclenché par `if (!token) return;`
dans un effet, sans aucun geste de l'utilisateur.

### 2.2 Cause 2 — le backend rendait tout rejeu « invalide » (le vrai piège)

`auth.service.ts` faisait, en cas de succès :

```ts
data: { emailVerified: true, emailVerificationToken: null, emailVerificationExpiresAt: null }
```

Conséquence : **un rejeu du même lien ne trouvait plus aucun compte**, et
tombait sur `EMAIL_INVALID_OR_EXPIRED`. Autrement dit, le message que vous voyiez
était **structurellement faux**.

Et le correctif était déjà écrit, juste… inatteignable :

```ts
if (found.emailVerified) {
  throw new BadRequestException('Cette adresse email est déjà vérifiée.');
}
```

Cette branche ne pouvait **jamais** s'exécuter, puisque `emailVerificationToken`
venait d'être mis à `null` — aucun compte ne pouvait plus correspondre au token.
**C'est du code mort**, et c'est ce qui explique que le message « déjà vérifiée »
apparaisse si rarement alors qu'il existe dans le code.

### 2.3 Pourquoi le message était trompeur

| Situation | Ancien comportement | Nouveau |
|---|---|---|
| Scanner a consommé le token, clic réel ensuite | 400 « invalide ou expiré » | 200 `alreadyVerified: true` → message positif |
| Clic suivant le premier | 400 « invalide ou expiré » | idem |
| Rejoué des jours plus tard, token expiré | 400 « invalide ou expiré » | idem (vérifié **avant** l'expiration) |

---

## 3. CORRECTIFS

### 3.1 Backend — `00d8090`

| Changement | Raison |
|---|---|
| `verifyEmail` retourne `{ user, alreadyVerified }` | Distinguer un vrai rejeu d'un échec |
| Le token **n'est plus effacé** après vérification | Il reste une clé de recherche, donc le rejeu trouve le compte |
| « Déjà vérifié » testé **avant** l'expiration | Le cas courant est un lien rejoué des jours plus tard ; l'inverse renverrait au message trompeur |
| Le contrôleur répond **200** + `alreadyVerified` et **repose la session** | Rouvrir un lien est un no-op, pas un cul-de-sac |

**Sécurité du token conservé** : il n'accorde **rien** une fois `emailVerified ===
true` — c'est ce seul drapeau qui autorise quoi que ce soit. Un nouveau lien
(renvoi ou relance) **écrase** l'ancien, donc un token fuite ne reste pas
exploitable. Le commentaire dans le code le documente.

### 3.2 Frontend — `b3498bf`

| Changement | Raison |
|---|---|
| **Écran de confirmation** quand `?token=` est présent | Le token n'est consommé qu'au clic |
| Le panneau explique **pourquoi** | « Certains services de messagerie testent automatiquement les liens » |
| `useRef` `verificationAttempted` | Double clic / re-render / remontage = une seule requête |
| `alreadyVerified` → « Votre email est déjà vérifié » + action | Plus d'alerte à tort |
| Message d'erreur reformulé | « Ce lien a expiré… demandez un nouveau lien », bouton de renvoi **conservé** |
| « Adresse inconnue » **supprimé** | Rien n'était « inconnu » : c'est nous qui ne savions pas à quel e-mail renvoyer |
| Le garde n'est relâché qu'**en cas d'erreur métier** | Relâché après un succès, un second POST renverrait 400 |

Le parcours technicien utilise le même composant et hérite de tout.

---

## 4. FICHIERS MODIFIÉS

| Dépôt | Chemin | Nature |
|---|---|---|
| backend | `src/auth/auth.service.ts` | `verifyEmail` idempotent, token conservé, ordre des vérifications |
| backend | `src/auth/auth.controller.ts` | Réponse `alreadyVerified` + session reposée |
| backend | `src/auth/auth.spec.ts` | 2 tests obsolètes mis à jour + 3 ajoutés |
| frontend | `src/components/auth/verification-panel.tsx` | Écran de confirmation, `useRef`, messages |
| frontend | `src/lib/api/auth-service.ts` | Type `VerifyEmailResult` |
| frontend | `src/lib/verification-confirm.test.ts` | **Nouveau** — 16 tests |

Aucun changement dans `email-templates.ts` : **le lien ne change pas**, seul le
comportement du frontend — comme le demandait le brief.

---

## 5. VÉRIFICATIONS

| Contrôle | Résultat |
|---|---|
| Backend `tsc -p tsconfig.build.json` | **OK** |
| Backend `oxlint` | **propre** |
| Backend suite | **944 passés**, 3 échecs pré-existants (noms identiques) |
| Frontend `tsc --noEmit` | **OK** |
| Frontend `oxlint` | **propre** |
| Frontend nouveau fichier | **16/16** |
| Frontend total (5 suites) | **93/93** |
| Fichiers `.js` parasites | **aucun** |

### Production

| Contrôle | Résultat |
|---|---|
| Railway | vert, `POST /verify-email` token inconnu → **400** (contrat inchangé) |
| Vercel | vert, `/client/verification?token=…` → **200** |

---

## 6. DÉFAUTS TROUVÉS DANS MES PROPRES TESTS

| # | Défaut | Correction |
|---|---|---|
| 1 | Assertion inversée : j'attendais que l'`update` **contienne** le token, alors que l'implémentation ne touche pas la colonne et conserve la valeur existante | Test réécrit sur l'invariant réel : l'`update` ne doit contenir **ni** le token **ni** l'expiration |
| 2 | Regex sur l'apostrophe droite alors que le JSX utilise `&apos;` | Recherche sur la source **littérale** |
| 3 | Chemin relatif erroné vers la page technicien (`../../` au lieu de `../`) | Corrigé |
| 4 | Fenêtre de contexte trop courte (200 → 500 caractères) autour de l'appel | Élargie |
| 5 | **`doesNotMatch` sur la source brute** : « Adresse inconnue » ne survivait que dans un **commentaire** expliquant sa suppression — le test échouait à tort | Comparaison sur le code, commentaires retirés |

Le défaut **n° 5** est le plus subtil : un test trop naïf échouait pour la bonne
raison. Le corriger demandait de distinguer « le texte dit à l'utilisateur » de
« le mot apparaît dans le dépôt ».

---

## 7. POINTS D'ATTENTION

### 7.1 L'utilisateur qui reclique un lien ancien hors contexte `from=demande`

Il sera déconnecté (`logout()`) et renvoyé vers la connexion : c'est le
comportement historique, conservé volontairement.

### 7.2 Le maillon `from=demande` reste rompu

**Non traité dans ce chantier** (hors périmètre demandé) : les trois liens e-mail
(`auth.service.ts:260`, `:339`, `verification-reminder.scheduler.ts:201`) ne
portent toujours **pas** `from=demande`. Conséquence : le clic depuis l'e-mail
atterrit sur `/client/verification?token=…` sans ce paramètre → `logout()` →
redirection manuelle vers la connexion, et non vers `/client/demandes`.

Ce correctif **n'empêche pas** cela : il rend le clic fiable, pas la
redirection intelligente. **Bug connu, à traiter.**

### 7.3 Le token vit plus longtemps en base

Until was nulled immediately; now it persists until the next resend/reminder.
Impact négligeable sur le stockage, mais à garder en tête pour un futur
« purge des tokens de vérification ».

### 7.4 Non testé en production par moi

Je ne peux pas recevoir d'e-mail. Le scénario ci-dessous vous revient.

---

## 8. QUESTIONS BLOQUANTES

**Aucune.**

Le point 7.2 reste la prochaine question ouverte — c'est le dernier maillon
manquant de la chaîne D2.5.

---

## 9. SCÉNARIO DE TEST PRODUCTION

| # | Étape | Attendu |
|---|---|---|
| 1 | `/demande` → 4 étapes → s'inscrire | — |
| 2 | Ouvrir l'e-mail, **cliquer sur le lien** | Page **Confirmez votre adresse email** + bouton |
| 3 | **DevTools → Network, SANS cliquer le bouton** | **Aucun** `POST /auth/verify-email` |
| 4 | Cliquer « Confirmer mon email » | `POST` → **200**, puis redirection selon le contexte |
| 5 | Recliquer le lien dans l'e-mail | **« Votre email est déjà vérifié »**, aucune erreur |
| 6 | Attendre ~10 s avant de cliquer (le scanner a eu le temps) | Le bouton fonctionne toujours |

L'étape **3** valide la cause 1 ; l'étape **5** valide la cause 2.