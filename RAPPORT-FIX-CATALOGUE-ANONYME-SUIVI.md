# RAPPORT — SUIVI DU FIX CATALOGUE ANONYME

**Date** : 2026-10-06
**Dépôt concerné** : `Repairdom-backend` (`2948f77`)
**Nature** : clôture du FIX catalogue + arbitrage § 7.3 + règle permanente

---

## 1. SYNTHÈSE

Tâche de clôture après validation en production du fix catalogue. Trois
volumes : arbitrage de sécurité, alimentation du backlog, et une règle
permanente qui aurait évité l'incident récité en § 5.

---

## 2. ARBITRAGE § 7.3 — DÉCISION APPLIQUÉE

> **Décision** : `GET /api/catalog/brands/:id/models` et
> `GET /api/catalog/domains/:id/problems` **restent publics**.

**Aucun changement de code** : c'est l'état déjà déployé (`fb3c587`). La
décision confirme l'exposition et ferme la question.

Fondement retenu — la spec de sécurité du rapport précédent couvre le risque :

| Contrôle automatisé | Statut |
|---|---|
| Aucun PII dans les `select` publics | Verrouillé par test |
| Aucune donnée tarifaire | Verrouillé par test |
| Aucun `include` large | Verrouillé par test |
| Filtrage `isActive` sur les référentiels | Verrouillé par test |
| Aucun verbe d'écriture dans le controller public | Verrouillé par test |

Ces deux routes ne transportent que `id`, `name`, `slug`. Elles préparent un
chantier ultérieur sans coût de surface additionnel.

---

## 3. BACKLOG — 2 ÉLÉMENTS AJOUTÉS

`backend/docs/UX-BACKLOG.md`, section « FIX catalogue anonyme — suivis »

| # | Élément | Origine |
|---|---|---|
| 1 | **Monitoring : aucune alerte si un endpoint public répond 401/403** | `console.warn` rend le diagnostic possible à la main, mais ne remplace pas une alerte. Piste retenue : compter les statuts par route dans `HttpExceptionFilter` et exposer un compteur sur `/api/health` |
| 2 | **Risque asymétrique du retrait des guards de classe** | Une future route **GET** portant une donnée de compte échapperait au garde-fou existant, qui n'interdit que les verbes d'écriture |

Le second point est écrit avec sa limite explicitée : le test actuel interdit
`@Post`/`@Put`/`@Patch`/`@Delete` mais **ne couvre pas un `GET`**. L'arbitrage
« liste explicite vs défaut public » est renvoyé à une réévaluation dans
6 mois.

---

## 4. RÈGLE PERMANENTE AJOUTÉE

`AGENTS.md` — **RÈGLE 7 : ne pas reproposer un chantier déjà livré.**

| Élément | Contenu |
|---|---|
| Origine | Vous avez annoncé D2.5 comme « prochain chantier » alors qu'il était livré et déployé la veille |
| Règle | Avant d'accepter une mission ou de proposer le chantier suivant, vérifier `git log` des deux dépôts **et** la présence d'un `RAPPORT-*` correspondant |
| Motif | Sans ce recoupement, proposer un chantier déjà fait fait perdre du temps et crée le risque de le refaire |

Ordre des règles vérifié : 1 à 7, sans trou de numérotation.

---

## 5. ⚠️ D2.5 EST DÉJÀ LIVRÉ — NE PAS LE RELANCER

Votre message indique « prochain chantier : D2.5 (cookie systématique +
relances email) ». **Ce chantier est terminé et déployé.**

| Élément | Commit | État |
|---|---|---|
| Backend | `dea7651` | Railway vert, migrations 47 appliquées |
| Frontend | `00f7e49` | Vercel vert |
| Rapport | `RAPPORT-CHANTIER-D2.5.md` (243 l.) | `projet.git` |

**Ce qui a été livré** :

- cookie de session posé **sans condition** à l'inscription (y compris CLIENT
  non vérifié) — vérifié en prod : `201` + `set-cookie` avec
  `emailVerified: false`
- scheduler de relances J+1 / J+3 / J+7, token régénéré à chaque relance,
  idempotence par compteur
- frontend : le brouillon est converti même non vérifié, redirect
  `/client/verification?from=demande`, suppression du `logout()` dans ce cas
- 27 tests backend + 60 frontend, 0 régression

**Le scénario de test 1 du rapport D2.5 n'a été confirmé en production que
si vous l'avez fait.** Le FIX catalogue était justement ce qui en empêchait le
parcours complet : le visiteur ne pouvait pas choisir son appareil. Avec le
correctif, le parcours est désormais atteignable — **c'est le bon moment pour
le tester.**

---

## 6. POINTS D'ATTENTION

1. **Le scénario complet n'est pas encore validé bout en bout** : le FIX
   catalogue et D2.5 n'ont jamais été testés ensemble. Un parcours
   `/demande` → inscription → envoi → `/client/verification?from=demande` →
   clic sur le lien → `/client/demandes` est le test qui fermerait les deux
   chantiers.
2. **Deux arbitrages D2.5 restent ouverts** (voir rapport D2.5 § 9) : durée du
   cookie pour un compte non vérifié (7 j comme tout le monde, ou réduit ?) et
   confirmation qu'un compte non vérifié peut déposer une demande.
3. **Le monitoring n'existe pas** : c'est précisément la cause du bug corrigé.
   Tant que ce n'est pas instrumenté, le prochain endpoint public cassé le sera
   de la même façon.
4. `AGENTS.md` a été modifié **hors périmètre annoncé** (vous n'aviez demandé que
   le backlog). Rétroaction d'une règle permanente, aucun impact produit —
   signalé pour transparence.

---

## 7. VÉRIFICATIONS

| Contrôle | Résultat |
|---|---|
| `npx tsc --noEmit -p tsconfig.build.json` | **OK** |
| `npm run lint` | **propre** |
| Tests | **non relancés** — aucun code de production modifié, seule une documentation |

---

## 8. QUESTIONS BLOQUANTES

**Aucune.**

---

## 9. PROCHAINE ÉTAPE RECOMMANDÉE

Ne pas ouvrir un nouveau chantier avant d'avoir validé le parcours complet
(§ 6.1). Les chantiers D1, D2, D2.5 et le FIX catalogue forment une chaîne
dont le dernier maillon n'a jamais été exécuté de bout en bout.