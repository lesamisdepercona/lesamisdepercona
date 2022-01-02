+++
title = "Une approche simple de dépannage usage elevé de CPU sur MySQL"
description = "Une approche simple de dépannage usage elevé de CPU sur MySQL par Juan Arruti et Leonardo Bacchi Fernandes"
author = "Francis"
date = 2021-07-09T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = ""
images = ["thumbnail/amisdepercona21-006.jpg"]
slug = "Une-approche-simple-de-dépannage-usage-CPU-MySQL"
+++


Un de nos clients a récemment demandé s&#39;il était possible d&#39;identifier, du côté de MySQL , la requête qui provoque une utilisation élevée du processeur sur son système. L&#39;utilisation d&#39;outils de système d&#39;exploitation simples pour trouver le coupable est une technique largement utilisée depuis longtemps par les administrateurs de base de données PostgreSQL et Oracle, mais cela n&#39;a pas fonctionné pour MySQL car historiquement, nous manquions d&#39;instrumentation pour faire correspondre un thread de système d&#39;exploitation avec un thread processlist - jusqu&#39;à récemment.

Percona a ajouté la prise en charge du mappage des identifiants de liste de processus aux identifiants de thread du système d&#39;exploitation via la colonne TID de la table information\_schema.processlist à partir de Percona Server pour MySQL 5.6.27. Avec la sortie de 5.7, MySQL a suivi sa propre implémentation en étendant la table PERFORMANCE\_SCHEMA.THREADS et en ajoutant une nouvelle colonne nommée THREAD\_OS\_ID, que Percona Server pour MySQL a adoptée à la place de la sienne, car elle reste généralement aussi proche de l&#39;upstream que possible.

L&#39;approche suivante est utile dans les cas où une requête surcharge un processeur particulier alors que d&#39;autres cœurs fonctionnent normalement. Pour les cas où il s&#39;agit d&#39;un problème général d&#39;utilisation du processeur, différentes méthodes peuvent être utilisées, comme celle décrite dans cet autre article de blog [Réduire le processeur élevé sur MySQL : une étude de cas ](https://www.percona.com/blog/2019/03/07/reducing-high-cpu-on-mysql-a-case-study/)

Comment pouvons-nous utiliser cette nouvelle colonne pour savoir quelle session utilise le plus de ressources CPU dans ma base de données ?

Utilisons un exemple :

Pour résoudre les problèmes de CPU, nous pouvons utiliser plusieurs outils, tels que top ou pidstat (nécessite le package sysstat ). Dans l&#39;exemple suivant, nous utiliserons pidstat . L&#39;outil a une option (-t) qui change sa vue de processus (la valeur par défaut) en threads, où il affiche les threads associés au sein d&#39;un processus donné. Nous pouvons l&#39;utiliser pour savoir quel thread consomme le plus de CPU sur notre serveur. Ajout du paramètre -p avec l&#39;identifiant du processus mysql afin que l&#39;outil n&#39;affiche que les threads MySQL, ce qui nous permet de résoudre plus facilement les problèmes. Le dernier paramètre (1) est d&#39;afficher un échantillon par seconde :

La commande est pidstat -t -p < mysqld\_pid > 1 :   

```
shell> pidstat -t -p 31258 1
03:31:06 PM   UID      TGID       TID    %usr %system  %guest    %CPU   CPU  Command
[...]
03:31:07 PM 10014         -     32039    5.00    1.00    0.00    6.00    22  |__mysqld
03:31:07 PM 10014         -     32040    5.00    1.00    0.00    6.00    23  |__mysqld
03:31:07 PM 10014         -     32042    6.00    1.00    0.00    7.00     8  |__mysqld
03:31:07 PM 10014         -     32047    5.00    1.00    0.00    6.00     6  |__mysqld
03:31:07 PM 10014         -     32048    5.00    1.00    0.00    6.00    15  |__mysqld
03:31:07 PM 10014         -     32049    5.00    1.00    0.00    6.00    14  |__mysqld
03:31:07 PM 10014         -     32052    5.00    1.00    0.00    6.00    14  |__mysqld
03:31:07 PM 10014         -     32053   94.00    0.00    0.00   94.00     9  |__mysqld
03:31:07 PM 10014         -     32055    4.00    1.00    0.00    5.00    10  |__mysqld
03:31:07 PM 10014         -      4275    5.00    1.00    0.00    6.00    10  |__mysqld
03:31:07 PM 10014         -      4276    5.00    1.00    0.00    6.00     7  |__mysqld
03:31:07 PM 10014         -      4277    6.00    1.00    0.00    7.00    15  |__mysqld
03:31:07 PM 10014         -      4278    5.00    1.00    0.00    6.00    18  |__mysqld
03:31:07 PM 10014         -      4279    5.00    1.00    0.00    6.00    10  |__mysqld
03:31:07 PM 10014         -      4280    5.00    1.00    0.00    6.00    12  |__mysqld
03:31:07 PM 10014         -      4281    5.00    1.00    0.00    6.00    11  |__mysqld
03:31:07 PM 10014         -      4282    4.00    1.00    0.00    5.00     2  |__mysqld
03:31:07 PM 10014         -     35261    0.00    0.00    0.00    0.00     4  |__mysqld
03:31:07 PM 10014         -     36153    0.00    0.00    0.00    0.00     5  |__mysqld
```

Nous pouvons voir que le thread 32053 consomme le plus de CPU de loin, et nous nous sommes assurés de vérifier que la consommation est constante sur plusieurs échantillons de pidstat . En utilisant ces informations, nous pouvons nous connecter à la base de données et utiliser la requête suivante pour savoir quel thread MySQL est le fautif:  

```
mysql > select * from performance_schema.threads where THREAD_OS_ID = 32053 \G
*************************** 1. row ***************************
          THREAD_ID: 686
               NAME: thread/sql/one_connection
               TYPE: FOREGROUND
     PROCESSLIST_ID: 590
   PROCESSLIST_USER: msandbox
   PROCESSLIST_HOST: localhost
     PROCESSLIST_DB: NULL
PROCESSLIST_COMMAND: Query
   PROCESSLIST_TIME: 0
  PROCESSLIST_STATE: Sending data
   PROCESSLIST_INFO: select * from test.joinit where b = 'a a eveniet ut.'
   PARENT_THREAD_ID: NULL
               ROLE: NULL
       INSTRUMENTED: YES
            HISTORY: YES
    CONNECTION_TYPE: SSL/TLS
       THREAD_OS_ID: 32053
1 row in set (0.00 sec)
```

C&#39;est parti ! Nous savons maintenant que la consommation élevée du processeur provient d&#39;une requête dans la table joinit , exécutée par l&#39;utilisateur msandbox depuis localhost dans le test de la base de données. En utilisant ces informations, nous pouvons dépanner la requête et vérifier le plan d&#39;exécution avec la commande EXPLAIN pour voir s&#39;il y a place à amélioration.  

```
mysql > explain select * from test.joinit where b = 'a a eveniet ut.' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: joinit
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 7170836
     filtered: 10.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

Dans ce cas, c&#39;était un simple index qui manquait !  

```
mysql > alter table test.joinit add index (b) ;
Query OK, 0 rows affected (15.18 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

Après avoir créé l&#39;index, nous ne voyons plus de pics de processeur :  

```
shell> pidstat -t -p 31258 1
03:37:53 PM   UID      TGID       TID    %usr %system  %guest    %CPU   CPU  Command
[...]
03:37:54 PM 10014         -     32039   25.00    6.00    0.00   31.00     0  |__mysqld
03:37:54 PM 10014         -     32040   25.00    5.00    0.00   30.00    21  |__mysqld
03:37:54 PM 10014         -     32042   25.00    6.00    0.00   31.00    20  |__mysqld
03:37:54 PM 10014         -     32047   25.00    4.00    0.00   29.00    23  |__mysqld
03:37:54 PM 10014         -     32048   25.00    7.00    0.00   32.00    22  |__mysqld
03:37:54 PM 10014         -     32049   23.00    6.00    0.00   29.00     4  |__mysqld
03:37:54 PM 10014         -     32052   23.00    7.00    0.00   30.00    14  |__mysqld
03:37:54 PM 10014         -     32053   10.00    2.00    0.00   12.00    11  |__mysqld
03:37:54 PM 10014         -     32055   24.00    6.00    0.00   30.00     1  |__mysqld
03:37:54 PM 10014         -      4275   25.00    6.00    0.00   31.00     7  |__mysqld
03:37:54 PM 10014         -      4276   25.00    6.00    0.00   31.00     1  |__mysqld
03:37:54 PM 10014         -      4277   24.00    5.00    0.00   29.00    14  |__mysqld
03:37:54 PM 10014         -      4278   24.00    6.00    0.00   30.00     9  |__mysqld
03:37:54 PM 10014         -      4279   25.00    5.00    0.00   30.00     6  |__mysqld
03:37:54 PM 10014         -      4280   26.00    5.00    0.00   31.00    14  |__mysqld
03:37:54 PM 10014         -      4281   24.00    6.00    0.00   30.00    10  |__mysqld
03:37:54 PM 10014         -      4282   25.00    6.00    0.00   31.00    10  |__mysqld
03:37:54 PM 10014         -     35261    0.00    0.00    0.00    0.00     4  |__mysqld
03:37:54 PM 10014         -     36153    0.00    0.00    0.00    0.00     5  |__mysqld
```

Pourquoi ne pas utiliser cette approche pour résoudre les problèmes d&#39;E/S et de mémoire ?

Le problème avec la mesure des threads IO du côté du système d&#39;exploitation est que la plupart des opérations MySQL IO sont effectuées par des threads d&#39;arrière-plan, tels que les threads read, write et nettoyeur de page. Pour mesurer les threads IO, vous pouvez utiliser des outils tels que pidstat avec l&#39;option -d (IO au lieu de CPU) ou iostat avec -H (par thread). Si vous avez un thread très gourmand en E/S, vous pourrez peut-être le voir, mais sachez que les résultats peuvent être trompeurs en raison des opérations de thread en arrière-plan.

La consommation de mémoire est une ressource encore plus délicate à mesurer du côté du système d&#39;exploitation, car toute la mémoire est allouée sous le processus MySQL , et puisque c&#39;est MySQL qui gère son accès mémoire, il est transparent pour le système d&#39;exploitation quel thread consomme le plus de mémoire. Pour cela, nous pouvons utiliser l&#39; [instrumentation de mémoire perfomance\_schema disponible à partir de 5.7 ](https://dev.mysql.com/doc/refman/5.7/en/memory-summary-tables.html).

Conclusion

Il existe de nombreuses approches pour résoudre les problèmes d&#39;utilisation élevée du processeur, mais nous présentons ici une approche simple et largement utilisée sur les bases de données Oracle et PostgreSQL qui peut être adaptée à MySQL , à partir de la version 5.7. En traçant la consommation des threads du système d&#39;exploitation jusqu&#39;au côté de la base de données, nous pouvons détecter rapidement les requêtes gourmandes en CPU qui affectent les performances du système.

Page source : [Percona](https://www.percona.com/blog/2020/04/23/a-simple-approach-to-troubleshooting-high-cpu-in-mysql/)
