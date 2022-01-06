+++
title = "Comparer la performance de Graviton (ARM) vs celle d'Intel et AMD pour MySQL"
description = "Voici le resultat de nos recherches sur les performances de Graviton, en le comparant avec d'autres processeurs (Intel et AMD) directement pour MySQL."
author = "Francis"
date = 2022-01-06
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/2022/article01-01.jpg"
images = ["thumbnail/2022/article01-01.jpg"]
slug = "performance-graviton-arm-vs-intel-vs-amd-for-mysql"
+++

Récemment, AWS a présenté sa propre architecture CPU sur ARM pour les solutions serveur.

C'était Graviton. En conséquence, ils mettent à jour certaines lignes de leurs instances EC2 avec le nouveau suffixe « g » (par exemple, m6g.small, r5g.nano, etc.). Dans leur examen et présentation, AWS a montré des résultats impressionnants qu'il est plus rapide dans certains benchmarks jusqu'à 20%. D'un autre côté, certains critiques ont déclaré que Graviton n'affichait aucun résultat significatif et, dans certains cas, affichait moins de performances qu'Intel.

Nous avons décidé de l'étudier et de faire nos recherches sur les performances de Graviton, en le comparant avec d'autres processeurs (Intel et AMD) directement pour MySQL.

## Avertissement

1. Le test est conçu pour être lié au processeur uniquement, nous allons donc utiliser un test en lecture seule et nous assurer qu'il n'y a pas d'activité d'E/S pendant le test.
1. Les tests ont été effectués sur des instances EC2 m5.\* (Intel), m5a.\* (AMD), m6g.\* (Graviton) dans la région US-EAST-1. (Liste des EC2 voir en annexe).  
1. Le suivi a été réalisé avec [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management) (PMM).  
1. Système d'exploitation : Ubuntu 20.04 TLS.
1. Outil de chargement (sysbench) et base de données cible (MySQL) installés sur la même instance EC2.
1. MySQL – 8.0.26-0 – installé à partir des packages officiels.
1. Outil de chargement : sysbench — 1.0.18
1. innodb\_buffer\_pool\_size=80 % de la RAM disponible.
1. La durée du test est de cinq minutes pour chaque thread, puis de 90 secondes avant la prochaine itération.
1. Les tests ont été exécutés trois fois (pour lisser les valeurs aberrantes ou avoir des résultats plus reproductibles), puis nous prenons les résultats moyens pour les graphiques. 
1. Nous allons utiliser des scénarios de simultanéité élevée pour les scénarios où le nombre de threads serait supérieur au nombre de vCPU. Et scénario faiblement simultané avec des scénarios où le nombre de threads serait inférieur ou égal à un nombre de vCPU sur EC2.
1. Scripts pour reproduire les résultats sur notre GitHub.

## Cas de test

### Prérequis:

1. Créez une base de données avec 10 tables et 10 000 000 lignes par table

```
sysbench oltp_read_only --threads=10 --mysql-user=sbtest --mysql-password=sbtest --table-size=10000000 --tables=10 --db-driver=mysql --mysql-db=sbtest prepare

```


2. Chargez toutes les données dans LOAD\_buffer

```
sysbench oltp_read_only --time=300 --threads=10 --table-size=1000000 --mysql-user=sbtest --mysql-password=sbtest --db-driver=mysql --mysql-db=sbtest run

```

### Test:

Exécuter en boucle pour le même scénario mais un THREAD de simultanéité différent (1,2,4,8,16,32,64,128) sur chaque EC2.

```
sysbench oltp_read_only --time=300 --threads=${THREAD} --table-size=100000 --mysql-user=sbtest --mysql-password=sbtest --db-driver=mysql --mysql-db=sbtest run

```

### Résultats:

L'examen des résultats a été divisé en 3 parties :

1. pour les « petits » EC2 avec 2, 4 et 8 vCPU
1. pour EC2 « moyen » avec 16 et 32 vCPU
1. pour "grand" EC2 avec 48 et 64 vCPU 

Les divisions « petite », « moyenne » et « grande » ne sont que des noms synthétiques pour une meilleure évaluation. Elles dépendent de la quantité de vCPu par EC2.

Il y aurait quatre graphiques pour chaque test :

1. Débit (requêtes par seconde) qu'EC2 pourrait effectuer pour chaque scénario (nombre de threads)
1. Latence 95 percentile qu'EC2 pourrait effectuer pour chaque scénario (nombre de threads) 
1. Comparaison relative Graviton et Intel
1. Comparaison absolue de Graviton et Intel

La validation de toute la charge que va au CPU, et non aux DISK I/O, a également été effectuée à l'aide de PMM ( [Percona Monitoring and Management](https://www.percona.com/doc/percona-monitoring-and-management/2.x/index.html) ). 

![image01](/posts/2022/article01/part01/img01.jpeg)

*Pic 0.1 - Surveillance du système d'exploitation pendant toutes les étapes de test*

A partir de l'image 0.1, nous pouvons voir qu'il n'y avait pas d'activité DISK I/O pendant les tests, seulement une activité CPU. L'activité principale avec les disques était pendant la phase de création de la base de données.

## Résultat pour EC2 avec 2, 4 et 8 vCPU

![image02](/posts/2022/article01/part01/img02.png)

*Photo 1.1. Débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image03](/posts/2022/article01/part01/img03.png)

*Photo 1.2. Latences (95 percentile) pendant le test pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads* 

![image04](/posts/2022/article01/part01/img04.png)

*Photo 1.3. Comparaison en pourcentage du débit des processeurs Graviton et Intel (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image05](/posts/2022/article01/part01/img05.png)


*Photo 1.4. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

APERÇU:

1. AMD a les plus grandes latences dans tous les scénarios et pour toutes les instances EC2. Nous ne répéterons pas cette information dans tous les aperçus futurs, et c'est la raison pour laquelle nous l'excluons de la comparaison avec d'autres processeurs en pourcentage et en valeurs numériques (dans les graphiques 1.3 et 1.4, etc.).
1. Les instances avec deux et quatre vCPU Intel présentent un certain avantage pour moins de 10 % dans tous les scénarios.
1. Cependant, une instance avec 8 vCPU Intel montre un avantage uniquement sur les scénarios avec des threads qui ont une quantité inférieure ou égale de vCPU sur EC2.
1. Sur EC2 avec huit vCPU, Graviton a commencé à montrer un avantage. Il montre de bons résultats dans les scénarios où le nombre de threads est supérieur à la quantité de vCPU sur EC2. Il augmente jusqu'à 15 % dans les scénarios à forte concurrence avec 64 et 128 threads, qui sont 8 et 16 fois plus importants que la quantité de vCPU disponible pour l'exécution.
1. Graviton commence à montrer un avantage sur EC2 avec huit vCPU et avec des scénarios où les threads dépassent la quantité de vCPU. Cette fonctionnalité apparaîtrait dans tous les scénarios futurs - plus de charge que le processeur, meilleur résultat qu'elle montre.



## Résultat pour EC2 avec 16 et 32 vCPU

![image06](/posts/2022/article01/part01/img06.png)


*Photo 2.1. Débit (requêtes par seconde) pour EC2 avec 16 et 32 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image07](/posts/2022/article01/part01/img07.png)


*Photo 1.2. Latences (95 percentile) lors du test pour EC2 avec 16 et 32 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image08](/posts/2022/article01/part01/img08.png)


*Photo 2.3. Comparaison en pourcentage du débit des CPU Graviton et Intel (requêtes par seconde) pour EC2 avec 16 et 32 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image09](/posts/2022/article01/part01/img09.png)

*Photo 2.4. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 16 et 32 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

### APERÇU:

1. Dans les scénarios avec la même charge pour ec2 avec 16 et 32 vCPU, Graviton continue d'avoir des avantages lorsque la quantité de threads est plus importante que la quantité de vCPU disponible sur les instances.
1. Graviton montre un avantage allant jusqu'à 10 % dans les scénarios à forte concurrence. Cependant, Intel a jusqu'à 20 % dans les scénarios de faible concurrence.
1. Dans les scénarios à forte concurrence, Graviton pourrait montrer une différence incroyable dans le nombre de transactions (lecture) par seconde jusqu'à 30 000 TPS.

## Résultat pour EC2 avec 48 et 64 vCPU

![image10](/posts/2022/article01/part01/img10.png)

*Photo 3.1. Débit (requêtes par seconde) pour EC2 avec 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*  

![image11](/posts/2022/article01/part01/img11.png)

*Photo 3.2. Latences (95 percentile) lors du test pour EC2 avec 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads* 

![image12](/posts/2022/article01/part01/img12.png)

*Photo 3.3. Comparaison en pourcentage des CPU Graviton et Intel en débit (requêtes par seconde) pour EC2 avec 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image13](/posts/2022/article01/part01/img13.png)

*Photo 3.4. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

### APERÇU:

1. Il semble qu'Intel présente un avantage significatif dans la plupart des scénarios lorsque son nombre de threads est inférieur ou égal à la quantité de vCPU. Il semble qu'Intel soit vraiment bon pour ce genre de tâche. Quand il a du vCPU supplémentaire, ce serait mieux, et cet avantage pourrait aller jusqu'à 35%.
1. Cependant, Graviton affiche des résultats exceptionnels lorsque la quantité de threads est supérieure à la quantité de vCPU. Il montre un avantage de 5 à 14% sur Intel.
1. En chiffres réels, l'avantage de Graviton pourrait atteindre 70 000 transactions par seconde par rapport aux performances d'Intel dans les scénarios à forte concurrence.

## Aperçu des résultats totaux

![image14](/posts/2022/article01/part01/img14.png)

*photo 4.2. Latences (95 percentile) lors du test pour EC2 avec 2,4,8,16,32,48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image15](/posts/2022/article01/part01/img15.png)

*photo 4.3. Comparaison en pourcentage des CPU Graviton et Intel en débit (requêtes par seconde) pour EC2 avec 2,4,8,16,32,48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image16](/posts/2022/article01/part01/img16.png)

*Photo 4.4. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 2,4,8,16,32,48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

### Conclusion

1. Les processeurs ARM affichent de meilleurs résultats sur EC2 avec plus de vCPU et avec une charge plus élevée, en particulier dans les scénarios à forte concurrence.
1. En raison des petites instances EC2 et de la faible charge, les processeurs ARM affichent des performances moins impressionnantes. Nous ne pouvons donc pas voir ses avantages par rapport à Intel EC2
1. Intel est toujours le leader dans le domaine des scénarios à faible concurrence. Et c'est définitivement gagnant sur EC2 avec une petite quantité de vCPU.
1. AMD ne montre pas de résultats compétitifs dans tous les cas.

## Conclusion Finale

1. AMD — nous avons beaucoup de questions sur les instances EC2 sur AMD. Ce serait donc une bonne idée de vérifier ce qui se passait sur cet EC2 pendant le test et de vérifier les performances générales des processeurs sur ces EC2.
1. Nous avons découvert que dans certaines conditions spécifiques, Intel et Graviton pouvaient se concurrencer. Mais le revers de la médaille est économique. Qu'est-ce qui est moins cher à utiliser dans chaque situation ? Le prochain article en parlera.
1. Ce serait une bonne idée d'essayer d'utiliser EC2 avec Graviton pour une vraie base de données à forte concurrence.
1. Il semble qu'il faut exécuter des scénarios supplémentaires avec 256 et 512 threads pour vérifier l'hypothèse selon laquelle Graviton pourrait mieux fonctionner lorsque les threads sont supérieurs à vCPU.

[Découvrez la deuxième partie de nos tests !](https://www.percona.com/blog/comparing-graviton-arm-performance-to-intel-and-amd-for-mysql-part-2/)



## ANNEXE:##

Liste des EC2 utilisés dans la recherche :

|Type de processeur|EC2|Prix EC2 par heure (USD)|vCPU |RAM|
| :- | :- | :- | :- | :- |
|Graviton|m6g.large|0,077|2|8 Go|
|Graviton|m6g.xlarge|0,154|4|16 GB|
|Graviton|m6g.2xlarge|0,308|8|32 Go|
|Graviton|m6g.4xlarge|0,616|16|64 Go|
|Graviton|m6g.8xlarge|1.232|32|128 Go|
|Graviton|m6g.12xlarge|1,848|48|192 Go|
|Graviton|m6g.16xlarge|2.464|64|256 Go|
|Intel|m5.large|0,096|2|8 Go|
|Intel|m5.xlarge|0,192|4|16 GB|
|Intel|m5.2xlarge|0,384|8|32 Go|
|Intel|m5.4xlarge|0,768|16|64 Go|
|Intel|m5.8xlarge|1.536|32|128 Go|
|Intel|m5.12xlarge|2.304|48|192 Go|
|Intel|m5.16xlarge|3.072|64|256 Go|
|AMD|m5a.large|0,086|2|8 Go|
|AMD|m5a.xlarge|0,172|4|16 GB|
|AMD|m5a.2xlarge|0,344|8|32 Go|
|AMD|m5a.4xlarge|0,688|16|64 Go|
|AMD|m5a.8xlarge|1,376|32|128 Go|
|AMD|m5a.12xlarge|2.064|48|192 Go|
|AMD|m5a.16xlarge|2,752|64|256 Go|

### my.cnf

```
my.cnf:
[mysqld]
ssl=0
performance_schema=OFF
skip_log_bin
server_id = 7

# general
table_open_cache = 200000
table_open_cache_instances=64
back_log=3500
max_connections=4000
 join_buffer_size=256K
 sort_buffer_size=256K

# files
innodb_file_per_table
innodb_log_file_size=2G
innodb_log_files_in_group=2
innodb_open_files=4000

# buffers
innodb_buffer_pool_size=${80%_OF_RAM}
innodb_buffer_pool_instances=8
innodb_page_cleaners=8
innodb_log_buffer_size=64M

default_storage_engine=InnoDB
innodb_flush_log_at_trx_commit  = 1
innodb_doublewrite= 1
innodb_flush_method= O_DIRECT
innodb_file_per_table= 1
innodb_io_capacity=2000
innodb_io_capacity_max=4000
innodb_flush_neighbors=0
max_prepared_stmt_count=1000000 
bind_address = 0.0.0.0
[client]

```

Source : [Percona Blog](https://www.percona.com/blog/comparing-graviton-performance-to-arm-and-intel-for-mysql/)
