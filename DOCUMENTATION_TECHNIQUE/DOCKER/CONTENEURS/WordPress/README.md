Installation WordPress avec Docker, Traefik 3.7 et DNS Windows
1. Création du dossier WordPress
sudo mkdir -p /opt/wordpress
sudo chown -R $USER:$USER /opt/wordpress
cd /opt/wordpress


2. Création du fichier .env
nano .env


DOMAIN=wordpress.home.arpa
MARIADB_DATABASE=wordpress
MARIADB_USER=wpuser
MARIADB_PASSWORD=REMPLACER_PAR_UN_MOT_DE_PASSE_FORT
MARIADB_ROOT_PASSWORD=REMPLACER_PAR_UN_AUTRE_MOT_DE_PASSE_FORT


chmod 600 .env


3. Création du réseau Docker
docker network ls


Si le réseau proxy n'existe pas :
docker network create proxy


4. Création du fichier Docker Compose
nano docker-compose.yml


services:
  wordpress-db:
    image: mariadb:11.4
    container_name: wordpress-db
    restart: unless-stopped

    environment:
      MARIADB_DATABASE: ${MARIADB_DATABASE}
      MARIADB_USER: ${MARIADB_USER}
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}

    volumes:
      - wordpress_db_data:/var/lib/mysql

    networks:
      - wordpress-internal

    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10

  wordpress:
    image: wordpress:php8.3-apache
    container_name: wordpress-web
    restart: unless-stopped

    depends_on:
      wordpress-db:
        condition: service_healthy

    environment:
      WORDPRESS_DB_HOST: wordpress-db:3306
      WORDPRESS_DB_USER: ${MARIADB_USER}
      WORDPRESS_DB_PASSWORD: ${MARIADB_PASSWORD}
      WORDPRESS_DB_NAME: ${MARIADB_DATABASE}

    volumes:
      - wordpress_data:/var/www/html

    networks:
      - wordpress-internal
      - proxy

    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy"
      - "traefik.http.routers.wordpress.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.wordpress.entrypoints=websecure"
      - "traefik.http.routers.wordpress.tls=true"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"

networks:
  wordpress-internal:
    internal: true

  proxy:
    external: true

volumes:
  wordpress_db_data:
  wordpress_data:


Attention : remplacer proxy par le nom réel du réseau Docker de Traefik si différent, et websecure par le nom réel de son EntryPoint HTTPS si nécessaire.
5. Vérification de la configuration
docker compose config


6. Téléchargement des images
docker compose pull


7. Démarrage des conteneurs
docker compose up -d


8. Vérification des conteneurs
docker ps


docker compose ps


9. Consultation des journaux
docker compose logs -f


docker logs wordpress-web


docker logs wordpress-db


10. Vérification du réseau Docker
docker network inspect proxy


11. Configuration DNS Windows Server
Dans PowerShell :
Add-DnsServerPrimaryZone -Name "home.arpa" -ReplicationScope "Domain"


Créer l'enregistrement DNS :
Add-DnsServerResourceRecordA -ZoneName "home.arpa" -Name "wordpress" -IPv4Address "IP_DEBIAN"


Remplacer IP_DEBIAN par l'adresse IP réelle du serveur hébergeant Traefik.
12. Vérification DNS depuis Windows
nslookup wordpress.home.arpa


13. Vérification de l'accès HTTPS
curl -I https://wordpress.home.arpa


14. Administration Docker
Arrêter les conteneurs :
docker compose stop


Redémarrer les conteneurs :
docker compose restart


Arrêter et supprimer les conteneurs :
docker compose down


Redémarrer le déploiement :
docker compose up -d


15. Sauvegarde de la base de données
docker compose exec -T wordpress-db sh -c \
'exec mariadb-dump -u"$MARIADB_USER" -p"$MARIADB_PASSWORD" "$MARIADB_DATABASE"' \
> sauvegarde-wordpress.sql


16. Mise à jour des conteneurs
docker compose pull
docker compose up -d


Adresse du site :
https://wordpress.home.arpa


Important : pour que HTTPS fonctionne sans avertissement, Traefik doit disposer d'un certificat signé par ton autorité de certification locale, avec wordpress.home.arpa comme nom DNS, et les postes Windows doivent faire confiance à cette autorité. Le DNS seul ne suffit pas à activer HTTPS.
