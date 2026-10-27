
## Étape 1 : Installer Virt-Viewer sur votre ordinateur

• Si votre PC est sous Windows :  
	1. Téléchargez l'installateur officiel : Virt-Viewer Windows MSI.  
	2. Installez le fichier .msi téléchargé.  
• Si votre PC est sous Linux (Ubuntu / Debian / Mint) :  
	• Ouvrez votre terminal et installez le paquet :  

```bash
sudo apt install virt-viewer
```
• Si votre PC est sous macOS :  
	• Installez-le via Homebrew :  

```bash
brew install virt-viewer
```

## Étape 2 : Ouvrir la console

Une fois le logiciel installé :  
1. Retournez sur Proxmox et cliquez sur Console -> SPICE.  
2. Le fichier .vv se télécharge à nouveau.  
3. Double-cliquez sur ce fichier .vv dans votre dossier de téléchargements.  
4. Il s'ouvrira automatiquement avec Remote Viewer (le composant de Virt-Viewer) et la fenêtre de votre Debian 13 va apparaître en direct


## Étape 3 : Activer enfin le copier-coller (Côté Debian)

Puisque vous utilisez SPICE, pour que le copier-coller fonctionne entre votre PC et la VM, il faut installer l'agent à l'intérieur de votre Debian 13.  
Dans la console qui vient de s'ouvrir (ou via SSH), tapez ces commandes :  

```bash
sudo apt update
sudo apt install spice-vdagent
sudo systemctl enable --now spice-vdagentd.service
```
