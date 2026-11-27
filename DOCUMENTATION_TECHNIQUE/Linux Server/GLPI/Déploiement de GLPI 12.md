# Déploiement de GLPI 12 sur Proxmox : helpdesk, inventaire et annuaire Active Directory

Documentation du projet : installation de GLPI 12 sur une VM Debian, mise en place d'un flux de support (tickets), inventaire automatique du parc via le GLPI Agent déployé par GPO, et authentification via l'annuaire Active Directory.

## Sommaire

1. [Contexte et architecture](#1-contexte-et-architecture)
2. [Installation de GLPI 12](#2-installation-de-glpi-12)
3. [Configuration post-installation](#3-configuration-post-installation)
4. [Structure du support et flux de tickets](#4-structure-du-support-et-flux-de-tickets)
5. [Inventaire et GLPI Agent](#5-inventaire-et-glpi-agent)
6. [Intégration de l'annuaire Active Directory](#6-intégration-de-lannuaire-active-directory)
7. [Problèmes rencontrés et solutions](#7-problèmes-rencontrés-et-solutions)
8. [Pistes d'amélioration](#8-pistes-damélioration)

---

## 1. Contexte et architecture

Petit projet d'infrastructure virtualisée sous **Proxmox VE**. Objectif : disposer d'un outil de gestion de parc et de support (ITSM) simple, relié à l'annuaire de l'entreprise.

| Élément | Détail |
|---|---|
| Hyperviseur | Proxmox VE |
| Serveur GLPI | VM Debian `SRV-DEB-GLPI` (`10.0.0.13`) |
| Application | GLPI 12.0.0 |
| Pile web | Apache 2.4 + PHP (8.2 minimum requis) + MariaDB |
| Contrôleur de domaine | VM Windows Server `SRV-WIN-DC1` |
| Domaine AD | `claas.lan` |
| Poste client de test | VM Windows 11 (`CLI-WIN-ADM`) |

```
[Poste Windows]--(GLPI Agent, HTTP)-->[SRV-DEB-GLPI : Apache + PHP + MariaDB]
                                              |
                                              +--(LDAP 389)-->[SRV-WIN-DC1 : Active Directory]
[GPO] déploie l'agent sur les postes du domaine
```

---

## 2. Installation de GLPI 12

### 2.1 Prérequis

- PHP 8.2 ou supérieur avec les extensions : `curl, gd, intl, mysql, mbstring, xml, zip, bz2, ldap, opcache`
- Apache avec le module `rewrite`
- MariaDB

```bash
php -v
a2enmod rewrite
```

### 2.2 Base de données

```sql
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE>';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
GRANT SELECT ON mysql.time_zone_name TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
```

### 2.3 Fichiers de l'application

Depuis GLPI 11, l'application est servie par un contrôleur frontal situé dans le dossier `public/`. L'ancienne URL `/install/install.php` n'existe plus : l'assistant se lance depuis la racine du site.

```bash
mv /var/www/html/glpi /var/www/html/glpi_old   # si une ancienne version existe
cd /var/www/html
tar -xzf /tmp/glpi-12.0.0.tgz
chown -R www-data:www-data /var/www/html/glpi
chmod -R 755 /var/www/html/glpi
```

### 2.4 VirtualHost Apache

Fichier `/etc/apache2/sites-available/000-default.conf` :

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/glpi/public

    <Directory /var/www/html/glpi/public>
        Require all granted

        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

La règle de réécriture redirige toutes les URL qui ne correspondent pas à un fichier vers `index.php`. Sans elle, Apache renvoie un 404 sur les routes de GLPI.

```bash
apachectl configtest
systemctl restart apache2
```

### 2.5 Assistant d'installation

Ouvrir `http://srv-deb-glpi/` (la racine, pas `/install/install.php`), choisir la langue, accepter la licence, puis renseigner la connexion à la base (serveur `localhost`, utilisateur `glpiuser`, base `glpidb`).

Comptes créés par défaut à l'installation : `glpi`, `tech`, `normal`, `post-only` (mots de passe par défaut à changer ou désactiver, voir section 3).

---

## 3. Configuration post-installation

### 3.1 Cohérence des fuseaux horaires

Trois horloges interviennent : le système, PHP et MariaDB. Si elles diffèrent, les dates d'exécution des tâches planifiées sont décalées. Tout a été aligné sur `Europe/Paris`.

**Chargement des fuseaux dans MariaDB :**

```bash
mariadb-tzinfo-to-sql /usr/share/zoneinfo | mysql -u root -p mysql
```

**PHP** (Apache et CLI) :

```bash
sed -i 's|^;*date.timezone.*|date.timezone = Europe/Paris|' /etc/php/*/apache2/php.ini /etc/php/*/cli/php.ini
```

**MariaDB** :

```bash
cat > /etc/mysql/mariadb.conf.d/99-timezone.cnf <<'EOF'
[mysqld]
default-time-zone = 'Europe/Paris'
EOF
systemctl restart mariadb
systemctl restart apache2
```

**Activation dans GLPI**, puis choix du fuseau dans le profil utilisateur :

```bash
cd /var/www/html/glpi
sudo -u www-data php bin/console database:enable_timezones
```

Vérification : `SELECT NOW(), UTC_TIMESTAMP(), @@global.time_zone;` doit afficher l'heure locale et `Europe/Paris`.

### 3.2 Tâches automatiques (cron)

Les actions automatiques de GLPI (notifications, alertes, etc.) sont exécutées par le cron système, à condition que leur mode d'exécution soit **CLI**.

```bash
crontab -u www-data -e
```

```
* * * * * /usr/bin/php /var/www/html/glpi/front/cron.php >/dev/null 2>&1
```

Dans GLPI : *Configuration > Actions automatiques*, passer le mode d'exécution des actions en **CLI**.

Contrôle : la colonne « Dernière exécution » doit afficher une heure récente, ou en base :

```sql
SELECT name, mode, frequency, lastrun FROM glpi_crontasks WHERE state=1;
```

### 3.3 Sécurisation des comptes

- Changer le mot de passe du compte `glpi` (Super-Admin).
- Désactiver les comptes de démonstration `tech`, `normal` et `post-only`.
- Conserver un compte local Super-Admin en secours, utilisable si l'annuaire est indisponible.

---

## 4. Structure du support et flux de tickets

### 4.1 Catégories ITIL

*Configuration > Intitulés > Assistance > Catégories ITIL*

| Catégorie | Sous-catégories |
|---|---|
| Incident | Réseau, Serveur, Poste de travail, Logiciel, Accès / Compte |
| Demande | Nouveau matériel, Nouveau compte, Installation logiciel |

Les cases de visibilité (incident / demande) doivent être cochées sur chaque catégorie, sinon elle n'apparaît pas dans le formulaire de ticket.

### 4.2 Groupe et utilisateurs

- **Groupe** `Support N1` (*Administration > Groupes*), avec l'option « chargé de » les **Tickets** cochée.
- **Technicien** : utilisateur avec le profil **Technician** sur l'entité racine, membre de `Support N1`.
- **Demandeur de test** : utilisateur avec le profil **Self-Service**.

Un utilisateur peut avoir plusieurs habilitations et basculer d'un profil à l'autre depuis le menu en haut à droite.

### 4.3 Droits du profil Technician

Pour pouvoir attribuer un ticket, le profil Technician doit disposer des droits suivants (*Administration > Profils > Technician > Assistance*) :

- Mettre à jour (tickets)
- Attribuer un ticket
- Se faire attribuer un ticket

Sans ces droits, le champ « Attribué à » est affiché en lecture seule. Les droits sont rechargés à la connexion : il faut se déconnecter et se reconnecter après modification.

### 4.4 Cycle de vie d'un ticket

```
Nouveau  ->  En cours (attribué)  ->  En attente  ->  Résolu  ->  Clos
```

Test réalisé : création d'un ticket avec un compte Self-Service, attribution au groupe `Support N1` par le technicien, ajout d'une solution, puis clôture.

---

## 5. Inventaire et GLPI Agent

### 5.1 Activation côté serveur

*Administration > Inventaire* : cocher **Activer l'inventaire** et vérifier le réglage d'authentification des agents (voir le point 7 sur l'erreur 403).

### 5.2 Déploiement par GPO sur les postes Windows

Le `.msi` du GLPI Agent et un script d'installation sont déposés dans le partage **NETLOGON** du contrôleur de domaine, qui est lisible par tous les ordinateurs du domaine (`\\SRV-WIN-DC1\netlogon`, équivalent du dossier `scripts` de SYSVOL).

Script `install-glpi-agent.bat` :

```batch
@echo off
sc query "glpi-agent" >nul 2>&1
if %errorlevel%==0 exit /b 0
msiexec /i "\\SRV-WIN-DC1\netlogon\GLPI-Agent-x.x.x-x64.msi" /qn SERVER="http://10.0.0.13/front/inventory.php" RUNNOW=1 ADD_FIREWALL_EXCEPTION=1
```

- Le test `sc query` évite de réinstaller l'agent à chaque démarrage.
- `RUNNOW=1` lance un inventaire dès l'installation.
- Remplacer `x.x.x` par la version exacte du fichier.

**Création de la GPO** (`gpmc.msc`) :

1. Créer la GPO `Deploiement GLPI Agent` et la lier à l'OU contenant les comptes ordinateurs (OU `claas`).
2. *Configuration ordinateur > Stratégies > Paramètres Windows > Scripts (démarrage/arrêt) > Démarrage*, ajouter le script avec son chemin UNC complet.
3. Vérifier que le lien de la GPO est bien **activé** (colonne « Appliqué » à *Oui*).

**Test sur un poste :**

```cmd
gpupdate /force
shutdown /r /t 0
gpresult /r /scope computer
```

Le script de démarrage ne s'exécute qu'au redémarrage. Après le reboot, le service **GLPI Agent** doit être présent dans `services.msc`.

### 5.3 Postes Linux et hôte Proxmox

Les GPO ne s'appliquent pas aux systèmes Linux. Installation manuelle :

```bash
apt install ./glpi-agent_x.x.x-1_all.deb
# /etc/glpi-agent/agent.cfg
#   server = http://10.0.0.13/front/inventory.php
systemctl restart glpi-agent
glpi-agent --force
```

### 5.4 Vérification

- Sur le serveur : les requêtes de l'agent doivent apparaître avec un code 200.

```bash
grep inventory /var/log/apache2/access.log | tail -n 5
```

- Dans GLPI : le poste apparaît dans *Parc > Ordinateurs* (exemple : `CLI-WIN-ADM`, détecté comme machine virtuelle QEMU).

---

## 6. Intégration de l'annuaire Active Directory

### 6.1 Prérequis

Sur le serveur GLPI, extension PHP LDAP et accès au contrôleur de domaine sur le port 389 :

```bash
php -m | grep -i ldap || apt install php-ldap
systemctl restart apache2
nc -zv <IP_DU_DC> 389
```

### 6.2 Compte de service

Dans *Utilisateurs et ordinateurs Active Directory*, création d'un compte `svc-glpi` dans l'OU `COMPTES DE SERVICE` :

- Créer un objet de type **Utilisateur** (et non *Contact*, qui n'a pas de mot de passe et ne peut pas s'authentifier).
- Mot de passe solide, « le mot de passe n'expire jamais » coché.
- Aucun droit supplémentaire : la lecture de l'annuaire est accordée aux utilisateurs du domaine.

### 6.3 Configuration dans GLPI

*Configuration > Authentification > Annuaires LDAP > Ajouter*

| Champ | Valeur |
|---|---|
| Nom | `AD claas` |
| Serveur par défaut | Oui |
| Serveur / Port | IP du DC / `389` |
| Filtre de connexion | `(&(objectClass=user)(objectCategory=person)(!(userAccountControl:1.2.840.113556.1.4.803:=2)))` |
| BaseDN | `DC=claas,DC=lan` |
| RootDN | `svc-glpi@claas.lan` |
| Mot de passe | mot de passe du compte de service |
| Champ identifiant | `samaccountname` |
| Champ e-mail | `mail` |

Le filtre exclut les comptes désactivés. L'onglet **Test** valide la connexion.

### 6.4 Import et droits

- Import : *Administration > Utilisateurs > Liaison annuaire LDAP > Importation de nouveaux utilisateurs*.
- Un utilisateur importé n'a aucun profil par défaut. Une règle d'habilitation lui attribue un profil : *Administration > Règles > Règles d'affectation d'habilitations à un utilisateur* (par exemple profil **Self-Service** sur l'entité racine, récursif).
- Connexion : choisir la source **AD claas** sur la page de login, identifiant sans le suffixe `@claas.lan`.

---

## 7. Problèmes rencontrés et solutions

| Symptôme | Cause | Solution |
|---|---|---|
| 404 sur `/install/install.php` | Cette URL n'existe plus depuis GLPI 11 | Ouvrir la racine du site, DocumentRoot sur `public/`, règle de réécriture Apache |
| « Access denied for user 'glpi'@'localhost' » à l'installation | Mauvais nom d'utilisateur de base de données saisi | Utiliser `glpiuser` |
| Actions automatiques qui ne s'exécutent pas | Horloges système, PHP et MariaDB désynchronisées | Aligner les trois sur `Europe/Paris` et activer les fuseaux dans GLPI |
| `mysql_tzinfo_to_sql: command not found` | Sous MariaDB, la commande porte un autre nom | Utiliser `mariadb-tzinfo-to-sql` |
| Champ « Attribué à » en lecture seule sur un ticket | Droits manquants dans le profil Technician | Cocher les droits d'attribution (section 4.3), puis se reconnecter |
| GPO sans effet sur les postes | Lien de la GPO désactivé (« Appliqué : Non ») | Activer le lien |
| Poste Windows absent du parc malgré l'agent installé | Inventaire désactivé côté serveur, puis réponse HTTP 403 sur `/front/inventory.php` | Activer l'inventaire et ajuster le réglage d'authentification des agents dans *Administration > Inventaire* ; les requêtes sont ensuite passées en 200 |
| Compte de service AD sans mot de passe | Objet créé comme *Contact* au lieu d'*Utilisateur* | Supprimer le contact et créer un vrai utilisateur |

Méthode de diagnostic efficace : lire le journal `access.log` d'Apache pour voir le code HTTP renvoyé à l'agent, puis les journaux de `files/_log/` côté GLPI.

---

## 8. Pistes d'amélioration

- Configurer le **SMTP** (*Configuration > Notifications*) pour l'envoi d'e-mails de tickets.
- Mettre en place des **règles d'attribution automatique** des tickets (par catégorie vers un groupe).
- Passer en **HTTPS** (certificat) pour GLPI, et en **LDAPS** (port 636) pour l'annuaire.
- Renforcer l'authentification des agents d'inventaire plutôt que de la laisser ouverte.
- Ajouter des **règles d'habilitation par groupe AD** pour attribuer automatiquement le profil Technician.
- Définir des **SLA** et des niveaux de service.
- Planifier des **sauvegardes** régulières de la base `glpidb` et des dossiers `config/` et `files/`.

---

## Ressources

- Documentation GLPI : <https://help.glpi-project.org>
- GLPI Agent : <https://github.com/glpi-project/glpi-agent>
