# Ansible — Déploiement automatisé d'Apache HTTPS et de GLPI

Automatisation complète, avec **Ansible**, du déploiement de deux services sur des machines virtuelles Linux :

- un serveur web **Apache2 en HTTPS** (certificat autosigné, redirection HTTP → HTTPS) sur Ubuntu Server ;
- un serveur **GLPI** (gestion de parc et helpdesk) avec PHP et MariaDB sur AlmaLinux ;
- un playbook de **mises à jour** qui fonctionne sur toutes les distributions (apt ou dnf selon l'OS).

Aucune intervention manuelle n'est nécessaire : chaque playbook installe, configure, puis **vérifie lui-même** que le service répond.

> Projet réalisé dans le cadre de mon programme en réseautique et cybersécurité (CÉGEP de Saint-Hyacinthe).

---

## Architecture

Toutes les VM sont sur un réseau **NAT**.

| Machine | OS | Rôle |
|---|---|---|
| Nœud de contrôle | Ubuntu Desktop | Ansible installé, lance les playbooks via SSH |
| `apache01` | Ubuntu Server | Apache2 + HTTPS |
| `glpi01` | AlmaLinux | Apache (httpd) + PHP + MariaDB + GLPI |

```
             ┌──────────────────────────┐
             │  Ubuntu Desktop (Ansible)│
             └────────────┬─────────────┘
                     SSH  │  (clé)
          ┌───────────────┴───────────────┐
          ▼                               ▼
 ┌─────────────────┐             ┌─────────────────┐
 │ apache01        │             │ glpi01          │
 │ Ubuntu Server   │             │ AlmaLinux       │
 │ Apache2 :443    │             │ httpd :80       │
 │ (80 → 443)      │             │ PHP + MariaDB   │
 └─────────────────┘             └─────────────────┘
```

## Structure du projet

```
.
├── ansible.cfg
├── hosts.ini                  # inventaire : groupes [https] et [glpi]
├── requirements.yml           # collection community.mysql
├── playbooks/
│   ├── https.yml              # applique le rôle apache_https
│   ├── glpi.yml               # applique le rôle glpi
│   └── update.yml             # met à jour tous les serveurs (apt / dnf)
├── roles/
│   ├── apache_https/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/vhost.conf.j2
│   │   └── files/index.html
│   └── glpi/
│       ├── defaults/main.yml  # version GLPI, nom de la base, utilisateur
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── templates/
│           ├── glpi.conf.j2
│           └── config_db.php.j2
└── docs/
    ├── NOTES.md               # explication bloc par bloc
    └── captures/
```

## Ce que fait chaque rôle

### `apache_https` (Ubuntu Server)
1. Installe `apache2`, `openssl`, `ssl-cert`.
2. Active les modules `ssl` et `headers`.
3. Génère un **certificat autosigné** RSA 2048 valide 1 an (Let's Encrypt est impossible : la VM est en NAT, donc injoignable depuis Internet).
4. Crée le DocumentRoot et copie la page `index.html`.
5. Déploie le VirtualHost à partir d'un **template Jinja2** : port 443 en HTTPS + port 80 qui redirige tout vers HTTPS.
6. Active le site, désactive le site par défaut, redémarre Apache via un **handler**.
7. **Valide** automatiquement : HTTPS répond `200`, HTTP répond `301`.

### `glpi` (AlmaLinux)
1. Installe httpd, MariaDB, PHP et ses extensions (`dnf`).
2. Démarre et active les services au boot.
3. Crée la base `glpi` (utf8) et l'utilisateur dédié (`community.mysql`).
4. Télécharge et décompresse GLPI (seulement s'il n'est pas déjà présent).
5. Donne les droits à l'utilisateur `apache` et corrige les permissions (755 dossiers / 644 fichiers).
6. Déploie le VirtualHost (`AllowOverride All` pour les `.htaccess` de GLPI).
7. Génère `config_db.php` à partir d'un template.
8. Initialise la base en CLI : `php bin/console db:install --no-interaction`.
9. **Valide** que GLPI répond.

### `update.yml`
Détecte la famille d'OS (`ansible_os_family`) : `apt` pour Debian/Ubuntu, `dnf` pour RHEL/AlmaLinux. Redémarre seulement si le système l'exige.

## Utilisation

```bash
# 1. Installer la collection nécessaire
ansible-galaxy collection install -r requirements.yml

# 2. Vérifier la connexion SSH
ansible all -m ping
ansible all -a "ip a"

# 3. Déployer
ansible-playbook playbooks/https.yml  --ask-become-pass
ansible-playbook playbooks/glpi.yml   --ask-become-pass -e glpi_db_password='MotDePasseFort'

# 4. Mettre à jour tous les serveurs
ansible-playbook playbooks/update.yml --ask-become-pass
```

Le mot de passe de la base n'est **pas** stocké dans le dépôt : il se passe avec `-e` ou se chiffre avec `ansible-vault`.

Les playbooks sont **idempotents** : on peut les relancer autant de fois qu'on veut, Ansible ne refait que ce qui manque (`state: present`, `creates:`, handlers).

## Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| GLPI affichait des erreurs d'écriture (logs, fichiers) | Les fichiers appartenaient à `root` après l'extraction / l'installation | `owner: apache` récursif + permissions 755/644, appliqué aussi **après** `db:install` |
| Erreur « access config table » | GLPI ne trouvait pas sa connexion à la base | Générer `config/config_db.php` via template **avant** `db:install` |
| Avertissement utf8mb4 de GLPI | Encodage de la base | Base créée explicitement en `utf8` / `utf8_unicode_ci` |
| Pas de certificat Let's Encrypt possible | VM en NAT, non joignable depuis Internet | Certificat autosigné généré par `openssl` |

## Ce que j'ai appris

- Organiser un projet Ansible en **rôles réutilisables** (tasks, templates, handlers, defaults).
- Utiliser des **templates Jinja2** pour générer des fichiers de configuration par machine.
- Rendre un playbook **idempotent** et le faire se **valider lui-même**.
- Gérer deux familles de distributions (Debian/apt et RHEL/dnf) dans un même inventaire.
- Diagnostiquer des problèmes de permissions Linux et de configuration applicative (GLPI).

## Améliorations possibles

- Chiffrer les variables sensibles avec `ansible-vault`.
- Ajouter une tâche `firewalld` (ouvrir 80/443) et gérer les contextes **SELinux** pour GLPI.
- Passer GLPI en HTTPS lui aussi et utiliser le dossier `public/` comme DocumentRoot.
- Supprimer `install/install.php` après l'installation.
- Passer la base en `utf8mb4`.

## Captures d'écran

_À ajouter dans `docs/captures/` : exécution des playbooks, page HTTPS d'apache01, page de connexion GLPI._
