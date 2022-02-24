+++
title = "Percona PMM2 - Test avec VirtualBox et Tunnels SSH"
description = "Une méthode pour tester PMM2 à l'aide de VirtualBox de votre ordinateur portable, des tunnels SSH sans installer des agents sur les serveurs de base de données Percona Monitoring and Management 2 "
author = "Francis"
date = 2022-02-22
tags = ['PMM']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article09.jpg"
images = ["thumbnail2022/article09.jpg"]
slug = "postgresql-sur-kubernetes-ha-haute-disponibilite-et-recuperation"
+++

![thumbnail](/thumbnail2022/article09.jpg)

[Percona Monitoring and Management 2 ](https://www.percona.com/software/database-tools/percona-monitoring-and-management)(PMM2) est la suite de surveillance de base de données assemblée et développée par Percona. Il est basé sur des composants open source standard et des intégrations logicielles sur mesure. PMM2 vous aide à réduire la complexité, à optimiser les performances et à améliorer la sécurité de vos environnements de bases de données stratégiques, quel que soit leur emplacement ou leur déploiement.

Cet article de blog décrira une méthode pour tester PMM2 à l'aide de la VirtualBox de votre ordinateur portable, des tunnels ssh et sans installer des agents sur les serveurs de base de données. Il s'agit d'un environnement de test et d'évaluation, non destiné à la production. Si vous souhaitez effectuer un test complet de PMM2, nous vous recommandons un environnement aussi similaire que possible à votre configuration de production finale: virtualisation d'entreprise, conteneurs Docker ou AWS. Nous supposons que votre ordinateur portable n'a pas de connectivité directe aux bases de données, c'est pourquoi nous utilisons des tunnels ssh.

L'architecture PMM2 se compose de 2+1 composants:

![image01](/posts/2022/article07/img01.png)

Deux composants s'exécutent sur votre infrastructure: les agents PMM et le serveur PMM. Les agents collectent les métriques au niveau de la base de données et du système d'exploitation. PMM Server s'occupe du traitement, du stockage, du regroupement et de l'affichage de ces métriques. Il peut également effectuer des opérations supplémentaires telles que la capture de métriques de bases de données sans serveur, des sauvegardes et l'envoi d'alertes (les deux dernières fonctionnalités sont en préversion technique au moment de la rédaction de cet article). L'autre composant pour compléter la formule est la plate-forme Percona, qui ajoute plus de fonctionnalités au PMM, des conseillers au DBaaS. Avis de non-responsabilité: la plate-forme Percona est en version préliminaire avec des fonctionnalités limitées; elle convient aux premiers utilisateurs, au développement et aux tests. Outre les fonctionnalités étendues ajoutées à PMM, la plateforme Percona rassemble les distributions de MySQL, PostgreSQL et MongoDB , y compris une gamme d'outils open source pour la sauvegarde, la disponibilité et la gestion des données. Vous pouvez en savoir plus sur la plateforme [ici ](https://www.percona.com/software/percona-platform).

Pour faciliter la configuration, PMM2 Server peut être exécuté soit comme un conteneur docker , soit en important une image OVA, exécutée à l'aide de VMWare VSphere , VirtualBox , ou tout autre hyperviseur. Si vous exécutez votre infrastructure dans AWS, vous pouvez déployer PMM à partir d' [AWS Marketplace ](https://aws.amazon.com/marketplace/pp/prodview-uww55ejutsnom).

Pour exécuter les agents, vous avez besoin d'une machine Linux. Nous vous recommandons d'exécuter les agents et la base de données sur le même nœud. PMM peut également collecter les métriques à l'aide d'une connexion directe à une base de données sans serveur ou en exécutant un système d'exploitation qui ne prend pas en charge l'agent.

Souvent, l'installation des agents est un frein pour certains DBA qui souhaitent tester PMM2. De plus, alors que les conteneurs sont fréquents dans les grandes organisations, nous trouvons la virtualisation et les conteneurs relégués aux environnements de développement et d'assurance qualité. Ces environnements n'ont généralement pas d'accès direct aux bases de données de production.

## Transfert TCP sur les connexions SSH

AllowTcpForwarding est l'option de configuration du démon ssh qui permet de transférer les ports TCP via la connexion ssh. À première vue, cela peut sembler un risque pour la sécurité, mais comme l'indique la documentation ssh : "la désactivation du transfert TCP n'améliore pas la sécurité à moins que les utilisateurs ne se voient également refuser l'accès au shell, car ils peuvent toujours installer leurs redirecteurs".

Si vos administrateurs système n'autorisent pas le transfert TCP, les autres options disponibles pour obtenir les mêmes résultats sont socat ou netcat . Mais nous ne les aborderons pas ici.

Si votre ordinateur portable a un accès direct aux bases de données, vous pouvez ignorer tous les tunnels ssh et utiliser la méthode d'accès direct décrite plus loin dans cet article.

## Installer PMM 2 Ova

Téléchargez l'image compatible Open Virtualization Format sur [https://www.percona.com/downloads/pmm2/ ](https://www.percona.com/downloads/pmm2/)ou utilisez la ligne de commande :

```
$ wget https://www.percona.com/downloads/pmm2/2.25.0/ova/pmm-server-2.25.0.ova
```

Vous pouvez importer le fichier OVA à l'aide de l'interface utilisateur, avec l'option d'importation du menu Fichier ou à l'aide de la ligne de commande :

```
$ VBoxManage import pmm-server-2.25.0.ova --vsys 0 --vmname "PMM Testing"
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Interpreting /Users/pep/Downloads/pmm-server-2.25.0.ova...
OK.
Disks:
vmdisk1 42949672960 -1 http://www.vmware.com/interfaces/specifications/vmdk.html#streamOptimized PMM2-Server-2021-12-13-1012-disk001.vmdk -1 -1
vmdisk2 429496729600 -1 http://www.vmware.com/interfaces/specifications/vmdk.html#streamOptimized PMM2-Server-2021-12-13-1012-disk002.vmdk -1 -1

Virtual system 0:
0: Suggested OS type: "RedHat_64"
(change with "--vsys 0 --ostype "; use "list ostypes" to list all possible values)
1: VM name specified with --vmname: "PMM Testing"
2: Suggested VM group "/"
(change with "--vsys 0 --group ")
3: Suggested VM settings file name "/Users/Pep/VirtualBox VMs/PMM2-Server-2021-12-13-1012/PMM2-Server-2021-12-13-1012.vbox"
(change with "--vsys 0 --settingsfile ")
4: Suggested VM base folder "/Users/Pep/VirtualBox VMs"
(change with "--vsys 0 --basefolder ")
5: Product (ignored): Percona Monitoring and Management
6: Vendor (ignored): Percona
7: Version (ignored): 2021-12-13
8: ProductUrl (ignored): https://www.percona.com/software/database-tools/percona-monitoring-and-management
9: VendorUrl (ignored): https://www.percona.com
10: Description "Percona Monitoring and Management (PMM) is an open-source platform for managing and monitoring MySQL and MongoDB performance"
(change with "--vsys 0 --description ")
11: Number of CPUs: 1
(change with "--vsys 0 --cpus ")
12: Guest memory: 4096 MB
(change with "--vsys 0 --memory ")
13: Network adapter: orig NAT, config 3, extra slot=0;type=NAT
14: SCSI controller, type LsiLogic
(change with "--vsys 0 --unit 14 --scsitype {BusLogic|LsiLogic}";
disable with "--vsys 0 --unit 14 --ignore")
15: Hard disk image: source image=PMM2-Server-2021-12-13-1012-disk001.vmdk, target path=PMM2-Server-2021-12-13-1012-disk001.vmdk, controller=14;channel=0
(change target path with "--vsys 0 --unit 15 --disk path";
disable with "--vsys 0 --unit 15 --ignore")
16: Hard disk image: source image=PMM2-Server-2021-12-13-1012-disk002.vmdk, target path=PMM2-Server-2021-12-13-1012-disk002.vmdk, controller=14;channel=1
(change target path with "--vsys 0 --unit 16 --disk path";
disable with "--vsys 0 --unit 16 --ignore")
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Successfully imported the appliance.
```


Une fois la machine importée, nous la connecterons à un réseau hôte uniquement. Ce réseau limite le trafic réseau uniquement entre l'hôte et les machines virtuelles. Mais d'abord, trouvons un réseau approprié:

```
$ VBoxManage list hostonlyifs
Name: vboxnet0
GUID: 786f6276-656e-4074-8000-0a0027000000
DHCP: Disabled
IPAddress: 192.168.56.1
NetworkMask: 255.255.255.0
IPV6Address:
IPV6NetworkMaskPrefixLength: 0
HardwareAddress: 0a:00:27:00:00:00
MediumType: Ethernet
Wireless: No
Status: <strong>Up</strong>
VBoxNetworkName: HostInterfaceNetworking-vboxnet0
```

Sélectionnez le premier dont l'état est activé, notez le nom et l'adresse IP. Assurez-vous ensuite qu'un serveur DHCP est affecté à cette interface :

```
$ VBoxManage list dhcpservers
NetworkName: HostInterfaceNetworking-vboxnet0
Dhcpd IP: 192.168.56.100
LowerIPAddress: 192.168.56.101
UpperIPAddress: 192.168.56.254
NetworkMask: 255.255.255.0
Enabled: <strong>Yes</strong>
Global Configuration:
minLeaseTime: default
defaultLeaseTime: default
maxLeaseTime: default
Forced options: None
Suppressed opts.: None
1/legacy: 255.255.255.0
Groups: None
Individual Configs: None

```

Nous allons maintenant attribuer deux interfaces réseau à notre machine virtuelle PMM. L'un est attribué au réseau interne et l'autre utilise NAT pour se connecter à Internet et, par exemple, vérifier les mises à niveau.

```
$ VBoxManage modifyvm "PMM Testing" --nic1 hostonly --hostonlyadapter1 vboxnet0
$ VBoxManage modifyvm "PMM Testing" --nic2 natnetwork
```

Une fois la mise en réseau configurée, nous pouvons démarrer la machine virtuelle.

```
$ VBoxManage startvm "PMM Testing"
```

L'étape suivante consiste à récupérer l'adresse IP attribuée à notre box PMM. Tout d'abord, nous allons obtenir l'adresse MAC de la carte réseau que nous avons récemment ajoutée :

```
$ VBoxManage showvminfo "PMM Testing" | grep -i vboxnet0
NIC 1: MAC: <strong>08002772600D</strong>, Attachment: Host-only Interface 'vboxnet0', Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
```

En utilisant l'adresse MAC récupérée, nous pouvons rechercher les baux DHCP :

```
$ VBoxManage dhcpserver findlease --interface=vboxnet0 --mac-address=08002772600D
IP Address: <strong>192.168.56.112</strong>
MAC Address: 08:00:27:72:60:0d
State: acked
Issued: 2021-12-21T22:15:54Z (1640124954)
Expire: 2021-12-21T22:25:54Z (1640125554)
TTL: 600 sec, currently 444 sec left
```

Il s'agit de l'adresse IP que nous utiliserons pour accéder au serveur PMM. Ouvrez un navigateur pour vous connecter à [**https://192.168.56.112** ](https://192.168.56.112/) avec les identifiants par défaut : admin/admin.

![image02](/posts/2022/article09/img02.jpg)

L'étape suivante configure les tunnels pour se connecter aux bases de données que nous surveillons.

## Configurer les tunnels SSH

Voici la topologie de notre réseau :

![image03](/posts/2022/article09/img03.png)

Nous allons ouvrir deux connexions ssh par serveur auquel nous voulons accéder depuis PMM. Ouvrez une session de terminal et exécutez la commande suivante, en la remplaçant par le nom d'utilisateur que vous utilisez normalement pour vous connecter à votre hôte de saut :

```
$ ssh -L 192.168.56.1:3306:10.0.0.1:3306 @192.168.55.100
```

Cela crée un tunnel qui connecte le port 3306 du serveur MySQL à notre adresse interne locale dans le même port. Si vous souhaitez vous connecter à plusieurs instances MySQL , vous devez utiliser des ports différents. Pour ouvrir le tunnel pour le serveur MongoDB , utilisez la commande suivante :

```
$ ssh -L 192.168.56.1:27017:10.0.0.2:27017 @192.168.55.100
```

Testez la connectivité du tunnel à l' hôte MySQL en utilisant netcat :

```
$ nc -z 192.168.56.1 3306
Connection to 192.168.56.1 port 3306 [tcp/mysql] succeeded!
```

Et testez également la connectivité à l'hôte MongoDB :

```
$ nc -z 192.168.56.1 27017
Connection to 192.168.56.1 port 27017 [tcp/*] succeeded!
```

Il s'agit de la topologie de notre réseau, y compris les tunnels ssh .

![image04](/posts/2022/article09/img04.png)

## Configurer les comptes

Suivez la documentation PMM et créez un compte MySQL (ou utilisez un compte déjà existant) avec les privilèges requis :

```
CREATE USER 'pmm'@'10.0.0.100' IDENTIFIED BY '' WITH MAX_USER_CONNECTIONS 10;
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BACKUP_ADMIN ON *.* TO 'pmm'@'10.0.0.100';
```

Notez que nous devons utiliser l'adresse IP interne pour l'hôte de saut. Si vous ne connaissez pas l'adresse IP, utilisez le caractère générique '%'.

Ajoutez également les informations d'identification pour MongoDB , exécutez ceci dans une session Mongo :

```
db.getSiblingDB("admin").createRole({
role: "explainRole",
privileges: [{
resource: {
db: "",
collection: ""
},
actions: [
"listIndexes",
"listCollections",
"dbStats",
"dbHash",
"collStats",
"find"
]
}],
roles:[]
})

db.getSiblingDB("admin").createUser({
user: "pmm_mongodb",
pwd: "",
roles: [
{ role: "explainRole", db: "admin" },
{ role: "clusterMonitor", db: "admin" },
{ role: "read", db: "local" }
]
})
```

## Ajouter les services à PMM

![image05](/posts/2022/article09/img05.jpg)


Nous ne pouvons pas installer les agents car nous n'avons pas accès à notre environnement de test PMM à partir des serveurs de base de données. Au lieu de cela, nous allons configurer les deux services en tant qu'instances distantes. Allez dans le menu "Configuration" ![image06](/posts/2022/article09/img-06-small.png), sélectionnez "Inventaire PMM" ![image07](/posts/2022/article09/img-07-small.jpeg) , puis "Ajouter une instance" ![image08](/posts/2022/article09/img-08-small.jpeg) . Choisissez ensuite MySQL Ajouter une instance distante.

Complétez les champs suivants : 
```
![image09](/posts/2022/article09/img09.jpg)
```

Nom d'hôte : 192.168.56.1 (Il s'agit de l'adresse interne Host-Only VirtualBox ) 

Nom du service : MySQL8Port : 3306Username : pmm 

Password : <password>

Et appuyez sur le bouton. Il vérifiera la connectivité et, si tout est correct, le service MySQL sera ajouté à l'inventaire. S'il y a une erreur, vérifiez que la connexion ssh est toujours ouverte et que vous avez entré les informations d'identification correctes. Assurez-vous que l'hôte que vous avez spécifié pour créer l’utilisateur MySQL est correct.

Nous utiliserons un processus similaire pour MongoDB :

![image10](/posts/2022/article09/img10.jpg)

Voici les champs que vous devez remplir avec les informations correctes : Nom d'hôte : 192.168.56.1 (Encore une fois, l'adresse interne de VirtualBox Host-Only ) 

Nom du service : Port MongoDB

` `: 27017 Nom d'utilisateur : pmm\_mongodb 

Mot de passe : <mot de passe>

Et appuyez sur le bouton. Il vérifiera la connectivité et, si tout est correct, le service MongoDB sera ajouté à l'inventaire. S'il y a une erreur, revérifiez à nouveau que la connexion ssh est ouverte et que vous avez saisi les informations d'identification correctes. Vous pouvez également utiliser l'application cliente MongoDB pour vérifier l'accès.

Une fois que vous avez ajouté les deux services, il vous suffit d'attendre quelques minutes pour avoir le temps de collecter des données et de commencer à tester PMM2.

![image11](/posts/2022/article09/img11.jpg)

Source : [Percona](https://www.percona.com/blog/percona-monitoring-and-management-2-test-drive-using-virtualbox-and-ssh-tunnels/)
