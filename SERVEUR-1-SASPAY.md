# SERVEUR 1 — Relais SasPay (production)

> **Rôle unique : relais payout SasPay.** Ce serveur ne doit **rien** héberger
> d'autre. Ni API Relio, ni frontend, ni base, ni outil de développement.

## Identité

| | |
|---|---|
| Nom | `api.relioo.space` (DNS public) |
| IP publique | `13.61.214.149` |
| Hôte interne | `ip-172-31-34-232` |
| Fournisseur / région | AWS — `eu-west-3` (Paris) |
| Type d'instance | `t3.micro` (2 vCPU, 908 Mo RAM, 2 Go swap) |
| OS | Ubuntu 26.04.1 LTS |
| Disque | 6,7 Go — **96 % occupé** (317 Mo libres) |

## Accès

- **Utilisateur** : `ubuntu`
- **Clé** : `Relio.pem` (racine du dépôt de travail, ignorée par Git) — copie de travail : `~/.ssh/relio.pem` avec ACL restreintes.
- **Mot de passe** : aucun. Authentification par clé uniquement, pas de passphrase.
- **`sudo`** : sans mot de passe.
- **`root`** : connexion refusée (« Please login as the user "ubuntu" »).

```bash
ssh -i Relio.pem ubuntu@api.relioo.space
# ou, si le nom DNS n'est pas résolu :
ssh -i Relio.pem ubuntu@13.61.214.149
```

> La même clé ouvre aussi le **serveur 2** (voir `SERVEUR-2-BACKEND.md`). Une
> compromission de la clé expose les deux machines.

## Ce qui tourne

| Élément | Détail |
|---|---|
| `nginx` | actif — écoute 80 et 443, TLS Let's Encrypt (`CN=api.relioo.space`, expire le **2026-12-23**) |
| `relio-payout-relay.service` | actif depuis le 24/10/2026, `Restart=always`, 0 redémarrage |
| Code du relais | `/opt/relio-payout-relay/relay.py` (Python 3), utilisateur `relio-relay` |
| Écoute applicative | `127.0.0.1:8787` (jamais exposé directement) |
| Exposition HTTP | **une seule route** : `POST /payout/initialize` → proxy vers `127.0.0.1:8787`. Tout le reste renvoie 404. |
| `fail2ban` | actif, 1 jail (`sshd`) |
| `docker` | installé et actif, **aucun conteneur** |
| `node` | installé (`/usr/bin/node`), **non utilisé** par le relais |

Chaîne complète : Internet → nginx (TLS) → `127.0.0.1:8787` → relais Python.

## Contrôles (lecture seule)

```bash
systemctl status relio-payout-relay --no-pager
journalctl -u relio-payout-relay -n 20 --no-pager
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://api.relioo.space/payout/initialize
```

`401` sans secret d'authentification est le **résultat attendu**, pas une panne.
Un `404` sur `https://api.relioo.space/` est également normal.

## Points d'attention

- **Disque à 96 %** — 317 Mo libres. C'est le seul incident connu et il est bloquant à terme : un disque plein arrête nginx et le relais. `sudo apt clean` libère l'essentiel de `/var/cache` (342 Mo).
- 43 paquets en attente de mise à jour (relevé du 2026-10-30).
- `/opt/relio/backend` existe (2,5 Mo, propriétaire `ubuntu`) mais **n'est exécuté par aucun service** : résidu d'une tentative antérieure, sans effet.
- `docker.service` et `containerd.service` tournent sans conteneur : RAM consommée inutilement sur une instance à 908 Mo.
- **Ne jamais y déployer l'API Relio.** L'API vit sur Railway (`backend/` du dépôt), le frontend sur Vercel.

## Interdits sur ce serveur

- Déployer le backend NestJS, le frontend, une base de données ou un cron métier.
- Exposer un autre chemin que `/payout/initialize`.
- Y stocker des sauvegardes ou des artefacts de build.

## État relevé

- **2026-10-31 03:24 UTC** : up 5 semaines 21 h, `relay=active`, `nginx=active`, disque 96 %, RAM 409 Mo utilisés / 908 Mo.

---

*Toute divergence entre ce document et l'état réel doit être signalée dans un `RAPPORT-*.md`.*
