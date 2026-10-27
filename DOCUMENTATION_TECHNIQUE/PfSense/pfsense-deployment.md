# Guide de Déploiement : Pare-feu pfSense sous Proxmox & Intégration Active Directory

Ce document détaille la procédure pas à pas pour déployer et configurer un pare-feu virtuel **pfSense** sous **Proxmox VE**, relier un réseau local d'entreprise (`10.0.0.0/24`) à un réseau WAN pédagogique/externe, et configurer la résolution DNS sur un contrôleur de domaine Active Directory.

---

## 1. Architecture Réseau & Prérequis

* **Hyperviseur** : Proxmox VE
* **Réseau Interne (LAN)** : `10.0.0.0/24`
  * Passerelle pfSense (LAN) : `10.0.0.254`
  * Serveur Active Directory (DC1) : `10.0.0.10`
* **Réseau Externe (WAN)** : Réseau pédagogique / Virtuel (ex: `10.100.0.0/24`)
  * IP WAN pfSense : `10.100.0.40`
  * Passerelle WAN : `10.100.0.1`

---

## 2. Configuration matérielle de la VM pfSense sur Proxmox

Dans l'onglet **Hardware** de la machine virtuelle pfSense, deux cartes réseau virtuelles (`virtio`) doivent être configurées :

1. **`net0` (Interface WAN)** : 
   * Connectée au bridge Proxmox relié au réseau externe / box / réseau du formateur (ex: `vmbr100`).
   * *Note : Veiller à ce que le pare-feu virtuel Proxmox n'interdise pas les flux si `firewall=1` est actif.*
2. **`net1` (Interface LAN)** : 
   * Connectée au bridge interne du lab (ex: `vmbr4000`).

---

## 3. Configuration initiale de pfSense (Assistant Web)

Une fois la VM démarrée et l'accès Web ouvert sur `https://10.0.0.254` :

1. **Informations Générales** : Renseigner le Hostname (`pfsense`), le domaine (`claas.lan`) et les serveurs DNS de secours.
2. **Configuration WAN** : 
   * Mode : Statique (ou DHCP selon l'infrastructure).
   * Renseigner l'adresse IP statique (`10.100.0.40`) et la passerelle amont (`10.100.0.1`).
   * **Point de vigilance critique (Réseaux Privés)** : Comme l'adresse IP du WAN est une IP privée (plage RFC 1918), il faut impérativement **décocher** tout en bas de la page WAN :
     * 🔲 *Block private networks and loopback addresses*
     * 🔲 *Block bogon networks*
   * Sans cela, pfSense bloquera tout le trafic provenant de la passerelle en amont.
3. **Configuration LAN** : Conserver l'adresse IP `10.0.0.254/24`.
4. **Mot de passe Admin** : Sécuriser l'accès avec un mot de passe robuste.
5. Cliquer sur **Reload** pour appliquer la configuration.

---

## 4. Validation de la connectivité WAN

Pour s'assurer que le pare-feu accède correctement à Internet :
1. Aller dans **Diagnostics** > **Ping**.
2. Choisir la source : **WAN** (`10.100.0.40`).
3. Tester une cible externe (ex: `1.1.1.1` ou `8.8.8.8`). Si les paquets passent avec 0% de perte, le routage et le NAT sortant (*Outbound NAT*) fonctionnent.

---

## 5. Configuration des Redirecteurs DNS sur l'Active Directory

Par défaut, un serveur Active Directory (`SRV-WIN-DC1` en `10.0.0.10`) gère sa propre zone locale et ne sait pas résoudre les noms de domaine sur Internet (ce qui bloque les mises à jour ou les téléchargements sur les autres serveurs du réseau comme Zabbix).

### Procédure :
1. Se connecter sur le serveur Active Directory (`10.0.0.10`).
2. Ouvrir la console **DNS** (`dnsmgmt.msc` via `Exécuter` ou le Gestionnaire de serveur > Outils > DNS).
3. Faire un clic droit sur le nom du serveur (`SRV-WIN-DC1`) dans l'arborescence, puis cliquer sur **Propriétés**.
4. Se rendre dans l'onglet **Redirecteurs** (*Forwarders*).
5. Cliquer sur **Modifier** (*Edit*).
6. Ajouter une adresse IP de DNS public de confiance (ex: `1.1.1.1` ou `8.8.8.8`) ou l'adresse IP du pfSense (`10.0.0.254`).
7. Valider par **OK** et **Appliquer**.

### Test de validation :
Depuis une machine du LAN (ou un serveur membre comme Zabbix pointant vers l'AD), tester la résolution de nom et l'accès Internet :
```cmd
ping 1.1.1.1
ping google.com