# RAPPORT — État du serveur Linux (VPS relais payout)

## Statut

- **Date** : 2026-10-30
- **Dépôts concernés** : aucun. Investigation en lecture seule, aucun fichier versionné modifié.
- **Commits** : aucun.
- **Push** : aucun (rapport non poussé, en attente de validation — RÈGLE 6).
- **Déploiement** : aucun.

## Synthèse

Le serveur Linux accessible avec `Relio.pem` est **en marche et sain**. Il s'agit du VPS `api.relioo.space` (`13.61.214.149`, AWS `eu-west-3`/Paris, Ubuntu 26.04.1 LTS), allumé depuis **35 jours** sans redémarrage. Il n'héberge **que le relais payout SasPay** : `relio-payout-relay.service` est actif depuis le 24/10, sans redémarrage, sans erreur au journal, et répond correctement en 401 sans secret d'authentification. Le point de vigilance réel est le **disque à 96 % d'occupation** (317 Mo libres sur 6,7 Go).

## Fichiers

Créés (hors dépôt) :

- `C:\Users\Docker\.ssh\relio.pem` — copie de travail de la clé, ACL restreintes à `DOCKERW-VBDCHVN\Docker` (lecture) + SYSTEM/Administrateurs. **L'original `Relio.pem` à la racine n'a pas été modifié.**
- `RAPPORT-SERVEUR-VPS.md` (racine) — ce rapport
- `%LOCALAPPDATA%\Temp\relio-diag.sh`, `relio-diag2.sh` — scripts de diagnostic temporaires

Modifiés : aucun. Supprimés : aucun.

Vérification RÈGLE 5 : `Relio.pem` est ignoré par `.gitignore` (règle ligne 9) et **n'est pas suivi** par Git (`git ls-files --error-unmatch` → échec attendu). Aucun secret n'apparaît dans ce rapport.

## Migrations

Aucune.

## Vérifications

| Élément | Valeur observée |
|---|---|
| Hôte | `13.61.214.149` (`ip-172-31-34-232`), DNS `api.relioo.space` |
| Utilisateur SSH | `ubuntu` (connexion par clé, sans passphrase) ; `root` refusé, message « Please login as the user "ubuntu" » |
| Accès sudo | sans mot de passe |
| OS / noyau | Ubuntu 26.04.1 LTS |
| Uptime | 35 jours 20 h, load 0.27 |
| CPU / RAM | 2 vCPU, 908 Mo (470 Mo utilisés, 437 Mo disponibles), swap 2 Go |
| Disque `/` | 6,7 Go — **6,3 Go utilisés, 317 Mo libres, 96 %** |
| Unités systemd en échec | **aucune** |
| `nginx` | active (running), écoute 80 / 443 |
| `relio-payout-relay` | active (running) depuis 2026-10-24 06:45 UTC, `NRestarts=0` |
| Erreurs journal relais | `No entries` (niveau err) |
| Relais en écoute | `127.0.0.1:8787` (python3, utilisateur `relio-relay`) |
| `POST https://api.relioo.space/payout/initialize` (sans secret) | **401** — comportement attendu |
| `GET https://api.relioo.space/` | 404 — voulu, nginx ne sert que `/payout/initialize` |
| Certificat TLS | `CN=api.relioo.space`, Let's Encrypt, expire le **2026-12-23** |
| IP publique sortante | `13.61.214.149` (identique à l'entrante → IP fixe confirmée) |
| `fail2ban` | actif, 1 jail (`sshd`) |
| Synchronisation horaire | `System clock synchronized: yes` |
| Docker | installé et actif, **aucun conteneur** |
| Paquets en attente de mise à jour | **43** |
| `relioo.space` (Vercel) | 308 vers la version canonique — normal |
| `pm2` | absent (le relais est un service systemd) |

### Preuve

Les 401 observés côté public sont bien tracés côté serveur dans le journal du relais, avec l'horodatage exact des deux appels de test :

```
Oct 31 03:03:58 relay INFO POST /payout/initialize -> 401 0ms idempotency=***
Oct 31 03:04:18 relay INFO POST /payout/initialize -> 401 0ms idempotency=***
```

Cela confirme que la chaîne complète fonctionne : Internet → nginx (TLS) → `127.0.0.1:8787` → relais. Le 401 ne vient pas d'une panne mais du secret d'authentification absent, exigé par conception.

## Bugs trouvés

1. **Disque à 96 % (317 Mo libres) — risque de saturation.** Impact : un remplissage complet bloquerait nginx, le journal et le service relais. Répartition mesurée : `/usr` 3,3 Go, `/var` 1,1 Go (dont `/var/cache` 342 Mo, `/var/log` 38 Mo), `/opt` 2,6 Mo. Le volume ne fait que 6,7 Go au total. Correctif non appliqué (je n'ai rien supprimé sans votre accord) : `sudo apt clean` libérerait l'essentiel de `/var/cache`, et une extension du volume EBS est à envisager pour un serveur de production.
2. **43 paquets en attente de mise à jour**, dont les correctifs de sécurité éventuels. Non appliqués : un `apt upgrade` peut redémarrer des services et sort du périmètre d'une simple vérification.

## Points d'attention

- **`/opt/relio/backend` existe** (2,5 Mo, propriétaire `ubuntu`) à côté de `/opt/relio-payout-relay`. Aucun service ne l'exécute et aucun conteneur Docker ne tourne : c'est vraisemblablement une ancienne piste de déploiement. Rien n'écoute dessus, donc sans effet sur le fonctionnement actuel, mais sa présence est ambiguë pour un futur intervenant.
- **Aucun conteneur Docker** alors que le démon tourne : `docker.service` et `containerd.service` consomment de la RAM pour rien sur une instance à 908 Mo. Non arrêtés (décision à votre main).
- **Le relais est le seul service exposé.** L'API applicative complète n'est pas sur ce VPS : elle reste sur Railway (conforme à `.env.example`, « Railway uniquement »). Ce serveur n'est donc pas utilisable comme preuve de disponibilité de l'application Relio.
- **Certificat Let's Encrypt** valide jusqu'au 2026-12-23 : si le renouvellement automatique (Certbot) est actif, rien à faire ; je n'ai pas vérifié le timer `certbot.timer`.
- **Clé privée `Relio.pem`** : RSA sans passphrase, stockée à la racine du dépôt de travail. Elle est ignorée par Git, mais ce n'est pas un emplacement idéal pour un secret d'accès serveur.
- Non vérifié : contenu et logique du script `relay.py` (lecture de code métier hors périmètre), accès à une base de données depuis le VPS, comportement du relais avec un secret valide (exigerait de manipuler le secret de production).

## Questions bloquantes

Aucune. Trois décisions vous reviennent :

1. Faut-il libérer de l'espace disque (`apt clean`, purge des noyaux/paquets obsolètes) et/ou étendre le volume ?
2. Faut-il appliquer les 43 mises à jour ?
3. Faut-il pousser ce rapport (et celui de l'initialisation) sur `projet.git` ?
