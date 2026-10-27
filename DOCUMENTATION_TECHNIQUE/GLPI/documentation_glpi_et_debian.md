# Documentation : Déploiement de GLPI sur Debian (LAMP Stack)

Ce document répertorie les étapes pas à pas pour installer et configurer un serveur GLPI sur une machine virtuelle Debian (hébergée sur Proxmox), en utilisant une pile LAMP (Linux, Apache, MariaDB, PHP 8.3).

---

## 1. Préparation du Système et Dépôts

### 1.1 Configuration des sources APT
Vérifiez et mettez à jour le fichier `/etc/apt/sources.list` pour vous assurer d'avoir les dépôts officiels nécessaires.

```bash
apt update
apt upgrade -y
```

### 1.2 Installation du dépôt PHP de Sury (pour PHP 8.3)
GLPI 10 nécessite une version de PHP compatible (PHP 8.3). La version par défaut de Debian pouvant être trop récente ou divergente, nous utilisons le dépôt de référence d'Ondřej Surý.

```bash
apt install apt-transport-https lsb-release ca-certificates wget curl -y
curl -sS https://packages.sury.org/php/apt.gpg | gpg --dearmor | tee /etc/apt/trusted.gpg.d/php.gpg > /dev/null
sh -c 'echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
apt update
```

---

## 2. Installation de la Pile LAMP

### 2.1 Installation d'Apache et MariaDB
```bash
apt install apache2 mariadb-server -y
```

### 2.2 Installation de PHP 8.3 et des extensions requises
GLPI requiert plusieurs extensions PHP pour fonctionner correctement (GD, cURL, Mbstring, XML, Intl, Zip, LDAP, etc.).

```bash
apt install php8.3 php8.3-mysql php8.3-gd php8.3-curl php8.3-mbstring php8.3-xml php8.3-intl php8.3-zip php8.3-ldap php8.3-bcmath php8.3-apcu libapache2-mod-php8.3 -y
```

Activez le module PHP 8.3 pour Apache :
```bash
a2dismod php* 2>/dev/null || true
a2enmod php8.3
systemctl restart apache2
```

---

## 3. Configuration de la Base de Données MariaDB

Il est indispensable de créer un utilisateur dédié et une base de données pour GLPI afin d'éviter les erreurs de connexion lors de l'assistant d'installation.

1. Connectez-vous à la console MariaDB en tant que `root` :
   ```bash
   mysql -u root -p
   ```

2. Exécutez les requêtes SQL suivantes pour créer la base, l'utilisateur et lui attribuer les droits :
   ```sql
   CREATE DATABASE IF NOT EXISTS glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'VotreMotDePasseSecurise';
   GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
   FLUSH PRIVILEGES;
   EXIT;
   ```

---

## 4. Téléchargement et Déploiement de GLPI

1. Téléchargez l'archive officielle de GLPI (version 10.0.16 par exemple) dans le dossier `/tmp` :
   ```bash
   cd /tmp
   wget https://github.com/glpi-project/glpi/releases/download/10.0.16/glpi-10.0.16.tgz
   ```

2. Décompressez l'archive dans le répertoire racine d'Apache :
   ```bash
   tar -zxvf glpi-*.tgz -C /var/www/html/
   ```

3. Attribuez les droits de lecture/écriture au serveur web (`www-data`) :
   ```bash
   chown -R www-data:www-data /var/www/html/glpi
   ```

---

## 5. Finalisation via l'Assistant Web

1. Ouvrez un navigateur web et rendez-vous à l'adresse : `http://<IP_DU_SERVEUR>/glpi`
2. Suivez les instructions de l'assistant graphique :
   * **Langue** : Choisissez votre langue.
   * **Licence** : Acceptez les termes.
   * **Prérequis** : Vérifiez que tous les voyants sont au vert (notamment grâce à PHP 8.3).
   * **Connexion à la base de données** : 
     * **Serveur SQL** : `localhost` (ou `127.0.0.1`)
     * **Utilisateur SQL** : `glpiuser`
     * **Mot de passe SQL** : `VotreMotDePasseSecurise`
   * **Sélection de la base** : Choisissez `glpidb`.
3. Terminez l'installation.

### 🔑 Identifiants par défaut (première connexion) :
* **Compte administrateur** : `glpi` / `glpi`
* **Compte technicien** : `tech` / `tech`
* **Compte normal** : `normal` / `normal`
* **Compte post-only** : `post-only` / `post-only`

*(Il est fortement recommandé de modifier ou de sécuriser ces mots de passe dès la première connexion).*