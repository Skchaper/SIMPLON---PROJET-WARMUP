# Installation et configuration des pare-feu pfSense (Site A & Site B)

## Prérequis pour pfSense

Pour la suite, il faut disposer d'un hyperviseur **Proxmox VE** sur lequel les deux machines virtuelles pare-feu (`GRP4-FW-PfSense1` pour le Site A et `GRP4-FW-PfSense2` pour le Site B) sont déployées. L'architecture repose sur une topologie **Router-on-a-Stick** : un lien virtuel unique en mode Trunk (IEEE 802.1Q) transporte l'ensemble des VLANs étiquetés entre le commutateur virtuel Proxmox et chaque pare-feu pfSense.

Le tutoriel présente chaque procédé avec **un exemple complet** (une interface, un alias, une règle, un tunnel). Pour les autres éléments de la matrice, il suffit de reproduire la même démarche en adaptant les valeurs.

## Prise en main de pfSense et préparation Proxmox

**Configurer le matériel réseau dans Proxmox**

Nous allons commencer par préparer les cartes réseau des VMs pfSense dans l'onglet **Hardware** de Proxmox afin d'assurer la compatibilité avec les trames étiquetées 802.1Q.

Sur le commutateur virtuel interne de Proxmox (pont réseau du groupe), vérifiez que la case **VLAN aware** est cochée. Sur les deux VMs pfSense, sélectionnez le modèle de carte réseau **Intel E1000** (et non VirtIO) : FreeBSD interprète alors les interfaces sous les noms **em0** (WAN) et **em1** (LAN / Trunk).

```text
VM pfSense (ex: 4003 - GRP4-FW-PfSense1) :
- net0 (WAN) : Bridge WAN (vmbr100), Model = Intel E1000, VLAN Tag = (vide)
- net1 (Trunk) : Bridge interne du groupe, Model = Intel E1000, VLAN Tag = (vide)

VMs clientes et serveurs (ex: 4008 - CLI-WIN-ADM) :
- net0 (Access) : Bridge interne du groupe, VLAN Tag = 10
```

Deux réglages matériels sont essentiels au niveau de l'hyperviseur et du système :

**Model = Intel E1000 :** Évite les pertes silencieuses de trames VLAN 802.1Q rencontrées avec le pilote VirtIO (`vtnet`) sous FreeBSD.  
**VLAN Tag = (vide) sur net1 :** Laisse passer toutes les trames étiquetées vers pfSense (mode Trunk), tandis que chaque VM invitée porte son propre tag VLAN (mode Access).

**Désactiver l'accélération matérielle et débloquer le WAN**

Une fois connecté à l'interface Web de pfSense (`https://192.168.1.1` initialement, puis via l'IP de votre interface d'administration), nous allons appliquer deux paramètres indispensables en architecture cascade (pare-feu derrière le pare-feu maître de l'école).

Allez dans **System > Advanced > Networking**, descendez à la section **Network Interfaces**, cochez **Disable hardware checksum offload**, cliquez sur **Save** puis redémarrez la VM (**Diagnostics > Reboot**).

Ensuite, allez dans **Interfaces > WAN**, descendez à la section **Reserved Networks** et décochez la case **Block private networks and loopback addresses**, puis cliquez sur **Save** et **Apply Changes**.

**Disable hardware checksum offload :** Empêche FreeBSD de rejeter les paquets VLAN entrants/sortants dont la somme de contrôle est déléguée à la carte virtuelle.  
**Déblocage RFC1918 sur le WAN :** Autorise le trafic provenant du réseau privé d'interconnexion de l'école (indispensable pour le routage sortant et l'établissement du tunnel VPN IPsec).

## Création et assignation des interfaces VLAN

**Déclarer un tag VLAN 802.1Q**

Dans pfSense, chaque segment réseau doit d'abord être déclaré en tant que sous-interface étiquetée sur le port physique interne **em1**.

Allez dans **Interfaces > Assignments**, ouvrez l'onglet **VLANs** et cliquez sur **+ Add**. Nous prenons ici l'exemple du **VLAN 10 (ADMIN)** :

```text
Parent Interface : em1
VLAN Tag : 10
VLAN Priority : 0 (par défaut)
Description : VLAN_ADMIN
```

Cliquez sur **Save**. *(Pour les autres VLANs du Site A — 20, 30, 40, 50 — et le VLAN 100 du Site B sur PfSense2, pensez à répéter l'opération en remplaçant uniquement le numéro de Tag et la description).*

**Assigner et activer l'interface logique**

Une fois le VLAN déclaré, il faut l'associer à une interface logique pfSense et lui attribuer son adresse de passerelle.

Retournez dans **Interfaces > Assignments**. Dans la liste déroulante **Available network ports**, sélectionnez **VLAN 10 on em1 (VLAN_ADMIN)** puis cliquez sur **+ Add**. Cliquez ensuite sur le nom de la nouvelle interface créée (ex. **OPT1**) pour ouvrir sa configuration :

```text
Enable : [x] Enable interface
Description : ADMIN
IPv4 Configuration Type : Static IPv4
IPv4 Address : 10.4.10.254 / 24
IPv4 Upstream gateway : None
```

Cliquez sur **Save** puis **Apply Changes**. *(Pour les autres interfaces — `INFRA_AD` en `10.4.20.254/24`, `SRV_LINUX` en `10.4.30.254/24`, `DATAS` en `10.4.40.254/24`, `BACKUP` en `10.4.50.254/24` et `LAN` Site B en `10.40.10.254/24` — pensez à remplacer le port réseau `VLAN X on em1`, le nom et l'adresse IP de passerelle correspondante. Une fois la migration terminée, pensez également à désactiver l'ancienne interface `LAN` non taguée).*

## Centralisation par Alias

**Créer un Alias d'hôtes ou de réseaux**

Sur pfSense, les règles de pare-feu sont plus lisibles et maintenables lorsqu'elles s'appuient sur des objets nommés plutôt que sur des adresses IP en dur. Allez dans **Firewall > Aliases**, onglet **IP**, et cliquez sur **+ Add**.

Exemple pour le réseau global d'isolation **RFC1918** :

```text
Name : RFC1918
Description : Plages d'adresses IP privées (Isolation inter-VLAN et inter-groupes)
Type : Network(s)
Network / FQDN :
  - 10.0.0.0 / 8
  - 172.16.0.0 / 12
  - 192.168.0.0 / 16
```

Cliquez sur **Save**. *(Pour les autres alias IP de type `Host(s)` — `IP_ADM`, `IP_DC`, `IP_ZABBIX`, `IP_GLPI`, `IP_DOCKER`, `IP_DATAS`, `IP_VEEAM`, `IP_SITEB` — ou de type `Network(s)` comme `RESEAUX_LAB` en `10.4.0.0/16`, pensez à remplacer le nom, le type et les adresses IP correspondantes).*

**Créer un Alias de ports**

Pour regrouper plusieurs ports non contigus dans une même règle (par exemple les ports d'administration distante), allez dans **Firewall > Aliases**, onglet **Ports**, et cliquez sur **+ Add** :

```text
Name : PORTS_ADMIN
Description : Ports d'administration distante (SSH, RDP, WinRM)
Type : Port(s)
Port :
  - 22
  - 3389
  - 5985
  - 5986
```

Cliquez sur **Save** puis **Apply Changes**. *(Pour les autres alias de ports — `PORTS_WEB` : `80, 443` ; `PORTS_AD` : `53, 88, 135, 389, 445, 464, 636, 3268, 3269, 49152:65535` ; `PORTS_VEEAM` : `22, 135, 445, 2500:3300, 6160:6162` — pensez à remplacer le nom et la liste des ports).*

## Configuration des règles de filtrage

**Comprendre la logique d'évaluation pfSense**

Sur pfSense, les règles de pare-feu sont **stateful** (avec suivi d'état : le trafic retour est automatiquement autorisé) et s'évaluent **de haut en bas sur l'interface d'entrée (source) du paquet**. La première règle qui correspond au paquet est appliquée.

L'ordre de construction sur chaque onglet VLAN suit toujours la même structure en trois blocs :

1. **Règles Pass spécifiques (haut de liste) :** Autorisent uniquement les flux légitimes vers les cibles et ports précis.
2. **Règle Block RFC1918 (avant-dernière position) :** Bloque tout le reste du trafic vers les réseaux privés (`RFC1918`), garantissant l'étanchéité entre vos VLANs et vis-à-vis des autres groupes du lab.
3. **Règle Pass Internet (dernière position) :** Autorise la sortie vers Internet (`Any` sur `PORTS_WEB`), uniquement pour les adresses publiques puisque `RFC1918` a été bloqué juste au-dessus.

**Ajouter une règle de filtrage (Exemple sur l'onglet ADMIN)**

Allez dans **Firewall > Rules**, sélectionnez l'onglet **ADMIN** et cliquez sur **+ Add** (flèche vers le haut pour insérer en tête de liste). Exemple pour autoriser le poste d'administration à joindre les interfaces Web des serveurs Linux :

```text
Action : Pass
Interface : ADMIN
Address Family : IPv4
Protocol : TCP
Source : Single host or alias -> IP_ADM
Destination : SRV_LINUX net
Destination Port Range : From: PORTS_WEB | To: PORTS_WEB
Description : Accès Web Zabbix, GLPI, Docker
```

Cliquez sur **Save**. Juste avant la règle d'accès Internet, ajoutez la règle d'isolation inter-VLAN (en cochant **Log packets that are handled by this rule** pour faciliter le diagnostic dans **Status > System Logs > Firewall**) :

```text
Action : Block
Interface : ADMIN
Address Family : IPv4
Protocol : Any
Source : ADMIN net
Destination : Single host or alias -> RFC1918
Log : [x] Log packets that are handled by this rule
Description : Isolation inter-VLAN et inter-groupes
```

Cliquez sur **Save** puis **Apply Changes**. *(Pour toutes les autres règles de la matrice sur les onglets `ADMIN`, `INFRA_AD`, `SRV_LINUX`, `DATAS`, `BACKUP`, `WAN`, `OpenVPN` et `IPsec`, pensez à reproduire cette procédure en remplaçant l'onglet source, le protocole, la source, la destination et les ports selon votre matrice de flux).*

## Interconnexion VPN Site-à-Site (IPsec Full Tunnel)

**Configurer la Phase 1 (Authentification des passerelles)**

Le tunnel IPsec relie `GRP4-FW-PfSense1` (Site A) et `GRP4-FW-PfSense2` (Site B) via leurs interfaces WAN respectives. Allez dans **VPN > IPsec > Tunnels** et cliquez sur **+ Add P1** (exemple sur `PfSense1`) :

```text
Key Exchange version : IKEv2
Internet Protocol : IPv4
Interface : WAN
Remote Gateway : <IP_WAN_PfSense2>
Authentication Method : Mutual PSK
Pre-Shared Key : <Clé_Secrète_Identique_Des_Deux_Côtés>
Encryption Algorithm : AES (256 bits) | Hash : SHA256 | DH Group : 14 (2048 bit)
```

Cliquez sur **Save**. *(Sur `PfSense2`, pensez à remplacer `Remote Gateway` par `<IP_WAN_PfSense1>` avec la même clé pré-partagée et les mêmes paramètres cryptographiques).*

**Configurer la Phase 2 en mode Full Tunnel (Trafic chiffré)**

En mode **Full Tunnel**, tout le trafic sortant du Site B (y compris sa navigation Internet) est encapsulé vers le Site A afin d'être filtré par `PfSense1`. Sous la Phase 1 créée, cliquez sur **Show Phase 2 Entries** puis **+ Add P2** (exemple sur `PfSense2`) :

```text
Mode : Tunnel IPv4
Local Network : LAN subnet (10.40.10.0/24)
Remote Network : Network -> 0.0.0.0 / 0
Protocol : ESP
Encryption Algorithms : AES (256 bits)
Hash Algorithms : SHA256
```

Cliquez sur **Save** puis **Apply Changes**. *(Sur `PfSense1`, pensez à inverser les réseaux en renseignant `Local Network : Network -> 0.0.0.0 / 0` et `Remote Network : Network -> 10.40.10.0 / 24`).*

**Activer le NAT sortant hybride et autoriser le flux IPsec**

Pour que le trafic Internet du Site B (`10.40.10.0/24`) puisse sortir par le WAN de `PfSense1` :

1. **Sur les deux pare-feu (Onglet WAN) :** Ajoutez une règle **Pass** autorisant l'IP WAN du pare-feu distant en **UDP ports 500, 4500** ainsi qu'une règle **Pass** pour le protocole **ESP**.
2. **Sur PfSense1 (Firewall > NAT > Outbound) :** Sélectionnez **Hybrid Outbound NAT rule generation**, cliquez sur **Save**, puis ajoutez une règle en bas de liste (**+ Add**) :

```text
Interface : WAN
Address Family : IPv4
Protocol : Any
Source : Network -> 10.40.10.0 / 24
Destination : Any
Translation Address : WAN address
Description : NAT Internet pour le Site B via tunnel IPsec
```

3. **Sur PfSense1 (Firewall > Rules > IPsec) :** Autorisez les flux du Site B (`10.40.10.10`) vers les serveurs autorisés (`IP_DC`, `IP_DATAS`, `IP_GLPI`, `IP_ZABBIX`), placez une règle **Block** vers `RFC1918`, puis terminez par une règle **Pass TCP** vers `Any` sur `PORTS_WEB` pour la sortie Internet.

## Mise à jour du VPN d'accès distant (OpenVPN)

**Adapter le routage et le DNS du serveur OpenVPN**

Si vous utilisez un serveur OpenVPN d'administration nomade sur `PfSense1`, mettez à jour sa configuration dans **VPN > OpenVPN > Servers** (icône crayon) afin de router l'ensemble des nouveaux sous-réseaux VLAN vers les clients connectés :

```text
Tunnel Settings :
- IPv4 Tunnel Network : 10.8.0.0/24
- IPv4 Local network(s) : 10.4.0.0/16

Client Settings :
- DNS Server 1 : 10.4.20.11
- DNS Server 2 : 10.4.20.12
```

Cliquez sur **Save**. *(Dans **Firewall > Rules > OpenVPN**, pensez également à vérifier qu'une règle `Pass` autorise le réseau tunnel `10.8.0.0/24` vers `RFC1918` sur `PORTS_ADMIN`, et réexportez le fichier `.ovpn` si l'IP WAN de `PfSense1` a changé).*

## Vérification et diagnostic de l'état des pare-feu

**Contrôler l'établissement des tunnels et les journaux**

Depuis l'interface Web de `PfSense1` et `PfSense2`, vous pouvez vérifier immédiatement le bon fonctionnement de l'ensemble de la chaîne réseau :

* **État du tunnel IPsec :** Allez dans **Status > IPsec**. Le statut de la connexion doit afficher **ESTABLISHED** en vert, avec les compteurs de paquets *Bytes-In* et *Bytes-Out* qui s'incrémentent lors d'un test depuis `CLI-WIN-SITEB` (`tracert 8.8.8.8`).
* **Journaux de filtrage en temps réel :** Allez dans **Status > System Logs > Firewall**. Les flux bloqués par vos règles `Block RFC1918` apparaissent avec une croix rouge, permettant de valider l'isolation entre les VLANs ou d'identifier immédiatement un port manquant lors d'un test applicatif.
