+++
title = "Comparer la performance de Graviton (ARM) vs celle d'Intel et AMD pour MySQL - Partie 02"
description = "Voici la deuxième partie de resultat de nos recherches sur les performances de Graviton, en le comparant avec d'autres processeurs (Intel et AMD) directement pour MySQL."
author = "Francis"
date = 2022-01-06
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article01-02.jpg"
images = ["thumbnail2022/article01-02.jpg"]
slug = "performance-graviton-arm-vs-intel-vs-amd-for-mysql-part02"
+++

Récemment, nous avons [publié la première partie de la recherche comparant Graviton (ARM) avec AMD et Intel CPU sur AWS](https://www.lesamisdepercona.fr/posts/performance-graviton-arm-vs-intel-vs-amd-for-mysql/) . Dans la première partie, nous avons sélectionné des instances EC2 à usage général avec les mêmes configurations (quantité de vCPU). L'objectif principal était de voir la tendance et de faire une comparaison générale des types de processeurs sur la plate-forme AWS uniquement pour MySQL. Nous ne nous sommes pas fixé pour objectif de comparer les performances de différents types de processeurs. Notre expertise réside dans le réglage des performances MySQL. Nous partageons la recherche « telle quelle » avec tous les scripts, et toute personne intéressée peut la réexécuter et la reproduire. Tous les scripts, journaux bruts et tracés supplémentaires sont disponibles sur GitHub : ( [2021_10_arm_cpu_comparison_c5](https://github.com/Percona-Lab-results/2021_10_arm_cpu_comparison_c5) , [csv_file_with_all_data](https://github.com/Percona-Lab-results/2021_10_arm_cpu_comparison_c5/blob/83bcff23ada279ff9a09890b79e2f6cca2b31573/report/oltp_test_result.csv) ).

Nous avons été heureux de voir les réactions de nos lecteurs du blog Percona à notre recherche, et nous sommes ouverts à tout commentaire. Si quelqu'un a des idées sur la mise à jour de notre méthodologie, nous serions heureux de les corriger.

Cet article est une continuation de la recherche basée sur notre intérêt pour l'EC2 optimisé pour le calcul (et, bien sûr, parce que nous avons vu que notre public voulait le voir). Aujourd'hui, nous allons parler de (AWS) Compute Optimized EC2 : C5, C5a, C6g (liste complète en annexe).

La prochaine fois, nous partagerons nos découvertes sur l'efficacité économique des instances m5 et c5.

## Courte conclusion :

1. Dans la plupart des cas, pour les instances c5, c5a et c6g, Intel affiche de meilleures performances de débit pour les transactions de lecture MySQL.
1. Parfois, Intel pouvait montrer un avantage significatif - plus de près de 100 000 rps que les autres processeurs.
1. Si nous pouvions dire en quelques mots : les instances c5 (avec Intel) sont meilleures dans leur catégorie que les autres instances c5a, c6g (en performances). Et cet avantage commence à partir de 5% et peut aller jusqu'à 40% par rapport aux autres processeurs. 

### Détails et avis de non-responsabilité :

1. Les tests ont été effectués sur des instances C5.\* (Intel) , C5a.\* (AMD),  C6g.\*(Graviton) EC2 dans la région US-EAST-1. (voir la liste des EC2 en annexe.)
1. Le suivi a été fait avec PMM
1. Système d'exploitation : Ubuntu 20.04 TLS 
1. Outil de chargement (sysbench) et base de données cible (MySQL) installés sur la même instance EC2
1. Oracle MySQL Community Server — 8.0.26-0 — installé à partir des packages officiels.
1. Outil de chargement : sysbench — 1.0.18 
1. innodb\_buffer\_pool\_size=80% de RAM disponible
1. La durée du test est de 5 minutes pour chaque thread, puis de 90 secondes avant la prochaine itération.
1. Les tests ont été exécutés 3 fois (pour lisser les valeurs aberrantes / pour avoir des résultats plus reproductibles). Ensuite, les résultats ont été moyennés pour les graphiques.
1. Nous allons utiliser une définition de scénario « à forte concurrence » pour les scénarios où le nombre de threads serait supérieur au nombre de vCPU. Et définition de scénario « faiblement simultané » avec des scénarios où le nombre de threads serait inférieur ou égal à un nombre de vCPU sur EC2.
1. Nous comparons le comportement de MySQL sur la même classe d'EC2, pas les performances du processeur.

## Cas de test:

### Prérequis:

\1. Créez une base de données avec 10 tables et 10 000 000 lignes par table

```
sysbench oltp_read_only --threads=10 --mysql-user=sbtest --mysql-password=sbtest --table-size=10000000 --tables=10 --db-driver=mysql --mysql-db=sbtest prepare

```

\2. Chargez toutes les données dans LOAD\_buffer

```
sysbench oltp_read_only --time=300 --threads=10 --table-size=1000000 --mysql-user=sbtest --mysql-password=sbtest --db-driver=mysql --mysql-db=sbtest run
```

\3. Test :

Exécuter en boucle pour le même scénario mais une simultanéité différente THREAD (1,2,4,8,16,32,64,128) sur chaque EC2

```
sysbench oltp_read_only --time=300 --threads=${THREAD} --table-size=100000 --mysql-user=sbtest --mysql-password=sbtest --db-driver=mysql --mysql-db=sbtest run
```

## Résultats:

L'examen des résultats a été divisé en trois parties :

1. Pour les « petits » EC2 avec 2, 4 et 8 processeurs virtuels
1. Pour EC2 "moyen" avec 16 et pour EC2 "grand" avec 48 et 64 vCPU (AWS n'a pas de C5 EC2 avec 64 vCPU)
1. Pour tous les scénarios pour voir l'image globale.

Il y aurait quatre graphiques pour chaque test :

1. Débit (requêtes par seconde) qu'EC2 pourrait effectuer pour chaque scénario (nombre de threads).
1. Latence 95 percentile qu'EC2 pourrait effectuer pour chaque scénario (nombre de threads)
1. Comparaison relative entre Graviton et Intel, Graviton et AMD.
1. Comparaison absolue entre Graviton et Intel, Graviton et AMD.

La validation que toute la charge va au CPU, et non aux DISK I/O, a également été effectuée à l'aide de PMM ( [Percona Monitoring and Management](https://www.percona.com/doc/percona-monitoring-and-management/2.x/index.html) ). 

![image01](/posts/2022/article01/part02/img01.jpeg)


*Photo 0.1. Surveillance du système d'exploitation pendant toutes les étapes de test (l'image est par exemple)*

## Résultat pour EC2 avec 2, 4 et 8 vCPU :

![image02](/posts/2022/article01/part02/img02.png)

*Photo 1.1. Débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image03](/posts/2022/article01/part02/img03.png)

*Photo 1.2. Latences (95 percentile) lors du test pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image04](/posts/2022/article01/part02/img04.png)

*Photo 1.3.1 Comparaison des pourcentages entre Graviton et CPU Intel en débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image05](/posts/2022/article01/part02/img05.png)

*Photo 1.3.2 Comparaison en pourcentage du débit des CPU Graviton et AMD (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads* 

![image06](/posts/2022/article01/part02/img06.png)

*Photo 1.4.1. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image07](/posts/2022/article01/part02/img07.png)

*Photo 1.4.2. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 2, 4 et 8 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

### APERÇU:

1. Sur la base du tracé 1.1, nous pourrions dire que EC2 avec Intel a un avantage absolu par rapport à Graviton et AMD.
1. Cet avantage dans la plupart des scénarios oscille entre 10 % et 20 %.
1. En chiffres, c'est plus de 3 000 requêtes par seconde. 
1. Il existe un scénario où Graviton devient meilleur EC2 avec 8 vCPU (c6g.2xlarge). Mais l'avantage est si minime (près de 2 %) qu'il pourrait s'agir d'une erreur statistique. On ne peut donc pas dire que les prestations sont pertinentes.

## Résultat pour EC2 avec 16, 48 et 64 vCPU :

![image08](/posts/2022/article01/part02/img08.png)


*Photo 2.1. Débit (requêtes par seconde) pour EC2 avec 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image09](/posts/2022/article01/part02/img09.png)

*Photo 2.2. Latences (95 percentile) lors du test pour EC2 avec 16, 48 et 64 vCPU pour les scénarios avec 1,2 4,8,16,32,64,128 threads*

![image10](/posts/2022/article01/part02/img10.png)

*Photo 2.3.1 Comparaison des pourcentages Graviton et CPU Intel en débit (requêtes par seconde) pour EC2 avec 16, 48 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image11](/posts/2022/article01/part02/img11.png)

*Photo 2.3.2 Comparaison en pourcentage du débit des CPU Graviton et AMD (requêtes par seconde) pour EC2 avec 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image12](/posts/2022/article01/part02/img12.png)

*Photo 2.4.1. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 16, 48 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image13](/posts/2022/article01/part02/img13.png)


*Photo 2.4.2. Comparaison des nombres Graviton et CPU AMD en débit (requêtes par seconde) pour EC2 avec 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

## APERÇU:

1. Le tracé 2.1 montre qu'il a un avantage sur les autres vCPU dans nos conditions (il n'y a pas d'EC2 avec 64 vCPU d'Intel pour avoir une image complète de comparaison). 
1. Cet avantage pourrait être proche de 20 % pour EC2 avec 16 vCPU et jusqu'à 40 % pour EC2 avec 48 vCPU. Cependant, il est possible de voir que cet avantage diminue avec l'augmentation du nombre de threads.
1. En nombres réels, Intel pourrait exécuter jusqu'à 100 000 transactions de lecture de plus que les autres processeurs (tracé 2.1. , tracé 2.4.1).
1. D'un autre côté, dans un scénario haute performance, nous pourrions voir un petit avantage (3 %) de Graviton. Cependant, il est si petit qu'il pourrait s'agir à nouveau d'une erreur statistique (graphique 2.3.1.). 
1. Dans la plupart des cas, Graviton montre de meilleurs résultats qu'AMD (tracé 2.1, tracé 2.3.2, tracé 2.4.2).

## Vue d'ensemble des résultats :

![image14](/posts/2022/article01/part02/img14.png)

*Photo 3.1. Débit (requêtes par seconde) pour EC2 avec 2, 4, 8, 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image15](/posts/2022/article01/part02/img15.png)

*Photo 3.2. Latences (95 percentile) lors du test pour EC2 avec 2, 4, 8, 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image16](/posts/2022/article01/part02/img16.png)

*Photo 3.3.1. Comparaison en pourcentage des CPU Graviton et Intel en débit (requêtes par seconde) pour EC2 avec 2, 4, 8, 16 et 48 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image17](/posts/2022/article01/part02/img17.png)

*Photo 3.3.2. Comparaison en pourcentage du débit des CPU Graviton et AMD (requêtes par seconde) pour EC2 avec 2, 4, 8, 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image18](/posts/2022/article01/part02/img18.png)

*Photo 3.4.1. Comparaison des nombres Graviton et Intel CPU en débit (requêtes par seconde) pour EC2 avec 2, 4, 8, 16 ET 48 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

![image19](/posts/2022/article01/part02/img19.png)

*Photo 3.4.2. Comparaison des nombres Graviton et CPU AMD en débit (requêtes par seconde) pour EC2 avec 2, 4, 8, 16, 48 et 64 vCPU pour les scénarios avec 1,2,4,8,16,32,64,128 threads*

## Conclusion Finale

1. Nous comparons les instances ec2 (c5, c5a, c6g) optimisées pour le calcul de la plate-forme AWS et leur comportement pour MySQL.
1. La question de l'efficacité économique de tous ces EC2 reste ouverte. Nous ferons des recherches sur ce sujet et répondrons à cette question un peu plus tard.
1. Dans ces tests, AMD ne fournit aucun résultat compétitif pour MySQL. Il est possible que dans d'autres tâches, il puisse montrer des résultats bien meilleurs et compétitifs.

### ANNEXES:

Liste des EC2 utilisés dans la recherche :

|Type de CPU|EC2|Quantité de vCPU|Mémoire GB|Prix EC2 par heure (USD)|
| :- | :- | :- | :- | :- |
|AMD|c5a.large|2|4|0,077|
|AMD|c5a.xlarge|4|8|0,154|
|AMD|c5a.2xlarge|8|16|0,308|
|AMD|c5a.4xlarge|16|32|0,616|
|AMD|c5a.12xlarge|48|96|1,848|
|AMD|c5a.16xlarge|64|128|2.464|
|Intel|c5.large|2|4|0,085|
|Intel|c5.xlarge|4|8|0,170|
|Intel|c5.2xlarge|8|16|0,340|
|Intel|c5.4xlarge|16|32|0,680|
|Intel|c5.12xlarge|48|96|2.040|
|Graviton|c6g.large|2|4|0,068|
|Graviton|c6g.xlarge|4|8|0,136|
|Graviton|c6g.2xlarge|8|16|0,272|
|Graviton|c6g.4xlarge|16|32|0,544|
|Graviton|c6g.12xlarge|48|96|1.632|
|Graviton|c6g.16xlarge|64|128|2.176|

### my.cnf:
```
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

Source : [Percona Blog](https://www.percona.com/blog/comparing-graviton-arm-performance-to-intel-and-amd-for-mysql-part-2/)
