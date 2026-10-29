


## Créer un hôte sur l'interface web de ZABBIX pour la communication AGENT-SERVEUR  

Sur la page web d'administration ZABBIX, allez dans **Surveillance** puis dans **Hôtes** :  

![]()

Une fois arrivé sur la page principale des Hôtes, cliquez sur **Créer un hôte** : 

![]()

La page **Nouvel hôte** s'ouvre, remplissez là comme suit et en l'adaptant à l'équipement que vous souhaitez ajouter à la supervision :  

![]()

1. **Nom de l'hôte :** Nom de la machine que vous allez superviser /!\DOIT CORRESPONDRE A LA VALEUR INDIQUEE DANS LE FICHIER DE CONFIGURATION : zabbix_agent2.conf/!\  
2. **Modèles :** Cliquez sur **Sélectionner**, dans la nouvelle fenêtre cliquez également sur **Sélectionner**, choisissez le groupe qui correspond à votre type d'équipement (ici **Templates/Operating systems**), cochez Windows by Zabbix agent active, recliquez sur **Sélectionner**.  
3. **Groupes d'hôtes :** Cliquez sur **Sélectionner** et adaptez votre choix à l'équipement, puis **Ajouter**  
4. **Interfaces :** Cliquez sur **Ajouter**, puis sur **Agent**, renseignez l'**adresse IP** de l'équipement sur lequel l'agent a été installé, le **nom DNS** renseigné en amont dans le DNS, laissez **IP** grisé et le port 10050 par défaut.  
5. **Description :** Ajoutez une description au besoin.  
6. **Surveillé par :** laissez **Serveur**
7. Et laisser la case **Activé** cochée.
8. Puis cliquez sur **Ajouter**

Une fois l'hôte renseigné et l'ajout préalablement installé (cf. Installation de l'agent dans la documentation install.md de ZABBIX), les informations de l'équipement supervisé devraient remonter.  




