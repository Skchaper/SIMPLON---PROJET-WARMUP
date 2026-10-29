# I. Présentation

Dans ce tutoriel, nous allons voir comment installer Docker sur Debian 12 ou Debian 13 afin d'utiliser ses premiers containers sous  Linux ! Cette installation pas-à-pas repose sur la méthode officielle décrite dans la documentation de Docker. Au-delà des dépendances que nous allons installer et de Docker en lui-même, les ressources de votre machine seront déterminantes pour exécuter plus ou moins de containers Docker.  

L'utilisation d'un container reste très pratique pour tester rapidement un  logiciel sans affecter la machine hôte, en phase de développement, etc. Un container Docker va permettre d'empaqueter facilement des applications afin de les rendre utilisables sur différentes plateformes. De nos jours, c'est une technologie populaire et appréciée et que l'on utilise aussi bien on-premise que sur les environnements Cloud.  

Pour fonctionner, Docker s'appuie sur différents composants qu'il est important de connaître :  

- **Docker Engine** ou **Docker Daemon** correspondant au processus qui fait tourner Docker sur le système, en charge de la génération et l'exécution des containers  
- **Docker Registry** est un emplacement de stockage pour héberger les images de containers Docker (il peut être public ou privé)  
- **Docker Image** correspondant à un fichier qui contient la définition d'un container Docker (dépendances, configuration, etc.)  
- **Docker Client** correspondant à l'utilitaire en ligne de commande qui va permettre de gérer les containers (commande "docker")  
- **Docker Container** correspondant aux containers en eux-mêmes, tout en sachant qu'une image peut permettre de créer plusieurs containers avec chacun un identifiant unique  

La documentation de Docker est disponible à cette adresse : docs.docker.com  

# II. Installer Docker sur Debian 13

Ce tutoriel a été testé sur la version suivante de Debian : Debian 13.0. Il fonctionne également sur Debian 11 et Debian 12, version pour laquelle il a été écrit initialement.  

## A. Installer les dépendances de Docker

Premièrement, nous devons installer les dépendances nécessaires au bon fonctionnement de Docker. Commençons par mettre à jour le cache des paquets :  

```bash
sudo apt-get update
```

Puis, exécutez la commande ci-dessous pour installer les paquets :  

```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg2
```

Une fois cette étape effectuée, passez à la suite.

## B. Ajouter le dépôt officiel Docker

Deuxièmement, nous devons ajouter le dépôt officiel de Docker à notre machine Debian afin de pouvoir récupérer les sources. Commençons par récupérer la clé GPG qui nous permettra de valider les paquets récupérés depuis le dépôt Docker :  

```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Ensuite, on ajoute le dépôt Docker à la liste des sources de notre machine :  

```bash
sudo echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

Pour finir, nous devons mettre à jour le cache des paquets pour prendre en compte les paquets de ce nouveau dépôt :  

```bash
sudo apt-get update
```

Dans la sortie de la commande précédente, vous devriez voir le nom de votre version de Debian apparaître. Si vous travaillez sur Debian 13, vous verrez debian trixie.  

Nous allons pouvoir passer à l'installation de Docker.  

## C. Installation des paquets Docker

Troisièmement, c'est l'installation de Docker qui doit être réalisée. Trois paquets sont à installer sur notre hôte pour bénéficier de l'ensemble des composants. Voici la commande à exécuter :  

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Si vous souhaitez que Docker démarre automatiquement avec votre machine Debian, la commande suivante doit être exécutée :  

```bash
sudo systemctl enable docker
```

## D. Docker est-il bien installé ?

L'installation des paquets est terminée, mais Docker est-il correctement installé ? Pour répondre à cette question, vous pouvez regarder le statut de Docker, ce qui sera une première indication si le service est identifié sur la machine.  

```bash
sudo systemctl status docker
```

Ensuite, le meilleur moyen de vérifier si Docker est installé, c'est d'exécuter le container nommé hello-world. La commande ci-dessous permettra de télécharger l'image de ce container et de l'exécuter.  

```bash
sudo docker run hello-world
```

Lorsque ce container sera exécuté, le message Hello from Docker! sera retourné dans la console. C'est le signe que les différents composants sont opérationnels et que Docker a pu générer et exécuter le container.  

## E. Quelle est la version de Docker installée ?

Pour finir, sachez qu'à tout moment, vous pouvez voir quelle est la version de Docker que vous utilisez grâce à la commande suivante :  

```bash
docker --version
```
EX : Docker version 28.4.0, build d8eb465




