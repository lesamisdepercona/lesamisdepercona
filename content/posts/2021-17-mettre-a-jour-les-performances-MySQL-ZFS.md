+++
title = "Mise à jour des performances MySQL/ZFS"
description = "Traduit à partir de l'article de Yves Trudeau intitulé, MySQL/ZFS Performance Update"
author = "Francis"
date = 2021-09-24T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle17.jpg"
images = ["thumbnail/thumbnailarticle17.jpg"]
slug = "mise-a-jour-des-performances-MySQL-ZFS"
+++

Comme certains d'entre vous le savent probablement, j'ai une opinion favorable de ZFS et en particulier de MySQL sur ZFS. Comme je l'ai [publié ](https://www.percona.com/blog/2018/05/15/about-zfs-performance/) il y a quelques années, l'argument en faveur de ZFS concernait moins les performances que ses fonctionnalités utiles telles que la compression de données et les instantanés. À l'époque, ZFS était significativement plus lent que xfs et ext4, sauf lorsque le L2ARC était utilisé.

Depuis lors, cependant, ZFS sur Linux a beaucoup progressé et j'ai également appris à mieux le régler. De plus, j'ai découvert que le benchmark sysbench que j'utilisais à l'époque n'était pas un choix juste car l'ensemble de données qu'il génère se compresse beaucoup moins qu'un fichier réaliste. Pour toutes ces raisons, je pense qu'il est temps de revisiter l'aspect performance de MySQL sur ZFS.

**Évolution de ZFS**

En 2018, j'ai rapporté les résultats de performances ZFS basés sur la version 0.6.5.6, la version par défaut disponible dans Ubuntu Xenial. Le présent article utilise la version 0.8.6-1 de ZFS, la version par défaut disponible sur Debian Buster. Entre les deux versions, il y a plus de 3600 commits en ajoutant un certain nombre de nouvelles fonctionnalités comme le support des opérations *trim* et l'ajout de algorithme de compression *zstd*.

ZFS 0.8.6-1 n'est pas à la pointe, il y a eu plus de 1700 commits depuis et après la 0.8.6, le numéro de version de ZFS est passé à 2.0. Le gros ajout inclus dans la version 2.0 est le cryptage natif.

**Outils de référence**

Les benchmarks classiques de la base de données MySQL sysbench ont un ensemble de données contenant principalement des données aléatoires. De tels ensembles de données ne compressent pas beaucoup, moins que la plupart des ensembles de données du monde réel avec lesquels j'ai travaillé. La compressibilité de l'ensemble de données est importante car les caches ZFS, l'ARC et L2ARC, stockent les données compressées. Un meilleur taux de compression signifie essentiellement que plus de données sont mises en cache et moins d'opérations IO seront nécessaires. 

Un outil bien connu pour comparer le volume de travail transactionnel est TPCC. De plus, l'ensemble de données créé par TPCC se compresse assez bien, ce qui le rend plus réaliste dans le contexte de cet article. Nous utilisons l’[implémentation sysbench TPCC](https://github.com/Percona-Lab/sysbench-tpcc)

**Environnement Test**

Comme je connais déjà AWS et Google cloud, j'ai décidé d'essayer Azure pour ce projet. J'ai lancé ces deux machines virtuelles : 

tpcc :

- hôte de référence
- Instance D2ds\_v4 standard
- 2 vCpu , 8 Go de RAM et 75 Go de stockage temporaire
- Debian Buster

db :

- Hôte de la base de données
- Instance E4-2ds-v4 standard
- 2 vCpu , 32 Go de RAM et 150 Go de stockage temporaire
- 256 Go SSD Premium (SSD Premium LRS P15 – 1100 IOPS (3500 burst), 125 Mo/s)
- Debian Buster
- Percona serveur 8.0.22-13

**Configuration**

Par défaut et sauf indication contraire, les systèmes de fichiers ZFS sont créés avec :
```
zpool create bench /dev/sdc
zfs set compression=lz4 atime=off logbias=throughput bench
zfs create -o mountpoint=/var/lib/mysql/data -o recordsize=16k \
           -o primarycache=metadata bench/data
zfs create -o mountpoint=/var/lib/mysql/log bench/log
```

Il existe deux systèmes de fichiers ZFS. *bench/data* est optimisé pour l'ensemble de données InnoDB tandis que *bench/log* est réglé pour les fichiers journaux InnoDB . Les deux sont compressés à l'aide de lz4 et le paramètre *logbias* est défini sur le *débit,* ce qui modifie la façon dont le ZIL est utilisé. Avec ext4, l'option *noatime* est utilisée.

ZFS a également un certain nombre de paramètres de noyau, ceux définis sur des valeurs autres que celles par défaut sont :
```
zfs_arc_max=2147483648
zfs_async_block_max_blocks=5000
zfs_delete_blocks=1000
```
Essentiellement, les paramètres ci-dessus limitent la taille de l'ARC à 2 Go et réduisent l'agressivité de ZFS pour les suppressions. Enfin, la configuration de la base de données est légèrement différente entre ZFS et ext4. Il y a une section commune :

```
[mysqld]
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
log-error = /var/log/mysql/error.log
skip-log-bin
datadir = /var/lib/mysql/data
innodb_buffer_pool_size = 26G
innodb_flush_log_at_trx_commit = 1 # TPCC reqs.
innodb_log_file_size = 1G
innodb_log_group_home_dir = /var/lib/mysql/log
innodb_flush_neighbors = 0
innodb_fast_shutdown = 2
```

et lorsque ZFS est utilisé :

```
innodb_flush_method = O_DIRECT
```

et lorsque ZFS est utilisé :
```
innodb_flush_method = fsync
innodb_doublewrite = 0 # ZFS is transactional
innodb_use_native_aio = 0
innodb_read_io_threads = 10
innodb_write_io_threads = 10
```
ZFS ne prend pas en charge *O\_DIRECT* mais il est ignoré avec un message dans le journal des erreurs. J'ai choisi de définir explicitement la méthode flush sur *fsync*. Le tampon à double écriture n'est pas nécessaire avec ZFS et j'avais l'impression que l'implémentation IO asynchrones natives de Linux n'était pas bien prise en charge par ZFS. Je l'ai donc désactivée et augmenté le nombre de threads d'IO. Nous reviendrons sur la question des IO asynchrones dans un prochain article.

**Base de données**

J'utilise la commande suivante pour créer l'ensemble de données :

```
./tpcc.lua --mysql-host=10.3.0.6 --mysql-user=tpcc --mysql-password=tpcc --mysql-db=tpcc \
--threads=8 --tables=10 --scale=200 --db-driver=mysql prepare
```

L'ensemble de données résultant a une taille d'environ 200 Go. L'ensemble de données est beaucoup plus volumineux que le pool de mémoire tampon, de sorte que les performances de la base de données sont essentiellement liées aux IO.

**Procédure Test**

L'exécution de chaque benchmark a été scriptée et a suivi ces étapes :

1. Arrêter MySQL
1. Supprimer tous les fichiers de données
1. Ajuster le système de fichiers
1. Copier l'ensemble de données
1. Ajuster la configuration MySQL
1. Démarrer MySQL
1. Enregistrer la configuration
1. Exécuter le benchmark

**Résultats**

Pour le benchmark, j'ai utilisé l'invocation suivante :
```
./tpcc.lua --mysql-host=10.3.0.6 --mysql-user=tpcc --mysql-password=tpcc --mysql-db=tpcc \
--threads=16 --time=7200 --report-interval=10 --tables=10 --scale=200 --db-driver=mysql ru
```
Le benchmark TPCC utilise 16 threads pour une durée de 2 heures. La durée est suffisamment longue pour permettre un état stationnaire et épuiser la capacité de stockage. Sysbench renvoie le nombre total de transactions TPCC par seconde toutes les 10 s. Ce nombre comprend non seulement les transactions de *nouvelle commande*, mais également les autres types de transactions comme le *paiement*, le *statut de la commande*, etc. Sachez-le si vous souhaitez comparer ces résultats avec d'autres références TPCC.

Dans ces conditions, la figure ci-dessous présente les taux de transactions TPCC au fil du temps pour ext4 et ZFS.

![image01](/posts/article17/img01.png)

*Résultats MySQL TPCC pour ext4 et ZFS*

Au cours des 15 premières minutes, le pool de mémoire tampon se réchauffe, mais à un moment donné, la charge de travail se déplace entre une lecture IO liée à une écriture IO et à un processeur. Ensuite, à environ 3000 s, la capacité de SSD Premium est épuisée et la charge de travail n'est liée qu'aux IO. J'ai été un peu surpris par les résultats, assez pour refaire les benchmarks pour m'en assurer. Les résultats pour ext4 et ZFS **sont qualitativement similaires.** Toute différence est dans la marge d'erreur. Cela signifie essentiellement que si vous configurez correctement ZFS, il peut être aussi efficace en IO que ext4.

Ce qui est intéressant, c'est la quantité de stockage utilisée. Alors que l'ensemble de données sur ext4 consommait 191 Go, la compression lz4 de ZFS a donné un ensemble de données de seulement 69 Go. C'est une énorme différence, un facteur de 2,8, qui pourrait économiser une somme d'argent décente au fil du temps pour les grands ensembles de données.

**Conclusion**

Il semble que c'était effectivement le bon moment pour revisiter les performances de MySQL avec ZFS. Dans un cas d'utilisation assez réaliste, ZFS est comparable à ext4 en termes de performances tout en offrant les avantages supplémentaires de la compression de données, des instantanés, etc. Dans le prochain article, j'examine l'utilisation du [stockage éphémère dans le cloud avec ZFS](https://www.percona.com/blog/mysql-zfs-in-the-cloud-leveraging-ephemeral-storage/) et je vois comment cela pourrait améliorer les performances.  

**Percona Distribution pour MySQL est la solution MySQL open source la plus complète, stable, évolutive et sécurisée disponible, offrant des environnements de base de données de niveau entreprise pour vos applications métier les plus critiques… et son utilisation est gratuite !**

[Télécharger Percona Distribution pour MySQL](https://www.percona.com/software/mysql-database)

Source : [Blog Percona](https://www.percona.com/blog/mysql-zfs-performance-update/) 
