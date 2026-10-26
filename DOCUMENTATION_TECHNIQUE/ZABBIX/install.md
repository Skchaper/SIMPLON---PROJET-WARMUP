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

Download/Zabbix ![Download/Zabbix(https://www.zabbix.com/fr/download)]

Sélectionnez « Packages Zabbix » puis choisissez la version de Zabbix, votre distribution Linux, sa version, les composants Zabbix « Server, Frontend, Agent 2 », sa base de données et le serveur web voulu. 

Pour ma part, je suis parti sur l’installation de Zabbix 7.4, sur Debian 13 avec une base de données MySQL et Apache comme serveur web.  

