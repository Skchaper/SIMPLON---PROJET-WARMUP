# Guide d'Installation d'un Second Contrôleur de Domaine (Redondance) sous Windows Server Core

Ce document décrit la procédure pour ajouter un **contrôleur de domaine secondaire (Replica DC)** à un domaine Active Directory existant, en utilisant une installation **Windows Server Core** (sans interface graphique).

## 📋 Table des Matières

1. [Prérequis et Hypothèses](#prérequis-et-hypothèses)

2. [Étape 1 : Configuration Réseau et Nommage](#étape-1--configuration-réseau-et-nommage)

3. [Étape 2 : Installation du rôle AD DS en Server Core](#étape-2--installation-du-rôle-ad-ds-en-server-core)

4. [Étape 3 : Promotion en tant que Contrôleur de Domaine Supplémentaire](#étape-3--promotion-en-tant-que-contrôleur-de-domaine-supplémentaire)

5. [Étape 4 : Vérification de la Réplication et de la Redondance](#étape-4--vérification-de-la-réplication-et-de-la-redondance)

## 🛠️ Prérequis et Hypothèses

* **Système d'exploitation :** Windows Server (2019 / 2022 / 2025) en édition **Core**.

* **Domaine existant :** `claas.lan` (géré par un premier DC, ex: `SRV-WIN-DC1` d'adresse IP `10.0.0.10`).

* **Droits requis :** Compte membre du groupe *Administrateurs du domaine* (*Domain Admins*).

## 🌐 Étape 1 : Configuration Réseau et Nommage

Connectez-vous à la console de votre serveur Core et ouvrez une session PowerShell.

### 1. Renommer le serveur
```powershell
Rename-Computer -NewName "SRV-WIN-DC2" -Restart
```
*(Le serveur va redémarrer).*

### 2. Configurer l'adresse IP statique et le DNS primaire
Récupérez l'index de votre carte réseau (ifIndex) avec `Get-NetAdapter`, puis configurez l'IPv4 :
```powershell
# Remplacer 'Ethernet' par le nom de votre interface si besoin
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "10.0.0.11" -PrefixLength 24 -DefaultGateway "10.0.0.254"

# Définir le DNS primaire (doit pointer vers le DC existant pour joindre le domaine)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.0.0.10","127.0.0.1")
```

## 📦 Étape 2 : Installation du rôle AD DS en Server Core

Installez le rôle Active Directory Domain Services ainsi que les outils d'administration compatibles avec Server Core :

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

## 👑 Étape 3 : Promotion en tant que Contrôleur de Domaine Supplémentaire

Pour intégrer ce serveur comme réplica du domaine existant, exécutez la commande de promotion. Il vous sera demandé de saisir vos identifiants d'administrateur du domaine et le mot de passe DSRM.

```powershell
Install-ADDSDomainController `
    -NoGlobalCatalog:$false `
    -CreateDnsDelegation:$false `
    -Credential (Get-Credential) `
    -DatabasePath "C:\Windows\NTDS" `
    -DomainName "entreprise.local" `
    -InstallDns:$true `
    -LogPath "C:\Windows\NTDS" `
    -ReplicationSourceDC "SRV-WIN-DC1.claas.lan" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -Force:$true
```

Le serveur va procéder à la réplication de l'annuaire depuis `SRV-WIN-DC1` puis redémarrer automatiquement.

## ✔️ Étape 4 : Vérification de la Réplication et de la Redondance

Après le redémarrage, reconnectez-vous au serveur Core (en PowerShell) et validez le bon état de santé de l'Active Directory :

### 1. Vérifier l'état général de la réplication
```powershell
Get-ADReplicationUpToDateServer -Scope Domain
repadmin /replsum
```

### 2. Tester la résolution DNS et l'inscription du DC
```powershell
dcdiag /v
```

Votre infrastructure dispose désormais d'une redondance Active Directory opérationnelle !
