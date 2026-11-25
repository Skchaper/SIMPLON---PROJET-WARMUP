# Procédure complète – Installation de WordPress avec Docker et Traefik 3.7

## 1. Présentation de l'installation

Installation de WordPress avec MariaDB sur Debian 13, derrière un reverse proxy Traefik 3.7 déjà installé.

| Élément | Configuration |
|---|---|
| Système | Debian 13 |
| Dossier | `/opt/wordpress` |
| Domaine local | `wordpress.claas.lan` |
| Serveur Docker | `10.0.0.14` |
| Conteneur Web | WordPress |
| Base de données | MariaDB 11.4 |
| Reverse proxy | Traefik 3.7 |
| DNS | Windows Server |
| Protocole | HTTP |

**Important :**
- Docker et Traefik sont déjà installés.
- Aucun nouveau Traefik ne doit être déployé.
- Le domaine `.lan` est réservé à l'environnement interne.
- Let's Encrypt ne délivre pas de certificat pour `wordpress.claas.lan`.
- Pour HTTPS local, utiliser un certificat émis par une autorité de certification interne.

## 2. Vérification de Traefik

Avant de créer les conteneurs, vérifier le fonctionnement de Docker et identifier le réseau utilisé par Traefik.

### 2.1 Vérifier les conteneurs

```bash
docker ps
```

### 2.2 Vérifier les réseaux Docker

```bash
docker network ls
```

Le réseau identifié pour Traefik est :

```text
traefik_default
```

### 2.3 Vérifier le réseau du conteneur Traefik

Remplacer `traefik` par le nom réel du conteneur si nécessaire.

```bash
docker inspect traefik --format '{{json .NetworkSettings.Networks}}'
```

Vérifier également les EntryPoints configurés dans Traefik. L'EntryPoint utilisé dans les labels WordPress doit correspondre à la configuration existante.

## 3. Création du dossier WordPress

```bash
sudo mkdir -p /opt/wordpress
cd /opt/wordpress
```

## 4. Création du fichier d'environnement `.env`

```bash
sudo nano /opt/wordpress/.env
```

Contenu :

```env
DOMAIN=wordpress.claas.lan

DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=CHANGE_MOI_MOT_DE_PASSE
DB_ROOT_PASSWORD=CHANGE_MOI_MOT_DE_PASSE_ROOT
```

Remplacer les mots de passe par des valeurs fortes et personnelles.

Enregistrer avec :

- `Ctrl + O`
- `Entrée`
- `Ctrl + X`

Protéger le fichier :

```bash
sudo chmod 600 /opt/wordpress/.env
```

## 5. Création du fichier Docker Compose

```bash
sudo nano /opt/wordpress/docker-compose.yml
```

Contenu :

```yaml
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
      - "traefik.docker.network=traefik_default"
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
    name: traefik_default
```

**Attention :** le réseau `traefik_default` est celui identifié dans l'installation. Vérifier que l'EntryPoint `web` existe réellement dans Traefik avant le démarrage.

## 6. Vérification et démarrage

Se placer dans le dossier :

```bash
cd /opt/wordpress
```

Vérifier la syntaxe et les variables :

```bash
docker compose config
```

Télécharger les images :

```bash
docker compose pull
```

Démarrer les conteneurs :

```bash
docker compose up -d
```

Vérifier les conteneurs :

```bash
docker compose ps
```

Consulter les journaux :

```bash
docker compose logs -f
```

## 7. Configuration du DNS Windows Server

Sur le contrôleur de domaine Windows Server :

1. Ouvrir le Gestionnaire DNS.
2. Développer le serveur DNS.
3. Ouvrir `Zones de recherche directes`.
4. Sélectionner la zone `claas.lan`.
5. Créer un nouvel hôte A.

Paramètres :

| Champ | Valeur |
|---|---|
| Nom | `wordpress` |
| Type | A |
| Adresse IP | `10.0.0.14` |

Le nom complet obtenu sera :

```text
wordpress.claas.lan
```

## 8. Vérification DNS

Depuis un poste Windows :

```powershell
nslookup wordpress.claas.lan
```

L'adresse retournée doit être :

```text
10.0.0.14
```

## 9. Accès à WordPress

Ouvrir un navigateur depuis le réseau local :

```text
http://wordpress.claas.lan
```

Suivre l'assistant d'installation WordPress :

1. Choisir la langue.
2. Définir le titre du site.
3. Créer le compte administrateur.
4. Définir un mot de passe sécurisé.
5. Finaliser l'installation.

## 10. Vérification finale

Sur Debian :

```bash
docker compose ps
```

```bash
docker compose logs wordpress
```

```bash
docker compose logs db
```

Vérifier que :

- Le conteneur WordPress est démarré.
- Le conteneur MariaDB est démarré.
- Le réseau Docker `traefik_default` est utilisé.
- Le DNS résout `wordpress.claas.lan`.
- L'interface WordPress est accessible depuis le réseau local.

## 11. Persistance des données

Les données sont conservées dans deux volumes Docker :

| Volume | Contenu |
|---|---|
| `db_data` | Base de données MariaDB |
| `wordpress_data` | Fichiers WordPress |

Ne pas supprimer ces volumes lors d'une opération de maintenance.

**Commande à éviter :**

```bash
docker compose down -v
```

Cette commande supprime les volumes associés au projet et peut entraîner la perte des données.

---

**Résultat attendu :**

WordPress est installé sur Debian 13, exécuté dans Docker, connecté à MariaDB et publié sur le réseau local à travers Traefik 3.7.

Adresse d'accès :

`http://wordpress.claas.lan`
