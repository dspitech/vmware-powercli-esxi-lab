# Laboratoire PowerCLI sur hôte ESXi autonome

Guide de référence et support de cours pour l'administration d'un hôte VMware ESXi avec VMware PowerCLI, réalisé sur la plateforme VMware Hands-on Labs (HOL).

## Informations sur le laboratoire

| Information | Détail |
|---|---|
| **Titre** | Laboratoire PowerCLI sur hôte ESXi |
| **Auteur** | LO Pape |
| **Date de publication** | 17 novembre 2026 |
| **Dernière mise à jour** | 17 novembre 2026 |
| **Version** | 1.0.0 |
| **Statut** | Première publication |
| **Type de document** | Laboratoire technique |
| **Technologie principale** | VMware PowerCLI |
| **Environnement** | VMware ESXi |
| **Langue** | Français |
| **Licence** | MIT |

<img width="750" height="420" alt="image" src="https://github.com/user-attachments/assets/1e29125f-19b5-4e9f-8273-83df6cf00793" />

---

## Table des matières

1. Présentation et objectifs
2. Configuration du laboratoire
3. Notions fondamentales
4. Phase 1 : Connexion et inventaire
5. Phase 2 : Réseau standard
6. Phase 3 : Stockage
7. Phase 4 : Cycle de vie et matériel des machines virtuelles
8. Phase 5 : Snapshots
9. Phase 6 : Montage d'une image ISO
10. Phase 7 : Déplacement d'une VM (cold migration)
11. Phase 8 : Durcissement et sécurité
12. Phase 9 : Esxcli avancé (pare-feu)
13. Phase 10 : Gestion des ressources
14. Phase 11 : Évolution du matériel et opérations sur les VM
15. Phase 12 : Réseau avancé
16. Phase 13 : Stockage avancé
17. Phase 14 : Gestion des accès
18. Phase 15 : Durcissement avancé
19. Phase 16 : Supervision, événements et journaux
20. Phase 17 : Sauvegarde et export
21. Phase 18 : Automatisation et rapport de santé
22. Nettoyage de fin de laboratoire
23. Journal de résolution des erreurs
24. Bonnes pratiques et limites du lab
25. Glossaire

---

## 1. Présentation et objectifs

Ce laboratoire a pour but de piloter entièrement un hôte ESXi en ligne de commande avec PowerCLI. Il couvre les compétences suivantes :

- Se connecter à un hôte et réaliser un inventaire.
- Configurer le réseau virtuel standard (vSwitch, port groups, VLAN, VMkernel).
- Gérer le stockage (LUN iSCSI, datastores VMFS).
- Créer et administrer des machines virtuelles (matériel, snapshots, ISO, migration à froid).
- Appliquer des mesures de durcissement (NTP, SSH, pare-feu).
- Utiliser `esxcli` via PowerCLI pour accéder aux fonctions avancées de l'hôte.
- Piloter les ressources CPU et mémoire (shares, réservations, limites, resource pools).
- Configurer le teaming réseau avancé et le multipathing de stockage.
- Mettre en place une gestion des accès fondée sur les rôles (RBAC) et un durcissement avancé.
- Superviser l'hôte, exploiter les événements et les journaux.
- Sauvegarder la configuration, exporter des VM et automatiser un rapport de santé.

Le guide intègre les contournements de plusieurs limitations rencontrées lorsque PowerCLI est connecté directement à un ESXi sans vCenter. Chaque contournement est signalé par la mention **Point d'attention**.

### Organisation du document

Chaque module suit la même structure :

1. **Objectifs pédagogiques** : ce que l'étudiant doit savoir faire à la fin du module.
2. **Cours** : les connaissances théoriques à maîtriser (concepts, architecture, tableaux comparatifs, bonnes pratiques, erreurs courantes).
3. **Commandes** : la mise en pratique, chaque commande étant suivie de son explication.
4. **Points clés à retenir** et **Questions de révision** avec réponses, pour s'auto évaluer.

Parcours conseillé : lire d'abord la section 3, puis, pour chaque phase, lire le cours avant d'exécuter les commandes, et répondre aux questions sans regarder les réponses.

---

## 2. Configuration du laboratoire

| Élément | Valeur |
| :--- | :--- |
| Hôte ESXi | `esx-03a.vcf.sddc.lab` (adresse IP `10.0.0.22` en cas d'échec DNS) |
| Identifiants | `root` / `VMware123!` |
| Datastore principal | `ds-iscsi01` (85 Go, FreeNAS) |
| Datastore secondaire | `ds-iscsi02` (45 Go, FreeNAS) |
| Cartes physiques | `vmnic0`, `vmnic1`, `vmnic2`, `vmnic3` (`vmnic3` libre) |
| Environnement | Terminal Linux, répertoire de travail `/home/holuser` |
| PowerCLI | Module `VMware.VimAutomation.Core` |
| Plateforme | VMware Hands-on Labs (environnement temporaire) |

Remarque de sécurité : le mot de passe est écrit en clair dans les scripts uniquement parce que l'environnement HOL est jetable. Dans un environnement réel, il faut utiliser `Get-Credential` ou un coffre de secrets.

---

## 3. Notions fondamentales

Cette section constitue le socle théorique du laboratoire. Elle doit être comprise avant d'aborder les phases pratiques.

### Objectifs pédagogiques

À la fin de cette section, l'étudiant sait expliquer ce qu'est la virtualisation, décrire l'architecture interne d'ESXi, situer ESXi dans l'écosystème vSphere, distinguer un hôte autonome d'un environnement géré par vCenter, comprendre comment PowerCLI dialogue avec l'hôte et utiliser les notions PowerShell indispensables pour lire et écrire les scripts du lab.

### 3.1 La virtualisation

La virtualisation consiste à abstraire les ressources physiques (processeur, mémoire, stockage, réseau) pour les présenter à plusieurs systèmes isolés, appelés machines virtuelles (VM). Une couche logicielle, l'hyperviseur, arbitre l'accès au matériel.

Les cinq propriétés qui expliquent son intérêt :

| Propriété | Signification | Bénéfice concret |
| :--- | :--- | :--- |
| Partitionnement | Plusieurs systèmes se partagent un même serveur physique | Consolidation, moins de serveurs et moins de consommation électrique |
| Isolation | Une VM ne voit ni n'altère les autres | Une panne ou une compromission reste confinée |
| Encapsulation | Une VM est un ensemble de fichiers | Copie, sauvegarde, export et déplacement simples |
| Indépendance matérielle | La VM voit un matériel virtuel standardisé | Migration entre serveurs de marques différentes |
| Agilité | Création et modification par logiciel | Déploiement en minutes, automatisation par scripts |

Les grands domaines de virtualisation sont celle des serveurs (vSphere), du stockage (VMFS, vSAN), du réseau (vSwitch, NSX) et des postes de travail (VDI).

### 3.2 Hyperviseur de type 1 et de type 2

| Critère | Type 1 (bare metal) | Type 2 (hébergé) |
| :--- | :--- | :--- |
| Installation | Directement sur le matériel | Comme une application sur un système existant |
| Exemples | VMware ESXi, Microsoft Hyper-V, KVM | VMware Workstation, VirtualBox |
| Performances | Élevées, chemin court vers le matériel | Moindres, passage par le système hôte |
| Usage typique | Production, datacenter | Poste de développeur, test |

ESXi est un hyperviseur de type 1 : il n'a pas besoin d'un système d'exploitation généraliste sous lui.

### 3.3 Architecture interne d'ESXi

```text
+--------------------------------------------------------------+
|  VM 1            VM 2            VM 3                        |
|  [OS invité]     [OS invité]     [OS invité]                 |
|  [VMX + VMM]     [VMX + VMM]     [VMX + VMM]                 |
+--------------------------------------------------------------+
|  Agents de gestion : hostd (API), vpxa (agent vCenter), DCUI |
+--------------------------------------------------------------+
|  VMkernel : ordonnanceur CPU, gestion mémoire,               |
|             pile stockage (PSA, VMFS), pile réseau (vSwitch) |
+--------------------------------------------------------------+
|  Matériel : CPU, RAM, cartes réseau, HBA, disques            |
+--------------------------------------------------------------+
```

Rôle de chaque composant :

- **VMkernel** : noyau de l'hyperviseur. Il ordonnance les vCPU sur les cœurs physiques, gère la mémoire, héberge la pile de stockage (chemins, VMFS) et la pile réseau (vSwitch, interfaces VMkernel).
- **VMM (Virtual Machine Monitor)** : composant qui exécute le code de la VM avec l'aide des extensions matérielles de virtualisation du processeur (Intel VT-x, AMD-V).
- **VMX** : processus d'espace utilisateur propre à chaque VM. Il gère les périphériques virtuels et l'interface avec le VMkernel.
- **hostd** : agent de gestion de l'hôte. Il expose l'API sur le port 443. C'est lui que contactent PowerCLI, le Host Client et l'ESXi autonome.
- **vpxa** : agent qui relie l'hôte à vCenter. Il est inactif sur un hôte autonome.
- **DCUI (Direct Console User Interface)** : console locale en mode texte pour configurer le réseau de gestion, redémarrer ou dépanner.
- **Ports usuels** : 443 (API et Host Client), 902 (flux de console et transfert de fichiers), 22 (SSH, désactivé par défaut).

### 3.4 L'écosystème vSphere

| Composant | Rôle |
| :--- | :--- |
| ESXi | Hyperviseur installé sur chaque serveur |
| vCenter Server | Gestion centralisée de plusieurs hôtes, active vMotion, DRS, HA |
| Host Client | Interface web d'un hôte ESXi seul |
| vSphere Client | Interface web de vCenter |
| VMFS | Système de fichiers cluster pour les datastores |
| vSAN | Stockage défini par logiciel à partir des disques locaux des hôtes |
| NSX | Virtualisation et sécurité réseau avancées |
| VMware Cloud Foundation (VCF) | Plateforme intégrée qui assemble vSphere, vSAN, NSX et l'automatisation |

Le lab s'exécute dans un environnement VCF de Hands-on Labs, mais on y pilote un hôte comme s'il était isolé.

### 3.5 Hôte autonome et vCenter

| Critère | ESXi autonome | Avec vCenter |
| :--- | :--- | :--- |
| Périmètre de gestion | Un seul hôte | Plusieurs hôtes et clusters |
| vMotion et Storage vMotion | Non | Oui |
| DRS (équilibrage) et HA (haute disponibilité) | Non | Oui |
| Content Library | Non exploitable directement | Oui |
| Clonage et modèles (templates) | Limités | Oui |
| Distributed Switch | Non | Oui |
| Authentification centralisée | Comptes locaux | Single Sign-On, Active Directory |
| Résolution des objets réseau par nom | Parfois instable | Fiable |

Cette différence explique la majorité des contournements que le document appelle **Point d'attention**.

### 3.6 Comment PowerCLI communique avec l'hôte

PowerCLI est un ensemble de modules PowerShell. Ses cmdlets envoient des requêtes SOAP sur HTTPS vers l'API vSphere exposée par hostd (hôte autonome) ou vCenter.

- Les cmdlets de haut niveau (`Get-VM`, `New-Datastore`) retournent des objets enveloppes, faciles à utiliser.
- Chaque objet possède une propriété `ExtensionData` qui donne accès à l'objet natif de l'API, plus complet.
- `Get-View` retourne directement les objets natifs de l'API. Il est plus rapide pour les grands volumes et donne accès à des propriétés absentes des enveloppes.
- `Get-EsxCli` donne accès aux commandes `esxcli` via l'API, sans SSH.
- Quand une cmdlet de haut niveau est défaillante, descendre au niveau de l'API est la technique de contournement standard. Elle est utilisée en phase 6.

Installation : `Install-Module VMware.PowerCLI -Scope CurrentUser`. Sous Linux, PowerCLI s'exécute dans PowerShell 7 (`pwsh`).

### 3.7 Notions PowerShell indispensables

| Notion | Explication | Exemple |
| :--- | :--- | :--- |
| Cmdlet | Commande de la forme Verbe-Nom | `Get-VM`, `New-Snapshot` |
| Objet | PowerShell manipule des objets, pas du texte | `(Get-VM).MemoryGB` |
| Pipeline | Le caractère `\|` transmet les objets à la commande suivante | `Get-VM \| Stop-VM` |
| Variable | Préfixe `$` | `$vmhost = Get-VMHost` |
| Filtrer | `Where-Object` conserve les objets qui répondent à une condition | `Where-Object { $_.CapacityGB -eq 45 }` |
| Sélectionner | `Select-Object` choisit les propriétés | `Select-Object Name, PowerState` |
| Propriété calculée | Colonne définie par un bloc de code | `@{N='Libre%';E={...}}` |
| Découvrir | `Get-Member` liste les propriétés, `Get-Help` la documentation | `Get-VM \| Get-Member` |
| Simulation | `-WhatIf` montre ce qui serait fait sans l'exécuter | `Remove-VM -VM x -WhatIf` |
| Confirmation | `-Confirm:$false` supprime la demande | utilisé partout dans le lab |
| Gestion d'erreur | `-ErrorAction SilentlyContinue` ou `Stop` | voir phase 18 |

À retenir : `$_` désigne l'objet courant dans un `Where-Object` ou un `ForEach-Object`. Certaines variables sont réservées (`$host`, `$args`, `$input`, `$error`) et ne doivent jamais être utilisées comme noms personnels.

### 3.8 Conventions de ce document

- Les blocs `powershell` sont à exécuter dans l'ordre d'une phase.
- Les lignes commençant par `#` sont des commentaires.
- Les chemins sont de type Linux (`/home/holuser/`).
- **Point d'attention** signale une limite connue ou un contournement.

### Points clés à retenir

- ESXi est un hyperviseur de type 1 dont l'API est portée par hostd sur le port 443.
- Un hôte autonome n'offre ni vMotion, ni DRS, ni HA.
- PowerCLI manipule des objets, et `ExtensionData` ou `Get-View` donne accès à l'API native.
- `-WhatIf` et `Get-Member` sont les deux meilleurs réflexes de prudence et de découverte.

### Questions de révision

1. Pourquoi dit-on qu'une VM est « encapsulée » ?
   Réponse : parce qu'elle est entièrement décrite par un ensemble de fichiers (configuration, disques, état), ce qui permet de la copier, la sauvegarder ou la déplacer comme de simples fichiers.
2. Quel composant d'ESXi PowerCLI contacte-t-il sur un hôte autonome ?
   Réponse : l'agent hostd, via HTTPS sur le port 443.
3. Quelle propriété d'un objet PowerCLI donne accès à l'objet natif de l'API ?
   Réponse : `ExtensionData` (ou la cmdlet `Get-View`).
4. Pourquoi ne faut-il pas nommer une variable `$host` ?
   Réponse : c'est une variable automatique de PowerShell qui ne peut pas être écrasée.

---

## 4. Phase 1 : Connexion et inventaire

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait établir une session PowerCLI sécurisée vers un hôte, expliquer pourquoi le certificat est refusé par défaut, maîtriser la notion de mode maintenance, comprendre le rôle du rescan de stockage et produire un inventaire structuré de l'hôte.

### le modèle de session

Une connexion `Connect-VIServer` ouvre une session authentifiée entre la machine d'administration et l'API de l'hôte. Tant que la session est ouverte, chaque cmdlet l'utilise implicitement. PowerCLI mémorise les connexions dans la variable globale `$global:DefaultVIServers`.

Points à connaître :

- Une session inactive finit par expirer. Il suffit alors de se reconnecter.
- Il est possible de se connecter à plusieurs serveurs. Le paramètre `-Server` d'une cmdlet précise sur lequel agir. Le mode par défaut se règle avec `Set-PowerCLIConfiguration -DefaultVIServerMode`.
- `Disconnect-VIServer -Server * -Confirm:$false` ferme proprement toutes les sessions.

### Cours : certificats et confiance

À l'installation, ESXi génère un certificat auto-signé. Aucune autorité reconnue ne l'ayant émis, PowerCLI le refuse par défaut afin d'éviter une attaque de l'homme du milieu. Le paramètre `InvalidCertificateAction` accepte quatre valeurs : `Prompt` (demander), `Warn` (avertir), `Ignore` (ignorer) et `Fail` (refuser).

| Contexte | Pratique recommandée |
| :--- | :--- |
| Lab jetable | `Ignore` est acceptable |
| Production | Remplacer le certificat de l'hôte par un certificat signé par une autorité de confiance, et ne jamais ignorer les erreurs |

La portée du réglage se choisit avec `-Scope` : `Session` (fenêtre en cours), `User` (utilisateur courant, persistant) ou `AllUsers`.

### Cours : l'authentification

Sur un hôte autonome, l'authentification repose sur des comptes locaux. Le compte `root` possède tous les droits. L'objet `PSCredential` transporte le couple identifiant et mot de passe sans le réafficher. La construction avec `ConvertTo-SecureString -AsPlainText -Force` écrit le mot de passe en clair dans le script : c'est toléré en lab, à proscrire en production, où l'on privilégie `Get-Credential` ou un coffre de secrets.

### Le mode maintenance

Le mode maintenance place l'hôte dans un état où il n'accepte plus de démarrage de VM. Sur un cluster vCenter, les VM sont évacuées automatiquement par vMotion. Sur un hôte autonome, il n'y a personne vers qui évacuer : les VM allumées doivent être éteintes ou suspendues avant d'entrer en maintenance.

L'état de connexion d'un hôte (`ConnectionState`) peut valoir `Connected`, `Maintenance`, `Disconnected` ou `NotResponding`. Un hôte qui sort d'un déploiement peut être resté en maintenance, d'où la commande `Set-VMHost -State Connected` du lab.

### Le rescan du stockage

Un rescan demande à l'hôte de redétecter ce qui est connecté à ses adaptateurs de stockage.

| Opération | Effet |
| :--- | :--- |
| Rescan HBA (`-RescanAllHba`) | Cherche de nouveaux périphériques (LUN) sur les adaptateurs |
| Rescan VMFS (`-RescanVmfs`) | Cherche de nouveaux volumes VMFS sur les périphériques déjà connus |

Après la création d'un LUN sur la baie ou après le démarrage d'un lab, l'hôte ne le voit pas tant qu'un rescan n'a pas été fait. La séquence HBA puis VMFS est donc la règle.

### Lire un inventaire

Un inventaire initial est la photographie de départ de l'infrastructure. Il sert de référence pour mesurer les changements. Les cmdlets `Get-*` ne modifient rien et peuvent être lancées sans risque. Pour découvrir toutes les propriétés d'un objet : `Get-VMHost | Format-List *`.

Indicateurs à relever dès le départ : version et build d'ESXi (support et correctifs), nombre de CPU et mémoire (capacité), état de connexion, datastores et espace libre, commutateurs et uplinks.

### Points clés à retenir

- Un certificat auto-signé se tolère uniquement en lab.
- Un hôte autonome n'évacue pas ses VM en entrant en maintenance.
- Rescan HBA pour les périphériques, rescan VMFS pour les volumes.
- Toujours commencer par un inventaire en lecture seule.

### Questions de révision

1. Pourquoi PowerCLI refuse-t-il la connexion par défaut ?
   Réponse : le certificat de l'hôte est auto-signé et donc non reconnu, ce qui pourrait masquer une usurpation.
2. Quelle différence entre `-RescanAllHba` et `-RescanVmfs` ?
   Réponse : le premier détecte de nouveaux périphériques, le second de nouveaux volumes VMFS.
3. Que devient une VM allumée quand un hôte autonome passe en maintenance ?
   Réponse : rien n'est évacué automatiquement, elle doit être arrêtée ou suspendue au préalable.
4. Où est mémorisée la liste des sessions PowerCLI ouvertes ?
   Réponse : dans `$global:DefaultVIServers`.

## Se connecter à la VM ESXI

<img width="1562" height="621" alt="image" src="https://github.com/user-attachments/assets/4888d73a-6d81-461d-9a6a-df4cb697559e" />
<img width="1537" height="551" alt="image" src="https://github.com/user-attachments/assets/8885d16d-5890-4518-b1d1-175ea79f73c3" />
<img width="1562" height="795" alt="image" src="https://github.com/user-attachments/assets/cc7507a2-83db-4907-9ad1-b12360b6852e" />
<img width="1456" height="755" alt="image" src="https://github.com/user-attachments/assets/ea029a1a-4362-4417-b1d1-29d739cad410" />
<img width="1350" height="757" alt="image" src="https://github.com/user-attachments/assets/dddf6927-1556-4435-bd4b-13af415cd0f5" />
<img width="987" height="737" alt="image" src="https://github.com/user-attachments/assets/9b183176-9799-4c27-af5a-5e8ef2673351" />
<img width="992" height="282" alt="image" src="https://github.com/user-attachments/assets/756796c3-0f55-4689-a807-aa327df8ee20" />


Configure PowerCLI pour accepter les certificats auto-signés. Le paramètre `-Scope User` applique le réglage à l'utilisateur courant de façon persistante. `-Confirm:$false` supprime la demande de confirmation.

```powershell
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Scope User -Confirm:$false
```
<img width="1226" height="262" alt="image" src="https://github.com/user-attachments/assets/2edb9393-ca01-4bce-92b6-499c2bf31177" />


Crée un objet d'identifiants. `ConvertTo-SecureString` transforme le mot de passe en chaîne sécurisée. L'option `-AsPlainText -Force` est nécessaire car la source est un texte en clair.

```powershell
$esxiCred = New-Object System.Management.Automation.PSCredential ("root", (ConvertTo-SecureString "VMware123!" -AsPlainText -Force))
```

<img width="1157" height="161" alt="image" src="https://github.com/user-attachments/assets/383cf70e-45d6-42de-b2e9-a42690734926" />


Ouvre la session entre PowerCLI et l'hôte. Toutes les cmdlets suivantes s'exécutent sur cette connexion. Si le DNS échoue, remplacer le nom par `10.0.0.22`.

```powershell
Connect-VIServer -Server "esx-03a.vcf.sddc.lab" -Credential $esxiCred
```

<img width="866" height="217" alt="image" src="https://github.com/user-attachments/assets/ac1b9290-2e35-4d5d-bb1c-7b967065d1f4" />


Fait sortir l'hôte du mode maintenance. Sans cette étape, le démarrage des VM est refusé.
<img width="1335" height="651" alt="image" src="https://github.com/user-attachments/assets/3e911ee9-22c3-4991-b6c3-04e409461d16" />

```powershell
Set-VMHost -VMHost (Get-VMHost) -State Connected -Confirm:$false
```

<img width="1057" height="201" alt="image" src="https://github.com/user-attachments/assets/afe9f414-3571-4a16-80fb-4de2207697ab" />
<img width="1332" height="652" alt="image" src="https://github.com/user-attachments/assets/b06db5ac-15df-4ab5-9924-507acbd50be6" />


Relance la détection sur tous les adaptateurs de stockage (`-RescanAllHba`) puis recherche les volumes VMFS (`-RescanVmfs`). Indispensable dans HOL où le stockage iSCSI peut apparaître après le démarrage.

```powershell
Get-VMHostStorage -VMHost (Get-VMHost) -RescanAllHba -RescanVmfs
```

<img width="902" height="172" alt="image" src="https://github.com/user-attachments/assets/0a43ce34-51a3-4d5a-aa27-cecc8d8ca1d2" />


Stocke l'objet hôte dans une variable réutilisable, puis affiche son nom, sa version, son numéro de build, son nombre de CPU, sa mémoire et son état de connexion.

```powershell
$vmhost = Get-VMHost
$vmhost | Select-Object Name, Version, Build, NumCpu, MemoryTotalGB, ConnectionState
```

<img width="795" height="212" alt="image" src="https://github.com/user-attachments/assets/9aac7937-3d25-410f-9c1c-89a475eabca8" />


Liste les datastores avec leur type (VMFS, NFS), leur capacité et leur espace libre.

```powershell
Get-Datastore | Select-Object Name, Type, CapacityGB, FreeSpaceGB
```

<img width="887" height="295" alt="image" src="https://github.com/user-attachments/assets/86eeb57d-73fa-4130-95e8-f08f69d46e8f" />
<img width="1342" height="610" alt="image" src="https://github.com/user-attachments/assets/e984eadb-495a-45cc-bbcd-b873ae1210df" />


Liste les commutateurs virtuels standard, les cartes physiques associées et la taille de MTU.

```powershell
Get-VirtualSwitch -VMHost $vmhost | Select-Object Name, Nic, Mtu
```

<img width="877" height="216" alt="image" src="https://github.com/user-attachments/assets/143230eb-36da-48a2-8f68-898117f6fad7" />
<img width="1317" height="612" alt="image" src="https://github.com/user-attachments/assets/0d90bc9f-c35c-40ec-ac0d-4a4b38308b14" />



---

## 5. Phase 2 : Réseau standard

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait décrire la chaîne réseau complète d'un hôte ESXi, créer un vSwitch avec uplinks, définir des port groups avec VLAN, expliquer les trois modes de marquage VLAN, configurer une interface VMkernel et appliquer une politique de sécurité adaptée.

### La chaîne réseau d'un hôte

```text
 VM (vNIC)  ---+
 VM (vNIC)  ---+--> Port group --> vSwitch --> Uplink (vmnic) --> Commutateur physique
 vmk0 (gestion)-+       (VLAN)
```

- **vmnic** : carte réseau physique vue par l'hôte (`vmnic0`, `vmnic1`, etc.).
- **vSwitch standard** : commutateur de niveau 2 logiciel, local à l'hôte. Il apprend les adresses MAC de ses VM, mais n'a pas de protocole d'arbre couvrant (pas de boucle possible entre vSwitch) et ne retransmet jamais entre deux uplinks.
- **Port group** : modèle de configuration appliqué aux ports du vSwitch (VLAN, sécurité, teaming, bande passante). Une VM se connecte à un port group, jamais directement au vSwitch.
- **Interface VMkernel (vmk)** : interface IP de l'hôte lui-même, qui porte les services.
- **vSwitch0** : créé à l'installation, avec le port group `VM Network` et l'interface de gestion `vmk0` dans `Management Network`.

Un vSwitch sans uplink est un réseau interne à l'hôte : les VM connectées peuvent communiquer entre elles sur cet hôte, mais pas vers l'extérieur.

### vSwitch standard et vSwitch distribué

| Critère | Standard (VSS) | Distribué (VDS) |
| :--- | :--- | :--- |
| Gestion | Hôte par hôte | Centralisée par vCenter |
| Cohérence | Risque de dérive entre hôtes | Configuration unique appliquée partout |
| Fonctions | De base | Mirroring de ports, NetFlow, LLDP, bande passante entrante et sortante |
| Disponibilité sur hôte autonome | Oui | Non |

Dans le lab, seul le vSwitch standard est possible.

### Les VLAN et le marquage 802.1Q

Un VLAN segmente un réseau physique en domaines de diffusion isolés. La trame porte une étiquette de 12 bits (valeurs 1 à 4094). Trois modes existent :

| Mode | Valeur du VLAN ID sur le port group | Qui pose l'étiquette |
| :--- | :--- | :--- |
| EST (External Switch Tagging) | 0 ou aucun | Le commutateur physique, le port est en accès |
| VST (Virtual Switch Tagging) | 1 à 4094 | Le vSwitch, le port physique doit être un trunk |
| VGT (Virtual Guest Tagging) | 4095 | Le système invité, qui reçoit tous les VLAN |

Le lab utilise VST : `VLAN10-Serveurs` porte le VLAN 10 et `VLAN20-Clients` le VLAN 20. Le port du commutateur physique doit être un trunk qui autorise ces VLAN, sinon le trafic n'a pas d'issue. Une erreur très courante est de configurer le VLAN dans ESXi sans autoriser ce VLAN sur le trunk.

### Les trois paramètres de sécurité

| Paramètre | Comportement si autorisé | Risque |
| :--- | :--- | :--- |
| Promiscuous mode | Une VM reçoit tout le trafic du port group | Écoute du trafic des autres VM |
| MAC address changes | Le système invité peut changer son adresse MAC | Usurpation de l'adresse d'une autre machine |
| Forged transmits | Le système invité peut émettre avec une MAC source différente | Usurpation et attaques de niveau 2 |

Le promiscuous mode est refusé par défaut. Pour MAC address changes et Forged transmits, la valeur par défaut dépend de la version et de la configuration (elle peut être Accept), d'où l'intérêt de les imposer explicitement à Reject, comme le fait le lab. Autoriser ces options est rarement nécessaire (analyseurs réseau, certains équilibreurs de charge ou clusters logiciels). La politique du port group remplace celle du vSwitch.

### Les interfaces VMkernel et leurs services

Une interface VMkernel peut porter un ou plusieurs services : gestion, vMotion, provisioning (migrations à froid et clones), Fault Tolerance, vSAN, réplication, trafic NFS et iSCSI. Chaque service doit être isolé autant que possible dans son VLAN et son sous-réseau. Une pile TCP/IP dédiée (par défaut, vMotion, provisioning) permet d'avoir des passerelles distinctes.

### MTU et formation de la bande passante

- La MTU par défaut est de 1500 octets. Les trames jumbo (9000) réduisent la charge CPU pour le stockage IP, mais exigent la même MTU sur toute la chaîne.
- Sur un vSwitch standard, la limitation de bande passante ne s'applique qu'au trafic sortant de l'hôte (traffic shaping : débit moyen, crête, taille de rafale).

### Bonnes pratiques

- Nommer les port groups de manière explicite (rôle et VLAN).
- Séparer gestion, stockage, vMotion et VM dans des VLAN distincts.
- Prévoir au moins deux uplinks par vSwitch de production.
- Documenter chaque VLAN et le vérifier côté commutateur physique.
- Garder la politique de sécurité la plus restrictive possible.

### Points clés à retenir

- La VM se connecte à un port group, pas au vSwitch.
- VST (VLAN ID de 1 à 4094) exige un trunk côté physique.
- Les trois options de sécurité doivent rester sur Reject sauf besoin justifié.
- Un export CSV est un livrable de documentation.

### Questions de révision

1. Quelle valeur de VLAN ID active le mode VGT ?
   Réponse : 4095.
2. Qu'est-ce qui distingue une interface VMkernel d'une carte réseau de VM ?
   Réponse : l'interface VMkernel appartient à l'hôte et porte ses services (gestion, vMotion, stockage), alors que la carte de VM appartient à un système invité.
3. Quel risque présente l'option Forged transmits ?
   Réponse : permettre à une VM d'émettre avec une adresse MAC source usurpée.
4. Que se passe-t-il si le port physique n'est pas un trunk et que le port group est en VLAN 10 ?
   Réponse : les trames étiquetées ne passent pas, la VM n'a pas de connectivité externe.

### Création d'un `vSwitch1`

Cherche `vSwitch1`. S'il n'existe pas, le crée avec `vmnic1` comme premier uplink et une MTU de 1500, puis recharge l'objet. Ce schéma conditionnel rend le script rejouable sans erreur (idempotence).

```powershell
$vs = Get-VirtualSwitch -VMHost $vmhost -Name 'vSwitch1' -ErrorAction SilentlyContinue
if (-not $vs) {
    New-VirtualSwitch -VMHost $vmhost -Name 'vSwitch1' -Nic 'vmnic1' -Mtu 1500
    $vs = Get-VirtualSwitch -VMHost $vmhost -Name 'vSwitch1'
}
```

<img width="932" height="232" alt="image" src="https://github.com/user-attachments/assets/af99c4ad-8c7c-4094-bbf3-cccf66ff167e" />
<img width="1317" height="606" alt="image" src="https://github.com/user-attachments/assets/88837836-bf68-480b-802a-531623c235fa" />


Ajoute `vmnic3` comme second uplink de `vSwitch1`. Le vSwitch dispose alors de deux cartes, ce qui constitue un teaming et une tolérance de panne.

```powershell
Add-VirtualSwitchPhysicalNetworkAdapter -VirtualSwitch $vs -VMHostPhysicalNic (Get-VMHostNetworkAdapter -VMHost $vmhost -Physical -Name 'vmnic3') -Confirm:$false
```

<img width="1222" height="97" alt="image" src="https://github.com/user-attachments/assets/4184041c-5b72-4b60-b3bc-2b7a25334696" />
<img width="1317" height="612" alt="image" src="https://github.com/user-attachments/assets/aca218ad-aee1-4810-a9e6-ec660dc5afa1" />


Crée deux port groups, l'un étiqueté VLAN 10 pour les serveurs, l'autre VLAN 20 pour les clients. Le trafic des VM connectées est isolé par VLAN.

```powershell
New-VirtualPortGroup -VirtualSwitch $vs -Name 'VLAN10-Serveurs' -VLanId 10
New-VirtualPortGroup -VirtualSwitch $vs -Name 'VLAN20-Clients' -VLanId 20
```

<img width="902" height="240" alt="image" src="https://github.com/user-attachments/assets/7c126d21-15f1-4bca-9346-90c85c1d6a06" />
<img width="1307" height="612" alt="image" src="https://github.com/user-attachments/assets/b77e73d4-cfe2-41a5-adfc-6b37614c238e" />

Récupère la politique de sécurité du port group puis désactive les trois comportements sensibles. C'est une mesure de durcissement qui limite l'écoute du trafic et l'usurpation d'adresse MAC.

```powershell
$pg10 = Get-VirtualPortGroup -VMHost $vmhost -Name 'VLAN10-Serveurs'
Get-SecurityPolicy -VirtualPortGroup $pg10 | Set-SecurityPolicy -AllowPromiscuous $false -ForgedTransmits $false -MacChanges $false -Confirm:$false
```

<img width="1402" height="377" alt="image" src="https://github.com/user-attachments/assets/27d58ece-016f-49c5-bc60-17fc1b4a0d32" />
<img width="1426" height="561" alt="image" src="https://github.com/user-attachments/assets/f623cd13-cdcd-4bc5-aad0-27f524d48de6" />


Crée une interface VMkernel connectée au port group VLAN 10 avec l'adresse `10.0.10.6/24`. L'hôte devient joignable dans ce réseau.

```powershell
New-VMHostNetworkAdapter -VMHost $vmhost -VirtualSwitch $vs -PortGroup 'VLAN10-Serveurs' -IP '10.0.10.6' -SubnetMask '255.255.255.0' -Confirm:$false
```
<img width="1405" height="387" alt="image" src="https://github.com/user-attachments/assets/89c07a9b-8a5d-4d7b-a13f-3a291cdd247d" />
<img width="1410" height="512" alt="image" src="https://github.com/user-attachments/assets/4024eb1f-0832-4121-a24d-4da1bdc19849" />


Exporte l'état du réseau en fichiers CSV. **Point d'attention** : le terminal est sous Linux, il faut des chemins de type `/home/holuser/` et non `C:\`. `-NoTypeInformation` retire la ligne d'en-tête de type PowerShell.

```powershell
Get-VirtualSwitch -VMHost $vmhost | Select-Object Name, Nic, Mtu | Export-Csv -Path "/home/holuser/Rapport-vSwitch.csv" -NoTypeInformation
Get-VirtualPortGroup -VMHost $vmhost | Select-Object Name, VirtualSwitchName, VLanId | Export-Csv -Path "/home/holuser/Rapport-PortGroups.csv" -NoTypeInformation
```

<img width="1418" height="512" alt="image" src="https://github.com/user-attachments/assets/441e3056-3cee-4655-b867-ab32133083a3" />


---

## 6. Phase 3 : Stockage

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait comparer les technologies de stockage de vSphere, expliquer l'architecture iSCSI, décrire les propriétés de VMFS, identifier un LUN par son identifiant canonique, créer un datastore en toute sécurité et suivre l'occupation du stockage.

### Cours : les familles de stockage

| Type | Protocole | Transport | Particularité |
| :--- | :--- | :--- | :--- |
| Local | SATA, SAS, NVMe | Bus interne | Disponible pour un seul hôte |
| Fibre Channel | FC | Réseau dédié (SAN) | Performant, coûteux |
| FCoE | FC sur Ethernet | Ethernet | Réseau convergé |
| iSCSI | SCSI sur IP | Ethernet | Économique, très répandu en lab |
| NFS | Système de fichiers réseau | Ethernet | Stockage fichier, pas de LUN ni de VMFS |
| vSAN | Logiciel | Disques locaux des hôtes | Stockage hyperconvergé |
| vVols | Gestion par VM | Selon la baie | Stratégies de stockage par VM |

On distingue le **stockage bloc** (l'hôte voit un disque brut et y pose VMFS : FC, iSCSI, local) du **stockage fichier** (l'hôte monte un partage : NFS).

### L'architecture iSCSI

```text
 Hôte ESXi (initiateur)                         Baie FreeNAS (cible)
 [adaptateur iSCSI logiciel]  ---- réseau IP ---> [portail : IP:3260]
   IQN : iqn.1998-01.com.vmware:...                 [cible : iqn...]
                                                       [LUN 0 : 85 Go]
                                                       [LUN 1 : 45 Go]
```

- **Initiateur** : le client (l'hôte). **Cible (target)** : le serveur de stockage. **Portail** : adresse IP et port (3260 par défaut) où écoute la cible.
- **IQN** (iSCSI Qualified Name) : nom unique de la forme `iqn.AAAA-MM.domaine-inversé:identifiant`.
- **Adaptateur logiciel** : fourni par ESXi, utilise les cartes réseau normales. Un adaptateur matériel déporte le traitement sur une carte dédiée.
- **Masquage de LUN** : la baie décide quels initiateurs voient quels LUN, grâce à des listes d'accès. Il évite qu'un hôte voie des LUN qui ne lui sont pas destinés.
- Le réseau iSCSI doit être dédié ou isolé (VLAN), idéalement avec des trames jumbo cohérentes de bout en bout.

### Les périphériques et leurs identifiants

Chaque LUN reçoit un identifiant stable. Le **nom canonique** commence généralement par `naa.` (Network Address Authority), parfois `t10.` ou `eui.`. Il est indépendant du chemin et ne change pas au redémarrage, contrairement aux noms de type `vmhba64:C0:T0:L1` qui désignent un chemin. En automatisation, il faut toujours cibler un LUN par son nom canonique et non par un nom de chemin.

### Le système de fichiers VMFS

VMFS est un système de fichiers cluster : plusieurs hôtes peuvent y lire et écrire simultanément, grâce à un verrouillage fin par fichier (ATS, Atomic Test and Set) qui a remplacé les anciennes réservations SCSI sur le volume entier.

| Caractéristique | VMFS 5 | VMFS 6 |
| :--- | :--- | :--- |
| Taille maximale d'un volume | 64 To | 64 To |
| Taille maximale d'un VMDK | 62 To | 62 To |
| Taille de bloc | 1 Mo unifiée | 1 Mo unifiée |
| Récupération d'espace automatique (UNMAP) | Manuelle | Automatique |
| Prise en charge des disques 512e | Non | Oui |
| Format de snapshot | VMFSsparse ou SEsparse selon la taille | SEsparse |

Pour un nouveau datastore, on choisit VMFS 6, qui est la version actuelle. Le disque est partitionné en GPT.

### Provisionnement et occupation

- Le provisionnement dynamique (thin) permet de promettre plus d'espace que n'en contient physiquement le datastore (sur-allocation). Il faut alors surveiller l'espace libre, car un datastore plein suspend les VM qui ont besoin d'écrire.
- Règle d'exploitation répandue : conserver au moins 20 pour cent d'espace libre, pour absorber les snapshots, les fichiers de swap et la croissance des disques thin.
- Le suivi se fait avec un rapport régulier, comme celui exporté en CSV dans le lab.

### Choisir le bon LUN avant de formater

Créer un datastore **détruit** les données présentes sur le LUN. Le lab filtre sur la capacité (45 Go) car il ne contient que deux LUN de taille différente. En production, il faut vérifier le nom canonique, le fabricant, le modèle et s'assurer que le LUN n'est pas déjà utilisé (`Get-Datastore` et `Get-ScsiLun` croisés).

### Bonnes pratiques

- Un LUN par datastore, de taille raisonnable.
- Ne pas mélanger des charges de profils très différents sur un même LUN.
- Nommer les datastores selon un schéma stable (type, site, numéro).
- Suivre l'espace libre et prévoir des alertes.

### Points clés à retenir

- Bloc (iSCSI, FC) et fichier (NFS) sont deux modèles différents.
- IQN identifie initiateur et cible, le portail écoute sur le port 3260.
- Le nom canonique `naa.` est l'identifiant à utiliser dans les scripts.
- VMFS 6 est le choix par défaut pour un nouveau datastore.

### Questions de révision

1. Quel port écoute par défaut une cible iSCSI ?
   Réponse : le port TCP 3260.
2. Pourquoi cibler un LUN par son nom canonique plutôt que par son chemin ?
   Réponse : le nom canonique est stable, le nom de chemin dépend de la topologie et peut changer.
3. Quel est le risque de la sur-allocation en thin ?
   Réponse : que le datastore se remplisse, ce qui suspend les VM qui doivent écrire.
4. Que se passe-t-il pour les données d'un LUN sur lequel on crée un datastore ?
   Réponse : elles sont effacées, le LUN est reformaté en VMFS.

Liste les LUN de type disque vus par l'hôte avec leur identifiant, leur capacité, leur fabricant et leur modèle. Sert à repérer le LUN libre.

```powershell
Get-ScsiLun -VMHost $vmhost -LunType disk | Select-Object CanonicalName, CapacityGB, Vendor, Model
```

<img width="1390" height="422" alt="image" src="https://github.com/user-attachments/assets/a9de37e5-9db1-444c-8bcf-7ee6a8290406" />
<img width="1431" height="572" alt="image" src="https://github.com/user-attachments/assets/5b515161-4421-453f-b773-19a5ce2e408a" />


Affiche les adaptateurs de stockage, leur type (iSCSI, bloc), leur état et l'IQN de l'initiateur.

```powershell
Get-VMHostHba -VMHost $vmhost | Select-Object Device, Type, Status, IScsiName
```

<img width="1407" height="367" alt="image" src="https://github.com/user-attachments/assets/43b46d9c-eb67-4466-8498-9cbe9d1fb0af" />
<img width="1417" height="582" alt="image" src="https://github.com/user-attachments/assets/02dffb44-321e-49bc-8653-c5b373322573" />


Sélectionne le LUN de 45 Go grâce à sa capacité, puis crée un datastore VMFS 6 nommé `ds-iscsi02` dessus. Le filtre par taille est pratique en lab mais, en production, il vaut mieux filtrer par identifiant pour éviter de formater un mauvais LUN.

```powershell
$lunLibre = Get-ScsiLun -VMHost $vmhost -LunType disk | Where-Object { $_.CapacityGB -eq 45 }
New-Datastore -VMHost $vmhost -Name 'ds-iscsi02' -Path $lunLibre.CanonicalName -Vmfs -FileSystemVersion "6"
```
<img width="1407" height="372" alt="image" src="https://github.com/user-attachments/assets/ad20eb23-fc69-450e-b99c-87f9f9443f35" />
<img width="1396" height="572" alt="image" src="https://github.com/user-attachments/assets/6785b8cd-6595-4292-bd43-ad75b9824143" />


Produit un rapport de stockage. La propriété calculée `%Utilisé` applique la formule (capacité moins espace libre) divisé par capacité, multiplié par 100, arrondi à deux décimales.

```powershell
Get-Datastore | Select-Object Name, Type, CapacityGB, FreeSpaceGB, @{N='%Utilisé';E={[math]::Round((($_.CapacityGB - $_.FreeSpaceGB) / $_.CapacityGB) * 100, 2)}} | Export-Csv -Path "/home/holuser/Rapport-Stockage.csv" -NoTypeInformation
```

<img width="1414" height="451" alt="image" src="https://github.com/user-attachments/assets/c7c47458-66aa-41b3-9c40-dafef98b1114" />
<img width="1412" height="502" alt="image" src="https://github.com/user-attachments/assets/c9b056e3-c20e-4deb-b0e1-6ed379a935f5" />




---

## 7. Phase 4 : Cycle de vie et matériel des machines virtuelles

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait décrire les fichiers qui composent une VM, choisir le matériel virtuel adapté, comparer les formats de disque, expliquer la gestion de la mémoire par l'hyperviseur, créer et démarrer une VM par script et contourner la limite de résolution réseau d'un hôte autonome.

### Cours : les fichiers d'une machine virtuelle

Une VM est un dossier sur un datastore contenant plusieurs fichiers :

| Fichier | Rôle |
| :--- | :--- |
| `nom.vmx` | Configuration de la VM (CPU, mémoire, périphériques) en texte |
| `nom.vmdk` | Descripteur du disque virtuel (métadonnées) |
| `nom-flat.vmdk` | Données brutes du disque (thick ou thin) |
| `nom-000001.vmdk` et `-delta.vmdk` | Disques de différence créés par les snapshots |
| `nom.nvram` | État du BIOS ou du firmware UEFI |
| `nom.vswp` | Fichier de swap de la VM, créé au démarrage |
| `nom.vmsd` et `nom-Snapshot1.vmsn` | Catalogue et état des snapshots |
| `vmware.log` | Journal de la VM |

La taille du fichier `.vswp` est égale à la mémoire configurée moins la réservation mémoire. Une VM de 4 Go sans réservation consomme donc 4 Go de datastore dès qu'elle démarre : il faut en tenir compte dans le dimensionnement.

### Le matériel virtuel

- **Version de matériel** (`vmx-NN`) : détermine les fonctions disponibles et la compatibilité avec les versions d'ESXi. On ne monte la version que lorsque nécessaire, car elle ne peut pas être abaissée.
- **vCPU** : processeur virtuel, ordonnancé sur un cœur physique. Plus de vCPU n'est pas toujours plus de performance : une VM surdimensionnée attend plus longtemps l'accès aux cœurs.
- **Contrôleurs de disque** : LSI Logic (compatible), **PVSCSI** (paravirtualisé, performances élevées, recommandé quand le système invité l'accepte), NVMe.
- **Cartes réseau** : E1000e (émulée, compatible) et **VMXNET3** (paravirtualisée, la plus performante, exige VMware Tools ou le pilote du noyau).
- **GuestId** : indique le système d'exploitation prévu (`ubuntu64Guest`). ESXi s'en sert pour proposer les périphériques et paramètres par défaut. Un GuestId incorrect n'empêche pas de démarrer mais peut dégrader les performances ou les fonctions.

### Les formats de disque

| Format (PowerCLI) | Allocation | Initialisation | Usage |
| :--- | :--- | :--- | :--- |
| `Thin` | À la demande | À la demande | Économie d'espace, lab, charges légères |
| `Thick` (lazy zeroed) | Complète à la création | Au premier accès à chaque bloc | Compromis |
| `EagerZeroedThick` | Complète à la création | Complète à la création | Performances constantes, requis pour Fault Tolerance et certains clusters |

Le format thin consomme l'espace réellement écrit : un disque de 10 Go peut n'occuper que quelques centaines de mégaoctets. C'est un choix économique, mais il exige une surveillance de l'espace du datastore.

### La gestion de la mémoire

Quand la demande dépasse la mémoire physique, ESXi dispose de plusieurs mécanismes, appliqués selon l'état de pression mémoire de l'hôte :

1. **Partage transparent de pages** : déduplication de pages identiques. Le partage entre VM différentes est désactivé par défaut depuis vSphere 6.0 pour des raisons de sécurité.
2. **Ballooning** : le pilote `vmmemctl` des VMware Tools gonfle un ballon dans la VM pour lui faire libérer des pages inutilisées. Sans VMware Tools, ce mécanisme est indisponible.
3. **Compression** : les pages à évacuer sont compressées en mémoire, ce qui est plus rapide qu'un swap disque.
4. **Swap de l'hôte** : dernier recours, les pages sont écrites dans le fichier `.vswp`. Il dégrade fortement les performances.

À retenir : la mémoire sur-allouée est possible, mais le swap de l'hôte est le symptôme d'un dimensionnement insuffisant.

### Le cycle de vie d'une VM

Création, configuration, démarrage, suspension, snapshot, clonage, migration, export, suppression. Les états d'alimentation sont `PoweredOn`, `PoweredOff` et `Suspended`. Une VM sans VMware Tools ne reçoit pas d'ordre d'arrêt propre : il faut la mettre hors tension.

### Pourquoi `-NetworkName` échoue sur un hôte autonome

À la création, `New-VM -NetworkName` cherche le port group dans l'inventaire PowerCLI. Sur un ESXi autonome, cette résolution par nom peut échouer (`network doesn't exist on host`) alors que le port group existe. La méthode fiable consiste à créer la VM sans réseau, puis à attacher la carte avec l'objet port group, qui évite toute résolution par nom. C'est un exemple typique de contournement par objet.

### Points clés à retenir

- Une VM est un dossier de fichiers, dont `.vmx` (configuration) et `.vmdk` (disque).
- Le fichier `.vswp` occupe de l'espace égal à la mémoire moins la réservation.
- Thin économise l'espace mais demande de la surveillance.
- Sur hôte autonome, rattacher la carte réseau par l'objet port group.

### Questions de révision

1. Quelle est la différence entre `.vmdk` et `-flat.vmdk` ?
   Réponse : le premier est le descripteur (métadonnées), le second contient les données du disque.
2. Quel format de disque consomme le moins d'espace à la création ?
   Réponse : le format thin.
3. Dans quel ordre ESXi applique-t-il les mécanismes de récupération mémoire ?
   Réponse : partage de pages, ballooning, compression, puis swap de l'hôte.
4. Pourquoi une VM sans VMware Tools est-elle limitée ?
   Réponse : pas d'arrêt propre, pas de ballooning, pas de pilotes paravirtualisés ni de remontée d'informations de l'invité.

Crée une VM d'un vCPU, 1 Go de RAM et 10 Go de disque sur `ds-iscsi01`, sans carte réseau.

```powershell
$ds = Get-Datastore -Name 'ds-iscsi01'
New-VM -Name 'VM-Network-Test' -VMHost $vmhost -Datastore $ds -NumCpu 1 -MemoryGB 1 -DiskGB 10 -GuestId 'ubuntu64Guest' -Confirm:$false
```

<img width="1416" height="420" alt="image" src="https://github.com/user-attachments/assets/2006a1d3-d43c-40d1-9e2d-55260cfd8e2c" />
<img width="1411" height="575" alt="image" src="https://github.com/user-attachments/assets/ec8a19ef-a0f8-455e-9cc6-01b57c39493f" />


Ajoute une carte réseau connectée au port group par défaut `VM Network`. `-StartConnected` connecte la carte au démarrage de la VM.

```powershell
$vm = Get-VM -Name 'VM-Network-Test'
New-NetworkAdapter -VM $vm -NetworkName 'VM Network' -StartConnected -Confirm:$false
```

<img width="1255" height="422" alt="image" src="https://github.com/user-attachments/assets/7d186fec-fbd1-4242-8084-7e8c16636883" />
<img width="1415" height="555" alt="image" src="https://github.com/user-attachments/assets/8b6e9ad2-ade3-4edd-a2d9-c723d5737225" />
<img width="1402" height="570" alt="image" src="https://github.com/user-attachments/assets/24792f35-8082-4322-a35a-e9ece6e0d043" />


Récupère l'objet port group, sélectionne la dernière carte de la VM, puis la bascule sur le VLAN 10 en passant l'objet et non le nom. C'est ce passage par objet qui contourne tout.

```powershell
$pg = Get-VirtualPortGroup -VMHost $vmhost -Name 'VLAN10-Serveurs'
$nic = Get-NetworkAdapter -VM $vm | Select-Object -Last 1
Set-NetworkAdapter -NetworkAdapter $nic -Portgroup $pg -Confirm:$false
```

<img width="1316" height="417" alt="image" src="https://github.com/user-attachments/assets/fcbf7262-3e85-4903-8a27-cb1536d1e1eb" />


Démarre la VM puis affiche, pour chaque carte, le réseau attaché et le type d'adaptateur, afin de valider la configuration.

```powershell
Start-VM -VM 'VM-Network-Test' -Confirm:$false
Get-NetworkAdapter -VM 'VM-Network-Test' | Select-Object NetworkName, Type
```

<img width="1207" height="422" alt="image" src="https://github.com/user-attachments/assets/9e4b9cce-2424-4a5b-aefb-dbfa0a7d96eb" />
<img width="1402" height="562" alt="image" src="https://github.com/user-attachments/assets/2f89f472-5f60-4ab4-8bb9-c63a1f6b995d" />



---

## 8. Phase 5 : Snapshots

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer le fonctionnement technique d'un snapshot, distinguer les options mémoire et quiesce, utiliser les opérations revert et delete à bon escient, et justifier pourquoi un snapshot n'est pas une sauvegarde.

### Cours : le principe technique

Quand on crée un snapshot, le disque d'origine devient en lecture seule. ESXi crée un **disque delta** (fichier `-000001.vmdk` et `-delta.vmdk`) qui reçoit toutes les écritures suivantes. Pour lire une donnée, l'hyperviseur consulte d'abord le delta puis, si le bloc n'y figure pas, le disque d'origine.

```text
 Avant         : [Disque de base]  <-- lectures et écritures
 Après snapshot: [Disque de base (figé)] <-- [Delta 1]  <-- lectures et écritures
 Deuxième snap : [Base] <-- [Delta 1 (figé)] <-- [Delta 2] <-- écritures
```

Un snapshot enregistre :

- l'état du disque (via les deltas),
- la configuration de la VM (fichier `.vmx` au moment de la capture),
- optionnellement l'état de la mémoire (fichier `.vmsn`).

C'est pourquoi un retour en arrière annule aussi les changements de matériel, comme l'ajout d'une carte réseau dans le lab.

### Les options de création

| Option | Effet | Coût | Quand l'utiliser |
| :--- | :--- | :--- | :--- |
| Memory | Sauvegarde le contenu de la RAM. Le retour restaure la VM allumée, dans son état d'exécution | Capture plus longue, fichier de la taille de la mémoire | Avant un test dont on veut revenir à l'instant exact |
| Quiesce | Demande aux VMware Tools de geler les systèmes de fichiers et les applications compatibles (VSS sous Windows) | Court gel de la VM | Pour obtenir un état cohérent des données |
| Aucune des deux | Capture l'état du disque, comme après une coupure brutale | Aucun | VM de test, VM éteinte |

Quiesce nécessite VMware Tools. Sans lui, l'option échoue ou n'a pas d'effet.

### Les opérations

| Opération | Effet |
| :--- | :--- |
| Revert (retour) | La VM revient à l'état du snapshot. Les données écrites depuis sont perdues |
| Delete (suppression) | Le snapshot disparaît et son delta est fusionné dans le disque parent. Aucun changement d'état de la VM |
| Delete all | Fusionne toute la chaîne |
| Consolidation | Fusion de deltas orphelins quand une suppression a échoué ou laissé des fichiers |

Une confusion fréquente est de croire que supprimer un snapshot revient en arrière. C'est l'inverse : la suppression valide l'état actuel en fusionnant les deltas, alors que le revert l'abandonne.

### Limites et risques

- Les performances se dégradent quand la chaîne s'allonge, car chaque lecture peut traverser plusieurs deltas.
- Un delta grossit à chaque écriture et peut atteindre la taille du disque d'origine. Il peut remplir un datastore.
- La suppression d'un snapshot ancien peut être longue et générer beaucoup d'activité disque.
- Un disque configuré en mode indépendant n'est pas concerné par les snapshots.
- Recommandation courante : peu de snapshots (un à trois), conservés quelques jours au plus.

### Pourquoi un snapshot n'est pas une sauvegarde

Les deltas dépendent du disque de base et sont stockés sur le même datastore. Si le datastore est perdu, la VM et ses snapshots disparaissent ensemble. Un snapshot sert de point de retour temporaire avant une opération risquée (mise à jour, changement de configuration), pas de protection durable des données.

### Points clés à retenir

- Un snapshot fige le disque et dirige les écritures vers un delta.
- Memory conserve l'état d'exécution, Quiesce garantit la cohérence des données.
- Revert abandonne les changements, Delete les valide.
- Un snapshot reste un état temporaire.

### Questions de révision

1. Que contient le disque delta ?
   Réponse : toutes les écritures faites depuis la création du snapshot.
2. Quelle opération annule les modifications faites depuis la capture ?
   Réponse : le revert.
3. Pourquoi un long maintien de snapshots est-il déconseillé ?
   Réponse : dégradation des performances, croissance des deltas, durée de la consolidation et risque de saturation du datastore.
4. L'ajout d'une carte réseau avant un revert est-il conservé ?
   Réponse : non, la configuration matérielle revient à celle du snapshot.

Crée un snapshot sans mémoire ni mise au repos du système de fichiers. Rapide, adapté à une VM de test.

```powershell
New-Snapshot -VM 'VM-Network-Test' -Name 'Snap-Avant-Modif' -Description 'Snapshot de test' -Memory:$false -Quiesce:$false
```

<img width="1411" height="362" alt="image" src="https://github.com/user-attachments/assets/b9dadab5-cfe4-413b-a50f-0e2b7ce2b7ca" />
<img width="1405" height="565" alt="image" src="https://github.com/user-attachments/assets/70059abe-6b5f-46f3-92c1-1a4b9ade8492" />


Effectue une modification de matériel (ajout d'une carte réseau) afin de pouvoir constater ensuite son annulation.

```powershell
New-NetworkAdapter -VM 'VM-Network-Test' -NetworkName 'VM Network' -Confirm:$false
```

<img width="1245" height="372" alt="image" src="https://github.com/user-attachments/assets/8b61d337-77c6-48c2-b9d3-47291d44a390" />
<img width="1411" height="571" alt="image" src="https://github.com/user-attachments/assets/54b71ead-e4cd-4ada-b04b-d70a3dfca709" />


Retrouve le snapshot puis y revient. La carte ajoutée à l'étape précédente disparaît.

```powershell
$snap = Get-Snapshot -VM 'VM-Network-Test' -Name 'Snap-Avant-Modif'
Set-VM -VM 'VM-Network-Test' -Snapshot $snap -Confirm:$false
```

<img width="1261" height="342" alt="image" src="https://github.com/user-attachments/assets/a90ddb8d-d8c2-452d-9a41-e0f39e614c23" />
<img width="1417" height="567" alt="image" src="https://github.com/user-attachments/assets/7e4415ed-b1f5-4c52-85a2-2dcef25568c1" />


Supprime le snapshot et consolide les disques delta.

```powershell
Remove-Snapshot -Snapshot $snap -Confirm:$false
```

<img width="891" height="150" alt="image" src="https://github.com/user-attachments/assets/a492216e-3d06-4487-aa15-fe3cce892d01" />



---

## 9. Phase 6 : Montage d'une image ISO

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer comment une VM accède à un média optique virtuel, comprendre le rôle de la Content Library et son incompatibilité avec le montage direct sur un hôte autonome, naviguer dans un datastore avec PowerShell, configurer le démarrage sur CD et détacher l'ISO après installation.

### Les médias d'une VM

Le lecteur CD/DVD virtuel d'une VM peut être associé à trois types de support :

| Type | Description |
| :--- | :--- |
| Fichier ISO sur datastore | Le fichier est lu depuis le stockage. Il est partagé et accessible à tout moment |
| Périphérique client | Un lecteur ou une ISO de la machine de l'opérateur, disponible tant que la console est ouverte |
| Périphérique de l'hôte | Le lecteur physique de l'hôte |

Le lab utilise le premier type, le seul adapté à l'automatisation. Le chemin se note `[datastore] dossier/fichier.iso`. L'option `StartConnected` connecte le lecteur à chaque démarrage de la VM.

### La Content Library

La Content Library est un service de vCenter qui centralise modèles de VM, OVF et ISO, avec possibilité de publication et d'abonnement entre sites. Physiquement, ses éléments sont stockés dans des dossiers du datastore nommés `contentlib-<UUID>`, avec des sous dossiers et des noms de fichiers à base d'identifiants. Ces fichiers sont gérés par le service : ils ne sont pas destinés à un montage direct par une VM.

Sur un hôte autonome, l'API refuse de monter une ISO située dans ces dossiers (`Invalid configuration for device '0'`). La solution est de copier l'ISO dans un dossier ordinaire du datastore, comme `ISO`, puis de monter la copie.

### Parcourir un datastore avec PowerShell

Le fournisseur `VimDatastore` expose un datastore comme un lecteur de fichiers. Après `New-PSDrive`, on utilise les commandes habituelles (`Get-ChildItem`, `New-Item`, `Copy-DatastoreItem`) avec des chemins de la forme `ds:\dossier\fichier`. Il faut démonter le lecteur avec `Remove-PSDrive` à la fin. Le même résultat est possible depuis l'onglet de navigation du datastore dans le Host Client.

### Le démarrage de la VM

- **Firmware** : une VM démarre en BIOS ou en UEFI. UEFI est nécessaire pour Secure Boot et certains systèmes récents.
- **Ordre de démarrage** : liste des périphériques testés (disque, CD, réseau). Quand le premier périphérique n'est pas amorçable, la VM passe au suivant de la liste.
- **`bios.bootDeviceClasses`** : paramètre avancé du fichier `.vmx` qui précise les classes de périphériques autorisées au démarrage. La valeur `allow:cd,hd` autorise le CD puis le disque. On l'écrit via l'objet API `VirtualMachineConfigSpec` et `extraConfig` quand la cmdlet de gestion de l'ordre de démarrage est défaillante.
- On modifie ce paramètre VM éteinte, pour qu'il soit pris en compte au démarrage suivant.

### Après l'installation

Une fois le système installé, il faut déconnecter l'ISO pour que la VM ne redémarre pas sur l'installateur à chaque fois :

```powershell
Get-CDDrive -VM 'VM-Network-Test' | Set-CDDrive -NoMedia -Confirm:$false
```

Cette commande retire le média du lecteur (`NoMedia`) sans supprimer le lecteur. Pour retirer aussi le lecteur : `Get-CDDrive -VM ... | Remove-CDDrive`.

### À propos de Photon OS

Photon OS est une distribution Linux légère de VMware, conçue pour les conteneurs et les environnements virtualisés. Sa version minimale se prête bien aux tests de VM en lab. L'ISO du lab provient de la Content Library de l'environnement.

### Points clés à retenir

- Le montage d'ISO automatisé passe par un fichier ISO du datastore.
- Les dossiers `contentlib-...` ne doivent pas être montés directement sur un hôte autonome.
- Le paramètre `bios.bootDeviceClasses` pilote le démarrage via l'API.
- Détacher l'ISO après installation.

### Questions de révision

1. Quel est le format d'un chemin de fichier sur datastore pour PowerCLI ?
   Réponse : `[nom-du-datastore] dossier/fichier`.
2. Pourquoi copier l'ISO hors de la Content Library ?
   Réponse : l'API d'un hôte autonome refuse de monter directement un fichier de dossier `contentlib-...`.
3. Comment retirer le média d'un lecteur sans le supprimer ?
   Réponse : `Set-CDDrive -NoMedia`.
4. Que fait `Remove-PSDrive -Name ds` ?
   Réponse : il démonte le lecteur PowerShell qui représentait le datastore.

### Configuration

Monte le datastore comme lecteur `ds:` puis crée le dossier `ISO` s'il n'existe pas. `Out-Null` masque la sortie du montage.

```powershell
$ds = Get-Datastore -Name 'ds-iscsi01'
New-PSDrive -Name ds -PSProvider VimDatastore -Root '\' -Location $ds | Out-Null
New-Item -Path ds:\ISO -ItemType Directory -ErrorAction SilentlyContinue
```

<img width="1176" height="300" alt="image" src="https://github.com/user-attachments/assets/70e219f7-0630-432c-86c8-a46cf8112356" />
<img width="1396" height="562" alt="image" src="https://github.com/user-attachments/assets/7ed66413-b918-4f40-a55e-b01f72740ed5" />


Copie l'ISO Photon OS depuis la Content Library vers `ds:\ISO\photon.iso`. 

```powershell
$sourceIso = "ds:\contentlib-1016f267-18c0-4796-bb78-98a2cf230a70\f63e27c2-89ce-47a3-a1b0-0218673c75ae\photon-minimal-5.0-dde71ec57.x86_64_27fd3db5-01d9-468e-aea4-8df41090cea9.iso"
$destIso = "ds:\ISO\photon.iso"
Copy-DatastoreItem -Item $sourceIso -Destination $destIso
```

<img width="1411" height="297" alt="image" src="https://github.com/user-attachments/assets/a059d353-089c-4f4d-af39-596e40cf2f1a" />
<img width="1405" height="557" alt="image" src="https://github.com/user-attachments/assets/021042fa-1670-41f3-be03-6688cdc03853" />


Arrête la VM (l'erreur est ignorée si elle l'est déjà) puis ajoute un lecteur CD pointant vers l'ISO. La syntaxe `[datastore] dossier/fichier` est le format de chemin natif de VMware.

```powershell
Stop-VM -VM 'VM-Network-Test' -Confirm:$false -ErrorAction SilentlyContinue
New-CDDrive -VM 'VM-Network-Test' -ISOPath '[ds-iscsi01] ISO/photon.iso' -StartConnected -Confirm:$false
```

<img width="1406" height="402" alt="image" src="https://github.com/user-attachments/assets/f4898aaf-a335-48bb-ba25-c684d58bbb18" />
<img width="1420" height="567" alt="image" src="https://github.com/user-attachments/assets/009487d0-ec26-4c96-a90d-8d35e9fcfd82" />


Utilise l'objet API natif (`Get-View`) pour écrire le paramètre `bios.bootDeviceClasses` avec la valeur `allow:cd,hd`. La VM peut alors démarrer sur le CD puis sur le disque. Cela contourne le dysfonctionnement de la cmdlet de gestion du boot order.

```powershell
$vm = Get-VM -Name 'VM-Network-Test'
$vmView = $vm | Get-View
$spec = New-Object VMware.Vim.VirtualMachineConfigSpec
$opt = New-Object VMware.Vim.OptionValue
$opt.Key = "bios.bootDeviceClasses"
$opt.Value = "allow:cd,hd"
$spec.extraConfig = @($opt)
$vmView.ReconfigVM($spec)
```

<img width="1155" height="322" alt="image" src="https://github.com/user-attachments/assets/92adf82e-49a4-46a7-835d-2c574eb44688" />
<img width="652" height="115" alt="image" src="https://github.com/user-attachments/assets/53929241-35fc-4c84-9e73-c8c75ace8167" />
<img width="1421" height="562" alt="image" src="https://github.com/user-attachments/assets/cf25c99b-c9cd-4b98-99fa-3600a22bddb5" />



Démarre la VM puis démonte le lecteur `ds:` pour libérer la ressource.
```powershell
Start-VM -VM 'VM-Network-Test' -Confirm:$false
Remove-PSDrive -Name ds
```

<img width="1206" height="372" alt="image" src="https://github.com/user-attachments/assets/a8bfe27e-6e10-498c-85d5-07f326faf8c7" />
<img width="1412" height="566" alt="image" src="https://github.com/user-attachments/assets/4042af47-580e-4a39-a5bc-816b6f727f97" />




---

## 10. Phase 7 : Déplacement d'une VM (cold migration)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait classer les types de migration de VM, expliquer pourquoi le Storage vMotion est indisponible sur un hôte autonome, décrire la structure d'un disque VMDK, appliquer une méthode correcte de migration à froid et mesurer ses conséquences.

### Les types de migration

| Type | VM | Ce qui bouge | Prérequis | Disponible sur hôte autonome |
| :--- | :--- | :--- | :--- | :--- |
| vMotion | Allumée | Calcul (hôte) | vCenter, stockage partagé, réseau vMotion | Non |
| Storage vMotion | Allumée | Stockage (datastore) | vCenter | Non |
| vMotion sans stockage partagé | Allumée | Calcul et stockage | vCenter | Non |
| Cold migration | Éteinte | Calcul et ou stockage | Aucun mécanisme spécifique | Oui, par copie ou recréation |
| Migration d'une VM suspendue | Suspendue | Calcul et ou stockage | vCenter | Non |

Les migrations à chaud reposent sur des services de vCenter. C'est pourquoi `Move-HardDisk` renvoie `The operation is not supported` sur un hôte seul.

### La structure d'un disque VMDK

Un disque est formé de deux fichiers qui forment un tout : le descripteur `.vmdk` et le fichier de données `-flat.vmdk`. Le descripteur référence le fichier de données par son nom. Copier uniquement le descripteur crée un disque qui pointe vers un fichier absent, d'où `disk path is not accessible`. Cette erreur illustre l'importance de traiter le couple comme un seul objet.

### Méthodes de déplacement à froid

| Méthode | Principe | Avantage | Limite |
| :--- | :--- | :--- | :--- |
| Recréer un disque (méthode du lab) | Détacher l'ancien disque, créer un disque neuf sur le datastore cible | Simple, fiable, scriptable | Le contenu n'est pas transféré |
| Clonage de disque avec `vmkfstools -i` | Copie correcte du couple VMDK, avec conversion de format | Conserve les données | Exige un accès shell à l'hôte |
| Copie via le navigateur de datastore | Copie des deux fichiers par l'interface | Conserve les données | Manuel |
| Passage par vCenter | Storage vMotion ou Migrate | Aucun arrêt | Exige vCenter |

Exemple de référence avec `vmkfstools` (à exécuter en SSH sur l'hôte, VM éteinte) :

```bash
vmkfstools -i /vmfs/volumes/ds-iscsi01/VM-Network-Test/VM-Network-Test.vmdk \
              /vmfs/volumes/ds-iscsi02/VM-Network-Test/VM-Network-Test.vmdk -d thin
```

Cette commande clone le disque en thin vers le nouveau datastore en gérant correctement le couple de fichiers. Il reste ensuite à rattacher ce disque à la VM avec `New-HardDisk -DiskPath`. Pour déplacer aussi les fichiers de configuration (`.vmx`), on copie le dossier de la VM, puis on enregistre la VM avec `New-VM -VMFilePath '[ds-iscsi02] dossier/VM.vmx' -VMHost $vmhost`.

### Conséquences de la méthode du lab

Le disque créé est vide. Le système installé et les données ne sont pas reportés. Dans le lab, l'objectif est de pratiquer les cmdlets, donc c'est sans conséquence. Dans un cas réel, cette méthode est inadaptée si la VM contient des données à conserver.

Remarque : `Remove-HardDisk` sans `-DeletePermanently` détache seulement le disque. Les fichiers restent sur l'ancien datastore et occupent de l'espace. Pour les supprimer définitivement, ajouter `-DeletePermanently`, ou les supprimer via le navigateur de datastore.

### Étapes d'une cold migration réelle

1. Planifier une fenêtre de maintenance et prévenir les utilisateurs.
2. Vérifier l'espace disponible sur la cible.
3. Éteindre proprement la VM.
4. Copier ou cloner correctement le disque et la configuration.
5. Enregistrer la VM sur la cible et contrôler le démarrage.
6. Supprimer la source seulement après validation.

### Points clés à retenir

- vMotion et Storage vMotion exigent vCenter.
- Un VMDK, ce sont deux fichiers, descripteur et flat.
- La méthode du lab ne conserve pas les données.
- `vmkfstools -i` est la méthode de clonage correcte en ligne de commande.

### Questions de révision

1. Pourquoi `Move-HardDisk` échoue-t-il sur un ESXi autonome ?
   Réponse : le Storage vMotion repose sur vCenter, indisponible sur un hôte seul.
2. Que contient le fichier `-flat.vmdk` ?
   Réponse : les données brutes du disque.
3. Quelle est la limite de la méthode « recréer un disque » ?
   Réponse : le nouveau disque est vide, les données de l'ancien ne sont pas transférées.
4. Comment enregistrer une VM déplacée dans l'inventaire ?
   Réponse : avec `New-VM -VMFilePath '[datastore] dossier/VM.vmx' -VMHost $vmhost`.

### Configuration

Éteint la VM, prérequis d'une migration à froid.

```powershell
Stop-VM -VM 'VM-Network-Test' -Confirm:$false
```

<img width="917" height="342" alt="image" src="https://github.com/user-attachments/assets/be586c34-7b75-46d9-b62a-319b38b59ab0" />


Détache le disque de la VM. Sans l'option `-DeletePermanently`, les fichiers restent présents sur l'ancien datastore, ils ne sont simplement plus référencés.

```powershell
Get-HardDisk -VM 'VM-Network-Test' | Remove-HardDisk -Confirm:$false -ErrorAction SilentlyContinue
```

<img width="1382" height="181" alt="image" src="https://github.com/user-attachments/assets/1a119ab7-16de-472c-942a-6771ccebf845" />


Crée un disque de 10 Go en provisionnement dynamique directement sur `ds-iscsi02` et l'attache à la VM.

```powershell
New-HardDisk -VM 'VM-Network-Test' -CapacityGB 10 -Datastore (Get-Datastore -Name 'ds-iscsi02') -StorageFormat Thin -Confirm:$false
```

<img width="1397" height="392" alt="image" src="https://github.com/user-attachments/assets/823849e7-b1c0-430b-8de9-fc9c7898bc93" />
<img width="1412" height="562" alt="image" src="https://github.com/user-attachments/assets/e21848a2-1b94-4070-94d9-8c807539cf2e" />


Vérifie que le chemin du disque pointe bien vers `[ds-iscsi02]`, puis redémarre la VM.

```powershell
Get-HardDisk -VM 'VM-Network-Test' | Select-Object Filename
Start-VM -VM 'VM-Network-Test' -Confirm:$false
```

<img width="1117" height="392" alt="image" src="https://github.com/user-attachments/assets/801817cd-f77a-4439-a721-401412cd5796" />



---

## 11. Phase 8 : Durcissement et sécurité

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer la défense en profondeur appliquée à ESXi, comprendre le rôle des services de l'hôte et de leurs politiques de démarrage, justifier l'importance de la synchronisation horaire et appliquer les premières mesures de durcissement.

### La défense en profondeur

La sécurité d'un hôte ne repose jamais sur une seule mesure. On empile des couches complémentaires :

| Couche | Exemples |
| :--- | :--- |
| Réseau | Isolation du réseau de gestion, VLAN, pare-feu de l'hôte |
| Hôte | Services minimaux, SSH désactivé, comptes nominatifs |
| Accès | Rôles, moindre privilège, verrouillage de compte |
| Système | Secure Boot, TPM, certificats, niveau d'acceptation des paquets |
| Supervision | Journaux centralisés, alertes, audit régulier |

La démarche est cyclique : définir une base de référence, mesurer l'écart, corriger, vérifier, recommencer. Les référentiels courants sont le Security Configuration Guide de VMware, le benchmark CIS pour ESXi et les guides DISA STIG.

### Les services de l'hôte

ESXi exécute des services identifiés par une clé. Les plus importants :

| Clé | Service | Remarque de sécurité |
| :--- | :--- | :--- |
| `TSM-SSH` | Accès SSH | Désactivé par défaut, ne l'activer que pour un dépannage |
| `TSM` | Shell ESXi local | Idem |
| `ntpd` | Synchronisation horaire | À activer, démarrage automatique |
| `DCUI` | Console locale | À protéger physiquement |
| `lbtd` | Équilibrage de charge du teaming | Service interne |
| `vmsyslogd` | Journalisation | Ne pas arrêter |

La **politique de démarrage** d'un service prend trois valeurs : `on` (démarre avec l'hôte), `off` (démarrage manuel) et `automatic` (démarre et s'arrête selon l'ouverture des ports du pare-feu). Changer la politique ne démarre ni n'arrête le service : c'est une action distincte (`Start-VMHostService`, `Stop-VMHostService`).

### Pourquoi le temps est critique

Une horloge dérivante provoque des problèmes difficiles à diagnostiquer :

- les journaux de plusieurs systèmes ne peuvent plus être corrélés,
- Kerberos et Active Directory refusent les authentifications si l'écart dépasse environ cinq minutes,
- les certificats peuvent sembler non valides ou expirés,
- les tâches planifiées et les licences à durée limitée se comportent de façon imprévisible.

On synchronise donc l'hôte sur une ou plusieurs sources NTP fiables. En production, on privilégie des serveurs internes, au moins trois pour permettre un vote majoritaire. Dans HOL, l'accès à des serveurs publics peut être filtré : l'absence de synchronisation n'est alors pas une erreur du script.

### Cours : variables réservées de PowerShell

`$host` est une variable automatique qui représente l'application hôte PowerShell. Tenter de la réaffecter provoque `Cannot overwrite variable Host`. D'autres noms à éviter : `$args`, `$input`, `$error`, `$pid`, `$home`, `$pwd`. Adopter un préfixe explicite comme `$esxi` ou `$vmhost` supprime ce risque.

### Bonnes pratiques

- Désactiver SSH et le shell quand ils ne sont pas nécessaires, et les arrêter après usage.
- Utiliser des comptes nominatifs plutôt que `root`.
- Synchroniser l'heure sur plusieurs sources.
- Vérifier chaque changement avec une commande de lecture.

### Points clés à retenir

- La sécurité est une superposition de couches.
- Politique de service et état d'exécution sont deux notions distinctes.
- NTP est une dépendance de l'authentification, des journaux et des certificats.
- Ne jamais utiliser une variable automatique de PowerShell comme nom personnel.

### Questions de révision

1. Que signifie la politique `automatic` d'un service ?
   Réponse : le service démarre et s'arrête selon l'ouverture des ports correspondants dans le pare-feu.
2. Changer la politique de `TSM-SSH` sur `off` arrête-t-il SSH ?
   Réponse : non, il empêche seulement le démarrage automatique. Il faut `Stop-VMHostService` pour l'arrêter.
3. Pourquoi plusieurs serveurs NTP ?
   Réponse : pour détecter une source fausse par comparaison et tolérer la panne d'une source.
4. Pourquoi l'écart d'horloge pose-t-il problème avec Active Directory ?
   Réponse : Kerberos rejette les tickets lorsque l'écart dépasse la tolérance, environ cinq minutes.

### Configuration

Déclare deux serveurs NTP publics puis les ajoute à l'hôte par une boucle. Prévoir une sortie Internet ou un serveur NTP interne ; dans HOL, l'accès externe peut être filtré.

```powershell
$esxi = Get-VMHost -Name "esx-03a.vcf.sddc.lab"
$ntpServers = "0.pool.ntp.org", "1.pool.ntp.org"
$ntpServers | ForEach-Object { Add-VMHostNtpServer -VMHost $esxi -NtpServer $_ }
```

<img width="1240" height="312" alt="image" src="https://github.com/user-attachments/assets/d41a5047-b393-41dc-bd9c-f42dbc64f117" />


Récupère le service `ntpd`, le démarre, puis règle sa politique sur `on` pour qu'il redémarre avec l'hôte.

```powershell
$svc = $esxi | Get-VMHostService | Where-Object { $_.Key -eq "ntpd" }
Start-VMHostService -HostService $svc
Set-VMHostService -HostService $svc -Policy "on"
```

<img width="1195" height="511" alt="image" src="https://github.com/user-attachments/assets/78bb2a18-7252-4fd0-bd0a-3bf32d7c1c26" />

Passe la politique du service SSH sur `off` pour qu'il ne démarre plus automatiquement. Cette commande ne l'arrête pas s'il est déjà en cours ; utiliser `Stop-VMHostService` pour cela.

```powershell
Get-VMHost | Get-VMHostService | Where-Object {$_.Key -eq "TSM-SSH"} | Set-VMHostService -Policy "off" -Confirm:$false
```
<img width="1407" height="307" alt="image" src="https://github.com/user-attachments/assets/c4d492cf-8916-48d2-a9c7-2550050619aa" />


Contrôle la liste des serveurs NTP configurés, puis l'état d'exécution (`Running`) et la politique du service `ntpd`.

```powershell
Get-VMHostNtpServer -VMHost $esxi
Get-VMHostService -VMHost $esxi | Where-Object { $_.Key -eq "ntpd" } | Select-Object Key, Running, Policy
```
<img width="1406" height="372" alt="image" src="https://github.com/user-attachments/assets/361fc89b-eba5-4712-8636-6a1302749c20" />


---

## 12. Phase 9 : Esxcli avancé (pare-feu)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer ce qu'est esxcli et pourquoi on l'utilise via PowerCLI, comparer les modes V1 et V2, explorer l'arborescence des commandes, décrire le fonctionnement du pare-feu de l'hôte et restreindre l'accès à un service par adresse source.

### Qu'est-ce qu'esxcli ?

`esxcli` est l'interface de commande native d'ESXi. Elle expose des fonctions de bas niveau que les cmdlets de haut niveau ne couvrent pas toujours. Avec `Get-EsxCli`, on l'utilise depuis PowerCLI par l'API, sans activer SSH. C'est préférable du point de vue de la sécurité.

Les commandes sont organisées en espaces de noms hiérarchiques :

| Espace de noms | Domaine | Exemple de commande |
| :--- | :--- | :--- |
| `system` | Système, version, paramètres | `system.version.get` |
| `hardware` | Matériel | `hardware.cpu.list` |
| `network` | Réseau, pare-feu, interfaces | `network.nic.list`, `network.firewall.get` |
| `storage` | Stockage, périphériques, chemins | `storage.core.device.list` |
| `software` | Paquets VIB, niveau d'acceptation | `software.vib.list` |
| `iscsi` | Configuration iSCSI | `iscsi.adapter.list` |
| `vm` | Processus de VM en cours | `vm.process.list` |

### Mode V1 et mode V2

| Critère | V1 | V2 (`-V2`) |
| :--- | :--- | :--- |
| Appel | Arguments positionnels, difficiles à lire | Arguments nommés via `CreateArgs()` |
| Lisibilité | Faible | Élevée |
| Découverte | Peu pratique | `$esxcli.network.firewall.ruleset.set` affiche la syntaxe |
| Recommandation | Anciens scripts | À utiliser pour tout nouveau script |

Le schéma V2 : récupérer l'objet commande, appeler `CreateArgs()`, renseigner les propriétés, puis `Invoke($args)`. Pour une commande sans argument : `Invoke()`.


### Le pare-feu d'ESXi

Le pare-feu d'ESXi est orienté services. Il repose sur trois notions :

- **Ruleset** : groupe de règles lié à un service (`sshServer`, `ntpClient`, `vSphereClient`).
- **Règle** : définit le sens (entrant ou sortant), le protocole et le port.
- **Liste d'adresses autorisées** : restreint un ruleset à certaines adresses ou réseaux.

La politique générale bloque le trafic qui n'est pas explicitement autorisé par un ruleset activé. Chaque ruleset possède l'indicateur **allowed-all** : s'il est vrai, toutes les adresses sont acceptées et la liste n'a aucun effet.

C'est pourquoi l'ajout d'une IP échoue tant que `allowed-all` est vrai (`Cannot add IP when allowed-all is true`) : il faut d'abord le désactiver, puis ajouter les adresses.

### Attention au verrouillage

Désactiver `allowed-all` et n'autoriser qu'un réseau peut vous couper l'accès si votre poste n'appartient pas à ce réseau. Avant de restreindre le service de gestion, il faut s'assurer que l'adresse de l'administrateur y figure. Pour le lab, le service concerné est SSH, qui n'est pas indispensable à PowerCLI (HTTPS), donc le risque est maîtrisé.

La notation CIDR `10.0.0.0/24` désigne les adresses de `10.0.0.0` à `10.0.0.255`. Le nombre après la barre est le nombre de bits du masque réseau.

### Bonnes pratiques

- Restreindre par adresse source tous les services d'administration.
- Contrôler régulièrement les rulesets actifs.
- Privilégier esxcli via l'API plutôt qu'un shell SSH.
- Utiliser le mode V2 pour la lisibilité.

### Points clés à retenir

- esxcli complète PowerCLI pour les fonctions avancées.
- V2 utilise `CreateArgs()` et `Invoke()`.
- `allowed-all` doit être désactivé avant d'ajouter des IP.
- Une restriction mal préparée peut couper l'accès.

### Questions de révision

1. Pourquoi préférer `Get-EsxCli` à une connexion SSH ?
   Réponse : l'appel passe par l'API sécurisée, sans activer un service d'accès supplémentaire.
2. Que représente `10.0.0.0/24` ?
   Réponse : un réseau de 256 adresses, de `10.0.0.0` à `10.0.0.255`.
3. Quelle séquence permet de restreindre SSH à un réseau ?
   Réponse : désactiver `allowedall` sur `sshServer`, puis ajouter l'adresse avec `allowedip.add`.
4. Comment connaître les arguments d'une commande en V2 ?
   Réponse : afficher l'objet de la commande ou appeler `CreateArgs()` et inspecter ses propriétés.

### Configuration

Obtient l'accès à l'espace de noms `esxcli` de l'hôte en mode V2.

```powershell
$esxcli = Get-EsxCli -VMHost "esx-03a.vcf.sddc.lab" -V2
```

<img width="990" height="155" alt="image" src="https://github.com/user-attachments/assets/fc7f9b05-4f8f-465a-aab1-ba7d09f2af86" />


Prépare les arguments, cible le ruleset `sshServer` et désactive `allowed-all`. Le service n'est plus ouvert à tous, il devient filtré par liste.


```powershell
$argsSet = $esxcli.network.firewall.ruleset.set.CreateArgs()
$argsSet.rulesetid = "sshServer"
$argsSet.allowedall = $false
$esxcli.network.firewall.ruleset.set.Invoke($argsSet)
```

<img width="995" height="256" alt="image" src="https://github.com/user-attachments/assets/f86f344d-f310-4c98-8c99-b67c26fcce0e" />


Ajoute le réseau `10.0.0.0/24` à la liste des adresses autorisées pour SSH. Seules les machines de ce sous réseau pourront s'y connecter. Pour contrôler le résultat :

```powershell
$argsAdd = $esxcli.network.firewall.ruleset.allowedip.add.CreateArgs()
$argsAdd.ipaddress = "10.0.0.0/24"
$argsAdd.rulesetid = "sshServer"
$esxcli.network.firewall.ruleset.allowedip.add.Invoke($argsAdd)
```

<img width="1257" height="247" alt="image" src="https://github.com/user-attachments/assets/bef75e64-31ea-49e9-93d0-9a6b72653be9" />


Affiche les adresses autorisées par ruleset.

```powershell
$esxcli.network.firewall.ruleset.allowedip.list.Invoke()
```
<img width="1407" height="647" alt="image" src="https://github.com/user-attachments/assets/99c3dec1-f090-49f9-8ebd-b9559a7586a0" />


---

## 13. Phase 10 : Gestion des ressources (shares, réservations, limites, resource pool)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer comment l'hyperviseur partage le CPU et la mémoire, interpréter shares, réservations et limites, calculer l'effet d'une contention, utiliser les resource pools à bon escient et repérer les indicateurs de saturation.

### L'ordonnancement du CPU

Chaque vCPU est un processus que le VMkernel planifie sur un cœur physique. Quand plus de vCPU veulent s'exécuter que de cœurs disponibles, certains attendent. Le temps d'attente s'appelle le **CPU Ready**. C'est le principal indicateur de saturation du processeur.

Règles de lecture :

- Un taux de CPU Ready qui dépasse environ 5 pour cent par vCPU mérite une investigation.
- Il peut être élevé alors que l'utilisation CPU de l'hôte paraît faible : la cause est souvent une VM avec trop de vCPU.
- Le surdimensionnement des VM est la première cause de dégradation, d'où la règle de commencer petit et d'augmenter selon la mesure.

Le rapport entre vCPU alloués et cœurs physiques s'appelle le taux de sur-allocation (overcommit). Des ratios modestes sont acceptables pour des charges peu actives, mais des charges intensives exigent un ratio proche de un pour un.

### Trois leviers

| Levier | Définition | Agit quand | Effet secondaire |
| :--- | :--- | :--- | :--- |
| Shares | Poids relatif | Uniquement en contention | Aucun en l'absence de contention |
| Réservation | Quantité garantie | Toujours | Réduit la capacité disponible pour les autres, peut empêcher le démarrage d'une VM (admission control) |
| Limite | Plafond | Toujours | Peut dégrader fortement la VM même si l'hôte est libre |

**Exemple de calcul.** Deux VM de même taille, A en shares High et B en shares Normal, se disputent un CPU saturé. Le rapport High/Normal est de 2 pour 1, donc A obtient environ deux tiers du temps CPU contesté et B un tiers. Sans contention, chacune obtient ce qu'elle demande.

À retenir : une limite est rarement une bonne idée. Elle s'applique même si l'hôte est inoccupé, et une limite mémoire inférieure à la mémoire configurée force le ballooning ou le swap.

### La mémoire sous contention

L'hôte raisonne en états de pression mémoire (haute, douce, forte, basse) et déclenche progressivement ballooning, compression puis swap. Les compteurs à surveiller sont :

| Compteur | Signification | Alerte si |
| :--- | :--- | :--- |
| Consumed | Mémoire physique allouée à la VM | Information, pas d'alerte |
| Active | Mémoire réellement touchée récemment | Sert à dimensionner |
| Ballooned | Mémoire récupérée par le ballon | Différent de zéro |
| Swapped | Mémoire envoyée sur disque par l'hôte | Différent de zéro |

La mémoire consommée élevée n'est pas un problème en soi, car ESXi garde volontiers la mémoire en cache. Ce sont le ballooning et surtout le swap qui signalent un vrai manque.

### Les resource pools

Un resource pool regroupe des VM et leur applique des paramètres communs. Les pools forment une hiérarchie, la racine étant l'hôte (ou le cluster).

```text
Hôte
 +-- RP-Production (shares High)
 |      +-- VM-A
 |      +-- VM-B
 +-- RP-Test (shares Low)
        +-- VM-C
```

Points d'attention :

- Les shares d'un pool ne s'appliquent qu'entre éléments de même niveau. Une VM placée à la racine peut avoir plus de poids qu'un pool entier, si ce dernier contient un grand nombre de VM.
- Un pool n'est pas un dossier d'organisation. Pour classer des VM, utiliser des dossiers ou des étiquettes (sous vCenter).
- Une réservation de pool est garantie à l'ensemble du pool. Une réservation extensible lui permet d'emprunter à son parent.

### Méthode de dimensionnement

1. Mesurer sur une période représentative.
2. Dimensionner d'après l'utilisation réelle (CPU actif, mémoire active) avec une marge.
3. Ajuster peu à peu, en comparant avec les indicateurs de saturation.
4. Documenter les réservations posées et leur justification.

### Points clés à retenir

- CPU Ready est l'indicateur central de saturation du processeur.
- Shares ne comptent que sous contention, limites et réservations toujours.
- Ballooning et swap signalent un manque réel de mémoire.
- Un resource pool n'est pas un dossier.

### Questions de révision

1. Quand les shares ont-elles un effet ?
   Réponse : uniquement quand les ressources sont saturées.
2. Pourquoi une limite peut-elle ralentir une VM sur un hôte inoccupé ?
   Réponse : la limite plafonne la consommation en permanence, quelle que soit la disponibilité de l'hôte.
3. Quel compteur indique que des vCPU attendent un cœur ?
   Réponse : CPU Ready.
4. Quelle est la différence entre mémoire consommée et active ?
   Réponse : la consommée est allouée, l'active est réellement utilisée récemment.

### Configuration

Stocke la VM dans une variable puis affiche sa configuration de ressources actuelle : niveaux de shares, réservations et limites CPU et mémoire. Une limite à `-1` signifie illimité.

```powershell
$vm = Get-VM -Name 'VM-Network-Test'
Get-VMResourceConfiguration -VM $vm | Select-Object VM, CpuSharesLevel, NumCpuShares, CpuReservationMhz, CpuLimitMhz, MemSharesLevel, MemReservationMB, MemLimitMB
```

<img width="1405" height="462" alt="image" src="https://github.com/user-attachments/assets/36db2d6a-66a1-4ca9-8290-09af034d2642" />

Passe les shares CPU de la VM au niveau High et lui réserve 512 Mo de mémoire. La VM est désormais prioritaire en cas de contention CPU et dispose d'une mémoire garantie.


```powershell
Get-VMResourceConfiguration -VM $vm | Set-VMResourceConfiguration -CpuSharesLevel High -MemReservationMB 512
```
<img width="1407" height="322" alt="image" src="https://github.com/user-attachments/assets/7ea84e9c-df6a-4794-8c87-b58fe1492d8a" />


Crée un resource pool nommé `RP-Production` directement sous l'hôte, avec des shares CPU et mémoire élevés.


```powershell
$rp = New-ResourcePool -Location $vmhost -Name 'RP-Production' -CpuSharesLevel High -MemSharesLevel High
```

<img width="1407" height="197" alt="image" src="https://github.com/user-attachments/assets/eff152cd-0f39-4362-bdf9-4c1b0c856150" />

Déplace la VM dans le pool puis liste les VM qu'il contient. **Point d'attention** : si l'hôte refuse l'opération via cette cmdlet, utiliser l'objet API : `$rp.ExtensionData.MoveIntoResourcePool(@($vm.ExtensionData.MoRef))`, ou recréer la VM avec le paramètre `-ResourcePool $rp`.

```powershell
Move-VM -VM $vm -Destination $rp
Get-VM -Location $rp | Select-Object Name, PowerState
```

<img width="1396" height="377" alt="image" src="https://github.com/user-attachments/assets/c52f19c0-7f69-4e3f-8288-4afe52b7cf71" />


---

## 14. Phase 11 : Évolution du matériel et opérations sur les VM

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait modifier le dimensionnement d'une VM en sécurité, comprendre l'intérêt du hot add et ses effets sur NUMA, étendre un disque jusqu'au système de fichiers invité, expliquer le rôle de VMware Tools et distinguer les opérations d'alimentation.

### Modifier le matériel d'une VM

| Ressource | À chaud | À froid | Remarque |
| :--- | :--- | :--- | :--- |
| vCPU | Si hot add activé | Toujours | Le hot add exige la prise en charge par le système invité |
| Mémoire | Si hot add activé | Toujours | Idem |
| Disque : ajout | Oui | Oui | Contrôleur SCSI requis |
| Disque : extension | Oui (selon les cas) | Oui | Impossible avec un snapshot |
| Disque : réduction | Non | Non | Non pris en charge |
| Carte réseau | Oui | Oui | Selon le type d'adaptateur |

Le hot add de vCPU désactive le vNUMA pour cette VM : l'hyperviseur n'expose plus la topologie NUMA à l'invité. Sur des VM de grande taille, cela peut dégrader les performances. C'est un compromis à connaître avant d'activer cette option.

### NUMA en bref

Sur un serveur à plusieurs sockets, chaque processeur accède plus vite à sa mémoire locale qu'à celle d'un autre socket. Une VM dont le nombre de vCPU et la mémoire tiennent dans un seul nœud NUMA a de meilleures performances. D'où l'intérêt de ne pas surdimensionner une VM.

### Etendre un disque, étape complète

L'extension se fait en trois niveaux :

1. **Disque virtuel** : `Set-HardDisk -CapacityGB` agrandit le VMDK.
2. **Partition** : le système invité doit agrandir sa partition (Linux : `growpart` ; Windows : Gestion des disques ou `diskpart`).
3. **Système de fichiers** : agrandir le système de fichiers (Linux : `resize2fs` pour ext4, `xfs_growfs` pour XFS ; Windows : l'extension du volume).

La première étape seule ne change rien pour l'utilisateur du système invité. Sur Linux avec LVM, on étend aussi le volume physique, puis le volume logique.

Prérequis : aucun snapshot sur la VM, sinon l'opération échoue.

### VMware Tools

VMware Tools est un ensemble de pilotes et de services installé dans le système invité :

- pilotes paravirtualisés (VMXNET3, PVSCSI) et pilote de ballooning,
- arrêt et redémarrage propres depuis l'hyperviseur,
- synchronisation de l'heure (optionnelle),
- remontée d'informations (adresse IP, nom d'hôte),
- mise au repos (quiesce) pour des snapshots cohérents.

Sous Linux, on installe de préférence le paquet de la distribution `open-vm-tools`. Photon OS l'embarque. L'état se lit dans `ExtensionData.Guest.ToolsStatus`, avec les valeurs `toolsOk`, `toolsOld`, `toolsNotRunning` et `toolsNotInstalled`.

### Les opérations d'alimentation

| Cmdlet | Effet | Équivalent physique | Tools requis |
| :--- | :--- | :--- | :--- |
| `Start-VM` | Démarre la VM | Bouton marche | Non |
| `Stop-VM` | Coupe l'alimentation | Débrancher le câble | Non |
| `Stop-VMGuest` | Arrêt propre de l'invité | Arrêt depuis le système | Oui |
| `Restart-VM` | Réinitialisation matérielle | Bouton reset | Non |
| `Restart-VMGuest` | Redémarrage propre | Redémarrer depuis le système | Oui |
| `Suspend-VM` | Enregistre l'état puis suspend | Veille prolongée | Non |

La mise hors tension brutale risque de corrompre des systèmes de fichiers. Préférer `Stop-VMGuest` dès que VMware Tools est présent.

### Points clés à retenir

- Le hot add est pratique mais désactive vNUMA.
- Étendre un disque demande trois étapes, dont deux dans le système invité.
- Une réduction de disque n'est pas supportée.
- `Stop-VM` coupe brutalement, `Stop-VMGuest` arrête proprement.

### Questions de révision

1. Pourquoi `Set-HardDisk` échoue-t-il parfois ?
   Réponse : la VM possède un snapshot.
2. Que fait l'agrandissement du VMDK pour l'utilisateur de la VM ?
   Réponse : rien tant que partition et système de fichiers n'ont pas été étendus dans l'invité.
3. Quel est l'inconvénient du hot add de vCPU ?
   Réponse : il désactive le vNUMA de la VM.
4. Quelle cmdlet arrête proprement une VM ?
   Réponse : `Stop-VMGuest`, si VMware Tools est installé.

### Configuration

Affiche l'état de la VM, son dimensionnement et l'espace disque provisionné et réellement utilisé. Cette vue sert de référence avant toute modification.

```powershell
Get-VM -Name 'VM-Network-Test' | Select-Object Name, PowerState, NumCpu, MemoryGB, ProvisionedSpaceGB, UsedSpaceGB
```

<img width="1406" height="415" alt="image" src="https://github.com/user-attachments/assets/759cac07-d7ca-4e92-b754-dc8516ae3eaa" />

Éteint la VM, passe à 2 vCPU et 2 Go de RAM et renseigne le champ de notes. L'arrêt préalable évite de dépendre du hot add.

```powershell
Stop-VM -VM 'VM-Network-Test' -Confirm:$false
Set-VM -VM 'VM-Network-Test' -NumCpu 2 -MemoryGB 2 -Notes 'VM de test du lab PowerCLI' -Confirm:$false
```

<img width="1380" height="391" alt="image" src="https://github.com/user-attachments/assets/31683ecc-ee92-4257-9611-8052807ccc75" />
<img width="1412" height="565" alt="image" src="https://github.com/user-attachments/assets/cb8bd273-8e1a-4089-ba7e-e5d6d91960b5" />


Étend le premier disque à 15 Go. **Point d'attention** : l'opération échoue si la VM possède un snapshot. Supprimer les snapshots avant d'étendre un disque.

```powershell
Get-HardDisk -VM 'VM-Network-Test' | Select-Object -First 1 | Set-HardDisk -CapacityGB 15 -Confirm:$false
```

<img width="1421" height="372" alt="image" src="https://github.com/user-attachments/assets/5083fb4d-ea6b-4185-b18a-d5f84167513d" />
<img width="1415" height="570" alt="image" src="https://github.com/user-attachments/assets/a2479ea0-8dc4-4f01-92d6-fa748682fec7" />


Ajoute un second disque de 5 Go en thin sur `ds-iscsi02` puis liste tous les disques de la VM avec leur format et leur emplacement.

```powershell
New-HardDisk -VM 'VM-Network-Test' -CapacityGB 5 -StorageFormat Thin -Datastore (Get-Datastore -Name 'ds-iscsi02') -Confirm:$false
Get-HardDisk -VM 'VM-Network-Test' | Select-Object Name, CapacityGB, StorageFormat, Filename
```

<img width="1406" height="537" alt="image" src="https://github.com/user-attachments/assets/94e0b88b-f8c4-4934-bf3e-f29e73c3086b" />
<img width="1421" height="565" alt="image" src="https://github.com/user-attachments/assets/5822a393-acb1-438f-be83-0fa8826d9ba0" />


Lit l'état de VMware Tools via l'objet API de la VM. La valeur `toolsNotInstalled` indique que les arrêts propres ne sont pas disponibles, ce qui est le cas d'une ISO Photon OS qui n'a pas été installée.

```powershell
Get-VM -Name 'VM-Network-Test' | Select-Object Name, @{N='ToolsStatus';E={$_.ExtensionData.Guest.ToolsStatus}}
```
<img width="1402" height="332" alt="image" src="https://github.com/user-attachments/assets/8699d644-8fa1-4224-9f81-93c314a20f90" />


Redémarre la VM avec son nouveau matériel.

```powershell
Start-VM -VM 'VM-Network-Test' -Confirm:$false
```

<img width="981" height="247" alt="image" src="https://github.com/user-attachments/assets/b2c81f21-fce4-4d52-8189-b2e3c4867718" />


---

## 15. Phase 12 : Réseau avancé (teaming, sécurité du vSwitch, audit)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait comparer les politiques de répartition de charge, choisir la détection de panne adaptée, définir un ordre de basculement par port group, comprendre l'impact du teaming sur le stockage iSCSI, et dépanner une connectivité réseau avec les outils de l'hôte.

### Le NIC teaming en profondeur

Le teaming associe plusieurs uplinks pour obtenir de la redondance et, selon la politique, une répartition de charge. Il est important de comprendre que la répartition se fait **par flux ou par port**, jamais paquet par paquet : une VM donnée n'utilise qu'un uplink à un instant donné, sauf avec le hachage IP et plusieurs destinations.

| Politique | Critère de choix de l'uplink | Exigence côté commutateur physique | Usage |
| :--- | :--- | :--- | :--- |
| Route based on originating virtual port (`LoadBalanceSrcId`) | Le port virtuel de la VM | Aucune | Choix par défaut, simple |
| Route based on source MAC hash (`LoadBalanceSrcMac`) | Adresse MAC de la VM | Aucune | Alternative proche de la précédente |
| Route based on IP hash (`LoadBalanceIP`) | Adresses IP source et destination | Agrégation statique de liens (EtherChannel) obligatoire | Une VM qui parle à de nombreuses destinations |
| Explicit failover order (`ExplicitFailover`) | Premier uplink actif disponible | Aucune | Redondance sans répartition |

Avec le hachage IP, une configuration incohérente côté physique provoque des pertes de paquets difficiles à diagnostiquer. En cas de doute, rester sur le port d'origine.

### Détection de panne et basculement

- **Link status** : l'hôte détecte uniquement les pannes du lien direct (câble débranché, port éteint). C'est le mode standard.
- **Beacon probing** : l'hôte envoie des sondes sur tous les uplinks pour détecter les pannes situées plus loin. Il demande au moins trois uplinks pour être fiable, car avec deux on ne peut pas savoir lequel est défaillant.
- **Failback** : si activé, le trafic revient sur l'uplink d'origine dès qu'il est rétabli. Peut provoquer des allers et retours en cas de lien instable.
- **Notify switches** : après un basculement, l'hôte envoie une trame pour que le commutateur mette à jour sa table MAC et redirige immédiatement le trafic. À laisser activé, sauf cas particulier (certains clusters à équilibrage de charge réseau en monodiffusion).
- **Ordre de basculement** : un uplink est Active (utilisé), Standby (utilisé si un actif tombe) ou Unused (jamais).

### Politique du vSwitch et politique du port group

La politique de teaming est définie sur le vSwitch, puis chaque port group peut la remplacer (override). Cela permet, avec deux uplinks, d'envoyer le VLAN 10 de préférence sur `vmnic1` et le VLAN 20 de préférence sur `vmnic3`, en utilisant chacun comme secours de l'autre. C'est une répartition manuelle efficace sans exigence côté commutateur.

### Cas particulier iSCSI

Quand on utilise l'association de ports iSCSI (iSCSI port binding), chaque interface VMkernel dédiée doit avoir **un seul uplink actif** et aucun uplink en secours. Le multipathing iSCSI assure la redondance à la place du teaming. Mélanger les deux est une erreur classique.

### Séparer les flux

| Flux | Raison de l'isolation |
| :--- | :--- |
| Gestion | Sécurité de l'administration |
| vMotion | Trafic volumineux, contenu mémoire non chiffré par défaut |
| Stockage IP | Latence et bande passante |
| VM | Selon les applications |

Sur un hôte autonome comme celui du lab, on ne peut pas faire de vMotion, mais la séparation reste le bon réflexe de conception.

### CDP et LLDP

Ces protocoles de découverte permettent de voir le commutateur physique voisin d'une carte (nom, port, VLAN). Un vSwitch standard prend en charge CDP. LLDP exige un commutateur distribué. L'information est très utile pour vérifier un câblage.

### Diagnostic réseau

| Besoin | Outil | Exemple |
| :--- | :--- | :--- |
| État des cartes physiques | esxcli | `$esxcli.network.nic.list.Invoke()` |
| Configuration IP des interfaces | esxcli | `$esxcli.network.ip.interface.list.Invoke()` |
| Test de joignabilité depuis une interface VMkernel | `vmkping` (SSH ou console) | `vmkping -I vmk1 10.0.10.1` |
| Test de trames jumbo | `vmkping` avec taille et non-fragmentation | `vmkping -I vmk1 -s 8972 -d 10.0.10.1` |
| Voisin physique | Vue CDP du vSwitch | Host Client |

Le test avec `-s 8972 -d` envoie un paquet de 8972 octets sans fragmentation : s'il échoue alors qu'un ping normal réussit, la MTU n'est pas cohérente sur le chemin.

### Points clés à retenir

- La répartition de charge se fait par flux, pas par paquet.
- IP hash exige une agrégation statique côté physique.
- Beacon probing nécessite au moins trois uplinks.
- Avec iSCSI port binding, un seul uplink actif par interface.

### Questions de révision

1. Quelle politique ne demande aucune configuration côté commutateur physique ?
   Réponse : la répartition par port d'origine (`LoadBalanceSrcId`), ainsi que la MAC source et l'ordre de basculement explicite.
2. Pourquoi le beacon probing demande-t-il trois uplinks ?
   Réponse : avec deux, on ne peut pas déterminer lequel est défaillant, car les sondes de chacun échouent.
3. À quoi sert Notify switches ?
   Réponse : à mettre à jour la table MAC du commutateur physique après un basculement.
4. Comment vérifier qu'une MTU de 9000 est cohérente ?
   Réponse : avec `vmkping -I vmkX -s 8972 -d adresse`.

### Configuration


Lit la politique de teaming du vSwitch puis affiche la répartition de charge, la détection de panne, les options de notification et de failback, ainsi que les cartes actives et en secours.

```powershell
$vs = Get-VirtualSwitch -VMHost $vmhost -Name 'vSwitch1'
$teaming = Get-NicTeamingPolicy -VirtualSwitch $vs
$teaming | Select-Object VirtualSwitch, LoadBalancingPolicy, NetworkFailoverDetectionPolicy, NotifySwitches, FailbackEnabled, ActiveNic, StandbyNic
```

<img width="1397" height="490" alt="image" src="https://github.com/user-attachments/assets/50deda6e-8686-421d-aeba-0a5e144c9784" />


Applique une politique standard et prévisible : répartition par port virtuel, détection par état du lien, notification des commutateurs et failback activés.

```powershell
Set-NicTeamingPolicy -VirtualSwitchPolicy $teaming -LoadBalancingPolicy LoadBalanceSrcId -NetworkFailoverDetectionPolicy LinkStatus -NotifySwitches $true -FailbackEnabled $true
```

<img width="1407" height="360" alt="image" src="https://github.com/user-attachments/assets/00a1f975-4262-4e4f-960e-bd9a1012de90" />

Définit un ordre de basculement propre au port group `VLAN20-Clients` : `vmnic1` est active et `vmnic3` en secours. `-InheritFailoverOrder $false` désactive l'héritage de la politique du vSwitch. **Point d'attention** : les deux cartes doivent déjà être des uplinks du vSwitch.

```powershell
$pg20 = Get-VirtualPortGroup -VMHost $vmhost -Name 'VLAN20-Clients'
$nic1 = Get-VMHostNetworkAdapter -VMHost $vmhost -Physical -Name 'vmnic1'
$nic3 = Get-VMHostNetworkAdapter -VMHost $vmhost -Physical -Name 'vmnic3'
Get-NicTeamingPolicy -VirtualPortGroup $pg20 | Set-NicTeamingPolicy -InheritFailoverOrder $false -MakeNicActive $nic1 -MakeNicStandby $nic3
```

<img width="1402" height="455" alt="image" src="https://github.com/user-attachments/assets/4ab17776-06b9-462b-89d4-21f6421a6299" />


Applique le durcissement de sécurité au niveau du vSwitch entier. Les port groups qui n'ont pas de politique propre en héritent.


```powershell
Get-SecurityPolicy -VirtualSwitch $vs | Set-SecurityPolicy -AllowPromiscuous $false -ForgedTransmits $false -MacChanges $false -Confirm:$false
```

<img width="1411" height="367" alt="image" src="https://github.com/user-attachments/assets/4612d19b-f3c1-47ad-9785-91fc1450aed8" />
<img width="1405" height="572" alt="image" src="https://github.com/user-attachments/assets/42b7d52d-595e-4f91-86c5-c69ecace078d" />
<img width="1415" height="570" alt="image" src="https://github.com/user-attachments/assets/11947cf5-0b42-4933-a777-f8c3de8d9134" />

Audite les interfaces VMkernel : adresse, masque, port group et services activés (gestion, vMotion). Permet de détecter une interface qui porte un service qu'elle ne devrait pas porter.

```powershell
Get-VMHostNetworkAdapter -VMHost $vmhost -VMKernel | Select-Object Name, IP, SubnetMask, PortGroupName, ManagementTrafficEnabled, VMotionEnabled
```

<img width="1401" height="602" alt="image" src="https://github.com/user-attachments/assets/3db8d7a8-8134-4546-9aa2-e013adf51702" />
<img width="1335" height="372" alt="image" src="https://github.com/user-attachments/assets/45b19202-c2e2-4aca-a1c2-95ee5d306f9d" />


Affiche la configuration réseau globale de l'hôte : nom, domaine, serveurs DNS, domaine de recherche et passerelle VMkernel par défaut. Cette commande est en lecture seule, car modifier le DNS ou la passerelle dans HOL pourrait couper l'accès à l'hôte.


```powershell
Get-VMHostNetwork -VMHost $vmhost | Select-Object HostName, DomainName, DnsAddress, SearchDomain, VMKernelGateway
```

<img width="1402" height="442" alt="image" src="https://github.com/user-attachments/assets/8e9d1633-a671-4bcc-8f22-d29919872b61" />



---

## 16. Phase 13 : Stockage avancé (iSCSI, multipathing, occupation)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait décrire une session iSCSI, comparer découverte dynamique et statique, expliquer le multipathing et ses politiques, interpréter les indicateurs de latence de stockage et connaître les situations APD et PDL.

### La configuration iSCSI de bout en bout

Les étapes pour connecter un hôte à une baie iSCSI :

1. Activer l'adaptateur iSCSI logiciel de l'hôte.
2. Préparer le réseau : interface VMkernel dédiée, VLAN et MTU adaptés.
3. Associer éventuellement les interfaces VMkernel à l'adaptateur (port binding).
4. Déclarer la cible : découverte dynamique (Send Targets) ou cible statique.
5. Configurer l'authentification CHAP si la baie l'exige.
6. Relancer un rescan, puis créer le datastore sur le LUN.

La découverte dynamique interroge le portail et récupère toutes les cibles : pratique et robuste aux changements. Les cibles statiques sont déclarées une par une avec leur IQN : plus de contrôle, plus de maintenance.

Le CHAP authentifie l'initiateur auprès de la cible (unidirectionnel) ou les deux parties l'une envers l'autre (mutuel). Le secret est partagé : il se protège comme un mot de passe.

### L'architecture de gestion des chemins

```text
        VM  ->  Datastore VMFS  ->  Périphérique (LUN)
                                        |
                          +-------------+-------------+
                          |                           |
                    Chemin A (vmhba64 -> cible)  Chemin B (vmhba65 -> cible)
```

- **PSA (Pluggable Storage Architecture)** : cadre qui gère l'accès aux périphériques.
- **NMP (Native Multipathing Plugin)** : module natif de VMware.
- **SATP (Storage Array Type Plugin)** : connaît le comportement de la baie (actif-actif, actif-passif, ALUA) et gère les basculements.
- **PSP (Path Selection Policy)** : choisit le chemin à utiliser pour chaque E/S.

| PSP | Comportement |
| :--- | :--- |
| `Fixed` | Utilise un chemin préféré, bascule en cas de panne, puis revient |
| `MostRecentlyUsed` | Utilise le dernier chemin qui a fonctionné, sans retour automatique |
| `RoundRobin` | Alterne entre tous les chemins actifs (par défaut toutes les 1000 E/S) |

La bonne politique dépend de la baie. On suit la recommandation du constructeur, souvent `RoundRobin` pour les baies actif-actif.

### Etats des chemins et situations de panne

- **Active** : chemin utilisable pour les E/S. **Standby** : disponible mais non utilisé. **Dead** : chemin perdu.
- **APD (All Paths Down)** : tous les chemins vers un périphérique sont perdus, mais on ne sait pas si c'est définitif. L'hôte réessaie un temps, puis arrête les E/S : les VM se figent.
- **PDL (Permanent Device Loss)** : la baie signale que le périphérique est définitivement perdu. L'hôte cesse immédiatement de l'utiliser.

Ces deux situations expliquent pourquoi un incident de stockage peut se manifester par des VM qui ne répondent plus sans panne visible de l'hôte.

### Mesurer les performances de stockage

La latence se décompose en couches :

| Indicateur | Où | Signification | Seuil indicatif d'investigation |
| :--- | :--- | :--- | :--- |
| DAVG | Périphérique et baie | Latence vue de la baie et du réseau | Au delà de 20 à 25 ms |
| KAVG | Noyau de l'hôte | Temps passé en file d'attente dans le VMkernel | Au delà de 2 ms |
| GAVG | Invité | Somme DAVG et KAVG, ce que ressent la VM | Au delà de 25 ms |

Un KAVG élevé indique une file d'attente saturée côté hôte, un DAVG élevé pointe la baie ou le réseau. Ces valeurs sont des repères, à confirmer par le contexte (type de charge, support).

### Accélérations matérielles (VAAI)

L'API VAAI permet à l'hôte de déléguer à la baie des opérations coûteuses : verrouillage atomique (ATS), copie de blocs (clonage), mise à zéro de blocs et libération d'espace (UNMAP). Elle réduit la charge sur l'hôte et le réseau.

### Contrôle des E/S et agrandissement

- **Storage I/O Control** : équilibre l'accès au stockage entre VM d'un datastore partagé selon des shares. Il exige vCenter et une licence adaptée.
- **Agrandir un datastore** : soit on agrandit le LUN sur la baie puis on étend le datastore dans l'espace libre du LUN, soit on ajoute un LUN supplémentaire comme extension. La première méthode est préférable, car l'extension multiplie les points de défaillance.

### Points clés à retenir

- La découverte dynamique récupère les cibles par le portail.
- PSP choisit le chemin, SATP gère la baie, NMP coordonne.
- APD est une perte de chemins d'issue incertaine, PDL une perte permanente.
- DAVG, KAVG et GAVG localisent la source d'une latence.

### Questions de révision

1. Quelle différence entre APD et PDL ?
   Réponse : APD, tous les chemins sont perdus avec une issue incertaine, PDL, la perte du périphérique est signalée comme définitive.
2. Que signifie un KAVG élevé ?
   Réponse : les commandes attendent longtemps en file dans le noyau de l'hôte.
3. Quel est le rôle du PSP ?
   Réponse : choisir le chemin utilisé pour chaque opération d'E/S.
4. Pourquoi préférer agrandir un LUN plutôt qu'ajouter une extension ?
   Réponse : l'extension multiplie les points de défaillance et complique la gestion.

### Configuration

Sélectionne l'adaptateur iSCSI logiciel de l'hôte puis affiche son nom de périphérique, son modèle, son IQN et son état.

```powershell
$iscsiHba = Get-VMHostHba -VMHost $vmhost -Type iScsi
$iscsiHba | Select-Object Device, Model, IScsiName, Status
```

<img width="1295" height="332" alt="image" src="https://github.com/user-attachments/assets/06093d0e-8e2c-45dc-863e-2359b41babf1" />

Liste les cibles iSCSI configurées, avec leur adresse, leur port et leur type (Send ou Static). Cette commande sert à documenter la connectivité vers FreeNAS.


```powershell
Get-IScsiHbaTarget -IScsiHba $iscsiHba | Select-Object Address, Port, Type
```

<img width="1166" height="357" alt="image" src="https://github.com/user-attachments/assets/f6bc51ba-80d7-4864-8b6f-cd29258f849b" />


Créer la NOUVELLE cible (avec une adresse qui n'existe pas encore)
```powershell
New-IScsiHbaTarget -IScsiHba $iscsiHba -Address '10.0.0.51' -Type Send
```

Forcer l'ESXi à aller chercher les nouveaux LUNs sur cette nouvelle cible

```powershell
Get-VMHostStorage -VMHost (Get-VMHost) -RescanAllHba -RescanVmfs
```

Vérifier que la nouvelle cible est bien enregistrée
```powershell
Get-IScsiHbaTarget -IScsiHba $iscsiHba | Select-Object Address, Port, Type
```

<img width="1382" height="577" alt="image" src="https://github.com/user-attachments/assets/6ce8d435-8603-441c-ac25-42bdcde1fd58" />

Affiche la politique de chemins de chaque LUN puis tous les chemins vers ces LUN avec leur état. Dans un lab, un LUN n'a souvent qu'un seul chemin.

```powershell
Get-ScsiLun -VMHost $vmhost -LunType disk | Select-Object CanonicalName, CapacityGB, MultipathPolicy
Get-ScsiLun -VMHost $vmhost -LunType disk | Get-ScsiLunPath | Select-Object Name, State, Preferred
```

<img width="1396" height="517" alt="image" src="https://github.com/user-attachments/assets/aafd713a-57de-468c-b52c-8ef07e45a8e1" />

Passe le LUN de 85 Go (celui de `ds-iscsi01`) en `RoundRobin`. Avec un seul chemin, le gain est nul, mais la commande illustre la méthode. **Point d'attention** : en production, suivre la politique recommandée par le constructeur de la baie.

```powershell
Get-ScsiLun -VMHost $vmhost -LunType disk | Where-Object { $_.CapacityGB -eq 85 } | Set-ScsiLun -MultipathPolicy RoundRobin
```
<img width="1406" height="347" alt="image" src="https://github.com/user-attachments/assets/67839977-b9f1-405c-9441-5b12c6925b6f" />
<img width="1422" height="562" alt="image" src="https://github.com/user-attachments/assets/9d13dec2-49bd-4d75-afce-5332ff1ed55f" />
<img width="1422" height="580" alt="image" src="https://github.com/user-attachments/assets/a448e13f-92d0-4009-88dc-b072d88f076d" />


Compte les VM présentes sur chaque datastore. Cela donne une vue rapide de la répartition de la charge de stockage.

```powershell
foreach ($d in Get-Datastore) {
    "{0} : {1} VM" -f $d.Name, @(Get-VM -Datastore $d).Count
}
```
<img width="1107" height="252" alt="image" src="https://github.com/user-attachments/assets/599b5b03-d3cc-416d-bf70-adc2a0e1e580" />


---

## 17. Phase 14 : Gestion des accès (comptes, rôles, permissions)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait expliquer le modèle d'autorisation d'ESXi, distinguer privilège, rôle et permission, appliquer le principe du moindre privilège, comprendre les règles de précédence et auditer les droits en place.

### Cours : le modèle d'autorisation

L'autorisation répond à la question « qui a le droit de faire quoi sur quel objet ? ». Elle repose sur quatre éléments :

```text
   Principal (qui)  +  Rôle (quoi)  +  Objet (où)  +  Propagation (jusqu'où)
        opsvm            Operateur-VM     VM-Network-Test       Non
```

- **Principal** : un utilisateur ou un groupe.
- **Privilège** : une action unitaire (`VirtualMachine.Interact.PowerOn`). On en compte plusieurs centaines.
- **Rôle** : un ensemble nommé de privilèges.
- **Permission** : association principal, rôle et objet, avec option de propagation aux objets enfants.

### Les rôles

| Rôle | Contenu |
| :--- | :--- |
| `Admin` (Administrator) | Tous les privilèges |
| `ReadOnly` | Lecture de l'inventaire, aucune modification |
| `NoAccess` | Aucun accès. Sert à retirer un droit hérité |
| Rôles personnalisés | Constitués sur mesure, comme `Operateur-VM` |

Il existe aussi des rôles système (`Anonymous`, `View`, `Read`) utilisés en interne. On ne les modifie pas.

Un rôle personnalisé se construit en partant d'un besoin métier (par exemple, un opérateur qui allume et éteint des VM) puis en ajoutant uniquement les privilèges nécessaires. Il est plus sûr de partir de rien que de retirer des droits à un rôle large.

### Propagation et précédence

- Avec **Propagate**, la permission s'applique à tous les objets enfants. Sans, elle ne concerne que l'objet ciblé.
- Une permission posée directement sur un objet l'emporte sur une permission héritée du parent.
- Si un utilisateur reçoit plusieurs permissions via ses groupes sur un même objet, ses droits sont l'union de ces permissions, sauf si l'une est `NoAccess`, qui l'emporte.
- Une permission explicite attribuée à l'utilisateur lui-même l'emporte sur celles de ses groupes.

### Comptes locaux d'un hôte autonome

- Il n'y a pas d'annuaire centralisé : chaque hôte a ses comptes. La gestion multi-hôtes est donc laborieuse, d'où l'intérêt d'automatiser.
- Le compte `root` a toujours le rôle `Admin`. On évite de l'utiliser au quotidien et on le réserve aux urgences. Ses actions sont difficiles à attribuer à une personne.
- Des comptes de service techniques existent (`vpxuser`, `dcui`). Ils ne doivent pas être supprimés.
- L'accès au shell est une autorisation distincte (`-GrantShellAccess`), qu'on n'accorde qu'en cas de besoin.
- Un hôte peut être joint à un domaine Active Directory pour utiliser des comptes de l'entreprise, mais en environnement vCenter c'est vCenter qui porte l'authentification.

### Politique de mot de passe

ESXi vérifie la complexité (classes de caractères, longueur) via le paramètre avancé `Security.PasswordQualityControl`. Un mot de passe refusé à la création d'un compte est le signe d'une politique non respectée. En production, on combine une longueur élevée, un historique et un verrouillage après échecs répétés (phase suivante).

### Auditer les droits

Un audit des accès compare l'état réel au besoin. Les questions à poser :

1. Qui a des droits d'administration ?
2. Chaque permission est-elle justifiée par une fonction ?
3. Existe-t-il des comptes inutilisés ou génériques ?
4. Les permissions sont-elles posées au bon niveau (le plus bas possible) ?

`Get-VIPermission` et `Get-VMHostAccount` fournissent les données, `Export-Csv` en fait un livrable de contrôle.

### Points clés à retenir

- Permission = principal + rôle + objet + propagation.
- Un rôle personnalisé se construit par ajout de privilèges nécessaires.
- `NoAccess` l'emporte sur les autres permissions.
- `root` est réservé aux urgences.

### Questions de révision

1. Quelle différence entre rôle et permission ?
   Réponse : le rôle est un ensemble de privilèges, la permission l'attribue à un principal sur un objet.
2. Que se passe-t-il si un utilisateur a `NoAccess` via un groupe et `ReadOnly` via un autre sur le même objet ?
   Réponse : `NoAccess` l'emporte.
3. Pourquoi éviter d'utiliser `root` ?
   Réponse : droits absolus et actions non nominatives, donc non traçables.
4. À quel niveau poser une permission ?
   Réponse : au niveau le plus bas qui répond au besoin, avec propagation seulement si nécessaire.

### Configuration

Crée deux comptes locaux : un compte en lecture seule et un compte d'opérateur. **Point d'attention** : si la création échoue, le mot de passe ne respecte probablement pas la politique de complexité de l'hôte.

```powershell
New-VMHostAccount -Id 'opsread' -Password 'Lab#Read2026!' -UserAccount -Description 'Compte exploitation lecture seule'
New-VMHostAccount -Id 'opsvm' -Password 'Lab#Oper2026!' -UserAccount -Description 'Compte operateur de VM'
```

<img width="1415" height="501" alt="image" src="https://github.com/user-attachments/assets/a8fd145e-987f-4863-8610-a86d0f19c137" />
<img width="1416" height="552" alt="image" src="https://github.com/user-attachments/assets/c2150e96-4c30-4ddd-964f-959bbd910aae" />


Récupère les privilèges d'allumage, d'extinction et de redémarrage, puis crée un rôle personnalisé qui ne contient que ceux-là. Ce rôle ne permet ni de modifier le matériel ni de supprimer une VM.

```powershell
$privs = Get-VIPrivilege -Id 'VirtualMachine.Interact.PowerOn','VirtualMachine.Interact.PowerOff','VirtualMachine.Interact.Reset'
New-VIRole -Name 'Operateur-VM' -Privilege $privs
```

<img width="1402" height="377" alt="image" src="https://github.com/user-attachments/assets/b681b560-dc57-40e7-87f9-a70f4b99f2aa" />


Accorde au compte `opsread` la lecture sur tout l'hôte, avec propagation. Accorde au compte `opsvm` le rôle d'opérateur uniquement sur la VM de test, sans propagation.


```powershell
New-VIPermission -Entity $vmhost -Principal 'opsread' -Role (Get-VIRole -Name 'ReadOnly') -Propagate:$true
New-VIPermission -Entity (Get-VM -Name 'VM-Network-Test') -Principal 'opsvm' -Role (Get-VIRole -Name 'Operateur-VM') -Propagate:$false
```

<img width="1407" height="480" alt="image" src="https://github.com/user-attachments/assets/68ddb8df-8e73-4cf7-baeb-af7eea5aa917" />
<img width="1412" height="557" alt="image" src="https://github.com/user-attachments/assets/0548946b-653d-4093-ad8c-0f778b67765d" />

<img width="1407" height="562" alt="image" src="https://github.com/user-attachments/assets/60f1f7d0-c3ab-45c4-8846-05c89c2c794e" />


Vérifie les permissions posées et la liste des comptes locaux. Le contrôle systématique des droits est une étape d'audit essentielle.

```powershell
Get-VIPermission -Entity $vmhost | Select-Object Principal, Role, Entity, Propagate
Get-VMHostAccount -User | Select-Object Id, Description
```

<img width="1402" height="592" alt="image" src="https://github.com/user-attachments/assets/318375ed-d654-408c-9f22-c13749b2d751" />



---

## 18. Phase 15 : Durcissement avancé (paramètres, journalisation, audit des ports)

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait situer chaque mesure de durcissement dans une couche de sécurité, régler les paramètres avancés de session et de verrouillage, centraliser les journaux, auditer les services et ports exposés, et expliquer les fonctions de sécurité matérielle et de signature des paquets.

### Les paramètres avancés

Les Advanced Settings sont des centaines de paramètres, regroupés par préfixe (`UserVars`, `Security`, `Syslog`, `Net`, `Mem`, `Disk`). Ils se lisent et se modifient avec `Get-AdvancedSetting` et `Set-AdvancedSetting`. Règle de prudence : consigner la valeur d'origine avant toute modification. Le lab le fait avec la variable `$avant`.

| Paramètre | Unité | Rôle | Valeur recommandée dans le lab |
| :--- | :--- | :--- | :--- |
| `UserVars.ESXiShellTimeOut` | secondes | Durée avant désactivation automatique des services shell et SSH | 600 |
| `UserVars.ESXiShellInteractiveTimeOut` | secondes | Inactivité avant fermeture d'une session shell | 300 |
| `UserVars.DcuiTimeOut` | secondes | Inactivité avant fermeture de la console locale | 600 |
| `Security.AccountLockFailures` | nombre | Échecs de connexion avant verrouillage | 5 |
| `Security.AccountUnlockTime` | secondes | Durée du verrouillage | 900 |
| `Annotations.WelcomeMessage` | texte | Message avant authentification | Avertissement légal |

Un timeout de zéro désactive la fonction. Le verrouillage de compte protège contre la force brute, mais peut aussi bloquer un administrateur légitime : prévoir une procédure de déverrouillage.

### La journalisation

Par défaut, les journaux de l'hôte sont conservés localement. Selon la configuration, ils peuvent résider dans un espace temporaire en mémoire et disparaître au redémarrage. D'où la nécessité d'un **serveur syslog distant** qui offre :

- la conservation après redémarrage de l'hôte,
- la corrélation entre systèmes,
- la preuve en cas d'incident (un attaquant ne peut pas effacer des journaux déjà déportés),
- l'alimentation d'un outil de détection.

Le paramètre `Syslog.global.logHost` prend une adresse sous la forme `protocole://hôte:port` (par exemple `udp://serveur:514`, ou `tcp://` et `ssl://` pour un transport fiable ou chiffré). Il faut aussi ouvrir la règle de pare-feu `syslog`, sinon rien n'est transmis.

### Audit de la surface d'attaque

Deux contrôles complémentaires :

1. **Services** : lister avec `Get-VMHostService` ceux qui tournent ou démarrent automatiquement. Tout service non nécessaire est à désactiver.
2. **Pare-feu** : lister les règles activées avec `Get-VMHostFirewallException`. Chaque port ouvert doit être justifié et, quand c'est possible, restreint par adresse.

### Sécurité du système

| Mécanisme | Fonction |
| :--- | :--- |
| Secure Boot | Vérifie la signature du chargeur et du noyau au démarrage, empêche l'exécution de code non signé |
| TPM 2.0 | Puce qui mesure l'état de démarrage de l'hôte et permet son attestation |
| `execInstalledOnly` | N'exécute que des binaires installés depuis des paquets signés |
| Niveau d'acceptation des VIB | Définit qui est autorisé à signer les paquets installables |
| Certificats | Chiffrent et authentifient les échanges. Sur un hôte seul, le certificat est auto-signé par défaut |
| Lockdown mode | Interdit l'accès direct à l'hôte pour imposer le passage par vCenter |

Les niveaux d'acceptation, du plus strict au plus permissif, sont : `VMwareCertified`, `VMwareAccepted`, `PartnerSupported`, `CommunitySupported`. On lit le niveau avec `$esxcli.software.acceptance.get.Invoke()`. En production, on évite `CommunitySupported`.

Le lockdown mode n'est pas adapté à un hôte autonome : avec lui, il n'y a plus de point de gestion de substitution, et un blocage peut couper définitivement l'accès à l'hôte.

### TLS et chiffrement

Les versions anciennes du protocole TLS (1.0 et 1.1) sont désactivées dans les versions récentes d'ESXi, qui privilégient TLS 1.2 au minimum. Le chiffrement protège les échanges de gestion. Les données de VM peuvent aussi être chiffrées (VM Encryption), ce qui exige un fournisseur de clés et vCenter.

### Méthode d'audit de conformité

1. Choisir un référentiel (CIS, Security Configuration Guide).
2. Extraire l'état actuel de l'hôte par script.
3. Comparer avec les valeurs attendues.
4. Corriger les écarts, documenter les exceptions justifiées.
5. Recommencer à intervalle régulier et après chaque mise à jour.

### Points clés à retenir

- Consigner les valeurs d'origine avant toute modification.
- Sans syslog distant, les journaux peuvent disparaître au redémarrage.
- Chaque service et chaque port ouvert doit se justifier.
- Le lockdown mode ne convient pas à un hôte autonome de lab.

### Questions de révision

1. Pourquoi enregistrer `$avant` ?
   Réponse : pour pouvoir restaurer les valeurs d'origine et documenter le changement.
2. Pourquoi un syslog distant protège-t-il en cas d'incident ?
   Réponse : les journaux déportés ne peuvent plus être effacés par un attaquant sur l'hôte et survivent au redémarrage.
3. Quel est le niveau d'acceptation le plus strict ?
   Réponse : `VMwareCertified`.
4. Quel est le risque du verrouillage de compte ?
   Réponse : qu'un administrateur légitime soit bloqué par des échecs répétés.

### Configuration

Enregistre les valeurs actuelles dans `$avant` avant toute modification, afin de pouvoir revenir en arrière. Il s'agit d'une bonne pratique de gestion du changement.

```powershell
$noms = 'UserVars.ESXiShellTimeOut','UserVars.ESXiShellInteractiveTimeOut','UserVars.DcuiTimeOut','Security.AccountLockFailures','Security.AccountUnlockTime'
$avant = Get-AdvancedSetting -Entity $esxi -Name $noms
$avant | Select-Object Name, Value
```

<img width="1401" height="421" alt="image" src="https://github.com/user-attachments/assets/f22a9ad8-a22f-43a3-a643-d1fe45881086" />


Fixe le timeout du shell à 10 minutes, le timeout des sessions shell interactives à 5 minutes, le timeout de la console DCUI à 10 minutes, le verrouillage après 5 échecs et le déblocage automatique après 15 minutes.

```powershell
Get-AdvancedSetting -Entity $esxi -Name 'UserVars.ESXiShellTimeOut' | Set-AdvancedSetting -Value 600 -Confirm:$false
Get-AdvancedSetting -Entity $esxi -Name 'UserVars.ESXiShellInteractiveTimeOut' | Set-AdvancedSetting -Value 300 -Confirm:$false
Get-AdvancedSetting -Entity $esxi -Name 'UserVars.DcuiTimeOut' | Set-AdvancedSetting -Value 600 -Confirm:$false
Get-AdvancedSetting -Entity $esxi -Name 'Security.AccountLockFailures' | Set-AdvancedSetting -Value 5 -Confirm:$false
Get-AdvancedSetting -Entity $esxi -Name 'Security.AccountUnlockTime' | Set-AdvancedSetting -Value 900 -Confirm:$false
```

<img width="1420" height="640" alt="image" src="https://github.com/user-attachments/assets/bc7d18d7-b11b-4f82-ad99-94dc15b36c9b" />



Définit le message d'avertissement affiché sur l'interface de l'hôte.

```powershell
Get-AdvancedSetting -Entity $esxi -Name 'Annotations.WelcomeMessage' | Set-AdvancedSetting -Value 'Acces reserve au personnel autorise. Toute activite est journalisee.' -Confirm:$false
```

<img width="1406" height="392" alt="image" src="https://github.com/user-attachments/assets/e56876f1-9388-423f-842c-b66927e3f34c" />


Vérifier si un serveur syslog est déjà configuré

```powershell
Get-AdvancedSetting -Entity $esxi -Name 'Syslog.global.logHost' | Select-Object Value
```

<img width="1356" height="317" alt="image" src="https://github.com/user-attachments/assets/cceb36dc-7ca2-47ab-a2ff-6b7237958832" />


Vérifier que l'exception 'syslog' est activée

```powershell
Get-VMHostFirewallException -VMHost $esxi -Name 'syslog' | Select-Object Name, Enabled
```

<img width="1237" height="270" alt="image" src="https://github.com/user-attachments/assets/4e6385d5-00b2-463c-8b11-caede6653ee0" />

Le résultat `Enabled` : `False` signifie que l'ESXi est configuré pour envoyer les logs, mais que le pare-feu bloque le trafic sortant.

Activer l'exception de pare-feu pour Syslog
```powershell
Get-VMHostFirewallException -VMHost $esxi -Name 'syslog' | Set-VMHostFirewallException -Enabled $true -Confirm:$false
```
<img width="1407" height="325" alt="image" src="https://github.com/user-attachments/assets/f496231f-a999-4cb8-a576-0fa9f8a634fd" />

Vérifier que c'est bien activé

```powershell
Get-VMHostFirewallException -VMHost $esxi -Name 'syslog' | Select-Object Name, Enabled
```

<img width="1316" height="237" alt="image" src="https://github.com/user-attachments/assets/e3d5e78a-3548-40c3-a273-6e085e0db1c4" />


Configurer l'envoi des logs vers le serveur Syslog (ici, le vCenter du lab)
- Remplacez l'adresse par l'IP ou le FQDN de votre serveur de logs

```powershell
$syslogServer = "vcsa-01a.vcf.sddc.lab"
Get-AdvancedSetting -Entity $esxi -Name 'Syslog.global.logHost' | Set-AdvancedSetting -Value "udp://$($syslogServer):514" -Confirm:$false
```

<img width="1392" height="382" alt="image" src="https://github.com/user-attachments/assets/ba8d7a3c-3ffe-44df-930e-8bb902c0dd43" />


- Autoriser le trafic sortant dans le pare-feu de l'ESXi (Port 514 UDP)

```powershell
Get-VMHostFirewallException -VMHost $esxi -Name 'syslog' | Set-VMHostFirewallException -Enabled $true -Confirm:$false
```

<img width="1402" height="327" alt="image" src="https://github.com/user-attachments/assets/ac714f86-73f5-463b-a977-ad9e8de8ff22" />



- Vérification

```powershell
Get-AdvancedSetting -Entity $esxi -Name 'Syslog.global.logHost' | Select-Object Value
Get-VMHostFirewallException -VMHost $esxi -Name 'syslog' | Select-Object Name, Enabled
```

<img width="1287" height="307" alt="image" src="https://github.com/user-attachments/assets/bcd73bc1-40b3-49b5-ae1f-e51ffaa20a76" />

<img width="1426" height="565" alt="image" src="https://github.com/user-attachments/assets/1942c0ef-f2ca-40c1-9643-e856ef87857d" />


Liste tous les services de l'hôte avec leur état et leur politique de démarrage, puis désactive le démarrage automatique du shell ESXi (`TSM`). Cette liste est le point de départ d'un audit de la surface d'attaque.

```powershell
Get-VMHostService -VMHost $esxi | Select-Object Key, Label, Running, Policy
Get-VMHostService -VMHost $esxi | Where-Object { $_.Key -eq 'TSM' } | Set-VMHostService -Policy 'off' -Confirm:$false
```

<img width="1405" height="650" alt="image" src="https://github.com/user-attachments/assets/661be37d-d5ae-4abb-b602-2ee0b6a83aff" />
<img width="1397" height="320" alt="image" src="https://github.com/user-attachments/assets/54b697c6-f6cc-4c5d-8fc0-3a68d63762b5" />


Affiche uniquement les règles de pare-feu activées, avec leurs ports et leurs protocoles. Chaque règle ouverte doit pouvoir être justifiée.

```powershell
Get-VMHostFirewallException -VMHost $esxi | Where-Object { $_.Enabled } | Select-Object Name, IncomingPorts, OutgoingPorts, Protocols, ServiceRunning
```
<img width="1402" height="647" alt="image" src="https://github.com/user-attachments/assets/aa865b1d-e0c5-4224-9b76-412525bdfd49" />
<img width="1406" height="607" alt="image" src="https://github.com/user-attachments/assets/aec0cd92-8453-40ba-b3d0-f12daf9745d4" />
<img width="1412" height="652" alt="image" src="https://github.com/user-attachments/assets/6418262a-45f3-406b-8364-b4b0f69736c8" />
<img width="1415" height="651" alt="image" src="https://github.com/user-attachments/assets/65e9e607-afb4-4a67-96d1-7b82ff03797c" />
<img width="1417" height="380" alt="image" src="https://github.com/user-attachments/assets/2f00e285-ad66-4b76-a60e-40f397b8d53d" />


---

## 19. Phase 16 : Supervision, événements et journaux

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait appliquer une méthode de supervision, lire les compteurs de performance essentiels, distinguer tâches, événements et journaux, localiser les fichiers de journaux de l'hôte et produire un inventaire exploitable.

### La méthode USE

Pour chaque ressource (CPU, mémoire, disque, réseau), on examine trois aspects :

| Aspect | Question | Exemple pour le CPU |
| :--- | :--- | :--- |
| Utilisation (Utilization) | Quelle part de la ressource est occupée ? | `cpu.usage.average` |
| Saturation | Du travail attend-il ? | CPU Ready |
| Erreurs (Errors) | Y a-t-il des défaillances ? | Paquets abandonnés, erreurs disque |

Une utilisation élevée sans saturation n'est pas forcément un problème. Une saturation, même avec une utilisation modérée, l'est presque toujours.

### Les compteurs essentiels

Une métrique se nomme `groupe.compteur.agrégation`. Les groupes principaux sont `cpu`, `mem`, `disk`, `net`, `sys` et `datastore`.

| Ressource | Compteur | À surveiller |
| :--- | :--- | :--- |
| CPU | `cpu.usage.average` | Utilisation en pourcentage |
| CPU | `cpu.ready.summation` | Saturation, voir phase 10 |
| Mémoire | `mem.usage.average` | Pourcentage de mémoire utilisée |
| Mémoire | `mem.vmmemctl.average` et `mem.swapped.average` | Ballooning et swap, doivent rester nuls |
| Disque | `disk.maxTotalLatency.latest` | Latence maximale observée |
| Réseau | `net.usage.average` | Débit |
| Réseau | `net.droppedRx.summation` | Paquets abandonnés, doit rester nul |

Sur un hôte autonome, seules les statistiques en temps réel sont disponibles : un échantillon toutes les 20 secondes, conservé environ une heure. Les cumuls historiques (jour, semaine, mois) nécessitent vCenter. Il faut donc exporter régulièrement ses mesures pour garder une trace.

### Esxtop

`esxtop` est l'outil interactif de performance de l'hôte (SSH ou console). Écrans courants : `c` (CPU), `m` (mémoire), `d` (disque), `n` (réseau). Le mode `esxtop -b -n 10 > fichier.csv` produit un enregistrement exploitable après coup. Il donne des indicateurs plus fins que l'API, mais PowerCLI suffit pour la plupart des besoins de suivi.

### Tâches, événements, alarmes

| Notion | Définition |
| :--- | :--- |
| Tâche (task) | Opération demandée (démarrer une VM) avec son état et son résultat |
| Événement (event) | Enregistrement horodaté de quelque chose qui s'est produit, avec utilisateur et message |
| Alarme | Règle qui déclenche une notification quand une condition est remplie. Surtout utilisée sous vCenter |

Les événements servent d'audit : qui a fait quoi, quand. Ils sont consultables avec `Get-VIEvent`.

### Les journaux de l'hôte

Ils se trouvent dans `/var/log` et sont accessibles par `Get-Log` sans SSH :

| Fichier | Contenu |
| :--- | :--- |
| `vmkernel.log` | Noyau, stockage, pilotes, réseau bas niveau |
| `hostd.log` | Agent de gestion, opérations sur les VM, API |
| `vpxa.log` | Communication avec vCenter |
| `vobd.log` | Événements du noyau (observations) |
| `auth.log` | Authentifications |
| `shell.log` | Commandes tapées dans le shell |
| `fdm.log` | Haute disponibilité (sous cluster) |
| `vmware.log` (dans le dossier de la VM) | Journal de la VM elle-même |

Ils tournent (rotation) et sont limités en taille : sans export ni syslog, les anciennes lignes sont perdues.

### Lire une anomalie

Démarche de diagnostic type :

1. Décrire le symptôme précis et son heure.
2. Consulter les événements autour de cette heure.
3. Lire `hostd.log` pour l'opération concernée, `vmkernel.log` pour un problème de stockage ou de réseau.
4. Corréler avec les compteurs de performance.
5. Formuler une hypothèse, la tester, noter la cause et la correction.

### L'inventaire comme document d'exploitation

Un inventaire régulier (VM, ressources, datastores) permet de détecter les VM oubliées, les disques qui grossissent et les écarts de configuration. Exporté en CSV, il se compare facilement d'une date à l'autre. Les propriétés calculées servent à le rendre lisible.

### Points clés à retenir

- USE : Utilisation, Saturation, Erreurs.
- Sur hôte autonome, seules les statistiques en temps réel existent.
- Les événements répondent à « qui, quoi, quand ».
- Sans syslog ni export, les journaux se perdent par rotation.

### Questions de révision

1. Que mesure l'aspect Saturation de la méthode USE ?
   Réponse : la quantité de travail qui attend la ressource.
2. Pourquoi exporter les statistiques sur un hôte autonome ?
   Réponse : l'historique est limité à environ une heure sans vCenter.
3. Dans quel fichier chercher un problème de stockage ?
   Réponse : `vmkernel.log`.
4. Quelle cmdlet donne la liste des journaux disponibles ?
   Réponse : `Get-LogType`.

### Configuration

Récupère les 10 derniers échantillons en temps réel d'utilisation du CPU et de la mémoire de l'hôte, exprimés en pourcentage.

```powershell
Get-Stat -Entity $vmhost -Stat 'cpu.usage.average','mem.usage.average' -Realtime -MaxSamples 10 | Select-Object Timestamp, MetricId, Value, Unit
```

<img width="1391" height="660" alt="image" src="https://github.com/user-attachments/assets/cbebf767-c597-41f0-8a00-84150da7c303" />

Même principe pour une VM. Elle doit être allumée pour fournir des échantillons.


```powershell
Get-Stat -Entity (Get-VM -Name 'VM-Network-Test') -Stat 'cpu.usage.average' -Realtime -MaxSamples 5 | Select-Object Timestamp, Value, Unit
```

<img width="1402" height="402" alt="image" src="https://github.com/user-attachments/assets/fb12e08d-721e-4d28-842f-5926577adadf" />


Lit les indicateurs agrégés de l'hôte. `Uptime` est exprimé en secondes.

```powershell
$stats = (Get-View -Id $vmhost.Id).Summary.QuickStats
$stats | Select-Object OverallCpuUsage, OverallMemoryUsage, Uptime
```

<img width="1130" height="312" alt="image" src="https://github.com/user-attachments/assets/f2e1070e-1cf1-4883-b737-f65bcb019ec1" />

Affiche les 20 derniers événements de l'hôte avec leur date, l'utilisateur concerné et le message. Utile pour reconstituer les actions réalisées pendant le lab.


```powershell
Get-VIEvent -Entity $vmhost -MaxSamples 20 | Select-Object CreatedTime, UserName, FullFormattedMessage
```

<img width="1402" height="657" alt="image" src="https://github.com/user-attachments/assets/cb17a73d-ea97-41b7-aff4-0fd799e08062" />


Liste les journaux disponibles sur l'hôte puis affiche les 20 dernières lignes du journal `vmkernel`. **Point d'attention** : si `Get-Log` ne renvoie rien, la clé est invalide. Utiliser les clés affichées par `Get-LogType`.

```powershell
Get-LogType -VMHost $esxi | Select-Object Key, Info
(Get-Log -VMHost $esxi -Key 'vmkernel' -NumLines 20).Entries
```

<img width="1412" height="651" alt="image" src="https://github.com/user-attachments/assets/bdbab1d0-9cf9-44f6-ab43-77b4f7d97fe0" />

Exporte un inventaire des VM avec leur état, leur dimensionnement, leur espace disque et leur datastore. Le résultat est un document d'exploitation réutilisable.


```powershell
Get-VM | Select-Object Name, PowerState, NumCpu, MemoryGB, ProvisionedSpaceGB, UsedSpaceGB, @{N='Datastore';E={(Get-Datastore -RelatedObject $_).Name -join ','}} | Export-Csv -Path "/home/holuser/Inventaire-VM.csv" -NoTypeInformation
```

<img width="1406" height="345" alt="image" src="https://github.com/user-attachments/assets/bd3766b0-65a5-4d38-bd9a-4752afd923f1" />



---

## 20. Phase 17 : Sauvegarde et export

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait distinguer sauvegarde de configuration et sauvegarde de données, définir RPO et RTO, expliquer le fonctionnement des sauvegardes d'images de VM, appliquer la règle 3-2-1 et exporter une VM au format OVF.

### Que protège-t-on ?

| Élément | Contenu | Outil |
| :--- | :--- | :--- |
| Configuration de l'hôte | Réseau, comptes, paramètres, licences | Bundle de configuration |
| Machines virtuelles | Disques, configuration | Sauvegarde d'images, export OVF, clonage |
| Données applicatives | Bases de données, fichiers | Sauvegarde applicative |
| Infrastructure de gestion | vCenter, bases | Sauvegardes dédiées |

Une erreur classique est de croire que sauvegarder l'hôte protège les VM, ou l'inverse. Ce sont deux protections indépendantes.

### RPO et RTO

- **RPO (Recovery Point Objective)** : perte de données maximale acceptable, exprimée en temps. Un RPO de 24 heures signifie qu'on tolère de perdre une journée de données, donc une sauvegarde au moins quotidienne.
- **RTO (Recovery Time Objective)** : durée maximale acceptable pour remettre le service en marche.

Ces deux objectifs viennent du besoin métier. Ils déterminent la fréquence des sauvegardes, la technologie et le budget.

### La règle 3-2-1

Trois copies des données, sur deux types de supports différents, dont une copie hors site. Une version moderne ajoute une copie immuable (non modifiable) ou isolée du réseau, pour résister aux rançongiciels. Une sauvegarde jamais testée en restauration n'est pas une sauvegarde : prévoir des tests de restauration réguliers.

### La sauvegarde de configuration d'un hôte

Le bundle est une archive compressée de l'état de configuration. Elle permet de reconstruire un hôte après réinstallation : on réinstalle la même version et build d'ESXi, puis on restaure le bundle. La restauration exige le mode maintenance et provoque un redémarrage. Les paramètres locaux sensibles y figurent, il faut donc **protéger le fichier** (accès restreint, chiffrement du support).

### La sauvegarde d'images de VM

Les logiciels de sauvegarde modernes utilisent l'API VADP (vSphere APIs for Data Protection) :

1. Un snapshot de la VM est créé pour figer son état (avec quiesce si possible).
2. Les données du disque figé sont lues sans interrompre la VM.
3. Le **CBT (Changed Block Tracking)** indique quels blocs ont changé depuis la dernière sauvegarde, ce qui permet des sauvegardes incrémentales rapides.
4. Le snapshot est supprimé.

Cela repose le plus souvent sur vCenter. Sur un hôte autonome, on recourt à des exports ou à des solutions qui acceptent les hôtes isolés.

### Le format OVF et OVA

- **OVF** : standard ouvert qui décrit une VM par un descripteur XML (`.ovf`), un manifeste de contrôle d'intégrité (`.mf`) et un ou plusieurs disques (`.vmdk`).
- **OVA** : même contenu regroupé dans une archive unique.

L'export est indépendant de la plateforme et convient au transfert, à l'archivage ponctuel et au déploiement d'un modèle. Il n'est pas adapté à une sauvegarde quotidienne de nombreuses VM, car il est lent et manuel. L'export d'une VM éteinte garantit la cohérence.

### Conservation et sécurité

- Définir une durée de rétention par type de donnée.
- Chiffrer les sauvegardes quittant le site.
- Séparer les identifiants du système de sauvegarde de ceux de la production.
- Surveiller les échecs de sauvegarde par alerte.

### Points clés à retenir

- Configuration de l'hôte et VM sont protégées séparément.
- RPO se rapporte à la perte de données, RTO au temps de reprise.
- Une sauvegarde non testée n'est pas fiable.
- Un snapshot n'est pas une sauvegarde (rappel de la phase 5).

### Questions de révision

1. Que couvre un bundle de configuration ?
   Réponse : la configuration de l'hôte (réseau, comptes, paramètres), pas le contenu des VM.
2. Que signifie un RTO de 4 heures ?
   Réponse : le service doit être rétabli en 4 heures au plus.
3. À quoi sert CBT ?
   Réponse : à ne sauvegarder que les blocs modifiés depuis la dernière sauvegarde.
4. Quels fichiers composent un export OVF ?
   Réponse : un descripteur `.ovf`, un manifeste `.mf` et les disques `.vmdk`.

### Configuration

Crée le dossier de destination puis télécharge le bundle de configuration de l'hôte. **Point d'attention** : le dossier doit exister avant l'appel, sinon la commande échoue.

```powershell
New-Item -Path "/home/holuser/backup" -ItemType Directory -Force | Out-Null
Get-VMHostFirmware -VMHost $esxi -BackupConfiguration -DestinationPath "/home/holuser/backup"
```

<img width="1390" height="477" alt="image" src="https://github.com/user-attachments/assets/993888b0-abe6-47aa-b942-52111bb3b598" />


Éteint la VM puis l'exporte au format OVF dans le dossier indiqué. L'export d'une VM éteinte garantit un état cohérent.


```powershell
Stop-VM -VM 'VM-Network-Test' -Confirm:$false
New-Item -Path "/home/holuser/export" -ItemType Directory -Force | Out-Null
Export-VApp -VM (Get-VM -Name 'VM-Network-Test') -Destination "/home/holuser/export" -Format Ovf
```

<img width="1392" height="557" alt="image" src="https://github.com/user-attachments/assets/8a6cb94e-918f-4d51-b665-dffdb726d211" />

Illustre la procédure de restauration : mise en maintenance puis restauration du bundle, qui redémarre l'hôte. Le nom exact du fichier `.tgz` dépend de la sauvegarde réalisée. Dans HOL, un redémarrage de l'hôte peut rendre l'environnement inutilisable.

####  Référence uniquement, à ne pas exécuter dans le lab

- Vérifier que l'hôte est bien en mode maintenance (obligatoire pour la restauration)

```powershell
Set-VMHost -VMHost $esxi -State Maintenance -Confirm:$false
```

- (Optionnel mais recommandé) Redémarrer l'hôte avant la restauration pour s'assurer que rien ne bloque

```powershell
Restart-VMHost -VMHost $esxi -Confirm:$false
```

- Lancer la restauration

**Note** : L'erreur "connexion fermée" qui peut apparaître à la fin est normale (l'hôte redémarre)

```powershell
Set-VMHostFirmware -VMHost $esxi -Restore -SourcePath "/home/holuser/backup/configBundle-esx-03a.vcf.sddc.lab.tgz" -HostCredential $esxiCred
```

- Attendre le redémarrage de l'hôte (environ 2-5 minutes), puis se reconnecter

```powershell
Start-Sleep -Seconds 180
Connect-VIServer -Server "esx-03a.vcf.sddc.lab" -Credential $esxiCred
```

- Vérifier que l'hôte est bien sorti du mode maintenance et que la configuration est appliquée

```powershell
Get-VMHost | Select-Object Name, ConnectionState, Version, Build
```
---

## 21. Phase 18 : Automatisation et rapport de santé

### Objectifs pédagogiques

À la fin de cette phase, l'étudiant sait écrire une fonction PowerShell robuste, gérer correctement les erreurs, journaliser l'exécution, éviter la présence de secrets en clair, optimiser les requêtes vers l'API et produire un rapport automatisable.

### Pourquoi automatiser

L'automatisation apporte la reproductibilité (même résultat à chaque exécution), la rapidité, la traçabilité et la réduction des erreurs manuelles. Elle impose en retour de la rigueur : un script mal écrit commet la même erreur sur cent hôtes en quelques secondes. D'où les règles qui suivent.

### Ecrire une fonction propre

| Principe | Application |
| :--- | :--- |
| Verbe-Nom approuvé | `Get-LabHealthCheck` (`Get-Verb` liste les verbes valides) |
| `[CmdletBinding()]` | Active les paramètres communs (`-Verbose`, `-ErrorAction`) |
| Paramètres typés et obligatoires | `[Parameter(Mandatory = $true)]` |
| Validation | `[ValidateSet('Thin','Thick')]`, `[ValidateRange(1,64)]` |
| Retourner des objets | Pas de `Write-Host` pour les données, utiliser la sortie standard |
| Pas de valeurs codées en dur | Paramètres ou fichier de configuration |
| Commentaires utiles | Expliquer le pourquoi, pas le comment |

Pour les fonctions qui modifient, ajouter `[CmdletBinding(SupportsShouldProcess)]` et appeler `$PSCmdlet.ShouldProcess()` permet de bénéficier de `-WhatIf` et `-Confirm`, comme les cmdlets natives.

### Gérer les erreurs

PowerShell distingue les erreurs **terminantes**, qui interrompent et sont interceptées par `catch`, et les erreurs **non terminantes**, qui s'affichent puis le script continue. Un `try` ne capture que les erreurs terminantes. Pour qu'une cmdlet déclenche le `catch`, il faut lui ajouter `-ErrorAction Stop` ou fixer `$ErrorActionPreference = 'Stop'` dans le bloc.

```powershell
try {
    $ErrorActionPreference = 'Stop'
    Get-VM -Name 'inexistante'
}
catch {
    Write-Warning "Échec : $($_.Exception.Message)"
}
finally {
    $ErrorActionPreference = 'Continue'
}
```

`finally` s'exécute dans tous les cas : c'est l'endroit pour nettoyer (fermer une session, supprimer un fichier temporaire).

### Journaliser

- `Start-Transcript` enregistre tout ce qui s'affiche dans la session.
- `Write-Verbose` fournit des messages détaillés activés à la demande avec `-Verbose`.
- Pour un suivi durable, écrire dans un fichier horodaté ou envoyer à un syslog.

### Les secrets

Ne jamais laisser un mot de passe en clair dans un script versionné. Options, par ordre croissant de robustesse : saisie interactive avec `Get-Credential`, variable d'environnement, module `Microsoft.PowerShell.SecretManagement` avec un coffre (KeePass, HashiCorp Vault, Azure Key Vault). Le chiffrement de fichiers d'identifiants par `Export-Clixml` repose sur DPAPI et ne fonctionne pas sous Linux.

### Ecrire des scripts performants

- Filtrer le plus tôt possible : utiliser les paramètres de la cmdlet (`Get-VM -Name`) avant `Where-Object`.
- Pour de gros inventaires, `Get-View` avec la propriété `-Property` est beaucoup plus rapide que `Get-VM` suivi d'accès à `ExtensionData`, qui multiplie les appels.
- Éviter les appels en boucle : récupérer une fois, réutiliser la variable.

### Rendre le script idempotent

Un script idempotent contrôle l'état avant d'agir : il crée un objet seulement s'il n'existe pas, change une valeur seulement si elle diffère. On peut le relancer sans crainte. C'est la propriété la plus importante pour un script de configuration.

### Industrialiser

| Étape | Outil |
| :--- | :--- |
| Contrôle de version | Git |
| Tests automatiques | Pester |
| Analyse statique | PSScriptAnalyzer |
| Planification sous Linux | cron avec `pwsh -File script.ps1` |
| Documentation | Aide commentée (`.SYNOPSIS`, `.EXAMPLE`) |

### Points clés à retenir

- `try` ne capture que les erreurs terminantes : utiliser `-ErrorAction Stop`.
- Pas de secret en clair, pas de valeur codée en dur.
- Filtrer côté serveur, éviter les accès répétés à `ExtensionData`.
- Idempotence et `-WhatIf` rendent un script sûr.

### Questions de révision

1. Pourquoi un `catch` peut-il ne pas se déclencher ?
   Réponse : l'erreur est non terminante, il faut `-ErrorAction Stop`.
2. À quoi sert le bloc `finally` ?
   Réponse : exécuter un nettoyage dans tous les cas, erreur ou non.
3. Quel est l'intérêt de `SupportsShouldProcess` ?
   Réponse : permettre `-WhatIf` et `-Confirm` sur une fonction qui modifie.
4. Qu'est-ce qu'un script idempotent ?
   Réponse : un script qui vérifie l'état avant d'agir et peut être rejoué sans effet indésirable.

### Configuration

Démarre l'enregistrement de la session dans un fichier. `-Append` ajoute à la suite du contenu existant.


```powershell
Start-Transcript -Path "/home/holuser/lab-session.log" -Append
```

<img width="1026" height="270" alt="image" src="https://github.com/user-attachments/assets/b87be310-b294-42cd-bf39-a410d06d26a2" />


Définit une fonction de contrôle de santé. Elle collecte l'état de l'hôte, le nombre de VM allumées, la consommation CPU et mémoire, les datastores dont l'espace libre est inférieur à 20 pour cent, ainsi que l'état de SSH et de NTP. Le bloc `catch` affiche un message clair en cas de problème. La ligne `$ErrorActionPreference = 'Stop'` rend les erreurs terminantes, ce qui permet au bloc `catch` de les intercepter, comme expliqué dans le cours. Les indicateurs choisis correspondent à ce que surveille un administrateur au quotidien.


```powershell
function Get-LabHealthCheck {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        $VMHost
    )
    try {
        $ErrorActionPreference = 'Stop'
        $view = Get-View -Id $VMHost.Id
        $dsFaibles = @(Get-Datastore -VMHost $VMHost | Where-Object { ($_.FreeSpaceGB / $_.CapacityGB) -lt 0.2 })
        $ssh = Get-VMHostService -VMHost $VMHost | Where-Object { $_.Key -eq 'TSM-SSH' }
        $ntp = Get-VMHostService -VMHost $VMHost | Where-Object { $_.Key -eq 'ntpd' }
        [PSCustomObject]@{
            Hote                = $VMHost.Name
            Etat                = $VMHost.ConnectionState
            VMAllumees          = @(Get-VM -Location $VMHost | Where-Object { $_.PowerState -eq 'PoweredOn' }).Count
            CpuUtiliseMHz       = $view.Summary.QuickStats.OverallCpuUsage
            MemoireUtiliseeMo   = $view.Summary.QuickStats.OverallMemoryUsage
            DatastoresSous20pct = $dsFaibles.Count
            SshActif            = $ssh.Running
            NtpActif            = $ntp.Running
            Horodatage          = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
        }
    }
    catch {
        Write-Error "Echec du controle : $($_.Exception.Message)"
    }
}
```
<img width="1412" height="650" alt="image" src="https://github.com/user-attachments/assets/4a8d5597-c37c-4bec-82b1-721ba7348bcb" />


Exécute la fonction sur l'hôte du lab puis affiche le résultat sous forme de liste lisible.

```powershell
$rapport = Get-LabHealthCheck -VMHost $vmhost
$rapport | Format-List
```

<img width="960" height="391" alt="image" src="https://github.com/user-attachments/assets/6716e3a1-3cdb-46fd-8766-1f8d2eb37e2d" />



Convertit le rapport en page HTML consultable dans un navigateur puis arrête l'enregistrement de la session. On obtient ainsi deux livrables : le rapport et le journal d'exécution.

```powershell
$rapport | ConvertTo-Html -Title 'Rapport de sante ESXi' | Out-File "/home/holuser/Rapport-Sante.html"
Stop-Transcript
```
<img width="1375" height="167" alt="image" src="https://github.com/user-attachments/assets/6a35f1e2-c7dc-4b2c-8b6b-eaaa98783fba" />


---

## 22. Nettoyage de fin de laboratoire

Dans HOL, l'environnement est détruit à la fin de la session. Ces commandes sont utiles si l'on souhaite rejouer le lab dans la même session.

```powershell
Stop-VM -VM 'VM-Network-Test' -Confirm:$false -ErrorAction SilentlyContinue
Remove-VM -VM 'VM-Network-Test' -DeletePermanently -Confirm:$false
```

Éteint puis supprime la VM et ses fichiers du datastore. Les fichiers de l'ancien disque détaché lors de la phase 7 restent à supprimer manuellement dans le dossier de la VM sur `ds-iscsi01` si l'on veut libérer l'espace.

```powershell
Get-VMHostNetworkAdapter -VMHost $vmhost -VMKernel | Where-Object { $_.IP -eq '10.0.10.6' } | Remove-VMHostNetworkAdapter -Confirm:$false
Get-VirtualPortGroup -VMHost $vmhost -Name 'VLAN10-Serveurs','VLAN20-Clients' | Remove-VirtualPortGroup -Confirm:$false
Remove-VirtualSwitch -VirtualSwitch (Get-VirtualSwitch -VMHost $vmhost -Name 'vSwitch1') -Confirm:$false
```

Supprime dans l'ordre l'interface VMkernel, les port groups puis le vSwitch. Cet ordre est obligatoire : on ne peut pas retirer un objet réseau encore utilisé.

Supprime les éléments de gestion des accès, de ressources et de configuration ajoutés pendant les phases 10 à 17.

```powershell
Get-VIPermission -Entity $vmhost | Where-Object { $_.Principal -eq 'opsread' } | Remove-VIPermission -Confirm:$false
Get-VIPermission -Entity (Get-VM -Name 'VM-Network-Test' -ErrorAction SilentlyContinue) -ErrorAction SilentlyContinue | Where-Object { $_.Principal -eq 'opsvm' } | Remove-VIPermission -Confirm:$false
Remove-VIRole -Role (Get-VIRole -Name 'Operateur-VM') -Confirm:$false
Get-VMHostAccount -User -Id 'opsread','opsvm' | Remove-VMHostAccount -Confirm:$false
Get-ResourcePool -Name 'RP-Production' | Remove-ResourcePool -Confirm:$false
$avant | ForEach-Object { Get-AdvancedSetting -Entity $esxi -Name $_.Name | Set-AdvancedSetting -Value $_.Value -Confirm:$false }
```

Retire les permissions, le rôle personnalisé et les deux comptes locaux, supprime le resource pool et restaure les paramètres avancés à leurs valeurs d'origine grâce à la variable `$avant` capturée en phase 15. Les permissions doivent être retirées avant le rôle et les comptes. Les fichiers générés dans `/home/holuser` (rapports, `backup`, `export`) peuvent être supprimés avec `Remove-Item -Recurse`.

```powershell
Disconnect-VIServer -Server * -Confirm:$false
```

Ferme toutes les sessions PowerCLI ouvertes.

---

## 23. Journal de résolution des erreurs

| Domaine | Erreur rencontrée | Cause | Solution |
| :--- | :--- | :--- | :--- |
| Réseau | `network doesn't exist on host` | Défaut de résolution de nom réseau sur ESXi autonome | Créer la VM sans réseau, puis `Set-NetworkAdapter -Portgroup $pg` avec l'objet |
| Stockage | `Invalid configuration for device '0'` | L'API refuse une ISO située dans `contentlib` | Copier l'ISO dans un dossier standard (`ds:\ISO`) avant le montage |
| Stockage | `The operation is not supported` | Storage vMotion indisponible sans vCenter | Recréer un disque sur le datastore cible avec `New-HardDisk` |
| Stockage | `disk path is not accessible` | Copie incomplète, fichier `-flat` manquant | Ne pas copier le VMDK à la main, passer par `New-HardDisk` ou `vmkfstools` |
| Sécurité | `Cannot overwrite variable Host` | `$host` est une variable réservée | Utiliser `$esxi` ou `$vmhost` |
| Esxcli | `Cannot add IP when allowed-all is true` | Configuration par défaut du pare-feu | Désactiver `allowedall` avec `ruleset.set` avant d'ajouter l'IP |
| Général | `Cannot find drive 'C:'` | Terminal Linux | Utiliser des chemins `/home/holuser/` |
| Connexion | Échec de résolution du nom de l'hôte | DNS non disponible | Se connecter avec `10.0.0.22` |
| Ressources | `Move-VM` refusé vers un resource pool | Limite de la cmdlet sur certains hôtes autonomes | Utiliser `ExtensionData.MoveIntoResourcePool` ou recréer la VM avec `-ResourcePool` |
| Disque | `Set-HardDisk` refuse l'extension | La VM possède un snapshot | Supprimer les snapshots avant d'étendre le disque |
| Réseau | `Set-NicTeamingPolicy` échoue avec `-MakeNicActive` | La carte n'est pas un uplink du vSwitch | Vérifier les uplinks avec `Get-VirtualSwitch` puis ajouter la carte |
| Accès | `New-VMHostAccount` refuse le mot de passe | Politique de complexité de l'hôte | Utiliser majuscule, minuscule, chiffre, caractère spécial et longueur suffisante |
| Sauvegarde | `Get-VMHostFirmware` échoue | Dossier de destination inexistant | Créer le dossier avec `New-Item` avant l'appel |
| Journaux | `Get-Log` ne renvoie rien | Clé de journal invalide | Lister les clés avec `Get-LogType` |
| Supervision | `Get-Stat` ne renvoie aucun échantillon | VM éteinte ou intervalle non disponible | Allumer la VM et utiliser `-Realtime` sur un hôte autonome |

---

## 24. Bonnes pratiques et limites du lab

- Rendre les scripts idempotents (tests d'existence avant création) pour pouvoir les rejouer.
- Toujours vérifier le résultat d'une commande avec une cmdlet `Get-` correspondante.
- Ne pas stocker de mot de passe en clair hors d'un environnement jetable.
- Désactiver SSH et restreindre le pare-feu quand l'accès distant n'est pas nécessaire.
- Ne pas considérer un snapshot comme une sauvegarde et ne pas le conserver longtemps.
- Privilégier un identifiant de LUN précis plutôt qu'un filtre par taille avant de formater un disque.
- Limites propres à un ESXi autonome : pas de vMotion, pas de Storage vMotion, pas de Content Library exploitable directement, et inventaire réseau parfois incohérent.
- La session HOL est temporaire : tout ce qui est configuré disparaît à l'expiration. Exporter les rapports CSV pendant la session si nécessaire.

---

## 25. Glossaire

| Terme | Définition |
| :--- | :--- |
| ESXi | Hyperviseur VMware de type 1 |
| vCenter | Serveur de gestion centralisée de plusieurs hôtes ESXi |
| PowerCLI | Modules PowerShell pour piloter vSphere |
| vSwitch | Commutateur virtuel logiciel de l'hôte |
| Port group | Groupe de ports virtuels partageant une politique réseau |
| VLAN | Segment réseau logique de niveau 2 |
| VMkernel | Interface réseau de l'hôte pour la gestion et le stockage |
| NIC teaming | Regroupement de cartes physiques pour redondance et charge |
| LUN | Volume de stockage présenté par une baie |
| iSCSI | Protocole de stockage bloc sur IP |
| VMFS | Système de fichiers cluster de VMware |
| Datastore | Conteneur logique de stockage pour les VM |
| VMDK | Fichier de disque virtuel |
| Thin provisioning | Allocation d'espace à la demande |
| Snapshot | Capture de l'état d'une VM à un instant donné |
| Cold migration | Déplacement d'une VM éteinte |
| NTP | Protocole de synchronisation de l'heure |
| esxcli | Interface en ligne de commande avancée de l'hôte |
| CIDR | Notation d'un réseau IP avec longueur de masque |
| HBA | Adaptateur reliant l'hôte au stockage |
| IQN | Nom unique d'un initiateur ou d'une cible iSCSI |
| Hands-on Labs | Plateforme VMware de laboratoires temporaires en ligne |
| Shares | Poids relatif d'une VM lors d'une contention de ressources |
| Réservation | Quantité de ressource garantie à une VM |
| Limite | Plafond de consommation d'une VM ou d'un pool |
| Resource pool | Conteneur logique de ressources CPU et mémoire |
| Hot add | Ajout de CPU ou de mémoire à chaud |
| VMware Tools | Agent installé dans le système invité |
| Load balancing | Règle de répartition des flux entre uplinks |
| Failback | Retour automatique du trafic sur la carte d'origine |
| Multipathing | Présence de plusieurs chemins vers un même LUN |
| PSP | Path Selection Policy, politique de choix du chemin |
| CHAP | Authentification entre initiateur et cible iSCSI |
| RBAC | Contrôle d'accès fondé sur des rôles |
| Privilège | Action unitaire autorisée sur un objet |
| Permission | Association d'un utilisateur, d'un rôle et d'un objet |
| Advanced Settings | Paramètres de configuration fine de l'hôte |
| Syslog | Protocole d'envoi des journaux vers un serveur central |
| Lockdown mode | Mode qui restreint l'accès direct à l'hôte |
| Bundle de configuration | Archive de la configuration de l'hôte |
| OVF et OVA | Formats ouverts d'export de machines virtuelles |
| Transcript | Enregistrement d'une session PowerShell dans un fichier |
| Idempotence | Propriété d'un script rejouable sans effet de bord |
| USE | Méthode de diagnostic : Utilisation, Saturation, Erreurs |
| CPU Ready | Temps pendant lequel un vCPU attend un cœur physique |
| NUMA | Architecture où l'accès mémoire dépend du socket du processeur |
| VST, EST, VGT | Trois modes de marquage VLAN (virtuel, externe, invité) |
| SATP | Plugin qui gère le comportement de la baie pour le multipathing |
| APD | All Paths Down, perte de tous les chemins avec issue incertaine |
| PDL | Permanent Device Loss, perte définitive signalée d'un périphérique |
| VAAI | API qui délègue des opérations de stockage à la baie |
| RPO | Perte de données maximale acceptable, exprimée en temps |
| RTO | Durée maximale acceptable de remise en service |
| CBT | Changed Block Tracking, suivi des blocs modifiés |
| VADP | API de sauvegarde de VM de vSphere |
