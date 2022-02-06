+++
title = "PostgreSQL : Pourquoi Linux HugePages est super important pour les serveurs de base de données?"
description = "Apprendre comment Linux Huge Pages et ses améliorations peuvent potentiellement sauver le serveur de base de données des OOM Killers et des plantages associés"
author = "Francis"
date = 2022-02-04
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article04.jpg"
images = ["thumbnail2022/article04.jpg"]
slug = "postgresql-utilite-de-linux-hugepages"
+++

![thumbnail](/thumbnail2022/article04.jpg)

Souvent, les utilisateurs viennent nous voir avec des incidents de plantage de base de données dus à OOM Killer. Le Out Of Memory Killer met fin aux processus PostgreSQL et reste la principale raison de la plupart des plantages de la base de données PostgreSQL qui nous sont signalés. Il peut y avoir plusieurs raisons pour lesquelles une machine hôte peut manquer de mémoire, et les problèmes les plus courants sont :

1. Mémoire mal réglée sur la machine hôte.
1. Une valeur élevée de work\_mem est spécifiée globalement (au niveau de l'instance). Les utilisateurs sous-estiment souvent l'effet multiplicateur de telles décisions globales.
1. Le nombre élevé de connexions. Les utilisateurs ignorent le fait que même une connexion non active peut contenir une bonne quantité d'allocation de mémoire.
1. D'autres programmes co-hébergés sur la même machine consomment des ressources.

Même si nous avions l'habitude d'aider à régler à la fois les machines hôtes et les bases de données, nous ne prenons pas toujours le temps d'expliquer comment et pourquoi les HugePages sont importantes et de le justifier avec des données. Grâce aux sondages répétés de mon ami et collègue [Fernando ](https://www.percona.com/blog/author/fernando-laudares/), je n'ai pas pu m'empêcher de le faire cette fois.

## Le problème

Permettez-moi d'expliquer le problème avec un cas testable et reproductible. Cela pourrait être utile si quelqu'un veut tester le cas à sa manière.

### Environnement d'essai

La machine de test est équipée de 40 cœurs de processeur (80 vCPU ) et de 192 Go de mémoire installée. Je ne veux pas surcharger ce serveur avec trop de connexions, donc seulement 80 connexions sont utilisées pour le test. Oui, seulement 80 connexions, ce à quoi nous devrions nous attendre dans n'importe quel environnement et c'est très réaliste. Transparent HugePages (THP) est désactivé. Je ne veux pas détourner le sujet en expliquant pourquoi ce n'est pas une bonne idée d'avoir THP pour un serveur de base de données, mais je m'engage à préparer un autre blog.

Afin d'avoir une connexion relativement persistante, tout comme celles des poolers côté application (ou même des poolers de connexions externes), pgBouncer est utilisé pour rendre les 80 connexions persistantes tout au long des tests. La suite est la configuration pgBouncer utilisée :

```
[databases]
sbtest2 = host=localhost port=5432 dbname=sbtest2

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /tmp/pgbouncer.log
pidfile = /tmp/pgbouncer.pid
admin_users = postgres
default_pool_size=100
min_pool_size=80
server_lifetime=432000
```
Comme nous pouvons le voir, le paramètre server\_lifetime est spécifié à une valeur élevée pour ne pas détruire la connexion du pooler à PostgreSQL. Après PostgreSQL , des modifications de paramètres sont incorporées pour imiter certains des paramètres d'environnement client courants.

```
logging_collector = 'on'
max_connections = '1000'
work_mem = '32MB'
checkpoint_timeout = '30min'
checkpoint_completion_target = '0.92'
shared_buffers = '138GB'
shared_preload_libraries = 'pg_stat_statements'
```

La charge de test est créée à l'aide de sysbench

```
sysbench /usr/share/sysbench/oltp_point_select.lua --db-driver=pgsql --pgsql-host=localhost --pgsql-port=6432 --pgsql-db=sbtest2 --pgsql-user=postgres --pgsql-password=vagrant --threads=80 --report-interval=1 --tables=100 --table-size=37000000 prepare
```

et puis

```
sysbench /usr/share/sysbench/oltp_point_select.lua --db-driver=pgsql --pgsql-host=localhost --pgsql-port=6432 --pgsql-db=sbtest2 --pgsql-user=postgres --pgsql-password=vagrant --threads=80 --report-interval=1 --time=86400  --tables=80 --table-size=37000000  run
```

La première étape de préparation place une charge en écriture sur le serveur et la seconde une charge en lecture seule.

Je n'essaie pas d'expliquer la théorie et les concepts derrière HugePages , mais je me concentre sur l'analyse d'impact. Veuillez vous référer à l'article LWN : [Five-Level Page Tables ](https://lwn.net/Articles/717293/)et l’article d'Andres Freund [Measuring the Memory Overhead of a Postgres Connection ](https://blog.anarazel.de/2020/10/07/measuring-the-memory-overhead-of-a-postgres-connection/)pour comprendre certains concepts.

## Test d’Observations

Pendant le test, la consommation de mémoire a été vérifiée à l'aide de la commande de l'utilitaire free Linux. Lors de l'utilisation du pool régulier de pages de mémoire, la consommation a commencé avec une valeur très faible. Mais il était en augmentation constante (voir la capture d'écran ci-dessous). La mémoire « disponible » est épuisée plus rapidement.

![image01](/posts/2022/article04/img01.png)


Vers la fin, il a également commencé l'activité d'échange. L'activité d'échange est capturée dans la sortie vmstat ci-dessous :

![image02](/posts/2022/article04/img02.png)

Informations de `/proc/meminfo` révèlent que la taille totale de la table de pages est passée de **25+Go à 45 Mo** .

![image03](/posts/2022/article04/img03.png)

Ce n'est pas seulement un gaspillage de mémoire ; il s'agit d'une surcharge énorme qui a un impact sur l'exécution globale du programme et du système d'exploitation. Cette taille correspond au **total des entrées inférieures de la PageTable** des 80+ processus PostgreSQL .

La même chose peut être vérifiée en vérifiant chaque processus PostgreSQL . Voici un exemple:

![image04](/posts/2022/article04/img04.png)

Ainsi, la taille totale de PageTable ( **25 Go** ) doit être d'environ cette valeur \* 80 (connexions). Étant donné que ce benchmark synthétique envoie une charge de travail presque similaire à travers toutes les connexions, tous les processus individuels ont des valeurs très proches de ce qui a été capturé ci-dessus.

La ligne Shell suivante peut être utilisée pour vérifier le Pss (Proportional set size). Étant donné que PostgreSQL utilise la mémoire partagée Linux, se concentrer sur Rss n'a aucun sens.

```
for PID in $(pgrep "postgres|postmaster") ; do awk '/Pss/ {PSS+=$2} END{getline cmd < "/proc/'$PID'/cmdline"; sub("\0", " ", cmd);printf "%.0f --> %s (%s)\n", PSS, cmd, '$PID'}' /proc/$PID/smaps ; done|sort -n
```


Sans informations Pss, il n'y a pas de méthode simple pour comprendre la responsabilité de la mémoire par processus.

Dans un système de base de données typique où nous avons une charge DML considérable, les processus d'arrière-plan de PostgreSQL tels que Checkpointer , Background Writer ou Autovaccum toucheront beaucoup de pages dans la mémoire partagée. Le Pss correspondant sera plus élevé pour ces processus.

![image05](/posts/2022/article04/img05.png)


**Cela devrait expliquer pourquoi Checkpointer , Background worker ou même le Postmaster devient souvent la victime/cible habituelle d'un OOM Killer. Comme nous pouvons le voir ci-dessus, ils portent la plus grande responsabilité de la mémoire partagée.**

Après plusieurs heures d'exécution, la session individuelle touchait plus de pages de mémoire partagée. En conséquence, par processus, les valeurs Pss ont été réorganisées : Checkpointer est moins responsable car les autres sessions partagent la responsabilité.

![image06](/posts/2022/article04/img06.png)

Cependant, le point de contrôle conserve la part la plus élevée.

Même si ce n'est pas important pour ce test, il convient de mentionner que ce type de modèle de charge est spécifique au benchmarking synthétique car chaque session fait à peu près le même travail. Ce n'est pas une bonne approximation de la charge d'application typique, où nous voyons généralement les pointeurs de contrôle et les rédacteurs d'arrière-plan porter la responsabilité principale.

## La solution : Activer les HugePages

La solution à ces tables de pages gonflées et aux problèmes associés consiste à utiliser HugePages à la place. Nous pouvons déterminer la quantité de mémoire à allouer à HugePages en vérifiant le VmPeak du processus postmaster. Par exemple, si 4357 est le PID du postmaster :

```
grep ^VmPeak /proc/4357/status
```

Cela donne la quantité de mémoire requise en Ko :

```
VmPeak: 148392404 kB
```
Tout cela doit s'intégrer dans d'énormes pages. Conversion de cette valeur en pages de 2 Mo :

```
postgres=# select 148392404/1024/2;
?column?
----------
    72457
(1 row)
```

Spécifiez cette valeur dans `/etc/sysctl.conf` pour `vm.nr\_hugepages` , par exemple :

```
vm.nr_hugepages = 72457
```

Arrêtons maintenant l'instance PostgreSQL et j’execute :

```
sysctl -p
```

Je dois vérifier si le nombre de huge pages demandé est créé ou non :

```
grep ^Huge /proc/meminfo
HugePages_Total:   72457
HugePages_Free:    72457
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:        148391936 kB
```

Si nous démarrons PostgreSQL à ce stade, nous pourrions voir que HugePages\_Rsvd est alloué.

```
$ grep ^Huge /proc/meminfo
HugePages_Total:   72457
HugePages_Free:    70919
HugePages_Rsvd:    70833
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:        148391936 kB
```

Si tout va bien, je préférerais m'assurer que PostgreSQL utilise toujours HugePages . car je préférerais un échec de démarrage de PostgreSQL plutôt que des problèmes/crashs plus tard.

```
postgres=# ALTER SYSTEM SET huge_pages = on;
```

La modification ci-dessus nécessite un redémarrage de l' instance PostgreSQL .

## Tests avec HugePages "ON"

Les HugePages sont créées à l'avance avant même le démarrage de PostgreSQL . PostgreSQL les alloue et les utilise. Il n'y aura donc même pas de changement notable dans la sortie `free` avant et après le démarrage. PostgreSQL alloue sa mémoire partagée dans ces HugePages si elles sont déjà disponibles. PostgreSQL `shared\_buffers` est le plus gros occupant de cette mémoire partagée.


![image07](/posts/2022/article04/img07.png)


La première sortie de `free - h` dans la capture d'écran ci-dessus est générée avant le démarrage de PostgreSQL et la seconde après le démarrage de PostgreSQL . Comme nous pouvons le voir, il n'y a pas de changement notable

J'ai fait le même test qui a duré plusieurs heures et il n'y a eu aucun changement; le seul changement notable, même après de nombreuses heures d'exécution, est le déplacement de la mémoire "libre" vers le cache du système de fichiers , ce qui est attendu et ce que nous voulons réaliser. La mémoire totale "disponible" est restée à peu près constante, comme nous pouvons le voir dans la capture d'écran suivante.

![image08](/posts/2022/article04/img08.png)

La taille totale des tableaux de pages est restée à peu près la même :

![image09](/posts/2022/article04/img09.png)

Comme nous pouvons le voir, la différence est énorme : seulement 61 Mo avec HugePages au lieu de 25 + Go auparavant. Le pss par session a également été considérablement réduit :

![image10](/posts/2022/article04/img10.png)


Le plus gros avantage que j'ai pu observer est que CheckPointer ou Background Writer n'est plus responsable de plusieurs Go de RAM.

![image11](/posts/2022/article04/img11.png)


Au lieu de cela, ils ne sont responsables que de quelques Mo de consommation. De toute évidence, ils ne seront plus une victime potentielle de l'OOM Killer.

## Conclusion

Dans cet article de blog, nous avons expliqué comment Linux Huge Pages peut potentiellement sauver le serveur de base de données des OOM Killers et des plantages associés. Nous pouvait voir deux améliorations :

1. La consommation globale de mémoire a été réduite par une grande marge. Sans HugePages, le serveur manquait presque de mémoire (mémoire disponible complètement épuisée et activité d'échange démarrée). Cependant, une fois que nous sommes passés à HugePages, 38 à 39 Go sont restés en tant que cache du système de fichiers disponible/Linux . C'est un énorme économie .
1. Lorsque HugePages est activé, les processus d'arrière-plan PostgreSQL ne sont pas pris en compte pour une grande quantité de mémoire partagée. Ainsi, ils ne seront pas facilement des candidats victimes/cibles pour OOM Killer.

Ces améliorations peuvent potentiellement sauver le système s'il est au bord de la condition OOM, mais je ne veux pas prétendre que cela protégera la base de données de toutes les conditions OOM pour toujours.

HugePage (hugetlbfs ) a atterri à l'origine dans le noyau Linux en 2002 pour répondre aux exigences des systèmes de bases de données qui doivent traiter une grande quantité de mémoire. J'ai pu voir que les objectifs de conception sont toujours valables.

HugePages présente d'autres avantages indirects supplémentaires :

1. Les HugePages ne sont jamais échangées. Lorsque les tampons partagés PostgreSQL sont dans HugePages , cela peut produire des performances plus cohérentes et prévisibles. je vais discuter cela dans un autre article.
1. Linux utilise une méthode [de recherche de page à plusieurs niveaux ](https://lwn.net/Articles/717293/). Les HugePages sont implémentées à l'aide de pointeurs directs vers les pages de la couche intermédiaire (une énorme page de 2 Mo se trouverait directement au niveau PMD, sans page PTE intermédiaire). La traduction d'adresse devient considérablement plus simple. Comme il s'agit d'une opération à haute fréquence dans un serveur de base de données avec une grande quantité de mémoire, les gains sont multipliés.

**Remarque :** les HugePages dont il est question dans cet article de blog concernent des pages énormes de taille fixe (2 Mo).

De plus, en passant, je tiens à mentionner qu'il y a beaucoup d'améliorations dans **Transparent HugePages (THP)** au fil des ans, ce qui permet aux applications d'utiliser HugePages sans aucune modification de code. THP est souvent considéré comme un remplacement des HugePages classiques (hugetlbfs ) pour une charge de travail générique. Cependant, l'utilisation de THP est déconseillée sur les systèmes de base de données car elle peut entraîner une fragmentation de la mémoire et des retards accrus. Je veux couvrir ce sujet dans un autre article et je veux juste mentionner qu'il ne s'agit pas d'un problème spécifique à PostgreSQL , mais qu'il affecte tous les systèmes de base de données. Par exemple,

1. Oracle recommande de désactiver TPH. [Lien de référence](https://docs.oracle.com/en/database/oracle/oracle-database/19/ladbi/disabling-transparent-hugepages.html#GUID-02E9147D-D565-4AF8-B12A-8E6E9F74BEEA)
1. MongoDB recommande de désactiver THP. [Lien de référence](https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/)  
1. "THP est connu pour entraîner une dégradation des performances avec PostgreSQL pour certains utilisateurs sur certaines versions de Linux." [Lien de référence](https://www.postgresql.org/docs/current/runtime-config-resource.html)

Alors que de plus en plus d'entreprises envisagent de migrer d'Oracle ou d'implémenter de nouvelles bases de données parallèlement à leurs applications, PostgreSQL est souvent la meilleure option pour ceux qui souhaitent s'exécuter sur des bases de données open source.

Source : [Percona](https://www.percona.com/blog/why-linux-hugepages-are-super-important-for-database-servers-a-case-with-postgresql/) 

A lire : [Pourquoi les utilisateurs utilisent Percona for PostgreSQL](https://learn.percona.com/why-customers-choose-percona-postgres)
