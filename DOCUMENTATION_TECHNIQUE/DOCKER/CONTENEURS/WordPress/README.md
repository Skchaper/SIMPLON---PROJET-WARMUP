# Installation WordPress avec Docker, Traefik 3.7 et DNS Windows
Procédure complète d’installation  WordPress avec Docker et Traefik 3.7 déjà installé, sur serveur Debian 13.

Élément
Configuration
Dossier
/opt/wordpress
Domaine local
wordpress.claas.lan
Serveur Docker
10.0.0.14
Conteneur Web
WordPress
Base de données
MariaDB
Reverse proxy
Traefik 3.7
DNS
Windows Server

Important : je pars sur un accès local en HTTP. Ton domaine .lan ne permet pas d'obtenir un certificat Let's Encrypt. On pourra configurer HTTPS avec un certificat de ton autorité de certification locale.
Avant de créer les conteneurs, il faut récupérer le réseau Docker et le nom de l'EntryPoint réellement utilisés par ton Traefik. On ne doit pas modifier sa configuration au hasard.
1. Vérifier Traefik
Sur Debian, exécute :
docker ps
Puis :
docker network ls
Ensuite, repère le nom du conteneur Traefik dans le résultat et exécute, en remplaçant traefik si nécessaire :
docker inspect traefik_default --format '{{json .NetworkSettings.Networks}}'
2. Créer le dossier WordPress
Une fois les informations de Traefik vérifiées :
sudo mkdir -p /opt/wordpress
cd /opt/wordpress
3. Créer le fichier .env
sudo nano /opt/wordpress/.env
Contenu :
DOMAIN=wordpress.claas.lan

DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=CHANGE_MOI_MOT_DE_PASSE
DB_ROOT_PASSWORD=CHANGE_MOI_MOT_DE_PASSE_ROOT
Enregistre avec Ctrl + O, Entrée, Ctrl + X.
4. Créer le fichier Docker Compose
sudo nano /opt/wordpress/docker-compose.yml
Voici la configuration qui sera utilisée, une fois les paramètres réels de Traefik confirmés :
services:

  db:
    image: mariadb:11.4
    container_name: wordpress-db
    restart: unless-stopped

    environment:
      MARIADB_DATABASE: ${DB_NAME}
      MARIADB_USER: ${DB_USER}
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}

    volumes:
      - db_data:/var/lib/mysql

    networks:
      - wordpress_internal

  wordpress:
    image: wordpress:php8.3-apache
    container_name: wordpress
    restart: unless-stopped

    depends_on:
      - db

    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}

      WORDPRESS_CONFIG_EXTRA: |
        define('WP_HOME', 'http://${DOMAIN}');
        define('WP_SITEURL', 'http://${DOMAIN}');

    volumes:
      - wordpress_data:/var/www/html

    networks:
      - wordpress_internal
      - proxy

    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.wordpress.entrypoints=web"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"

volumes:
  db_data:
  wordpress_data:

networks:
  wordpress_internal:
    driver: bridge

  proxy:
    external: true
Attention : proxy et web sont des exemples à confirmer avec ta configuration Traefik. Ne lance pas encore ce Compose si ces valeurs ne correspondent pas à ton installation.
5. Commandes de démarrage (après validation)
cd /opt/wordpress
Vérification :
docker compose config
Téléchargement des images :
docker compose pull
Démarrage :
docker compose up -d
Vérification :
docker compose ps
Logs :
docker compose logs -f
6. Configurer le DNS Windows
Dans la zone DNS claas.lan, créer un enregistrement A :
Nom : wordpress
IP : 10.0.0.14
Tester depuis Windows :
nslookup wordpress.claas.lan
Puis ouvrir :
http://wordpress.claas.lan


