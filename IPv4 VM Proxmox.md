MDP par défaut : Azerty1*  

**Réseau principal 10.0.0.0/24**  
**Plage DHCP1 :** 10.0.0.50 à 10.0.0.200 | garder : 10.0.0.1 à 10.0.0.49 pour les serveurs et .254 pour la passerelle  
**Réseau secondaire 10.0.1.0/24**  
**Plage DHCP2 :** 10.0.1.50 à 10.0.1.200 | garder : 10.0.1.1 à 10.0.1.49 pour les serveurs et 1.254 pour la passerelle  

Domaine Active Directory : claas.lan  
Hébergeur domaine web : duckdns  

**4001** - AD | DNS | DHCP | WDS | RDS - GRP4-SRV-WIN-DC1 - **10.0.0.10**  

**4002** - AD CORE - GR4-SRV-WIN-CORE - **10.0.0.11**  

**4003** - Firewall PfSense - GRP4-FW-Pfsense(1?) - **10.0.0.254** (passerelle)  

**4004** - Supervision Zabbix Debian 13 - GRP4-SRV-DEB-ZABBIX - **10.0.0.12**  

**4005** - Gestion de parc (ticketing, inventaire, ...) GLPI Debian 13 - GRP4-SRV-DEB-GLPI - **10.0.0.13**  

**4006** - Conteneurs DOCKER Debian 13 - GRP4-SRV-DEB-DOCKER - **10.0.0.14**  

**4007** - Serveur de fichier | WSUS (Windows Server) - GRP4-SRV-WIN-DATAS - **10.0.0.15**  

**4008** - PC Client administrateur - GRP4-CLI-WIN-ADM - **10.0.0.50**  

**4009** - PC Client utilisateur (test GPO, etc.) - GRP4-CLI-WIN-USR - **10.0.0.51**  

**4010** - Firewall2 PfSense - GRP4-FW-PfSense2 - **10.0.1.254** (passerelle)  

**4011** - Serveur de Sauvegardes Veeam - GRP4-SRV-BACKUP-VEEAM - **10.0.0.20**  

**4012** -   
