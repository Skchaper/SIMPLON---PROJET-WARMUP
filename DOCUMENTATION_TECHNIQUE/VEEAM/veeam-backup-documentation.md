# Mise en place des sauvegardes avec Veeam Backup & Replication

> Documentation du projet : déploiement de Veeam sur une infrastructure virtualisée Proxmox (domaine `claas.lan`), sauvegarde des serveurs Windows et Debian via agents.

## Sommaire

1. [Contexte et architecture](#1-contexte-et-architecture)
2. [Prérequis](#2-prérequis)
3. [Procédure de mise en place](#3-procédure-de-mise-en-place)
4. [Problèmes rencontrés et solutions](#4-problèmes-rencontrés-et-solutions)
5. [Résultat final](#5-résultat-final)
6. [Sécurité](#6-sécurité)
7. [Limites connues et améliorations](#7-limites-connues-et-améliorations)
8. [Commandes utiles](#8-commandes-utiles)

---

## 1. Contexte et architecture

| Élément | Détail |
|---|---|
| Hyperviseur | Proxmox VE |
| Serveur de sauvegarde | `SRV-BACKUP-VEEAM` (10.0.0.20), Windows, **hors domaine** (groupe de travail) |
| Version Veeam | Backup & Replication 13.0.2 (build 13.0.2.29), Community Edition |
| Domaine Active Directory | `claas.lan` |
| Machines Windows | contrôleur de domaine (`SRV-WIN-DC1`), `SRV-WIN-CORE`, `SRV-WIN-DATAS` |
| Machines Linux | Debian 13 (trixie) : `SRV-DEB-ZABBIX`, `SRV-DEB-GLPI`, `SRV-DEB-DOCKER` |
| Plage réseau | 10.0.0.0/24 |
| Protection Group | `PG-MES-VMS` |
| Jobs | `BCK-Windows`, `BCK-Linux` |

**Principe retenu :** sauvegarde **par agent**. Veeam déploie un agent sur chaque machine depuis une *Protection Group* (SMB/ADMIN$ pour Windows, SSH pour Linux), puis des jobs gérés par le serveur Veeam pilotent les sauvegardes.

```
                 ┌──────────────────────┐
                 │  SRV-BACKUP-VEEAM    │
                 │  10.0.0.20           │
                 └──────────┬───────────┘
        SMB (445) / SSH (22)│ déploiement + sauvegarde
      ┌──────────┬──────────┼──────────┬──────────┐
      ▼          ▼          ▼          ▼          ▼
  Windows     Windows     Debian     Debian     Debian
  (DC1...)   (CORE...)   (Zabbix)   (GLPI)    (Docker)
```

*(Ajouter ici une capture du schéma réseau / de la console Proxmox.)*

---

## 2. Prérequis

### Serveur Veeam
- Windows avec droits administrateur, IP fixe.
- Espace disque suffisant pour le repository.
- Pare-feu autorisant les flux entrants nécessaires (voir section 4.7).

### Machines Windows
- Compte administrateur (local ou domaine).
- Service **Serveur** (`LanmanServer`) actif, partage `ADMIN$` présent.
- Port **445/TCP** joignable depuis le serveur Veeam.

### Machines Linux (Debian)
- `openssh-server` installé et actif (port **22/TCP**).
- Compte avec droits **sudo** (ou root autorisé en SSH avec mot de passe).
- Accès aux dépôts apt (dépendances de l'agent).
- **Headers du noyau**, `dkms` et `build-essential` : nécessaires pour compiler le module `blksnap` (snapshots).

---

## 3. Procédure de mise en place

### Étape 1 : Préparer les machines Linux

```bash
apt update
apt install -y openssh-server
systemctl enable --now ssh
ss -tlnp | grep :22      # vérifier que SSH écoute
```

Puis les dépendances de l'agent Veeam :

```bash
apt update
apt full-upgrade -y
reboot
# après le redémarrage :
apt install -y dkms linux-headers-$(uname -r) build-essential
```

### Étape 2 : Préparer les machines Windows

PowerShell administrateur :

```powershell
Get-Service LanmanServer          # doit être Running
net share                         # ADMIN$ doit apparaître
New-NetFirewallRule -DisplayName "SMB-In Veeam" -Direction Inbound -Protocol TCP -LocalPort 445 -Action Allow -RemoteAddress 10.0.0.20
```

Pour un compte **local** autre que l'Administrateur intégré, désactiver le filtrage UAC distant :

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" -Name "LocalAccountTokenFilterPolicy" -Value 1 -PropertyType DWord -Force
```

### Étape 3 : Tester la connectivité depuis le serveur Veeam

```powershell
Test-NetConnection <IP> -Port 22     # Linux
Test-NetConnection <IP> -Port 445    # Windows
```

`TcpTestSucceeded : True` est requis avant d'aller plus loin.

### Étape 4 : Créer la Protection Group

**Inventory → Physical & Cloud Infrastructure → Add Group** :

1. Type : *Individual computers*.
2. Ajouter les machines **par IP**.
3. Associer les credentials adaptés :
   - Windows : compte standard, `.\Administrateur` (local) ou `CLAAS\compte` (domaine).
   - Linux : compte SSH (login + mot de passe), avec sudo.
4. Lancer le **Test Connection**, puis terminer l'assistant.

Veeam déploie ensuite automatiquement l'agent sur chaque machine. Attendre le statut **Success** partout (sinon **Rescan**).

*(Capture : Protection Group avec les agents « Installed » / « Online ».)*

### Étape 5 : Vérifier le repository

**Backup Infrastructure → Backup Repositories** : au moins un repository avec de l'espace libre.

### Étape 6 : Créer les jobs de sauvegarde

**Home → Backup Job → Windows computer** puis **Linux computer** :

| Paramètre | Valeur |
|---|---|
| Name | `BCK-Windows` / `BCK-Linux` |
| Mode | Managed by backup server |
| Computers | machines de la Protection Group |
| Backup Mode | Entire computer |
| Destination | repository Veeam |
| Application-aware processing | activé sur le contrôleur de domaine |
| Schedule | quotidien (22h) |
| Rétention | 7 points de restauration |

### Étape 7 : Lancer et valider

Clic droit sur le job → **Start**. Suivre l'avancement dans **Last 24 Hours → Running**, puis vérifier **Success** et la présence des fichiers dans **Backups → Disk**.

*(Capture : jobs en statut Working puis Success.)*

---

## 4. Problèmes rencontrés et solutions

### 4.1 Échec du Test Connection sur les Debian : « Cannot find any valid credentials »

- **Symptôme :** `Validating SSH credentials Error: Aucune connexion n'a pu être établie car...`
- **Diagnostic :** ce n'était **pas** un problème d'identifiants. `Test-NetConnection <IP> -Port 22` renvoyait `TcpTestSucceeded : False` alors que le ping passait : aucun service n'écoutait sur le port 22.
- **Cause :** `openssh-server` absent ou non démarré sur les Debian.
- **Solution :** installation et activation de SSH (étape 1).

### 4.2 Erreur trompeuse `Failed to write to admin share (ADMIN$)` sur une machine Debian

- **Symptôme :** sur le 10.0.0.12 (Zabbix, une Debian), Veeam affichait `Failed to write to admin share : Le nom réseau spécifié n'est plus disponible (\\10.0.0.12\ADMIN$)`.
- **Cause :** quand Veeam ne parvient pas à joindre SSH, il bascule sur le test Windows (SMB/ADMIN$), qui échoue forcément sur Linux. L'erreur affichée masque le vrai problème.
- **Leçon :** toujours chercher l'erreur **SSH** dans la liste des actions, et vérifier le port 22 avant de se fier au message ADMIN$.
- **Solution :** même correctif que 4.1.

### 4.3 Échec d'installation de l'agent Linux : paquets `veeam`, `veeam-libs`, `blksnap`

- **Symptôme :** l'agent est envoyé (paquets `.deb` uploadés, Data Mover installé) mais `Failed to install veeam_13.0.2.2_amd64.deb, veeam-libs... and blksnap...`.
- **Diagnostic :** `apt install dkms linux-headers-$(uname -r)` renvoyait `Impossible de trouver le paquet linux-headers-6.12.43+deb13-amd64`.
- **Cause :** le noyau en cours d'exécution (6.12.43) n'avait plus de headers dans les dépôts Debian trixie, qui proposaient un noyau plus récent. Or `blksnap` est un module noyau compilé via DKMS pour le noyau actif.
- **Solution :**
  ```bash
  apt full-upgrade -y
  reboot
  apt install -y dkms linux-headers-$(uname -r) build-essential
  ```
  puis **Rescan** dans Veeam.
- **Résultat :** installation de l'agent réussie sur les Debian.

### 4.4 Échec de déploiement de l'agent sur une machine Windows (10.0.0.50)

- **Symptôme :** `Unable to install backup agent: failed to connect to 10.0.0.50 ... Failed to connect with deployment kit on 10.0.0.50:6160 / 11731 ... Failed to connect to share '\\10.0.0.50\ADMIN$'`.
- **Constat :** le *Test Connection* peut réussir alors que le **déploiement** échoue : les deux ne vérifient pas les mêmes choses.
- **Pistes identifiées :**
  - la machine est dans le domaine `claas.lan` (profil réseau `DomainAuthenticated`) : le pare-feu en profil *Domaine* est probablement géré par **GPO**, ce qui peut bloquer SMB entrant ;
  - le serveur Veeam est hors domaine : il faut utiliser un compte de la forme `CLAAS\compte` ;
  - ports à autoriser depuis 10.0.0.20 : 445, 6160, 11731.
- **Remarque :** `Enable-NetFirewallRule -DisplayGroup "Partage de fichiers et d'imprimantes"` a échoué (nom de groupe dépendant de la langue). Les règles `New-NetFirewallRule` par numéro de port sont indépendantes de la langue.
- **Statut :** **non résolu** dans le cadre de ce projet (machine laissée hors périmètre).

### 4.5 Résolution du nom de domaine depuis le serveur Veeam

- **Constat :** `echo %USERDOMAIN%` sur le serveur Veeam renvoie `SRV-BACKUP-VEEAM` et `%USERDNSDOMAIN%` n'est pas défini : le serveur est **hors domaine**.
- **Conséquence :** les comptes du domaine doivent être saisis en entier (`CLAAS\utilisateur` ou `utilisateur@claas.lan`), et les machines ajoutées **par IP**.

### 4.6 Console Proxmox : pas de copier-coller

- **Symptôme :** une seule VM disposait du presse-papier noVNC.
- **Cause :** le panneau noVNC dépend du type d'affichage de la VM (**Hardware → Display**) et du type de console. Les conteneurs LXC utilisent xterm.js, SPICE passe par un autre client.
- **Contournement retenu :** travailler en **SSH** (Debian) ou **RDP** (Windows) depuis le poste d'administration plutôt que dans la console Proxmox.

### 4.7 Ping asymétrique entre l'AD et le serveur Veeam

- **Symptôme :** le serveur Veeam pingue l'AD, mais l'AD ne pingue pas le serveur Veeam.
- **Cause probable :** le pare-feu Windows du serveur Veeam bloque l'ICMP entrant (comportement par défaut), éventuellement un filtrage au niveau du pare-feu Proxmox.
- **Solution proposée :**
  ```powershell
  New-NetFirewallRule -DisplayName "ICMPv4 In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -RemoteAddress 10.0.0.0/24
  ```
  Autoriser aussi, si nécessaire, les ports Veeam entrants (6160, 6162, 10001) depuis le réseau.

---

## 5. Résultat final

- Protection Group `PG-MES-VMS` : **6 machines sur 7** avec agent installé et statut **Online** (3 Windows, 3 Debian).
- Jobs `BCK-Windows` et `BCK-Linux` créés et lancés (statut *Working* lors du premier passage).
- Un poste (10.0.0.50) reste sans agent (voir 4.4).

*(Capture : liste des jobs en Success, détail par machine.)*

---

## 6. Sécurité

L'ouverture du port 22 sur plusieurs machines augmente la surface d'attaque. Mesures recommandées :

| Mesure | Détail |
|---|---|
| Restreindre les sources | n'autoriser le port 22 / 445 que depuis 10.0.0.20 (pare-feu local ou règles pfSense / Proxmox) |
| Compte dédié | utiliser un compte Veeam dédié avec sudo plutôt que `root` |
| Clés SSH | préférer l'authentification par clé et mettre `PasswordAuthentication no` |
| fail2ban | bannir automatiquement les IP qui échouent en boucle |
| Pas d'exposition Internet | vérifier l'absence de redirection de port (NAT) sur le pfSense |
| Moindre privilège | sur Windows, compte de domaine dédié plutôt qu'Administrateur intégré |

---

## 7. Limites connues et améliorations

- **Règle 3-2-1 non respectée** : un seul repository, pas de copie hors site. Prévoir un *Backup Copy Job* vers un second support et une copie externe.
- **Poste 10.0.0.50** non sauvegardé (problème SMB/GPO non résolu).
- **Serveur Veeam hors domaine** : choix simple pour le projet ; en production, l'isoler volontairement du domaine est même une bonne pratique (limite la propagation d'une compromission AD), mais impose une gestion rigoureuse des comptes.
- **Test de restauration** à réaliser et documenter (*Restore guest files* ou *Entire computer*) : une sauvegarde n'est valide que si elle est restaurable.
- **Sauvegarde au niveau hyperviseur** : le plugin Veeam pour Proxmox VE permettrait de sauvegarder les VM sans agent.
- **Chiffrement** des sauvegardes et **immutabilité** du repository (anti-ransomware).
- **Supervision** : intégrer l'état des jobs à Zabbix.

---

## 8. Commandes utiles

### Windows (PowerShell)

```powershell
(Get-CimInstance Win32_ComputerSystem).Domain       # domaine de la machine
Get-NetConnectionProfile                            # profil réseau
Get-Service LanmanServer                            # service Serveur
net share                                           # partages (ADMIN$)
Test-NetConnection <IP> -Port 445                   # test SMB
dir \\<IP>\ADMIN$                                   # test d'accès au partage admin
gpresult /r                                         # GPO appliquées
```

### Debian

```bash
systemctl status ssh
ss -tlnp | grep :22
uname -r
dpkg -l | grep -i -E "veeam|blksnap"
journalctl -u ssh -n 30 --no-pager
```

---

*Documentation rédigée pendant la mise en œuvre. Les captures d'écran sont à ajouter aux emplacements indiqués.*
