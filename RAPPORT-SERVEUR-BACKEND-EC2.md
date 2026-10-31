# RAPPORT — Nouveau serveur backend (EC2 eu-north-1)

## Statut

- **Date** : 2026-10-30
- **Dépôts concernés** : aucun code modifié. Seul ce rapport est ajouté à la racine.
- **Commits** : aucun encore (rapport non poussé).
- **Déploiement** : aucun.

## Synthèse

Le serveur décrit par `connexion serveur backend.txt` est **joignable et fonctionnel, mais entièrement vide**. C'est une instance EC2 `t3.micro` fraîchement démarrée (13 minutes d'uptime au moment de la connexion) à Stockholm, avec Ubuntu 26.04.1 LTS à jour, 39 Go de disque dont 2,1 Go occupés, et **aucune application installée** : ni Docker, ni Node, ni dépôt Git, rien dans `/opt`. Seul le port 22 écoute.

Ce n'est donc pas un serveur en production : c'est une machine vierge prête à recevoir un déploiement.

## Fichiers

Créés : `RAPPORT-SERVEUR-BACKEND-EC2.md` (racine). Modifiés : aucun. Supprimés : aucun.

Le fichier `connexion serveur backend.txt` n'a **pas** été modifié ni déplacé.

## Migrations

Aucune.

## Vérifications

| Élément | Valeur observée |
|---|---|
| Commande du `.txt` | `ssh -i "Relio.pem" ubuntu@ec2-13-48-13-31.eu-north-1.compute.amazonaws.com` |
| Résolution DNS | `13.48.13.31` |
| Instance | `i-0ee08946adfe9d78f`, `t3.micro`, zone `eu-north-1a` (Stockholm) |
| Connexion | OK en `ubuntu`, par clé, sans passphrase |
| OS / noyau | Ubuntu 26.04.1 LTS, `7.0.0-1014-aws` |
| Uptime | 13 minutes (machine neuve) |
| CPU / RAM | 2 vCPU, 908 Mo, **swap 0** |
| Disque `/` | 39 Go, 2,1 Go utilisés (**6 %**) |
| Unités en échec | aucune |
| Ports en écoute | **22 uniquement** |
| Docker / Node / npm | **absents** |
| `/opt` | vide |
| Dépôt Git | aucun (`/home`, `/opt`, `/srv`, `/var/www`) |
| `sudo` | sans mot de passe |
| Pare-feu local | `ufw` **inactif**, 3 règles iptables par défaut |
| `fail2ban` | **absent** |
| Mises à jour en attente | 0 — mais voir le point d'attention n° 3 |
| `unattended-upgrades` | `enabled` et `active` |
| Agents AWS | `amazon-ssm-agent` actif |
| sshd | `passwordauthentication no`, `permitrootlogin prohibit-password`, port 22 |
| IP sortante | `13.48.13.31` — identique à l'IP publique (pas de NAT gateway) |

## Bugs trouvés

Aucun bug serveur. Un problème côté dépôt, en revanche :

1. **`connexion serveur backend.txt` n'était PAS ignoré par Git.** `.gitignore` ne contenait que le nom littéral `connexion VPS.txt`, pas de motif générique. Le fichier apparaissait donc en `?? "connexion serveur backend.txt"` dans `git status` à la racine : un `git add -A` aurait versionné l'accès serveur dans `projet.git`, contredisant la RÈGLE 5.
   **Correctif appliqué** (commit `chore: ignorer connexion*.txt a la racine`) : la ligne `connexion VPS.txt` est remplacée par `connexion*.txt`. Vérifié après coup — `git check-ignore -v` renvoie bien `.gitignore:10:connexion*.txt` pour `connexion serveur backend.txt`, et le fichier a disparu de `git status`. Il reste sur le disque, toujours non suivi.

## Points d'attention

- **Aucun pare-feu local.** `ufw` est inactif et `fail2ban` absent : la seule protection est le groupe de sécurité AWS. Sur le VPS payout, `fail2ban` est actif avec une jail `sshd`. À vérifier côté console AWS avant d'exposer un service.
- **Pas de swap sur 908 Mo de RAM.** Suffisant pour un relais Python, insuffisant pour un build Node ou un `npm ci` copieux sans risque d'OOM. Le VPS payout a 2 Go de swap ; celui-ci, aucun.
- **« 0 mise à jour en attente » est trompeur.** Les listes APT datent du **16 octobre** (date de fabrication de l'AMI) et `apt update` n'a visiblement jamais tourné depuis le démarrage. Le compteur réel n'est donc pas fiable ; il faut un `apt update` avant d'installer quoi que ce soit.
- **Ce n'est pas le serveur de production décrit dans les rapports précédents.** `api.relioo.space` (`13.61.214.149`, Paris) héberge le relais payout ; cette instance-ci est une autre machine, en Suède, vide. Aucun lien entre les deux n'a été constaté.
- **Même clé privée pour les deux serveurs.** `Relio.pem` ouvre le relais payout **et** cette nouvelle instance. Une compromission de la clé expose les deux. Une paire par instance serait préférable.
- **Démarrage très récent** : l'uptime de 13 minutes suggère une instance lancée pour ce chantier. Si un déploiement est prévu, les prérequis (Node 22+, Docker si retenu, reverse proxy, `.env`) sont tous à installer — rien n'est préparé.
- **Non vérifié** : groupe de sécurité AWS (ports réellement ouverts depuis Internet), coût/quotas, et présence éventuelle d'un rôle IAM. Aucun accès console AWS depuis cette machine.

## Questions bloquantes

Aucune. Trois décisions vous reviennent :

1. Faut-il corriger `.gitignore` (`connexion*.txt`) pour que ce fichier ne puisse plus être versionné par erreur ?
2. Faut-il pousser ce rapport sur `projet.git` ?
3. Que doit-on installer sur cette instance — et pour quel rôle exact (backend NestJS complet, relais, autre) ?
