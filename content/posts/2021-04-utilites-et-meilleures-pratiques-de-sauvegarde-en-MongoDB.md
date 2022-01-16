+++
title = "Pourquoi faire des sauvegardes?"
description = "Pratique et Conseil pour avoir une sauvegarde de base de donnée avec MongoDB"
author = "Francis"
date = 2021-06-01T11:43:01+04:00
tags = ['MongoDB']
Categories = ["Article de Percona"]
featured_image = ""
images = ["thumbnail/amisdepercona21-004.jpg"]
slug = "les-meilleures-pratiques-de-sauvegarde-MongoDB"
+++


Les sauvegardes régulières de la base de données sont essentielles pour se prémunir contre les événements de perte de données involontaires. Ils sont encore plus importants pour la restauration et la poursuite des opérations commerciales.

Dans ce blog, nous discuterons de différentes stratégies de sauvegarde et de leurs cas d&#39;utilisation, ainsi que des avantages et des inconvénients et de quelques autres conseils.

En règle générale, il existe deux types de sauvegardes utilisées avec des technologies de bases de données telles que MongoDB :

- **Sauvegardes logiques**
- **Sauvegardes physiques**

De plus, lorsque vous travaillez avec des sauvegardes logiques, nous avons également la possibilité d&#39;effectuer des sauvegardes incrémentielles où nous capturons les deltas ou les modifications de données incrémentielles effectuées entre les sauvegardes complètes pour minimiser la quantité de perte de données en cas de sinistre.

Nous discuterons de ces deux options de sauvegarde, de la manière de les utiliser et de celle qui convient le mieux en fonction des exigences et de la configuration de l&#39;environnement.

Nous examinerons également notre utilitaire de sauvegarde open source conçu sur mesure pour éviter les coûts et les logiciels propriétaires - **[Percona BackUp pour MongoDB](https://www.percona.com/software/mongodb/percona-backup-for-mongodb)** ou PBM. PBM est un outil de sauvegarde de communauté entièrement pris en charge, capable d&#39;effectuer des sauvegardes cohérentes côté cluster dans MongoDB pour les jeux de réplicas et les clusters partitionnés.

Sauvegardes logiques

Ce sont les types de sauvegardes où les données sont vidées des bases de données dans les fichiers de sauvegarde. Une sauvegarde logique avec MongoDB signifie que vous allez vider les données dans un fichier au format BSON.

Pendant les sauvegardes logiques, à l&#39;aide de l&#39;API client, les données sont lues depuis le serveur et renvoyées vers la même API qui sera sérialisée et écrite dans respectivement ». bson »,«. json »ou«. csv » sauvegarde des fichiers sur le disque en fonction du type d&#39;utilitaires de sauvegarde utilisé.

MongoDB propose ci-dessous l&#39;utilitaire pour effectuer les sauvegardes logiques :

**Mongodump** : prend le vidage / sauvegarde des bases de données dans &quot;. bson &quot;qui peut être restauré ultérieurement en rejouant les mêmes instructions logiques capturées dans les fichiers de vidage dans les bases de données.

```
mongodump --host=mongodb1.example.net --port=27017 --username=user --authenticationDatabase=admin --db=demo --collection=events --out=/opt/backup/mongodump-2011-10-24
```

**Remarque** : Si nous ne spécifions pas explicitement le nom de la base de données ou le nom de la collection dans la syntaxe « mongodump » ci-dessus, la sauvegarde sera effectuée pour la base de données entière ou les collections respectivement. Si «autorisation» est activée, nous devons spécifier « authenticationDatabase »

De plus, vous devez utiliser «- oplog » pour prendre les données incrémentielles pendant que la sauvegarde est toujours en cours, nous pouvons spécifier «- oplog » avec mongodump . Gardez à l&#39;esprit que cela ne fonctionnera pas avec –db et –collection car il ne fonctionnera que pour des sauvegardes de bases de données entières

``` 
mongodump --host=mongodb1.example.net --port=27017 --username=user --authenticationDatabase=admin --oplog --out=/opt/backup/mongodump-2011-10-24
```

**Avantages:**

1. Il peut prendre la sauvegarde à un niveau plus granulaire comme une base de données spécifique ou une collection qui sera utile lors de la restauration.
2. Ne vous oblige pas à interrompre les écritures sur un nœud spécifique sur lequel vous exécuterez la sauvegarde. Par conséquent, le nœud serait toujours disponible pour d&#39;autres opérations.

**Inconvénients** :

1. Comme il lit toutes les données, il peut être lent et nécessitera également des lectures de disque pour les bases de données plus grandes que la RAM disponible pour le cache WT. La pression du cache WT augmente, ce qui ralentit les performances.
2. Il ne capture pas les données d&#39;index dans le fichier de sauvegarde des métadonnées. Ainsi, lors de la restauration, tous les index doivent être construits à nouveau pour chaque collection après la réinsertion de la collection. Cela sera fait en série en un seul passage dans la collection une fois les inserts terminés, ce qui peut ajouter beaucoup de temps pour les restaurations de grandes collections.
3. La vitesse de sauvegarde dépend également des IOPS allouées et du type de stockage, car de nombreuses lectures / écritures se produiraient au cours de ce processus.
4. Les sauvegardes logiques telles que mongodump prennent en général beaucoup de temps pour les grands systèmes.

**Conseil de bonne pratique** : Il est toujours conseillé d&#39;utiliser des serveurs secondaires pour les sauvegardes afin d&#39;éviter une dégradation inutile des performances sur le nœud PRIMARY.

Comme nous avons différents types de configurations d&#39;environnement, nous devrions aborder chacune d&#39;elles comme ci-dessous.

1. **Jeu de réplicas** : il est toujours préférable de s&#39;exécuter sur des secondaires .
2. **Clusters de fragments** : effectuez une sauvegarde des réplicas du serveur de configuration et de chaque partition individuellement à l&#39;aide de leurs nœuds secondaires.

Puisque nous discutons de systèmes de bases de données distribuées comme des clusters fragmentés, nous devons également garder à l&#39;esprit que nous voulons avoir une cohérence dans nos sauvegardes à un moment donné. (Les sauvegardes des jeux de réplicas utilisant mongodump sont généralement cohérentes avec «- oplog »)

Parlons de ce scénario dans lequel l&#39;application écrit toujours des données et ne peut pas être arrêtée pour des raisons commerciales. Même si nous prenons une sauvegarde du serveur de configuration et de chaque partition séparément, les sauvegardes de chaque partition se termineront à des moments différents en raison du volume de données, de la distribution des données, de la charge, etc.

Vient maintenant la partie restauration en ce qui concerne les sauvegardes logiques. Comme pour les sauvegardes, MongoDB fournit les utilitaires ci-dessous à des fins de restauration.

**Mongorestore** : Restaure les fichiers de vidage créés par « mongodump ». La recréation d&#39;index n&#39;aura lieu qu&#39;après la restauration des données, ce qui entraîne l&#39;utilisation de ressources mémoire et de temps supplémentaires.

```mongorestore --host=mongodb1.example.net --port=27017 --username=user  --password --authenticationDatabase=admin --db=demo --collection=events /opt/backup/mongodump-2011-10-24/events.bson```

Pour la restauration du vidage incrémentiel, nous pouvons ajouter - oplogReplay dans la syntaxe ci-dessus pour rejouer également les entrées oplog .

**Conseil de** bonne **pratique** : Le «- oplogReplay » ne peut pas être utilisé avec les indicateurs –db et –collection car il ne fonctionnera que lors de la restauration de toutes les bases de données.

**Percona Backup pour MongoDB**

Il s&#39;agit d&#39;une solution distribuée à faible impact pour réaliser des sauvegardes cohérentes des clusters fragmentés MongoDB et des jeux de réplicas. Percona Backup for MongoDB permet de surmonter les problèmes de cohérence tout en effectuant des sauvegardes de clusters partitionnés. Percona Backup for MongoDB est un outil de ligne de commande simple de par sa conception qui se prête bien à la sauvegarde d&#39;ensembles de données plus volumineux. PBM utilise la bibliothèque «s2» plus rapide et les threads parallélisés pour améliorer la vitesse et les performances si des threads supplémentaires sont disponibles en tant que ressources.

**Les principaux avantages du PBM sont les suivants:** 

- Permet des sauvegardes avec jeu de réplicas et offre une cohérence fragmentées du cluster via oplog capture
- Fournit la cohérence des transactions distribuées avec MongoDB 4.2+
- Sauvegarde n&#39;importe où - dans le cloud (utilisation de n&#39;importe quel stockage compatible S3) ou sur site avec un système de fichiers distant monté localement
- Vous permet de choisir les algorithmes de compression à utiliser. Dans certaines expériences internes, la bibliothèque «s2» avec compression rapide exécutée en parallèle avec plusieurs threads était significativement plus rapide que gzip classique . Mise en garde: C&#39;est bon tant que vous disposez des ressources supplémentaires disponibles pour exécuter les threads parallèles.
- **Enregistre la journalisation de la progression de la sauvegarde.** Si vous souhaitez voir la vitesse de la sauvegarde (taux de téléchargement en Mo/s), vous pouvez consulter les journaux du nœud pbm -agent pour voir la progression actuelle. Si vous avez une sauvegarde volumineuse, vous pouvez suivre la progression de la sauvegarde dans les journaux de pbm -agent. Une ligne est ajoutée toutes les minutes indiquant les octets copiés par rapport à la taille totale de la collection actuelle.
- PBM permet des récupérations ponctuelles - restauration d&#39;une base de données jusqu&#39;à un moment précis. PIT-R restaure les données à partir d&#39;une sauvegarde, puis rejoue toutes les actions qui sont arrivées aux données jusqu&#39;au moment spécifié à partir des tranches oplog .
- Les PITR vous aident à éviter la perte de données lors d&#39;un sinistre tel qu&#39;une base de données en panne, la suppression accidentelle de données ou la suppression de tables et la mise à jour indésirable de plusieurs champs au lieu d&#39;un seul.
- PBM est optimisé pour permettre des sauvegardes et avoir un impact minimal sur vos performances de production.

**Conseil de bonne pratique** : utilisez PBM pour chronométrer d&#39;énormes jeux de sauvegarde. Beaucoup de gens ne réalisent pas combien de temps il faut pour sauvegarder de très grands ensembles de données. Et ils sont généralement très surpris du temps qu&#39;il faut pour les restaurer! Surtout si vous entrez ou sortez de types de stockage qui peuvent limiter la bande passante / le trafic réseau.

**Conseil de bonne pratique** : lors de l&#39;exécution de PBM à partir d&#39;un script non supervisé, nous vous recommandons d&#39;utiliser une chaîne de connexion de jeu de réplicas. Une chaîne de connexion directe ou autonome échouera si cet hôte mongod est indisponible ou temporairement indisponible.

Lorsqu&#39;une sauvegarde PBM est déclenchée, elle queues et capture les oplog de config jeu de réplicas du serveur et tous les tessons pendant que la sauvegarde est toujours en cours d&#39;exécution, assurant ainsi la cohérence une fois la sauvegarde terminée.

Il a une fonction de prendre des sauvegardes incrémentielles ainsi que la sauvegarde complète de la base de données avec le paramètre «PITR» activé. Il fait tout cela en exécutant « pbm -agent» sur les nœuds de base de données (« mongod ») du cluster et il est responsable des sauvegardes et des restaurations.

Comme nous pouvons le voir ci-dessous, la commande « pbm list» affiche la liste complète des sauvegardes dans la section Sauvegarde des instantanés ainsi que les sauvegardes incrémentielles dans la section «PITR».

Voici l&#39;exemple de sortie Output:

```
$ pbm list
Backup snapshots:
     2020-09-10T12:19:10Z
     2020-09-14T10:44:44Z
     2020-09-14T14:26:20Z
     2020-09-17T16:46:59Z
PITR <on>:
     2020-09-14T14:26:40 - 2020-09-16T17:27:26
     2020-09-17T16:47:20 - 2020-09-17T16:57:55
```

Si vous avez une sauvegarde volumineuse, vous pouvez suivre la progression de la sauvegarde dans les journaux de pbm-agent. Jetons également un œil à la sortie Output de « pbm-agent» pendant qu&#39;il effectue la sauvegarde.

```
Aug 19 08:46:51 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 08:46:51 Got command backup [{backup {2020-08-19T08:46:50Z s2} { } { 0} 1597826810}]
Aug 19 08:47:07 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 08:47:07 [INFO] backup/2020-08-19T08:46:50Z: backup started
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.891+0000        writing admin.system.users to archive on stdout
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.895+0000        done dumping admin.system.users (2 documents)
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.895+0000        writing admin.system.roles to archive on stdout
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.904+0000        done dumping admin.system.roles (1 document)
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.904+0000        writing admin.system.version to archive on stdout
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.914+0000        done dumping admin.system.version (5 documents)
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.914+0000        writing testmongo.col to archive on stdout
Aug 19 08:47:09 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:09.942+0000        writing test.collC to archive on stdout
Aug 19 08:47:13 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:13.499+0000        done dumping test.collC (1146923 documents)
Aug 19 08:47:13 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:13.500+0000        writing test.collA to archive on stdout
Aug 19 08:47:27 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:27.964+0000        done dumping test.collA (389616 documents)
Aug 19 08:47:27 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:27.965+0000        writing test.collG to archive on stdout
Aug 19 08:47:54 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:54.891+0000        done dumping testmongo.col (13280501 documents)
Aug 19 08:47:54 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T08:47:54.896+0000        writing test.collF to archive on stdout
Aug 19 08:48:09 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 08:48:09 [........................]  test.collG    1533/195563   (0.8%)
Aug 19 08:48:09 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 08:48:09 [####################....]  test.collF  116432/134747  (86.4%)
Aug 19 10:01:09 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:01:09 [#######################.]  test.collG  195209/195563  (99.8%)
Aug 19 10:01:17 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:01:17 [########################]  test.collG  195563/195563  (100.0%)
Aug 19 10:01:17 ip-172-30-2-122 pbm-agent[24331]: 2020-08-19T10:01:17.650+0000        done dumping test.collG (195563 documents)
Aug 19 10:01:20 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:01:20 [INFO] backup/2020-08-19T08:46:50Z: mongodump finished, waiting for the oplog
Aug 19 10:11:04 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:11:04 [INFO] backup/2020-08-19T08:46:50Z: backup finished
Aug 19 10:11:05 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:11:05 [INFO] pitr: streaming started from 2020-08-19 08:47:09 +0000 UTC / 1597826829
Aug 19 10:29:37 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:29:37 [INFO] pitr: created chunk 2020-08-19T08:47:09 - 2020-08-19T10:20:59. Next chunk creation scheduled to begin at ~2020-08-19T10:31:05
Aug 19 10:39:34 ip-172-30-2-122 pbm-agent[24331]: 2020/08/19 10:39:34 [INFO] pitr: created chunk 2020-08-19T10:20:59 - 2020-08-19T10:30:59. Next chunk creation scheduled to begin at ~2020-08-19T10:41:05 
```

Les trois dernières lignes de la sortie Output ci-dessus signifient que la sauvegarde complète est terminée et que la sauvegarde incrémentielle est démarrée avec un intervalle de veille de 10 minutes. Ceci est un exemple de la journalisation de la progression de la sauvegarde mentionnée ci-dessus.

Nous parlerons davanta de **Percona Backup pour MongoDB** dans un prochain article de blog. D&#39;ici là, vous pouvez trouver plus de détails sur la page de documentation [Percona Backup pour MongoDB](https://www.percona.com/doc/percona-backup-mongodb/index.html) sur notre site Web.

**Sauvegardes physiques / système de fichiers**

Cela implique de créer un instantané ou de copier les fichiers de données MongoDB sous-jacents ( -dbPath ) à un moment donné, et de permettre à la base de données de se restaurer proprement en utilisant l&#39;état capturé dans les fichiers instantanés . Ils contribuent à sauvegarder rapidement des bases de données volumineuses, en particulier lorsqu&#39;ils sont utilisés avec des instantanés de système de fichiers, tels que des [instantanés LVM](http://tldp.org/HOWTO/LVM-HOWTO/snapshots_backup.html) , ou des instantanés de volume de stockage en bloc.

Il existe plusieurs méthodes générales pour effectuer la sauvegarde au niveau du système de fichiers, également appelées sauvegardes physiques.

1. Copie manuelle de l&#39;intégralité des fichiers de données (à l&#39;aide de Rsync → Dépend de la bande passante N / W)
2. Instantanés basés sur LVM
3. Instantanés de disque basés sur le cloud (AWS / GCP / Azure ou tout autre fournisseur de cloud)
4. **Percona Server for MongoDB** comprend également un système de **sauvegarde à chaud** open source intégré qui crée une sauvegarde de données physiques sur un serveur en cours d&#39;exécution sans performances notables ni dégradation de fonctionnement. Vous pouvez trouver plus d&#39;informations sur **Percona Server for MongoDB Hot Backup** [ici](https://www.percona.com/doc/percona-server-for-mongodb/LATEST/hot-backup.html) .

Nous discuterons de toutes ces options ci-dessus, mais tout d&#39;abord, examinons les avantages et les inconvénients des sauvegardes physiques par rapport aux sauvegardes logiques.

**Avantages** : ** **

1. Elles sont au moins aussi rapides et généralement plus rapides que les sauvegardes logiques.
2. Peut être facilement copié ou partagé avec des serveurs distants ou un NAS connecté.
3. Recommandé pour les grands ensembles de données en raison de la vitesse et de la fiabilité
4. Peut être pratique lors de la création de nouveaux nœuds au sein du même cluster ou d&#39;un nouveau cluster

**Inconvénients** : ** **

1. Il n&#39;est pas possible de restaurer à un niveau moins granulaire, tel qu&#39;une restauration de base de données ou de collection spécifique
2. Les sauvegardes incrémentielles ne peuvent pas encore être réalisées
3. Un nœud dédié est recommandé pour la sauvegarde (il peut s&#39;agir d&#39;un nœud caché) car il nécessite l&#39;arrêt des écritures ou l&#39;arrêt proprement de « mongod » avant le cliché sur le nœud pour assurer la cohérence.

Vous trouverez ci-dessous la comparaison de la consommation de temps de sauvegarde pour le même jeu de données:

**Taille de la base de données: 267,6 Go**

**Taille de l&#39;index: \&lt;1 Mo** (car il était uniquement sur \_id pour les tests)

```
demo:PRIMARY> db.runCommand({dbStats: 1, scale: 1024*1024*1024})
{
        "db" : "test",
        "collections" : 1,
        "views" : 0,
        "objects" : 137029,
        "avgObjSize" : 2097192,
        "dataSize" : 267.6398703530431,
        "storageSize" : 13.073314666748047,
        "numExtents" : 0,
        "indexes" : 1,
        "indexSize" : 0.0011749267578125,
        "scaleFactor" : 1073741824,
        "fsUsedSize" : 16.939781188964844,
        "fsTotalSize" : 49.98826217651367,
        "ok" : 1,
        ...
}
demo:PRIMARY>
```

==============================

1. **Serveur Percona pour la sauvegarde à chaud de MongoDB** :

Syntaxe :

```
> use admin
switched to db admin
> db.runCommand({createBackup: 1, backupDir: "/my/backup/data/path"})
{ "ok" : 1 }
```

**Conseil de bonne pratique** : Le chemin de sauvegarde « backupDir » doit être absolu. Il prend également en charge le stockage des sauvegardes sur le système de fichiers et les compartiments AWS S3.

```
[root@ip-172-31-37-92 tmp]# time mongo  < hot.js
Percona Server for MongoDB shell version v4.2.8-8
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("c9860482-7bae-4aae-b0e7-5d61f8547559") }
Percona Server for MongoDB server version: v4.2.8-8
switched to db admin
{
        "ok" : 1,
        ...
}
bye
real    3m51.773s
user    0m0.067s
sys     0m0.026s
[root@ip-172-31-37-92 tmp]# ls
hot  hot.js  mongodb-27017.sock  nohup.out  systemd-private-b8f44077314a49899d0a31f99b31ed7a-chronyd.service-Qh7dpD  tmux-0
[root@ip-172-31-37-92 tmp]# du -sch hot
15G     hot
15G     total
```

Notez que le temps pris par « Percona Hot Backup» n&#39;était que de 4 minutes environ.

Ceci est très utile lors de la reconstruction d&#39;un nœud ou de la création de nouvelles instances / clusters avec le même jeu de données. La meilleure partie est qu&#39;il ne compromet pas les performances avec le verrouillage des écritures ou d&#39;autres performances.

**Conseil de bonne pratique** : il est recommandé de l&#39;exécuter contre les secondaires.

**2**.**Instantané du système de fichiers** :

Le temps approximatif nécessaire à la réalisation de l&#39;instantané n&#39;était que de 4 minutes.

```
[root@ip-172-31-37-92 ~]# aws ec2 describe-snapshots  --query "sort_by(Snapshots, &StartTime)[-1].{SnapshotId:SnapshotId,StartTime:StartTime}"
{
    "SnapshotId": "snap-0f4403bc0fa0f2e9c",
    "StartTime": "2020-08-26T12:26:32.783Z"
}
```

```
[root@ip-172-31-37-92 ~]# aws ec2 describe-snapshots \
> --snapshot-ids snap-0f4403bc0fa0f2e9c
{
    "Snapshots": [
        {
            "Description": "This is my snapshot backup",
            "Encrypted": false,
            "OwnerId": "021086068589",
            "Progress": "100%",
            "SnapshotId": "snap-0f4403bc0fa0f2e9c",
            "StartTime": "2020-08-26T12:26:32.783Z",
            "State": "completed",
            "VolumeId": "vol-0def857c44080a556",
            "VolumeSize": 50
        }
    ]
}
```



**3**. **Mongodump :**

```
[root@ip-172-31-37-92 ~]# time nohup mongodump -d test -c collG -o /mongodump/ &
[1] 44298
[root@ip-172-31-37-92 ~]# sed -n '1p;$p' nohup.out
2020-08-26T12:36:20.842+0000    writing test.collG to /mongodump/test/collG.bson
2020-08-26T12:51:08.832+0000    [####....................]  test.collG  27353/137029  (20.0%) 
```

**Résultats** : comme vous pouvez le voir dans cet exemple rapide utilisant le même ensemble de données - les méthodes de capture instantanée au niveau du système de fichiers et de Percona Server pour MongoDB Hot Backup n&#39;ont pris que 3 à 5 minutes. Cependant, « mongodump » a pris près de 15 minutes pour seulement 20% de la décharge. Par conséquent, la vitesse de sauvegarde des données avec mongodump est certainement très lente par rapport aux deux autres options discutées. C&#39;est là que la compression s2 et les threads parallélisés de Percona Backup pour MongoDB peuvent aider.

**Conclusion**

La meilleure méthode pour effectuer les sauvegardes dépend de plusieurs facteurs tels que le type d&#39;infrastructure, l&#39;environnement, les ressources disponibles, la taille du jeu de données, la charge, etc. Cependant, la cohérence et la complexité jouent également un rôle majeur lors de la sauvegarde des systèmes de base de données distribués.

En général, pour les petites instances, de simples sauvegardes logiques via mongodump conviennent. Lorsque vous atteignez des tailles de base de données un peu plus grandes au-dessus d&#39;environ 100G, utilisez des méthodes de sauvegarde telles que **[Percona Backup pour MongoDB](https://www.percona.com/software/mongodb/percona-backup-for-mongodb)** qui incluent des sauvegardes incrémentielles et capturent les oplogs afin de pouvoir effectuer des récupérations ponctuelles et minimiser les pertes de données potentielles.

PBM vous permet de sauvegarder partout - dans le nuage ou on-prem, il peut gérer vos sauvegardes plus grandes, et il est optimisé pour avoir un impact minimal sur les performances de votre production. Le PBM est également plus rapide grâce à l&#39;utilisation de la méthode de compression «s2» et à l&#39;utilisation de threads parallélisés. Enfin, PBM peut surmonter les problèmes de cohérence souvent rencontrés avec le jeu de réplicas et les clusters partitionnés en capturant les modifications dans le journal d&#39;opération.

Pour les systèmes très volumineux, c&#39;est-à-dire une fois que vous avez atteint environ 1To ou plus, vous devriez chercher à utiliser des sauvegardes d&#39;instantanés au niveau du système de fichiers physique. Un outil disponible pour le faire en open-source est le [Percona Serveur pour MongoDB](https://www.percona.com/software/mongodb/percona-server-for-mongodb). Il a la fonctionnalité intégrée de sauvegarde à chaud intégrée pour le moteur de stockage WiredTiger par défaut et prend à peu près le même temps que les autres instantanés physiques.

**_Telecharger gratuitement [Percona Backup pour MongoDB](https://www.percona.com/software/mongodb/percona-backup-for-mongodb) si vous souhaitez l&#39;essayer_**

_Page source :[Percona](https://www.percona.com/blog/2020/09/18/mongodb-backup-best-practices/)
