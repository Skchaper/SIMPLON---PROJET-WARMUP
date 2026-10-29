# Installation du reverse proxy Traefik

## Prérequis pour Traefik

Pour la suite, il faut un serveur (Linux) sur lequel Docker et Docker Compose sont installés. Le tutoriel s’applique aussi bien sur un VPS cloud que sur un serveur physique local.  

## Prise en main de Traefik

**Créer la racine du projet**  

Nous allons commencer par créer un dossier pour stocker les données de l'application Traefik.  

```bash
mkdir /opt/docker-compose/traefik
cd /opt/docker-compose/traefik
```

À la racine de ce dossier, nous allons créer le fichier **docker-compose.yml**.  

En guise de point de départ, nous allons utiliser le **fichier Docker Compose** disponible dans la documentation de Traefik. La **version 3.6 de Traefik** sera utilisée, il s'agit de la dernière version stable actuelle. **Deux ports sont exposés** au niveau de ce conteneur : **80**, pour bénéficier d'un entryPoint en HTTP (pour publier une application en HTTP) et **8080** pour l'accès au tableau de bord de Traefik.  

```bash
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.6
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Trois commandes sont aussi associées à l'exécution de ce conteneur en tant qu'arguments :  

**--api.insecure=true	:** Active le dashboard web de Traefik, sans authentification (sur le port 8080 par défaut).  
**--providers.docker=true	:** Active le Provider Docker de Traefik pour découvrir automatiquement les services / labels.  
**--entrypoints.web.address=:80	:** Déclare un entryPoint nommé web, qui écoute sur port 80 dans le conteneur. Idéal pour les accès HTTP.  

Enfin, une dernière ligne importante est intégrée à ce fichier : **/var/run/docker.sock:/var/run/docker.sock**. Elle permet à Traefik d’accéder au socket Docker pour "voir" en temps réel les autres conteneurs. C'est important pour l'intégration de Traefik avec Docker, notamment pour lire les **labels**.  

**Créer un réseau Docker pour Traefik**  

Vous remarquerez que ce Docker Compose ne contient pas de directives pour la connexion au réseau du conteneur. C'est la première évolution que nous allons lui apporter. Nous allons créer un réseau Docker nommé frontend, ce qui me semble être un nom adapté dans le contexte d'un reverse proxy.  

```bash
docker network create frontend
```

Une fois le réseau créé, vérifiez qu'il est bien présent :  

```bash
docker network ls
```

Dans le fichier Docker Compose, nous allons associer ce réseau au conteneur Traefik. Il sera déclaré comme un réseau externe (**external: true**) pour ne pas que Docker cherche à le créer.  

```bash
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.6
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - frontend
networks:
  frontend:
    external: true
```

**Le fichier traefik.yml**  

La configuration globale et statique de Traefik se déclare par l'intermédiaire du fichier **traefik.yml**, au format YAML.  

Nous allons ajouter une ligne sous **volumes** pour déclarer le fichier de configuration de Traefik, afin qu'il soit persistant et modifiable, tout en étant lu par le conteneur. Par la suite, nous manipulerons à plusieurs reprises le fichier **traefik.yml** pour ajouter différentes directives.  

```bash
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
```

Le fichier doit donc être créé à cet emplacement : **/opt/docker-compose/traefik/traefik.yml**.  

Éditez le fichier et ajoutez le code précisé ci-dessous pour définir les bases de **Traefik en mode proxy inverse pour HTTP et HTTPS**.  

Nous désactivons l’envoi de statistiques anonymisées (**sendAnonymousUsage**) et la vérification automatique de nouvelles versions (**checkNewVersion**). Nous activons le tableau de bord interne (port 8080) en mode non sécurisé (pas adapté à la production !) afin de pouvoir y accéder dans un premier temps pour visualiser les routes de Traefik. Enfin, on déclare deux entryPoints : un pour le trafic HTTP classique sur le port 80 (nommé **web**), et un second pour le trafic HTTPS sécurisé sur le port 443 (nommé **websecure**). Cela prépare Traefik à écouter sur ces deux ports et à accepter des requêtes entrantes que l’on pourra ensuite router vers nos conteneurs.  

```bash
# Configuration statique de Traefik
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true # Dashboard Traefik : 8080
  insecure: true

entryPoints:
  web:
    address: ":80" # EntryPoint pour HTTP
  websecure:
    address: ":443" # EntryPoint pour HTTPS
```

Enregistrez et fermez le fichier.  

**Pour éviter les doublons entre la configuration statique et les arguments en ligne de commande**, je vous invite à supprimer la ligne command et les trois lignes associées dans le fichier Docker Compose, à savoir :  

```bash
 command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
```

**Intégrer Docker à Traefik**  

Dans le fichier Docker Compose, le nécessaire a été fait pour que le conteneur Traefik dispose d'un accès au socket de Docker (**/var/run/docker.sock**). Mais ce lien n'est pas exploité en l'état actuel de la configuration. Dans le fichier de configuration⁣ **traefik.yml**, il est nécessaire d'ajouter un nouveau fournisseur (Provider) correspondant à Docker.  

Éditez de nouveau le fichier **traefik.yml** et ajoutez ce bloc à la fin du fichier :  

```bash
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

La directive **exposedByDefault** joue un rôle important ! Si elle est sur **false** comme ici, cela signifie que **Traefik va détecter les conteneurs Docker sans les exposer automatiquement** : seuls les conteneurs qui auront **des labels Traefik** seront pris en compte et routés, ce qui évite d’exposer par erreur un service qui ne devrait pas être accessible depuis l’extérieur.  

Désormais, enregistrez la configuration et lancez la construction du **conteneur Traefik** (depuis le répertoire **/opt/docker-compose/traefik**).  

```bash
docker compose up -d
```

**Accéder au tableau de bord Traefik**  

Ouvrez un navigateur web et **accédez au tableau de bord de Traefik** via l'adresse IP de l'hôte Docker sur le port **8080**. Par exemple : **http://iphôte:8080**. Ce tableau de bord ne permet pas d'effectuer la configuration en ligne, mais il présente l'avantage d'afficher l'état de la configuration actuelle. Vous devriez notamment **visualiser les 3 entryPoints de notre configuration Traefik** (celui du dashboard est automatique quand ce dernier est activé).  


