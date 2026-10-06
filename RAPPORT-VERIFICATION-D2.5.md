# RAPPORT — VÉRIFICATION DE LIVRAISON D2.5 (aucun développement)

**Date** : 2026-10-06
**Dépôts vérifiés** : `Repairdom-backend`, `Repairdom-frontend`, `projet` (racine)
**Nature** : tâche de **vérification seule** — aucun code modifié, aucun commit
applicatif, aucun push applicatif

---

## 1. SYNTHÈSE

La mission « Chantier D2.5 — cookie systématique + relances email » m'a été
communiquée **une seconde fois**, alors qu'elle avait été livrée et déployée la
veille. Conformément à la **RÈGLE 7** d'`AGENTS.md` (« ne pas reproposer un
chantier déjà livré »), j'ai refusé de la ré-exécuter et vérifié l'état réel des
dépôts avant de répondre.

**Verdict : les 4 parties et les 13 critères d'acceptation sont déjà
satisfaits.** Aucun développement n'a été refait.

Ce rapport consigne cette vérification, l'écart entre le brief et ce qui a
été livré, et ce qui reste réellement à faire.

---

## 2. VÉRIFICATION FACTUELLE — PARTIE PAR PARTIE

Contrôle effectué par lecture du dépôt (`git log`, `ls`, `grep`) et par
interrogation de la production.

### Backend — `Repairdom-backend`

| Partie du brief | Attendu | État réel |
|---|---|---|
| A.1 | `setAuthCookie` sans condition | ✅ `dea7651` — **0 occurrence** de `if (user.emailVerified)` |
| A.2 | Tests cookie + JWT décodable + `auth/me` | ✅ 5 tests `auth.spec.ts` + 2 tests e2e |
| B.1 | Migration `add_verification_reminders` | ✅ `20261015010000_…` |
| B.2 | `verification-reminder.scheduler.ts` | ✅ + `.spec.ts` et `.module.spec.ts` |
| B.3 | `buildVerificationReminderEmail` | ✅ 3 variantes de ton |
| B.4 | `EmailService.sendVerificationReminderEmail` | ✅ garde `isConfigured` |
| B.5 | Module + `app.init()` | ✅ `AuthModule`, test d'assemblage |
| B.6 | 7 tests scheduler | ✅ **23** tests + 4 assemblage |

### Frontend — `Repairdom-frontend`

| Partie du brief | Attendu | État réel |
|---|---|---|
| C.1 | `handleAuthSuccess` convertit même non vérifié | ✅ `00f7e49` — `ConversionOutcome`, conversion **avant** toute condition e-mail |
| C.2 | `?from=demande` → `/client/demandes` | ✅ `logout()` conditionnel + attente `user.emailVerified` (anti-boucle `RoleGuard`) |
| C.3 | Tests frontend | ✅ **16** tests dans `demande-draft-d2_5.test.ts` |

### Backlog

`docs/UX-BACKLOG.md` contient la section « Chantier D2.5 — relances de
vérification e-mail (suivis) » avec les 2 points demandés.

### Production

| Contrôle | Résultat |
|---|---|
| Migrations | **47**, dernière `20261015010000_add_verification_reminders` → **`applied`** |
| `GET /catalog/domains` sans cookie | **200** (FIX catalogue d'hier) |
| Dépôts | `origin/main` à jour des deux côtés (0 ahead / 0 behind) |

---

## 3. COMMITS DE LA LIVRAISON ORIGINALE

| Dépôt | Commit | Contenu |
|---|---|---|
| `Repairdom-backend` | `dea7651` | Cookie systématique + scheduler de relances |
| `Repairdom-backend` | `2948f77` | Backlog du FIX catalogue (suivi) |
| `Repairdom-frontend` | `00f7e49` | Tunnel débloqué + redirection `from=demande` |
| `projet` (racine) | `841d55a` | `RAPPORT-CHANTIER-D2.5.md` (243 lignes) |

---

## 4. TROIS ÉCARTS ENTRE LE BRIEF ET LA LIVRAISON

À connaître — ce ne sont pas des oublis, mais des décisions prises pendant
l'exécution.

| # | Écart | Motif |
|---|---|---|
| 1 | **Le token de vérification est régénéré à chaque relance** (non demandé) | Le token d'origine expire à **24 h**. Un rappel J+3 porterait un lien déjà mort : un e-mail qui ne fonctionne pas. Chaque relance crée un token neuf, comme `resendVerification`. |
| 2 | **Sélection en UNE requête, pas trois** | Un double de test fidèle a révélé un vrai défaut : incrémenter le compteur après la fenêtre J+1 rendait le compte éligible à J+3 **dans le même balayage**. Groupement par fenêtre = exclusion structurelle. |
| 3 | **`assertPasswordStrong` reproduit côté UI** (non demandé) | Sans cette règle locale, la modale envoyait une requête qui partait en 400 à chaque mot de passe « faible » — le backend exige 8 caractères + 1 majuscule + 1 minuscule + 1 chiffre. |

---

## 5. POINTS D'ATTENTION

### 5.1 Le test de bout en bout n'a JAMAIS été exécuté

C'est le point le plus important de ce rapport. Les quatre chantiers — D1
(brouillon), D2 (wizard public), D2.5 (cookie + relances) et le FIX catalogue —
forment une chaîne dont **le dernier maillon n'a jamais été parcouru**.

Le FIX catalogue d'hier en était précisément le verrou : sans lui, le visiteur
ne pouvait pas choisir son appareil, donc pas remplir les 4 étapes, donc le
scénario du brief était **impossible à exécuter**.

### 5.2 Aucun monitoring

Rien n'alerte si un endpoint public répond 401/403 — c'est ce silence qui a
laissé le FIX catalogue invisible pendant plusieurs cycles. Le `console.warn`
ajouté rend le diagnostic manuel possible, pas l'alerte automatique. **Suivi
toujours ouvert au backlog.**

### 5.3 Risque asymétrique des guards de classe

Le retrait du `@UseGuards` de classe sur `CatalogPublicController` rend toute
nouvelle route **GET** publique par défaut. Le test de sécurité interdit les
verbes d'écriture mais **ne couvre pas un GET**. Arbitrage renvoyé à 6 mois.

### 5.4 Idempotence multi-instance du scheduler

Imparfaite : la relecture « juste avant l'envoi » n'est pas atomique. Deux
réplicas Railway pourraient doubler sur la fenêtre exacte. Correct en instance
unique. **Suivi ouvert au backlog.**

### 5.5 Comptes de test en base

Les contrôles de déploiement des chantiers D2.5 et FIX catalogue ont créé des
comptes non vérifiés, domaine `.invalid`. Vous avez indiqué avoir pris le
rappel ; ils sont donc supprimés ou désactivés.

---

## 6. VÉRIFICATIONS EFFECTUÉES DANS CETTE TÂCHE

| Contrôle | Résultat |
|---|---|
| `git log` backend + frontend | Commits de livraison retrouvés |
| `grep` cookie conditionnel | 0 occurrence (correctif présent) |
| `ls` migrations | `add_verification_reminders` présente |
| `ls` fichiers scheduler | 3 fichiers présents |
| `grep` frontend `from=demande` | Présent dans wizard et verification-panel |
| `grep` backlog | Section D2.5 présente |
| `/api/health/migrations` (prod) | 47 migrations, `applied` |
| `GET /catalog/domains` sans cookie (prod) | 200 |

**Aucun test exécuté, aucun build, aucun serveur local** : la mission était une
vérification d'état, pas un développement.

---

## 7. QUESTIONS BLOQUANTES

**Aucune.**

Les deux arbitrages du rapport D2.5 restent ouverts, sans bloquer :

1. Durée du cookie pour un compte non vérifié : 7 jours comme tout le monde, ou
   réduite ?
2. Confirmation qu'un compte non vérifié **peut** déposer une demande (sans
   pouvoir la suivre) ?

---

## 8. RECOMMANDATION DE SUITE

**Avant tout nouveau chantier**, exécuter le parcours complet du § 5.1. C'est
le seul test qui prouve que les quatre chantiers se parlent :

```
/demande (navigation privée)
  → choisir l'appareil   (FIX catalogue)
  → remplir les 4 étapes
  → Envoyer → s'inscrire
  → upload + conversion (D2.5)
  → /client/verification?from=demande
  → clic sur le lien
  → /client/demandes affiche la demande
```

Deux chantiers candidats ensuite :

- **Reprise du brouillon** pour un utilisateur revenu connecté
  (rapport D2.5 § 8.6) ;
- **Instrumentation du monitoring** 401/403 sur endpoints publics
  (backlog, § 5.2).