# SERVEUR 2 — Déploiement backend

> **Rôle : héberger le déploiement du backend Relio** (`Repairdom-backend`,
> NestJS / Prisma). Le serveur est **actuellement vide** : rien n'y est encore
> installé ni déployé.

## Identité

| | |
|---|---|
| Nom DNS public | `ec2-13-48-13-31.eu-north-1.compute.amazonaws.com` |
| IP publique | `13.48.13.31` |
| Hôte interne | `ip-172-31-24-199` |
| Instance | `i-0ee08946adfe9d78f` |
| Fournisseur / région | AWS — `eu-north-1a` (Stockholm) |
| Type d'instance | `t3.micro` (2 vCPU, 908 Mo RAM, **aucun swap**) |
| OS | Ubuntu 26.04.1 LTS, noyau `7.0.0-1014-aws` |
| Disque | **39 Go — 6 % occupé** (2,1 Go utilisés, 37 Go libres) |
| Créé / démarré | 2026-10-31, image Ubuntu du 2026-10-16 |

## Accès

- **Utilisateur** : `ubuntu`
- **Clé** : `Relio.pem` (racine du dépôt de travail, ignorée par Git) — copie de travail : `~/.ssh/relio.pem` avec ACL restreintes.
- **Mot de passe** : aucun. `passwordauthentication no`, `permitrootlogin prohibit-password`.
- **`sudo`** : sans mot de passe.

```bash
ssh -i Relio.pem ubuntu@ec2-13-48-13-31.eu-north-1.compute.amazonaws.com
# ou, si le nom DNS n'est pas résolu :
ssh -i Relio.pem ubuntu@13.48.13.31
```

> La même clé ouvre aussi le **serveur 1** (voir `SERVEUR-1-SASPAY.md`).

## État actuel (relevé le 2026-10-31 03:24 UTC)

| Élément | État |
|---|---|
| Uptime | 18 minutes |
| Services en échec | aucun |
| Ports en écoute | **22 uniquement** |
| `nginx` | **absent** |
| `docker` | **absent** |
| `node` / `npm` | **absents** |
| Dépôt Git | aucun (`/home`, `/opt`, `/srv`, `/var/www` vides) |
| `ufw` | **inactif** (3 règles iptables par défaut) |
| `fail2ban` | **absent** |
| `unattended-upgrades` | `enabled` + `active` |
| `amazon-ssm-agent` | actif |
| IP sortante | `13.48.13.31` (identique à l'IP publique, pas de NAT gateway) |

**Rien n'est préparé pour un déploiement.** Tous les prérequis restent à installer.

## Prérequis à installer avant tout déploiement

1. **Node 22 minimum** (RÈGLE 4 du projet) — version LTS recommandée.
2. **Swap** : l'instance a 908 Mo de RAM et **0 swap**. Un `npm ci` sans swap risque l'OOM killer. Créer au minimum 2 Go de swap avant la première installation.
3. **Reverse proxy** (nginx ou Caddy) + certificat TLS.
4. **Process manager** : `systemd` (recommandé, comme le relais SasPay) ou `pm2`.
5. **Pare-feu** : activer `ufw` (22, 80, 443) et installer `fail2ban`. Aujourd'hui la seule protection est le groupe de sécurité AWS.
6. **`.env` de production** — jamais commité, jamais dans ce dépôt.

## Contrôles (lecture seule)

```bash
systemctl --failed --no-legend
ss -tlnp
df -h /
free -h
```

## Points d'attention

- **« 0 mise à jour en attente » est trompeur** : les listes APT datent du 2026-10-16 (fabrication de l'AMI), `apt update` n'a jamais tourné depuis le démarrage. Lancer `apt update` avant toute installation.
- **Pas de swap** — voir prérequis n° 2. C'est le risque le plus concret sur cette machine.
- **Adresse publique non garantie** : `13.48.13.31` est une IP publique d'instance, pas une Elastic IP confirmée. Si l'instance est arrêtée puis redémarrée, l'IP **et** le nom DNS public changent. À vérifier dans la console AWS avant de configurer un DNS ou des webhooks.
- **Pas de pare-feu local** : `ufw` inactif, `fail2ban` absent. À traiter avant d'exposer un port applicatif.
- Le nom d'hôte DNS AWS est long et change avec l'IP : prévoir un enregistrement DNS stable si ce serveur devient l'entrée d'API.
- Serveur **facturé** (`t3.micro`) alors qu'il n'héberge encore rien.

## Interdits sur ce serveur

- Y déployer le relais SasPay (il appartient au serveur 1, avec son IP fixe).
- Y committer ou y laisser un `.env` de production dans un dépôt Git.
- Y lancer `npm run dev` / `next build` en production (RÈGLE 3).

---

*Toute divergence entre ce document et l'état réel doit être signalée dans un `RAPPORT-*.md`.*
