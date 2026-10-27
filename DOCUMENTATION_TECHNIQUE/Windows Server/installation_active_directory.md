# Guide d'Installation et de Configuration d'Active Directory (AD DS)

Ce dépôt contient la documentation étape par étape pour installer, configurer et administrer un domaine **Active Directory Domain Services (AD DS)** sous Windows Server.

---

## 📋 Table des Matières
1. [Prérequis](#prérequis)
2. [Étape 1 : Configuration Réseau](#étape-1--configuration-réseau)
3. [Étape 2 : Installation du rôle AD DS](#étape-2--installation-du-rôle-ad-ds)
4. [Étape 3 : Promotion du Serveur en Contrôleur de Domaine](#étape-3--promotion-du-serveur-en-contrôleur-de-domaine)
5. [Étape 4 : Vérifications Post-Installation](#étape-4--vérifications-post-installation)
6. [Étape 5 : Bonnes Pratiques et Sécurité](#étape-5--bonnes-pratiques-et-sécurité)

---

## 🛠️ Prérequis

Avant de commencer l'installation, assurez-vous de disposer des éléments suivants :
*   **Système d'exploitation :** Windows Server (2016, 2019, 2022 ou versions ultérieures).
*   **Compte utilisateur :** Compte Administrateur local disposant des privilèges élevés.
*   **Ressources matérielles minimales :** 
    *   Processeur : 1.4 GHz 64-bit (recommandé 2 GHz ou plus).
    *   Mémoire RAM : 2 Go minimum (recommandé 4 Go+).
    *   Espace disque : 32 Go minimum.

---

## 🌐 Étape 1 : Configuration Réseau

Un Contrôleur de Domaine (DC) doit impérativement posséder une **adresse IP statique** et pointer vers lui-même (ou un autre DNS local) pour la résolution de noms.

1. Ouvrez les connexions réseau (`ncpa.cpl`).
2. Configurez les propriétés IPv4 de la carte réseau :
   * **Adresse IP :** `192.168.1.10` (exemple)
   * **Masque de sous-réseau :** `255.255.255.0`
   * **Passerelle par défaut :** `192.168.1.1`
   * **Serveur DNS préféré :** `127.0.0.1` (ou l'adresse IP fixe du serveur lui-même).
3. Renommez votre serveur avec un nom explicite (ex: `SRV-DC01`) via les propriétés système (`sysdm.cpl`), puis redémarrez si nécessaire.

---

## 📦 Étape 2 : Installation du rôle AD DS

L'installation peut s'effectuer via l'interface graphique (Server Manager) ou via PowerShell.

### Option A : Via l'Interface Graphique (GUI)
1. Ouvrez le **Gestionnaire de serveur** (*Server Manager*).
2. Cliquez sur **Ajouter des rôles et des fonctionnalités**.
3. Cliquez sur **Suivant** jusqu'à l'écran de sélection des rôles de serveur.
4. Cochez la case **Services de domaine Active Directory** (*Active Directory Domain Services*).
5. Cliquez sur **Ajouter des fonctionnalités** dans la fenêtre pop-up, puis sur **Suivant**.
6. Suivez les étapes de l'assistant sans modifier les options par défaut, puis cliquez sur **Installer**.

### Option B : Via PowerShell (Admin)
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

---

## 👑 Étape 3 : Promotion du Serveur en Contrôleur de Domaine

Une fois le rôle installé, vous devez promouvoir le serveur pour créer la forêt et le domaine.

### Via PowerShell (Recommandé)
Lancez PowerShell en mode Administrateur et exécutez la commande suivante (adaptez le nom du domaine selon vos besoins) :

```powershell
Install-ADDSForest `
    -CreateDnsDelegation:$false `
    -DatabasePath "C:\Windows\NTDS" `
    -DomainMode "WinThreshold" `
    -DomainName "entreprise.local" `
    -DomainNetbiosName "ENTREPRISE" `
    -ForestMode "WinThreshold" `
    -InstallDns:$true `
    -LogPath "C:\Windows\NTDS" `
    -NoRebootOnCompletion:$false `
    -SysvolPath "C:\Windows\SYSVOL" `
    -Force:$true
```
*Note : Il vous sera demandé de saisir et de confirmer un mot de passe pour le mode de restauration des services d'annuaire (DSRM).*

---

## ✔️ Étape 4 : Vérifications Post-Installation

Après le redémarrage automatique du serveur, vérifiez que l'AD fonctionne correctement :

1. **Vérification du service DNS :** Ouvrez le gestionnaire DNS et assurez-vous que la zone de votre domaine (`entreprise.local`) est bien présente avec les enregistrements SRV (_ldap, _kerberos).
2. **Vérification via PowerShell :**
   ```powershell
   Get-ADDomain
   Get-ADForest
   ```
3. **Ouverture des consoles d'administration :**
   * Appuyez sur `Windows + R`, tapez `dsa.msc` (Utilisateurs et ordinateurs Active Directory).
   * Tapez `gpmc.msc` (Gestion de stratégie de groupe - GPO).

---

## 🔒 Étape 5 : Bonnes Pratiques et Sécurité

*   **Second Contrôleur de Domaine :** Installez toujours au moins un second contrôleur de domaine pour assurer la haute disponibilité et la tolérance aux pannes.
*   **Comptes Administrateurs :** Ne travaillez jamais au quotidien avec le compte `Administrator` par défaut. Créez des comptes nominatifs dédiés à l'administration.
*   **Sauvegardes :** Planifiez des sauvegardes régulières de l'état du système (*System State*) et de la base de données NTDS.

---
*Généré avec ❤️ pour vos documentations d'infrastructure.*