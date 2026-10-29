# Quelques commandes Docker

Docker est installé sur notre serveur Debian 13, mais comment faire pour l'utiliser ? Pour finir, nous allons voir quelques commandes utiles qui vous permettront de débuter avec Docker.  

**Lister les containers Docker en cours d'exécution**

```bash
sudo docker ps
```

Le résultat de la commande permet d'avoir une liste avec différentes informations dont l'ID unique du container, le nom de l'image, et le statut.  

**Lister tous les containers Docker enregistrés sur votre machine, peu importe l'état**

```bash
sudo docker ps -a
```

**Supprimer un container Docker**

À partir d'un ID, il est possible de supprimer un container Docker. Par exemple, cela peut permettre de supprimer le container hello-world (ID 3c745b055853 sur ma machine) car il n'a plus d'intérêt.  

```bash
docker rm 3c745b055853
```

**Lister les images Docker (disponibles en local)**

```bash
docker images
```

Sur ma machine Debian, l'instance Docker dispose de trois images.  

**Supprimer une image Docker**

L'image du container hello-world peut être supprimée, après avoir arrêté puis supprimé le container associé, via cette commande :

```bash
docker rmi hello-world
```

**Démarrer un container Docker**

Avec Docker, on peut démarrer un container existant arrêté avec docker start mais aussi créer un container et le démarrer avec docker run. Un container peut être démarré en l'appelant par son identifiant :  

```bash
docker start d964015967b4
```

**Arrêter un container Docker**

La commande ci-dessus arrête le container Docker avec l'ID 6108d7c37298 récupéré via la commande docker ps.  

```bash
docker stop 6108d7c37298
```

**Télécharger une image Docker à partir de Docker Hub**

Le site Docker Hub référence les images Docker, et il est possible de télécharger une image à partir de cette source. Voici un exemple où l'image d'un container Docker Apache est téléchargée :  

```bash
docker pull httpd
```

Cette image sera ensuite exploitable pour créer un nouveau container Docker.  






