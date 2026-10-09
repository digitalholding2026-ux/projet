# RAPPORT — Lecture globale du projet (architecture, sans modification)

## Statut

| Champ | Valeur |
|---|---|
| Date | 2026-10-08 |
| Nature | **Lecture seule stricte** — aucune ligne de code modifiée dans `backend/` ni `frontend/` |
| Dépôts | `Repairdom-backend` (`da71861`), `Repairdom-frontend` (`50002d9`), `projet` (ce rapport) |
| Commits applicatifs | **aucun** — arbres de travail vérifiés propres (`git status --porcelain` vide) |
| Push | racine uniquement, après validation |
| Vérifications exécutées | `git status` ×3, `npm ci --dry-run` ×2, un `node --test` d'essai. **Aucun build, aucun test, aucun serveur.** |

## Synthèse

Relio est une plateforme camerounaise de mise en relation entre clients et
techniciens réparateurs, monétisée par commission, adossée à un rail de paiement
SasPay et à un moteur de fidélité based sur la marge. Le dépôt racine ne contient
que de la documentation ; tout le code est dans les deux dépôts imbriqués.

## Fichiers

Créés : ce rapport. Modifiés : aucun. Supprimés : aucun.

## Migrations

Aucune. 48 migrations Prisma, dernière `20261016020000_refonte_rewards_ltv`.

## Vérifications

- `git status --porcelain` sur les 3 dépôts → **aucune modification de code** (la
  racine affiche `?? backend/` et `?? frontend/`, ce qui est l'état normal et
  documenté en RÈGLE 2).
- `find backend/src frontend/src -name '*.js'` → aucun `.js` parasite (contrôle
  RÈGLE 4).
- `npm ci --dry-run` (backend) → **ÉCHEC** `EUSAGE`, voir Bugs §1.
- `npm ci --dry-run` (frontend) → OK, 267 paquets.
- `node --test` sur un fichier `.ts` → **ÉCHEC** `ERR_UNKNOWN_FILE_EXTENSION`,
  voir Bugs §2.

## Bugs trouvés

Les deux sont des blocages d'environnement, constatés en exécutant les
commandes, pas en lisant le code.

### 1. `npm ci` est cassé sur le backend

`Missing: typescript@5.9.3 from lock file`. Le backlog (`backend/docs/UX-BACKLOG.md`)
le documentait déjà et proposait `npm install` pour régénérer le lock — ce n'est
pas une surprise, c'est une dette confirmée. Conséquence directe : **aucune
installation de dépendances backend n'est possible sans `npm install`**, donc
`tsc`, `oxlint` et `vitest` backend sont inaccessibles en l'état.

### 2. Node 20 empêche l'exécution des tests frontend

`npm run test:unit` est un `node --test` sur 35 fichiers `.ts` purs. Node 20
refuse de les charger (`Unknown file extension ".ts"`). Confirmé par exécution,
pas déduit. RÈGLE 4 l'avait anticipé ; il reste à installer Node 22.

## Points d'attention

Les points ci-dessous sont des **constats de lecture**, pas des correctifs
appliqués. Aucun n'a été corrigé.

### Sécurité

1. **Deux routes de diagnostic publiques et non authentifiées** :
   `GET /api/health/db` et `GET /api/health/migrations`. Elles exposent la liste
   des colonnes de `User` et la liste des migrations avec leur statut
   (`applied`/`pending`/`rolled_back`). Elles sont marquées « À SUPPRIMER » dans
   le code et listées au backlog depuis le chantier reset-password.
2. **Aucune protection multi-instance** : le registre SSE, le rate-limit des
   brouillons et celui du tracking sont **en mémoire**. Le projet le documente et
   suppose une instance unique. Un second réplica Railway invaliderait la
   déduplication push anti-SSE et rendrait les quotas contournables.
3. **Cinq modèles IA présents en base sans implémentation** :
   `DemandeClassification`, `DiagnosticCatalogMatch`, `QuotePricingCheck`,
   `AiWarning`, `AiConversationFlag` — avec leurs enums et les types de
   notifications correspondants (`PRICING_WARNING`, `CONVERSATION_FLAG`).
   Vérifié par recherche exhaustive : **aucun service ne les lit ni ne les
   écrit**. Le seul agent IA réellement câblé est le backoffice Groq, en lecture
   seule.
4. **Secret de brouillon en `localStorage`**
   (`relio_demande_draft_token`). C'est un bearer token accessible à tout XSS.
   Le code est rigoureux ailleurs (jamais journalisé, jamais en URL).
5. **Protection des routes 100 % client-side** : le HTML et les assets de
   `/admin` sont téléchargeables sans session. Conséquence directe et assumée de
   l'architecture cross-domain — la sécurité réelle est portée par l'API.

### Correction

6. **`localStorage` contredit par son propre commentaire** :
   `demande-draft-storage.ts` annonce « une clé en *sessionStorage* » alors que
   `storage()` résout vers `localStorage`. Le commentaire affirme aussi que le
   token de vérification ne vit que dans l'URL — faux, il est aussi stocké.
7. **Duplications assumées** : `apiFetch` réimplémenté dans 15 fichiers de
   `src/lib/api/` (la classe `ApiError`, elle, a été centralisée — donc le
   problème est connu) ; `roleHomePath` dupliqué entre deux modules (protégé par
   un test statique) ; règles financières (5 000 / 500 / 4 % / 2 000) dupliquées
   côté front alors que le backend reste juge.
8. **Liste de tests figée** : `npm run test:unit` liste 35 chemins en dur. Un
   nouveau test non ajouté à la chaîne ne tournera jamais, sans erreur.
9. **Écarts dans `tsconfig.json`** côté frontend : 5 fichiers de test exécutés
   mais non exclus du typecheck, et 2 exclusions pointent vers des fichiers
   inexistants (restes d'une fonctionnalité IA retirée).

### Performance / taille

10. **~15 Mo d'assets dupliqués** trackés à la racine du frontend (`recompense/`,
    `hero/`, `MTN&ORANGE/`) : le code sert les copies de `public/`. Seul
    `animatio json/` est du code actif (importé par `lottie-animations.tsx`).
11. **Polling 5 s en parallèle du SSE** sur les pages mission : le SSE est
    documenté comme la source de signal, le polling comme le repli, mais les
    deux tournent simultanément.
12. **Fichiers géants** : `demande-wizard.tsx` (1 742 l.), `financial.service.ts`
    (2 740 l.), `catalog.service.ts` (2 205 l.). Surface de régression maximale,
    surtout pour l'ordonnancement des hooks (un `useMemo` conditionnel a déjà
    cassé toute une page technicien).
13. **`updatedAt` emprunté comme date de confirmation** dans le module
    récompenses (aucune colonne `confirmedAt`). Toute écriture ultérieure sur la
    mission fausserait la détection anti-fraude et le cumul de marge. Le code
    l'admet et juge le risque faible à l'échelle d'une fenêtre de 48 h.
14. **Déploiement** : `scripts/migrate-recover.js` cible une migration unique et
    codée en dur, et s'exécute à **chaque** déploiement Railway en tête du
    `startCommand`. Correctif ponctuel devenu étape permanente.
15. **Barèmes historiques coexistants** : commission 2 % legacy, `CLIENT_FEE`,
    `TECHNICIAN_PLATFORM_FEE` — conservés pour réconcilier les écritures
    immuables. Risque : une nouvelle écriture qui piocherait dans une constante
    « historique ».

## Questions bloquantes

Aucune. Rien n'a été modifié, donc rien n'est en attente de décision technique.

Décision d'arbitrage qui ne m'appartient pas : les trois questions de
`RAPPORT-AUDIT-FRAIS-SASPAY.md` (frais client, plafond de retrait après
majoration, nature du 3.5 %) sont toujours ouvertes et bloquent le chantier
transparence des frais.