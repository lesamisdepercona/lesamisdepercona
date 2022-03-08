+++
title = "Comparaison entre mysqldump vs MySQL Shell Utilities vs mydumper vs mysqlpump vs XtraBackup"
description = "Comparaison des performances d'une sauvegarde à partir d'une base de données MySQL à l'aide de mysqldump vs MySQL Shell Utilities vs mydumper vs mysqlpump vs XtraBackup"
author = "Francis"
date = 2022-03-08
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article10.jpg"
images = ["thumbnail2022/article10.jpg"]
slug = "sauvegarde-mysqldump-vs-mysql-shell-utilities-vs-mydumper-vs-mysqlpump-vs-xtrabackup"
+++

![thumbnail](/thumbnail2022/article10.jpg)

Dans cet article de blog, nous comparerons les performances d'une sauvegarde à partir d'une base de données MySQL à l'aide de [mysqldump ](https://dev.mysql.com/doc/refman/en/mysqldump.html), de la fonctionnalité MySQL Shell appelée [Instance Dump ](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-dump-instance-schema.html), [mysqlpump ](https://dev.mysql.com/doc/refman/8.0/en/mysqlpump.html), [mydumper ](https://github.com/maxbube/mydumper)et [Percona XtraBackup ](https://www.percona.com/software/mysql-database/percona-xtrabackup). Toutes ces options disponibles sont open source et gratuites pour toute la communauté.

Pour commencer, voyons les résultats du test.

## Résultats de Benchmark
Le benchmark a été exécuté sur une instance [m5dn.8xlarge ](https://aws.amazon.com/blogs/aws/new-m5n-and-r5n-instances-with-up-to-100-gbps-networking/), avec 128 Go de RAM, 32 vCPU et 2 disques NVMe de 600 Go (un pour la sauvegarde et l'autre pour les données MySQL ). La version MySQL était 8.0.26 et configurée avec 89 Go de pool de mémoire tampon, 20 Go de journal de rétablissement et une base de données exemple de 177 Go (plus de détails ci-dessous).

Nous pouvons observer les résultats dans le tableau ci-dessous :

![image01](/posts/2022/article10/img01.png)

Et si nous analysons le graphique uniquement pour les options multi-thread:

![image02](/posts/2022/article10/img02.png)

Comme nous pouvons le voir, pour chaque logiciel, j'ai exécuté chaque commande trois fois afin d'expérimenter en utilisant 16, 32 et 64 threads. L'exception à cela est mysqldump, qui n'a pas d'option parallèle et ne s'exécute qu'en mode monothread.

Nous pouvons observer des résultats intéressants :

1. Lors de l'utilisation de la compression zstd , mydumper brille vraiment en termes de performances. Cette option était ajouté il n'y a pas longtemps ( [MyDumper 0.11.3](https://www.percona.com/blog/mydumper-0-11-3-is-now-available/) ).
1. Lorsque mydumper utilise gzip , MySQL Shell est l'option de sauvegarde la plus rapide.
1. En 3ème nous avons Percona XtraBackup .
1. mysqlpump est le 4ème plus rapide suivi de plus près par mydumper lors de l'utilisation de gzip .
1. mysqldump est le style classique de la vieille école pour effectuer des vidages et est le plus lent des quatre outils.
1. Dans un serveur avec plus de processeurs, le parallélisme potentiel augmente, donnant encore plus d'avantages aux outils qui peuvent bénéficier de plusieurs threads.

## Spécifications matérielles et logicielles

Voici les spécifications du benchmark :

- 32 CPUs
- 128 Go de mémoire
- 2x NVMe disques 600 Go
- Centos 7.9
- MySQL 8.0.26
- MySQL Shell 8.0.26
- mydumper 0.11.5 – gzip
- mydumper 0.11.5 – zstd
- Xtrabackup 8.0.26

La configuration `my.cnf` :

```
[mysqld]
innodb_buffer_pool_size = 89G
innodb_log_file_size = 10G
```

## Test de performance

Pour le test, j'ai utilisé sysbench pour remplir MySQL. Pour charger les données, je choisis la méthode [tpcc ](https://github.com/Percona-Lab/sysbench-tpcc):

```
$ ./tpcc.lua  --mysql-user=sysbench --mysql-password='sysbench' --mysql-db=percona --time=300 --threads=64 --report-interval=1 --tables=10 --scale=100 --db-driver=mysql prepare
```

Avant de commencer la comparaison, j'ai exécuté mysqldump une fois et j'ai ignoré les résultats pour réchauffer le cache, sinon notre test serait biaisé car la première sauvegarde devrait récupérer les données du disque et non du cache.

Avec tout défini, j'ai démarré le mysqldump avec les options suivantes :

```
$ time mysqldump --all-databases --max-allowed-packet=4294967295 --single-transaction -R --master-data=2 --flush-logs | gzip > /backup/dump.dmp.gz
```

Pour l'utilitaire Shell:

```
$ mysqlsh
MySQL JS > shell.connect('root@localhost:3306');
MySQL localhost:3306 ssl test JS > util.dumpInstance("/backup", {ocimds: true, compatibility: ["strip_restricted_grants","ignore_missing_pks"],threads: 16})
```

Pour `mydumper` :

```
$ time mydumper  --threads=16  --trx-consistency-only --events --routines --triggers --compress --outputdir /backup/ --logfile /backup/log.out --verbose=2
```

*PS : Pour utiliser zstd , il n'y a aucun changement dans la ligne de commande, mais vous devez télécharger les [zstd binaires ](https://github.com/mydumper/mydumper/releases).*

Pour `mysqlpump` :

```
$ time mysqlpump --default-parallelism=16 --all-databases > backup.out
```

Pour `xtrabackup`:

```
$ time xtrabackup --backup --parallel=16 --compress --compress-threads=16 --datadir=/mysql_data/ --target-dir=/backup/
```

## Analyse des résultats

Et que nous disent les résultats ?

Les méthodes parallèles ont un débit de performances similaire. L' outil `mydumper` a réduit le temps d'exécution de 50 % lors de l'utilisation de `zstd` au lieu de `gzip` , de sorte que la méthode de compression fait une grande différence lors de l'utilisation de `mydumper` .

Pour l’utilitaire `util.dumpInstance`, l'un des avantages est que l'outil stocke les données au format binaire et texte et utilise la compression `zstd` par défaut. Comme mydumper, il utilise plusieurs fichiers pour stocker les données et a un bon taux de compression.

XtraBackup a obtenu la troisième place avec quelques secondes de différence par rapport au MySQL shell. Le principal avantage de XtraBackup est sa flexibilité, fournissant le PITR et le chiffrement par exemple.

Ensuite, `mysqlpump` est plus efficace que mydumper avec `gzip` , mais seulement par une petite marge. Les deux sont des méthodes de sauvegarde logiques et fonctionnent de la même manière. J'ai testé `mysqlpump` avec la compression `zstd`, mais les résultats étaient les mêmes, d'où la raison pour laquelle je ne l'ai pas ajouté au tableau. Une possibilité est que `mysqlpump` diffuse les données dans un seul fichier.

Enfin, pour `mysqldump`, nous pouvons dire qu'il a le comportement le plus prévisible et a des temps d'exécution similaires avec des exécutions différentes. Le manque de parallélisme et de compression est un inconvénient pour `mysqldump` ; cependant, comme il était présent dans les premières versions de MySQL, basé sur les cas Percona, il est toujours utilisé comme méthode de sauvegarde logique.

Veuillez laisser dans les commentaires ci-dessous ce que vous avez pensé de cet article de blog, si j'ai raté quelque chose ou si cela vous a aidé. Je serai ravi d'en discuter !

## Ressources utiles

Enfin, vous pouvez nous joindre via les réseaux sociaux, notre forum, ou accéder à notre matériel en utilisant les liens présentés ci-dessous :

- [**Blog** ](https://www.percona.com/blog/)
- [**Solution Briefs**](https://www.percona.com/resources/solution-brief)
- [**White Papers**](https://www.percona.com/resources/white-papers)
- [**Ebooks**](https://www.percona.com/resources/ebooks)
- [**Technical Presentations archive**](https://www.percona.com/resources/technical-presentations)
- [**Videos/Recorded Webinars**](https://www.percona.com/resources/videos)
- [**Forum**](https://www.percona.com/forums/)
- [**Knowledge Base (Percona Subscriber exclusive content)**](https://customers.percona.com/)

