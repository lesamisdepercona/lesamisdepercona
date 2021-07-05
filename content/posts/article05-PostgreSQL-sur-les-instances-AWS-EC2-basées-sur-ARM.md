+++
title = "PostgreSQL sur les instances AWS EC2 basées sur ARM c'est bien"
description = "un regard indépendant sur le rapport prix/performances des nouvelles instances du point de vue de l'exécution de PostgreSQL"
author = "Francis"
date = 2021-07-04T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = ""
slug = "PostgreSQL-sur-les-instances-AWS-EC2-basées-sur-ARM"
+++

La croissance attendue des processeurs ARM dans les centres de données est au centre de sujet de discussion depuis un certain temps, et nous étions curieux de voir comment cela fonctionnait avec PostgreSQL. La disponibilité générale des serveurs basés sur ARM pour les tests et l&#39;évaluation était un obstacle majeur. Le brise-glace a eu lieu lorsqu&#39;AWS a [annoncé son offre de processeurs basés sur ARM](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://aws.amazon.com/about-aws/whats-new/2018/11/introducing-amazon-ec2-a1-instances/) dans son cloud en 2018. Mais nous n&#39;avons pas pu voir beaucoup d&#39;enthousiasme dans l&#39;immédiat, car beaucoup considéraient qu&#39;il s&#39;agissait de choses plus «expérimentales». Nous étions également prudents quant à sa recommandation pour une utilisation critique et n&#39;avons jamais fait assez d&#39;efforts pour l&#39;évaluer. Mais lorsque la deuxième génération d&#39;[instances basées sur Graviton2 a été annoncée en mai 2020](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://aws.amazon.com/blogs/aws/new-m6g-ec2-instances-powered-by-arm-based-aws-graviton2/), nous l&#39;avions considéré davantage. Nous avons décidé de jeter un regard indépendant sur le rapport prix/performances des nouvelles instances du point de vue de l&#39;exécution de PostgreSQL.

**Important** : Notez que même s&#39;il est tentant d&#39;appeler cette comparaison de PostgreSQL sur x86 vs arm, ce ne serait pas correct. Ces tests comparent PostgreSQL sur deux instances de cloud virtuel, et cela inclut bien plus de pièces mobiles qu&#39;un simple processeur. Nous nous concentrons principalement sur le rapport qualité-prix de deux instances AWS EC2 particulières basées sur deux architectures différentes.

## Test Configuration

Pour ce test, nous avons sélectionné deux instances similaires. L&#39;un est l&#39;ancien m5d . 8xlarge , et l&#39;autre est le nouveau Graviton2 basé m6gd.8xlarge . Les deux instances sont livrées avec un stockage « éphémère » local que nous utiliserons ici. L&#39;utilisation de disques locaux très rapides devrait permettre d&#39;exposer les différences dans d&#39;autres parties du système et d&#39;éviter de tester le stockage en nuage. Les instances ne sont pas parfaitement identiques, comme vous le verrez ci-dessous, mais sont suffisamment proches pour être considérées comme de même niveau. Nous avons utilisé Ubuntu 20.04 AMI et PostgreSQL 13.1 du pgdg repo. Nous avons effectué des tests avec des bases de données de tailles petites (in-memory) et large (io-bound).

### Instances

Spécifications et tarification à la demande des instances selon les [informations de tarification AWS](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://aws.amazon.com/ec2/pricing/on-demand/) pour Linux dans la région de Virginie du Nord. Avec les prix actuellement affichés, m6gd.8xlarge est 25% moins cher.

##### Instance Graviton2 (ARM)
`` Instance : m6gd.8xlarge 	
Virtual CPUs : 32
RAM  : 128 GiB 	
Storage : 1 x 1900 NVMe SSD (1.9 TiB)
Price : $1.4464 per Hour
``
##### Instance Régulière (x86)
``
Instance : m5d.8xlarge
Virtual CPUs : 32
RAM : 128 GiB
Storage : 2 x 600 NVMe SSD (1.2 TiB)
Price : $1.808 per Hour
``
### Configuration du système d&#39;exploitation (OS) et de PostgreSQL

Nous avons sélectionné les AMI Ubuntu 20.04.1 LTS pour les instances et n&#39;avons rien changé du côté du système d&#39;exploitation. Sur l&#39;instance m5d.8xlarge, deux disques NVMe locaux ont été unifiés dans un seul périphérique raid0. PostgreSQL a été installé à l&#39;aide de .deb disponibles dans le repository PGDG.

La chaîne de version de PostgreSQL confirme l&#39;architecture du système d&#39;exploitation
``
postgres=# select version();
                                                                version                                                                 
----------------------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 13.1 (Ubuntu 13.1-1.pgdg20.04+1) on aarch64-unknown-linux-gnu, compiled by gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0, 64-bit
(1 row)
``

\*\* aarch64 signifie architecture ARM 64 bits

La configuration PostgreSQL suivante a été utilisée pour les tests.
``
max_connections = '200'
shared_buffers = '32GB'
checkpoint_timeout = '1h'
max_wal_size = '96GB'
checkpoint_completion_target = '0.9'
archive_mode = 'on'
archive_command = '/bin/true'
random_page_cost = '1.0'
effective_cache_size = '80GB'
maintenance_work_mem = '2GB'
autovacuum_vacuum_scale_factor = '0.4'
bgwriter_lru_maxpages = '1000'
bgwriter_lru_multiplier = '10.0'
wal_compression = 'ON'
log_checkpoints = 'ON'
log_autovacuum_min_duration = '0'
``

## Tests pgbench

Tout d&#39;abord, une première série de tests est effectuée à l&#39;aide de pgbench, l&#39;outil de micro-benchmarking disponible avec PostgreSQL. Cela nous permet de tester avec une combinaison différente d&#39;un certain nombre de clients et d&#39;emplois comme :
``
pgbench -c 16 -j 16 -T 600 -r
``
Où 16 connexions client et 16 tâches pgbench alimentant les connexions client sont utilisées.

### Lecture-écriture sans somme de contrôle (Checksum)

La charge par défaut créée par pgbench est une charge en lecture-écriture de type tpcb. Nous avons utilisé la même chose sur une instance PostgreSQL sur laquelle la somme de contrôle n&#39;est pas activée.

Nous pourrions voir un gain de performance de **19%** sur ARM.

| x86 (tps) | 28878 |
| --- | --- |
| ARM (tps) | 34409 |

![image 01](/images/article05/p05_image01.png)
### Lecture- écriture avec somme de contrôle (Checksum)

Nous étions curieux de savoir si le calcul de la somme de contrôle avait un impact sur les performances en raison de la différence d&#39;architecture si la somme de contrôle de niveau PostgreSQL est activée. Sur PostgreSQL 12 et les versions ultérieures, la somme de contrôle peut être activée à l&#39;aide de l&#39;utilitaire pg\_checksum comme suit :

``
pg_checksums -e -D $PGDATA
``


| x86 (tps) | 29402 |
| --- | --- |
| ARM (tps) | 34701 |

![image 02](/images/article05/p05_image02.png)

À notre grande surprise, les résultats étaient légèrement meilleurs! Comme la différence n&#39;est que de 1,7%, nous considérons cela comme un bruit. Au moins, nous pensons qu&#39;il est acceptable de conclure que l&#39;activation de la somme de contrôle n&#39;entraîne aucune dégradation notable des performances sur ces processeurs modernes.

### Lecture seule sans somme de contrôle (Checksum)

Les charges en lecture seule devraient être centrées sur le processeur. Étant donné que nous avons sélectionné une taille de base de données qui s&#39;intègre parfaitement dans la mémoire, nous pourrions éliminer les frais généraux liés aux E/S.

| x86 (tps) | 221436.05 |
| --- | --- |
| ARM (tps) | 288867.44 |

![image 03](/images/article05/p05_image03.png)

Les résultats ont montré un gain de **30%** en tps pour l&#39;ARM par rapport à l&#39;instance x86 .

### Lecture seule avec somme de contrôle (Checksum)

Nous voulions vérifier si nous pouvions observer un changement de tps si la somme de contrôle était activée lorsque la charge devient purement centrée sur le processeur.

| x86 (tps) | 221144.3858 |
| --- | --- |
| ARM (tps) | 279753.1082 |

![image 04](/images/article05/p05_image04.png)

Les résultats sont très proches du précédent, avec des gains de 26,5%.

Dans les tests de pgbench, nous avons observé que lorsque la charge devient centrée sur le processeur, la différence de performances augmente. Nous n&#39;avons pu observer aucune dégradation des performances avec la somme de contrôle.

### _Remarque sur les sommes de contrôle_

_PostgreSQL calcule et écrit la somme de contrôle des pages lorsqu&#39;elles sont écrites et lues dans le pool de mémoire tampon. De plus, les bits d&#39;indication sont toujours enregistrés lorsque les sommes de contrôle sont activées, ce qui augmente la pression WAL IO. Pour valider correctement la surcharge globale de la somme de contrôle, nous aurions besoin de tests plus longs et plus importants, comme nous l&#39;avons fait avec sysbench-tpcc ._

## Test avec sysbench-tpcc

Nous avons décidé d&#39;effectuer des tests plus détaillés en utilisant [sysbench-tpcc](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://github.com/Percona-Lab/sysbench-tpcc) . Nous nous sommes principalement intéressés au cas où la base de données s&#39;insère dans la mémoire. En passant, alors que PostgreSQL sur le serveur arm n&#39;a montré aucun problème, sysbench était beaucoup plus capricieux que celui de x86.

Chaque série de tests comportait quelques étapes :

1. Restaurer le répertoire de données de la balance nécessaire (10/200).
2. Exécutez un test de préchauffage de 10 minutes avec les mêmes paramètres que le test de grande envergure.
3. Point de contrôle côté PG.
4. Exécutez le test réel.

### En mémoire, 16 threads :

![image 05](/images/article05/p05_image05.png)

Avec cette charge modérée, l&#39;instance ARM affiche des performances environ **15,5%** meilleures que l&#39;instance x86. Ici et après, la différence en pourcentage est basée sur la valeur moyenne du tps.

Vous vous demandez peut-être pourquoi il y a une baisse soudaine des performances vers la fin du test. Il est lié au point de contrôle avec full\_page\_writes . Même si pour les tests en mémoire, nous avons utilisé la distribution pareto, une quantité considérable de pages va être écrite après chaque point de contrôle. Dans ce cas, l&#39;instance affichant plus de performances a déclenché un point de contrôle par WAL plus tôt que son homologue. Ces creux seront présents dans tous les tests effectués.

### En mémoire , 32 threads :

![image 06](/images/article05/p05_image06.png)

Lorsque la simultanéité est passée à 32, la différence de performances a été réduite à près de **8 %**.

### En mémoire, 64 threads :

![image 07](/images/article05/p05_image07.png)

En poussant les instances près de leur point de saturation (rappelez-vous, les deux sont des instances à 32 processeurs), nous voyons la différence se réduire encore à **4,5%**.

### En mémoire , 128 threads :

![image 08](/images/article05/p05_image08.png)

Lorsque les deux instances ont dépassé leur point de saturation, la différence de performances devient négligeable, bien qu&#39;elle soit toujours là à **1,4%.** De plus, nous avons pu observer une baisse de **6 à 7%** du débit ( tps ) pour ARM et une baisse de **4%** pour x86 en cas de concurrence augmenté de 64 à 128 sur ces 32 machines vCPU.

Tout ce que nous avons mesuré n&#39;est pas favorable à l&#39;instance basée sur Graviton2. Dans les tests liés aux E/S (ensemble de données ~ 200G, 200 entrepôts, distribution uniforme), nous avons vu moins de différence entre les deux instances, et à 64 et 128 threads, l&#39;instance m5d standard a mieux fonctionné. Vous pouvez le voir sur les graphiques combinés ci-dessous.

![image 09](/images/article05/p05_image09.png)

Une raison possible à cela, en particulier l&#39;effondrement important à 128 threads pour m6gd.8xlarge, est qu&#39;il lui manque le deuxième lecteur que m5d.8xlarge possède. Il n&#39;y a actuellement aucun couple d&#39;instances parfaitement comparables disponibles, nous considérons donc cela comme une comparaison équitable; chaque type d&#39;instance a un avantage. Davantage de tests et de profils sont nécessaires pour identifier correctement la cause, car nous nous attendions à ce que les disques locaux affectent de manière négligeable les tests. Des tests liés aux E/S avec EBS peuvent potentiellement être effectués pour essayer de supprimer les disques locaux de l&#39;équation.

Plus de détails sur la configuration de test, les résultats des tests, des scripts utilisés, et les données générées au cours de l&#39;essai sont disponibles sur [ce](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://github.com/arronax/scratch/tree/master/performance/graviton2-postgres)[GitHub](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://github.com/arronax/scratch/tree/master/performance/graviton2-postgres)[repo](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://github.com/arronax/scratch/tree/master/performance/graviton2-postgres) .

### Résumé

Il n&#39;y a pas eu beaucoup de cas où l&#39;instance ARM est devenue plus lente que l&#39; instance x86 dans les tests que nous avons effectués. Les résultats des tests étaient cohérents tout au long des tests des deux derniers jours. Alors que l&#39;instance basée sur ARM est 25 % moins chère, elle est capable d&#39;afficher un gain de performances de 15 à 20 % dans la plupart des tests par rapport aux instances x86 correspondantes. Ainsi, les instances basées sur ARM offrent un rapport qualité-prix nettement meilleur à tous égards. Nous devrions nous attendre à ce que de plus en plus de fournisseurs de cloud fournissent des instances basées sur ARM à l&#39;avenir. S&#39;il vous plaît laissez-nous savoir si vous souhaitez voir un autre type de tests de référence.

Source : [Percona](https://translate.google.com/translate?hl=en&amp;prev=_t&amp;sl=en&amp;tl=fr&amp;u=https://www.percona.com/blog/2021/01/22/postgresql-on-arm-based-aws-ec2-instances-is-it-any-good/)