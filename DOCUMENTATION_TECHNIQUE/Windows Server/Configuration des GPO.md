# CLAAS – Lecteurs réseau et GPO (Windows Server 2022)

Mise en place des partages réseau, des lecteurs et des stratégies de groupe pour l'entreprise fictive CLAAS (9 services, 35 salariés).

## Environnement

| Machine | Rôle | IP |
| --- | --- | --- |
| SRV-WIN-DC1 | Contrôleur de domaine `claas.lan`, DNS | 10.0.0.10 |
| SRV-WIN-DATAS | Serveur de fichiers, impression, FSRM | 10.0.0.15 |
| PC-COMPTA-01 | Poste client de test (PC de M. Wilson) | 10.0.0.52 |
| pfSense | Passerelle + VPN | 10.0.0.254 |

Organisation AD :
- Utilisateurs : `claas > Site Principal > USERS > <Service>` (et `Site Secondaire > USERS`)
- Groupes : `GG_RH`, `GG_Comptabilite`, `GG_Developpement`, `GG_Informatique`, `GG_SAV`, `GG_Marketing`, `GG_Direction`, `GG_Logistique`, `GG_Production`, `GG_Tous_Salaries`
- Postes clients : `claas > Site Principal > Ordinateurs`

## Utilisation

Sur **SRV-WIN-DATAS**, connecté avec un compte **administrateur du domaine** (`CLAAS\Administrateur`, pas le compte local) :

```powershell
Install-WindowsFeature RSAT-AD-PowerShell, GPMC
cd .\CLAAS-GPO\scripts
Get-ChildItem | Unblock-File
Set-ExecutionPolicy -Scope Process Bypass -Force
.\01_Partages.ps1
.\02_Comptes_AD.ps1
.\03_Imprimante_Quotas_Cliches.ps1
.\04_GPO.ps1
```

Les scripts lisent les membres des groupes directement dans l'AD : aucun nom d'utilisateur n'est écrit en dur.

| Script | Ce qu'il fait |
| --- | --- |
| `01_Partages.ps1` | Crée les dossiers dans `C:\Partages`, les droits NTFS, les partages et le script de connexion `mappage_perso.bat` (NETLOGON) |
| `02_Comptes_AD.ps1` | Horaires de connexion 7h–19h, limite M. Wilson à `PC-COMPTA-01` |
| `03_Imprimante_Quotas_Cliches.ps1` | Imprimante fictive COMMUN, modèle de quota 2 Mo, clichés instantanés |
| `04_GPO.ps1` | Crée les 11 GPO, les lie aux OU, règle le filtrage et les paramètres scriptables |

## Partages et droits

```
C:\Partages
├── Commun                 \\SRV-WIN-DATAS\Commun
├── Services\Commun_<Service>   \\SRV-WIN-DATAS\Commun_<Service>   (x9)
├── Perso\<login>          \\SRV-WIN-DATAS\<login>$   (caché, un par salarié)
└── Logiciels              \\SRV-WIN-DATAS\Logiciels$ (MSI pour les GPO)
```

| Dossier | Salariés | Informatique | Direction |
| --- | --- | --- | --- |
| Commun | lecture / écriture / modification du contenu, pas de suppression du dossier | contrôle total | comme les salariés |
| Commun_\<Service\> | idem, seulement pour leur service | contrôle total | seulement le sien |
| Perso\\\<login\> | contrôle total sur le sien, rien sur les autres | contrôle total | lecture |

**Protection du dossier racine** : le droit « Modification » est donné seulement sur les *sous-dossiers et fichiers*. Sur le dossier lui-même, les salariés peuvent lire et créer, mais pas supprimer ni renommer.

Les droits de partage sont en « Contrôle total » pour les utilisateurs authentifiés : c'est le NTFS qui filtre. L'énumération basée sur l'accès est activée, chacun ne voit que ce qu'il peut ouvrir.

## Lecteurs réseau

| Lecteur | Cible | Méthode |
| --- | --- | --- |
| P: | `\\SRV-WIN-DATAS\%USERNAME%$` | Script BAT (`mappage_perso.bat`) lancé par la GPO |
| T: | `\\SRV-WIN-DATAS\Commun` | GPO – Préférences, Mappages de lecteurs |
| S: | `\\SRV-WIN-DATAS\Commun_<Service>` | GPO – Préférences + ciblage par groupe `GG_<Service>` |

## GPO

| GPO | Rôle | S'applique à |
| --- | --- | --- |
| U_Lecteurs_Reseau | lecteurs P:, T:, S: | tous |
| U_Imprimante_COMMUN | imprimante `\\SRV-WIN-DATAS\COMMUN_IMP` par défaut | tous |
| U_Horaires_Session | message à 18h50, verrouillage à 19h00 | tous sauf Direction, SAV, Informatique |
| U_Restrictions_Salaries | panneau de configuration et USB/CD bloqués | tous sauf Informatique |
| U_Redirection_Documents | Documents → `\\SRV-WIN-DATAS\Perso$\%USERNAME%\Documents` | tous |
| U_Fond_Ecran | fond d'écran imposé (NETLOGON) | tous |
| U_Logiciels_Comptabilite | 7-Zip et Chrome (MSI) | Comptabilité |
| C_Admins_Locaux | GG_Informatique (+ GG_Direction) admins locaux | tous les PC |
| C_Restrictions_Installation_Heure | heure et installations réservées Direction / Informatique | tous les PC |
| C_Bureau_A_Distance | RDP activé + pare-feu + NLA | tous les PC |
| C_Parametres_Postes | attente du réseau, pilote d'imprimante autorisé depuis le serveur | tous les PC |

Le filtrage se fait par **inclusion** : les GPO restrictives s'appliquent uniquement aux groupes concernés (onglet *Délégation*).

### Réglages faits à la main dans GPMC
- **U_Lecteurs_Reseau** : script d'ouverture de session `\\claas.lan\NETLOGON\mappage_perso.bat` + 10 mappages (T: et 9 × S: avec ciblage par groupe).
- **U_Imprimante_COMMUN** : Préférences > Imprimante partagée `\\SRV-WIN-DATAS\COMMUN_IMP`, par défaut.
- **U_Horaires_Session** : 2 tâches planifiées quotidiennes (18h50 : popup PowerShell ; 19h00 : `rundll32.exe user32.dll,LockWorkStation`).
- **U_Redirection_Documents** : « Accorder des droits exclusifs » décoché (sinon Direction et Informatique perdent l'accès).
- **U_Logiciels_Comptabilite** : packages *Attribués*, installés à l'ouverture de session, chemin UNC obligatoire.
- **C_Admins_Locaux** : Préférences > Groupe local *Administrateurs (intégré)*, action Mettre à jour.
- **C_Restrictions_Installation_Heure** : droits « Modifier l'heure système » et « Changer le fuseau horaire » ; UAC « refuser automatiquement les demandes d'élévation » pour les utilisateurs standard ; Windows Installer « Interdire les installations par l'utilisateur ».

## Autres paramètres

- **Horaires** : attribut AD `logonHours` (7h–19h, stocké en UTC, le script convertit). La GPO ajoute le message et le verrouillage.
- **M. Wilson** : attribut `userWorkstations` = `PC-COMPTA-01`.
- **Clichés instantanés** : volume C:, 10 % max, lundi–vendredi à 7h et 12h.
- **Quota** : modèle FSRM `CLAAS_Limite_2Mo` (limite stricte, alertes à 85 % et 100 %) appliqué à `Commun_Marketing`.

## Vérification

Sur le poste client : `gpupdate /force` puis `gpresult /r`.

- Compte Informatique : 4 GPO utilisateur appliquées, `U_Restrictions_Salaries` et `U_Horaires_Session` refusées (filtrage), membre de *Administrateurs*.
- Compte Marketing : panneau de configuration et USB bloqués, quota atteint au-delà de 2 Mo dans S:, connexion refusée après 19h.
- M. Wilson : connexion refusée sur un autre PC que PC-COMPTA-01.

## Problèmes rencontrés

| Problème | Cause | Solution |
| --- | --- | --- |
| RDP impossible vers SRV-WIN-DATAS | service *TermService* bloqué (rien en écoute sur 3389) | redémarrage du serveur |
| Pas de droits sur l'AD | session ouverte avec l'administrateur **local** | se connecter en `CLAAS\Administrateur` |
| Erreur `0x80070846` à la création de l'imprimante | le nom de partage `COMMUN` est déjà pris par le dossier `Commun` (Windows ignore la casse) | imprimante nommée COMMUN, partagée en `COMMUN_IMP` |
| Liens GPO refusés sur l'OU Ordinateurs | OU créée sur un DC, liens demandés à un autre (réplication) | toutes les commandes envoyées au même DC (PDC) |
| Windows ne s'installe pas sur la VM cliente | disque IDE absent de l'ordre de démarrage Proxmox | disque activé dans *Boot Order* |
| Perte du VPN après modification DNS dans pfSense | mauvaise IP ajoutée comme DNS | restauration de la configuration (option 15 du menu console) |
