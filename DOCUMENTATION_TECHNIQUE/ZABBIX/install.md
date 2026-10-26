# Installation de ZABBIX

## 1. Prérequis

**ISO :** Distribution Linux (Red Hat, Debian, Oracle Linux, Alma Linux, Ubuntu ...)  

**Compatibilité agents :**  
- agent1 (original de Zabbix) : Linux / Windows / macOS / Unix (Solaris, IMB AIX, HP-UX, Solaris, freeBSD...)  
- agent2 : Linux / Windows  

**Taille de l'installation :** Petite  
    Nombre de métriques surveillées : 1000  
    Cœurs CPU/vCPU : 2  
    Mémoire (Gio) : 8  
    Base de données : Serveur Mysql, serveur Percona, serveur MariaDB, PostgreSQL  
    PHP : Version 8.0.0 minimum  
    Serveur Web : Apache 2.4 minimum / Nginx 1.20 minimum  

**Choix :** Zabbix, Apache2, PHP et MySQL  

**Installation choisie :** Dépôts officiels ZABBIX  

**Installations non retenue :**  
- Installer le code source et le compiler soi-même pour avoir un contrôle total de bout en bout sur l’installation.  
- Installation sous forme d’image Docker > Conteneur  
- Installation sous forme d’image Cloud > Images cloud pour son installation chez différents fournisseurs

## 2. Installation de ZABBIX sur Linux

https://www.zabbix.com/fr/download

Sélectionnez « Packages Zabbix » puis choisissez la version de Zabbix, votre distribution Linux, sa version, les composants Zabbix « Server, Frontend, Agent 2 », sa base de données et le serveur web voulu. 

Pour ma part, je suis parti sur l’installation de Zabbix 7.4, sur Debian 13 avec une base de données MySQL et Apache comme serveur web.  

### Récupérer le dépôt ZABBIX

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb  
dpkg -i zabbix-release_latest_7.4+debian13_all.deb  
apt update  
```

### Installer le serveur ZABBIX / le Frontend / l'AGENT

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

### Créer la base de données initiale

Assurez-vous d'avoir un serveur de base de données prêt à l'emploi et actif.  

Tapez les commandes suivantes dans la console de la base de données :  

```bash
# mysql -uroot -p
password
mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'password';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```

Sur le serveur Zabbix, importez le schéma initial et les données. Vous serez invité à saisir votre nouveau mot de passe :  

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Désactivez l'option log_bin_trust_function_creators après l'importation du schéma de base de données.  

```bash
# mysql -uroot -p
password
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

### Configurer la base de données pour le serveur ZABBIX

Modifiez le fichier /etc/zabbix/zabbix_server.conf  

```bash
DBPassword=password
```

### Démarrer les processus du serveur et de l'agent ZABBIX

Démarrez les processus du serveur et de l'agent Zabbix et configurez les pour qu'ils démarrent au démarrage du système.  

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

### Ouvrir la page Web de l'interface utilisateur de Zabbix

L'URL par défaut de l'interface utilisateur de Zabbix lors de l'utilisation du serveur web Apache est http://host/zabbix  







