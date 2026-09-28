# Notes — explication du code bloc par bloc

Mes notes de compréhension, rédigées pendant le labo.

## Playbook HTTPS — `playbooks/https.yml`

```yaml
- name: Apache HTTPS pour apache01
  hosts: https        # ce play s'applique au groupe [https] de l'inventaire
  become: true        # Ansible exécute les tâches avec sudo
  roles:
    - apache_https    # exécute roles/apache_https/tasks/main.yml
```

## Rôle `apache_https`

**Bloc 1 — Installer Apache et les outils SSL**
`apt` installe `apache2`, `openssl` et `ssl-cert`.
- `state: present` → installés seulement s'ils ne le sont pas déjà.
- `update_cache: yes` → met à jour la liste des paquets avant d'installer (équivalent de `apt update`).

**Bloc 2 — Activer les modules**
`a2enmod ssl` (HTTPS) et `a2enmod headers` (gestion des en-têtes HTTP).
`creates:` évite de relancer la commande si le module est déjà actif.

**Bloc 3 — Dossier du certificat**
`file` crée `/etc/apache2/ssl` s'il n'existe pas. `mode: '0755'` → rwx pour le propriétaire, rx pour les autres.

**Bloc 4 — Certificat autosigné**
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 ...
```
- `req -x509` → demande un certificat autosigné (format X.509).
- `-nodes` → « no DES » : la clé n'est pas chiffrée, donc pas de passphrase → Apache démarre sans mot de passe.
- `-days 365` → valide 1 an.
- `-subj` → remplit les champs du certificat sans poser de questions.
- La machine est en NAT : Let's Encrypt ne peut pas la valider, d'où le certificat autosigné.

**Bloc 5 — DocumentRoot et index**
`file` crée le dossier du site, `copy` copie `roles/apache_https/files/index.html` à la racine du site.

**Bloc 6 — VirtualHost (template Jinja2)**
- Premier bloc : écoute sur **443**, utilise le certificat autosigné, sert les fichiers du site.
- Deuxième bloc : écoute sur **80** et redirige tout vers HTTPS.
- `{{ inventory_hostname }}` est remplacé par le nom de la machine dans l'inventaire (`apache01`).
- `template` copie le fichier sur le serveur avec les variables remplacées.

**Bloc 7 — Activer le site**
- `a2ensite https01` → active le nouveau site.
- `a2dissite 000-default` → désactive le site par défaut d'Apache.
- Apache est redémarré par un **handler** (seulement si quelque chose a changé) et `enabled: true` le lance au boot.

## Playbook GLPI — `playbooks/glpi.yml`
Appelle le rôle `glpi` et l'applique au groupe `[glpi]` avec sudo.

## Rôle `glpi`

**Bloc 1 — Paquets**
`dnf` remplace `apt` sur AlmaLinux. `httpd` = Apache sur les distributions Red Hat.

**Bloc 2 — Services**
`service` démarre MariaDB et httpd. `enabled: true` → ils redémarrent automatiquement au boot.
Sans `enabled: true`, ça marche maintenant, mais après un reboot le service pourrait ne pas démarrer.

**Bloc 3 — Base de données**
- `mysql_db` crée la base `glpi` en `utf8` / `utf8_unicode_ci` (évite l'avertissement utf8mb4 de GLPI).
- `mysql_user` crée l'utilisateur `glpi` avec `priv: "glpi.*:ALL"` → tous les droits sur cette base seulement.

**Bloc 4 — Téléchargement**
- `get_url` (équivalent de `wget`) télécharge l'archive dans `/tmp` pour ne pas polluer `/var/www`.
- `unarchive` décompresse dans `/var/www/html/`.
  - `remote_src: true` → par défaut, le module pense que l'archive est sur la machine Ansible ; ici elle est sur la machine distante.
  - `creates:` → ne redécompresse pas si GLPI est déjà présent.

**Bloc 5 — Permissions** (l'erreur qui m'a le plus fait galérer)
GLPI doit pouvoir lire/écrire ses logs et ses fichiers : propriétaire `apache:apache`, dossiers en 755, fichiers en 644.
Équivalent manuel :
```bash
sudo chown -R apache:apache /var/www/html/glpi
sudo find /var/www/html/glpi -type d -exec chmod 755 {} \;
sudo find /var/www/html/glpi -type f -exec chmod 644 {} \;
```

**Bloc 6 — VirtualHost GLPI**
Apache écoute sur le port 80 et sert GLPI depuis `/var/www/html/glpi`.
`AllowOverride All` → Apache tient compte des fichiers `.htaccess` de GLPI (réécritures, protections).
`notify: Restart Apache GLPI` → déclenche le handler qui redémarre Apache.

**Bloc 7 — `config_db.php`**
Généré depuis un template dans `glpi/config/`. C'est ce fichier que `bin/console db:install` lit pour se connecter à la base (sinon : erreur « access config table »).

**Bloc 8 — Installation de la base en CLI**
```
php bin/console db:install --no-interaction
```
- `chdir` → Ansible se place dans `/var/www/html/glpi`.
- `php` → exécute l'interpréteur PHP en ligne de commande.
- `bin/console` → script qui contient toutes les commandes GLPI.
- `db:install` → lit `config_db.php`, se connecte à la base, crée toutes les tables, insère les données de base (dont `glpi_configs`) et crée les comptes par défaut.
- `--no-interaction` → installe sans poser de questions (indispensable avec Ansible).

**Bloc 9 — Handler**
`Restart Apache GLPI` redémarre httpd seulement quand une tâche l'a notifié.

## Concepts clés

- **`state: present`** → « je veux que cet élément existe ». S'il existe déjà, Ansible ne refait rien. On peut rejouer le playbook sans casser la configuration (**idempotence**).
- **Handler** → tâche lancée seulement si une autre tâche a changé quelque chose (ex. : redémarrer Apache après un changement de config).
- **Template Jinja2** → fichier de config avec des variables, rempli différemment pour chaque machine.
