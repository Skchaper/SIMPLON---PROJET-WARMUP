<div align="center"><H1> Projet WARMUP - AIS - FONDAMENTAUX DES SYSTEMES ET RESEAUX </H1></div>

## 1 Objectifs
• Révision et approfondissement des compétences fondamentales en systèmes et réseaux  
• Mise en place d’une infrastructure complète de maquettage  
• Acclimatation au travail en mode projet  
• Rédaction de documentation technique et projet  

## 2 Prérequis
• Accès au serveur Proxmox de la promo

## 3 Tâches à réaliser
• Conception du réseau  
• Mise en œuvre de la sécurité du réseau  
• Services Linux  
• Services Windows  

## 4 Organisation
• Le formateur jouera le rôle de la MOA, vous pourrez le solliciter autant que nécessaire en lui
proposant des RDV (prévoir 1 demi-journée d’avance)  
• L’ensemble des rendus devra être restitué au plus tard le 13 mars 2024 soit un total de 15 jours
de présence en formation)  

## 5 Livrables

• Dossier technique complet comportant :  
  o Argumentation des choix techniques  
  o Devis pour le matériel virtuel ou physique nécessaire  
  o Schéma réseau détaillé  
  o Plan d'adressage IP  
  o Informations nécessaires à l’administration de l’ensemble des services  

• Documentation projet comportant :  
Des modèles des documents suivants vous seront fournis par le formateur  
  o Charte de définition du projet  
  o Dossier de montage du projet  
  o Cahier des charges fonctionnel  
  o Tableau des parties prenantes  
  o RACI  
  o WBS  
  o Matrice des risques  
  o Post-mortem (à rédiger en toute fin de projet)  

## 6 ToDo

### A. Conception du Réseau
• Elaboration d'un schéma réseau détaillé, incluant la topologie physique et logique (plans des
locaux disponibles auprès de la MOA)  
o Se baser sur le fichier PacketTracer à récupérer auprès du formateur  
• Création d'un plan d'adressage IP pour les différents segments du réseau (à fournir dans le DT)  
• Configuration des équipements sur PacketTracer  
o Configuration de base des équipements (hostname, bannière, sécurisation console,
accès SSH...)  
o VLAN  
o Agrégats de lien  
o Sécurité MAC  
o NAT/PAT  
o Routage inter-vlan sur switch N3  
o Routage  
o ToIP  
o Services DHCP/TFTP/http  
o Relais DHCP  
• Souscription (factice) argumentée du service WAN (Wide Area Network) auprès d'un
fournisseur d'accès Internet (FAI) à intégrer dans le dossier technique  

### B. Sécurité réseau  
• Création d’une zone réseau pour le groupe dans le PfSense du Proxmox  
• Définition des règles de filtrage (matrice de flux + configuration)  
• Mise en place d'un proxy (indépendant du PfSense) avec whitelist pour contrôler l'accès à
Internet  
• Installation et configuration des VPN  
o OpenVPN pour le client-to-site  
o IPSec pour le site-to-site vers une VM de votre ordinateur personnel  

### C. Serveurs Linux
• Mise en place d'un serveur web avec WordPress
• Déploiement et configuration d'un serveur GLPI (Gestion Libre de Parc Informatique)
• Déploiement d'un serveur de fichiers
• Déploiement et configuration d’un serveur de supervision (mise en supervision des différents
serveurs de l’infrastructure)

### D.Serveurs Windows :
Un maximum d’opérations devra être réalisées en lignes de commandes PowerShell (joindre vos
commandes dans le dossier technique)
• Définition d’une convention de nommage pour l’ensemble des objets AD
• Configuration d'un Active Directory (AD) sur Windows Server 2022
• Mise en place de la réplication avec un serveur core
• Mise en place de services DNS et DHCP
• Service WDS, RDS, Serveur de fichiers
• Configuration d'un client Windows 10 et jonction au domaine pour les tests
• Configuration des ordinateurs, comptes, groupes, GPO, quotas
o Un objet PC créé et attribué par utilisateur (joindre un tableau d’attribution dans le
DT)

### Utilisateurs et groupes

<img width="855" height="249" alt="chrome_iuML2e1pnO" src="https://github.com/user-attachments/assets/b50df9e9-f514-486f-9bb8-e56346312251" />

### Lecteurs réseau
• Configuration des lecteurs réseau  
o Un dossier « Commun » doit être partagé. Le contenu doit être disponible en lecture
– écriture – modification pour tous les services. Le dossier « Commun » ne peut pas
être supprimé sauf par le service informatique  
o Chaque service doit avoir un répertoire partagé nommé « Commun_nom_du_service»
attribué à chacun des salariés lors de sa connexion réseau (permissions en lecture –
écriture - modification des sous dossiers mais pas du dossier « racine »)  
o Créer un dossier partagé personnel « caché » pour chaque salarié (contrôle total sur
celui-ci et aucun accès sur ceux des collègues) et configurer un lecteur réseau associé  
o Déployer les lecteurs réseaux nécessaires au bon fonctionnement de l’entreprise (pour
chacune des ressources nécessaires aux salariés – Dossier « COMMUN », « SERVICE »
et « PERSO »)  
o Seuls la direction et l’informatique peuvent accéder au dossier personnel des salariés
(en « lecture » pour la direction)  
o Déployer un lecteur réseau grâce à un script (BAT ou POWERSHELL) sur ces dossiers  
o Déployer les autres lecteur par GPO  

### GPOs
• Configuration des stratégies suivantes  
o Impression  
 Déployer une imprimante fictive nommée « COMMUN » sur tous les postes
des utilisateurs  
o Restriction d’horaires  
 Aucun salarié sauf la direction, le SAV et l’informatique ne peut se connecter
après 19 heures et avant 07 heures du matin  
 Un message d’alerte doit être implémenté sur les postes clients pour prévenir
les utilisateurs que la session se terminera dans 10 minutes et les invitera donc
à sauvegarder leurs données - La session doit donc se verrouiller à 19h00  

o Stratégies spécifiques :  
 En dehors de la direction et du service informatique, personne ne peut
installer de logiciels sur sa machine ni modifier l’heure  
 Les lecteurs CD et USB sont désactivés sur tous les postes informatiques, sauf
pour le service informatique  
 Le panneau de configuration ne doit pas être accessible sur le poste des
salariés qui ne font pas partie du service informatique  
 Les membres du service informatique doivent être « Administrateur Local »
de chaque machine  
 Le dossier local « Mes Documents » doit être redirigé vers un emplacement
de votre choix sur le serveur  
 Déployer le logiciel « 7zip » et « Google Chrome » pour le service «
comptabilité »  
 Tous les postes doivent être accessible à distance (activer le bureau à distance)  
 Forcer un fond d’écran de votre choix sur l’ensemble des PC de l’entreprise  
 Monsieur Wilson de la comptabilité ne peut ouvrir une session que sur le PC
qui lui a été attribué (il ne peut pas ouvrir de session sur un autre PC)  

### Autres paramètres

o Configurer les clichés instantanés sur un disque  
o Créez un modèle de quotas sur un dossier de votre choix et limitez le stockage à 2Mo
sur ce dossier  

## 7 Modalités d’évaluation

### Technique
• Réalisation et fonctionnement de l’infrastructure  
• Configuration  

### Organisation
• Répartitions des tâches  
• Prise d’initiatives  
• Implication  
• Utilisation des modèles de documents de gestion de projet mis à disposition  

### Écrit
• Qualité des documents (Dossier technique d’architecture et document projet)  
o Document professionnel  
o Orthographe  
o Page de garde (avec la charte graphique de votre entreprise)  
o Sommaire  
o Introduction du sujet  
o Présentation de l’entreprise  
o Rappel de la problématique, contexte, enjeux, ressources, contraintes...  
o Conclusion  
o Perspectives d’évolutions  
o Retour d’expérience  
o Glossaire  
