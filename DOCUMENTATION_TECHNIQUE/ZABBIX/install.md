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

Pour notre part, nous sommes parti sur l’installation de Zabbix 7.4, sur Debian 13 avec une base de données MySQL et Apache comme serveur web.  

![.png]()

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

## 3. Installation de l'agent ZABBIX  

### Installation sur un serveur Debian 13  

Passons maintenant à l’installation de l’agent 2 sur **SRV-DEB-DOCKER** par exemple, afin que des données commencent à remonter dans Zabbix. Retournez sur le site de Zabbix (https://www.zabbix.com/download_agents), puis allez dans « Packages Zabbix ». Sélectionnez les informations liées à votre distribution puis sélectionnez « Agent 2 » en composant :  

![.png]()

Puis suivez les commandes de la page de ZABBIX :  

**Install and configure Zabbix for your platform**  

**a. Install Zabbix repository**  

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.4+debian13_all.deb
dpkg -i zabbix-release_latest_7.4+debian13_all.deb
apt update
```

**b. Install Zabbix agent 2**  

Install zabbix-agent2 package.  

```bash
apt install zabbix-agent2
```

**c. Install Zabbix agent 2 plugins**  

You may want to install Zabbix agent 2 plugins.  

```bash
apt install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
```

**d. Start Zabbix agent 2 process**  

Start Zabbix agent 2 process and make it start at system boot.  

```bash
systemctl restart zabbix-agent2
systemctl enable zabbix-agent2
```

Une fois le service zabbix-agent2 téléchargé et démarré sans erreur, modifier le fichier suivant :  

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

Pour un agent passif cherchez la ligne Server= (CTRL + F dans nano) et remplacez l'IP par celle du serveur ZABBIX (ici, 10.0.0.12) :  

![.png]()

Pour un agent actif cherchez la ligne ServerActive= (CTRL + F dans nano) et remplacez l'IP par celle du serveur ZABBIX (ici, 10.0.0.12) et le Hostname= par le nom d'hôte renseigné préalablement dans la partie interface Web de ZABBIX ([cf. USER_GUIDE.md de ZABBIX partie création d'un hôte], ici, SRV-DEB-DOCKER):  

![.png]()

Enfin redémarrez le service pour qu'il charge la nouvelle configuration :  

```bash
sudo systemctl restart zabbix-agent2
```




### Installation sur un Windows Serveur 2022  


### Installation sur un PC Windows 11  





