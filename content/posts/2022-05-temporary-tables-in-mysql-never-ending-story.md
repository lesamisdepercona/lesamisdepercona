+++
title = "Tables temporaires dans MySQL - Une histoire sans fin?"
description = "Un résumé qui peut aider à dépanner l'utilisation des tables temporaires entre les principales versions de MySQL"
author = "Francis"
date = 2022-02-08
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article05.jpg"
images = ["thumbnail2022/article05.jpg"]
slug = "mysql-tables-temporaires"
+++

![thumbnail](/thumbnail2022/article05.jpg)

Si vous avez déjà dû faire face à des problèmes de performances et/ou d'espace disque liés aux tables temporaires, je parie que vous vous êtes finalement retrouvé perplexe. Il existe de nombreux scénarios possibles en fonction du type de table temporaire, des paramètres et de la version MySQL utilisée. Nous avons observé une assez longue évolution dans ce domaine pour plusieurs raisons. L'un d'eux était la nécessité d'éliminer complètement la nécessité d'utiliser le moteur obsolète MyISAM , et en même temps d'introduire des alternatives plus performantes et fiables. Un autre ensemble d'améliorations était nécessaire concernant InnoDB , où il était nécessaire de [réduire la surcharge ](https://docs.oracle.com/cd/E17952_01/mysql-5.7-relnotes-en/news-5-7-1.html)des tables temporaires à l'aide de ce moteur.

Pour cette raison, j'ai décidé de les rassembler dans une sorte de résumé qui peut aider à dépanner leur utilisation. En raison de vastes changements entre les principales versions de MySQL, j'ai divisé l'article en fonction de celles-ci.

## MySQL 5.6

(Si vous utilisez toujours cette version, nous vous encourageons à envisager de la mettre à niveau dès qu'elle a atteint la fin de [vie ](https://www.percona.com/blog/2020/12/07/not-ready-to-give-up-mysql-5-6-get-post-eol-support-from-percona/).)

### Tables temporaires créées par l'utilisateur

Lorsqu'une table est créée à l'aide de la clause CREATE TEMPORARY TABLE, elle utilisera le moteur défini par [default_tmp_storage_engine ](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_default_tmp_storage_engine)(par défaut à InnoDB ) s'il n'est pas explicitement défini autrement et sera stockée dans le répertoire défini par la variable [tmpdir ](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_tmpdir).

Un exemple peut ressembler à ceci :

```
mysql > create temporary table tmp1 (id int, a varchar(10));
Query OK, 0 rows affected (0.02 sec)

mysql > show create table tmp1\G
*************************** 1. row ***************************
Table: tmp1
Create Table: CREATE TEMPORARY TABLE `tmp1` (
`id` int(11) DEFAULT NULL,
`a` varchar(10) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
```

Mais comment trouver le fichier créé sur disque qui stocke les données de cette table ? Bien que cette requête puisse aider :

```
mysql > select table_id,space,name,path from information_schema.INNODB_SYS_DATAFILES join information_schema.INNODB_SYS_TABLES using (space) where name like '%tmp%'\G
*************************** 1. row ***************************
table_id: 21
space: 7
name: tmp/#sql11765a_2_1
path: /data/sandboxes/msb_5_6_51/tmp/#sql11765a_2_1.ibd
1 row in set (0.00 sec)
```

Nous ne voyons pas le nom de la table d'origine ici. Même en regardant le pool de mémoire tampon, nous n'avons toujours pas le vrai nom :

```
mysql > select TABLE_NAME from information_schema.INNODB_BUFFER_PAGE where table_name like '%tmp%';
+-------------------------+
| TABLE_NAME              |
+-------------------------+
| `tmp`.`#sql11765a_2_1`  |
+-------------------------+
1 row in set (0.07 sec)
```

Voici l'extension disponible dans la variante Percona Server pour MySQL 5.6 - table information\_schema supplémentaire : [GLOBAL_TEMPORARY_TABLES ](https://www.percona.com/doc/percona-server/5.6/diagnostics/misc_info_schema_tables.html#GLOBAL_TEMPORARY_TABLES). Avec celui-ci, nous pouvons créer une requête qui fournit un peu plus d'informations :

```
mysql > select SPACE,TABLE_SCHEMA,TABLE_NAME,ENGINE,g.NAME,PATH from information_schema.GLOBAL_TEMPORARY_TABLES g LEFT JOIN information_schema.INNODB_SYS_TABLES s ON s.NAME LIKE CONCAT('%', g.name, '%') LEFT JOIN information_schema.INNODB_SYS_DATAFILES USING(SPACE)\G
*************************** 1. row ***************************
SPACE: 16
TABLE_SCHEMA: test
TABLE_NAME: tmp1
ENGINE: InnoDB
NAME: #sql12c75d_2_0
PATH: /data/sandboxes/msb_ps5_6_47/tmp/#sql12c75d_2_0.ibd
*************************** 2. row ***************************
SPACE: NULL
TABLE_SCHEMA: test
TABLE_NAME: tmp3
ENGINE: MEMORY
NAME: #sql12c75d_2_2
PATH: NULL
*************************** 3. row ***************************
SPACE: NULL
TABLE_SCHEMA: test
TABLE_NAME: tmp2
ENGINE: MyISAM
NAME: #sql12c75d_2_1
PATH: NULL
3 rows in set (0.00 sec)
```

Ainsi, au moins pour la table temporaire InnoDB , nous pouvons corréler le nom exact de la table avec le chemin du fichier.

### Tables Temporaires Internes

Ce sont ceux créés par MySQL lors du processus d'exécution d'une requête. Nous n'avons pas accès à ces tables, mais voyons comment nous pouvons étudier leur utilisation.

Ce type est créé en mémoire (à l'aide [du moteur MEMORY ](https://dev.mysql.com/doc/refman/5.6/en/memory-storage-engine.html#memory-storage-engine-user-created-and-temporary-tables)) tant que sa taille ne dépasse pas les variables tmp\_table\_size ou max\_heap\_table\_size , et si aucune colonne TEXT/BLOB n'est utilisée. Si une telle table doit être stockée sur le disque, dans MySQL 5.6, elle utilisera le stockage MyISAM et également tmpdir comme emplacement. Exemple rapide, sur une table sysbench de 10 millions de lignes, requête produisant une grande table temporaire interne :

```
mysql > SELECT pad, COUNT(*) FROM sbtest1 GROUP BY pad;
```

Et nous pouvons voir les fichiers associés grandir:

```
$ ls -lh /data/sandboxes/msb_5_6_51/tmp/
total 808M
-rw-rw---- 1 przemek przemek 329M Sep 29 23:24 '#sql_11765a_0.MYD'
-rw-rw---- 1 przemek przemek 479M Sep 29 23:24 '#sql_11765a_0.MYI'
```

Cependant, il peut être difficile de corréler une table temporaire particulière et sa connexion client. Les seules informations que j'ai trouvées sont :

```
mysql > select FILE_NAME,EVENT_NAME from performance_schema.file_summary_by_instance where file_name like '%tmp%' \G
*************************** 1. row ***************************
FILE_NAME: /data/sandboxes/msb_5_6_51/tmp/Innodb Merge Temp File
EVENT_NAME: wait/io/file/innodb/innodb_temp_file
*************************** 2. row ***************************
FILE_NAME: /data/sandboxes/msb_5_6_51/tmp/#sql_11765a_0.MYI
EVENT_NAME: wait/io/file/myisam/kfile
*************************** 3. row ***************************
FILE_NAME: /data/sandboxes/msb_5_6_51/tmp/#sql_11765a_0.MYD
EVENT_NAME: wait/io/file/myisam/dfile
3 rows in set (0.00 sec)
```

## MySQL 5.7

### Tables temporaires créées par l'utilisateur

Comme précédemment, la variable default\_tmp\_storage\_engine décide du moteur utilisé. Mais deux changements se sont produits ici. Les tables temporaires InnoDB utilisent désormais un espace de table [partagé dédié commun - ibtmp1 ](https://dev.mysql.com/doc/refman/5.7/en/innodb-temporary-tablespace.html) sauf s'il est compressé. De plus, nous avons une vue information\_schema supplémentaire : INNODB\_TEMP\_TABLE\_INFO. Cela dit , nous pouvons obtenir des informations comme ci- dessous :

```
mysql > select name, FILE_NAME, FILE_TYPE, TABLESPACE_NAME, SPACE, PER_TABLE_TABLESPACE, IS_COMPRESSED from INFORMATION_SCHEMA.INNODB_TEMP_TABLE_INFO join INFORMATION_SCHEMA.FILES on FILE_ID=SPACE\G
*************************** 1. row ***************************
name: #sql12cf58_2_5
FILE_NAME: ./ibtmp1
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_temporary
SPACE: 109
PER_TABLE_TABLESPACE: FALSE
IS_COMPRESSED: FALSE
*************************** 2. row ***************************
name: #sql12cf58_2_4
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_4.ibd
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_file_per_table_110
SPACE: 110
PER_TABLE_TABLESPACE: TRUE
IS_COMPRESSED: TRUE
2 rows in set (0.01 sec)
```

Mais encore une fois, pour établir une corrélation avec un nom de table, l'extension Percona Server for MySQL doit être utilisée :

```
mysql > select g.TABLE_SCHEMA, g.TABLE_NAME, name, FILE_NAME, FILE_TYPE, TABLESPACE_NAME, SPACE, PER_TABLE_TABLESPACE, IS_COMPRESSED from INFORMATION_SCHEMA.INNODB_TEMP_TABLE_INFO join INFORMATION_SCHEMA.FILES on FILE_ID=SPACE join information_schema.GLOBAL_TEMPORARY_TABLES g using (name)\G
*************************** 1. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp1
name: #sql12cf58_2_5
FILE_NAME: ./ibtmp1
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_temporary
SPACE: 109
PER_TABLE_TABLESPACE: FALSE
IS_COMPRESSED: FALSE
*************************** 2. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp3
name: #sql12cf58_2_4
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_4.ibd
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_file_per_table_110
SPACE: 110
PER_TABLE_TABLESPACE: TRUE
IS_COMPRESSED: TRUE
2 rows in set (0.01 sec)
```

Alternativement, voir aussi MyISAM et les fichiers . frm , nous pouvons utiliser :

```
mysql > SELECT g.TABLE_SCHEMA, g.TABLE_NAME, NAME, f.FILE_NAME, g.ENGINE, TABLESPACE_NAME, PER_TABLE_TABLESPACE, SPACE FROM information_schema.GLOBAL_TEMPORARY_TABLES g join performance_schema.file_instances f ON FILE_NAME LIKE CONCAT('%', g.name, '%') left join INFORMATION_SCHEMA.INNODB_TEMP_TABLE_INFO using (name) left join INFORMATION_SCHEMA.FILES fl on space=FILE_ID order by table_name\G
*************************** 1. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp1
NAME: #sql12cf58_2_5
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_5.frm
ENGINE: InnoDB
TABLESPACE_NAME: innodb_temporary
PER_TABLE_TABLESPACE: FALSE
SPACE: 109
*************************** 2. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp2
NAME: #sql12cf58_2_6
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_6.MYD
ENGINE: MyISAM
TABLESPACE_NAME: NULL
PER_TABLE_TABLESPACE: NULL
SPACE: NULL
*************************** 3. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp2
NAME: #sql12cf58_2_6
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_6.MYI
ENGINE: MyISAM
TABLESPACE_NAME: NULL
PER_TABLE_TABLESPACE: NULL
SPACE: NULL
*************************** 4. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp2
NAME: #sql12cf58_2_6
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_6.frm
ENGINE: MyISAM
TABLESPACE_NAME: NULL
PER_TABLE_TABLESPACE: NULL
SPACE: NULL
*************************** 5. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp3
NAME: #sql12cf58_2_4
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_4.frm
ENGINE: InnoDB
TABLESPACE_NAME: innodb_file_per_table_110
PER_TABLE_TABLESPACE: TRUE
SPACE: 110
*************************** 6. row ***************************
TABLE_SCHEMA: test
TABLE_NAME: tmp3
NAME: #sql12cf58_2_4
FILE_NAME: /data/sandboxes/msb_ps5_7_33/tmp/#sql12cf58_2_4.ibd
ENGINE: InnoDB
TABLESPACE_NAME: innodb_file_per_table_110
PER_TABLE_TABLESPACE: TRUE
SPACE: 110
6 rows in set (0.01 sec)
```

### Tableaux Temporaires Interne

Pour les tables temporaires internes dans 5.7, c'est similaire en termes de tables en mémoire. Mais le moteur par défaut pour les tables temporaires sur disque est défini via une nouvelle variable : [internal_tmp_disk_storage_engine ](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_internal_tmp_disk_storage_engine), qui est désormais également par défaut InnoDB , et le tablespace ibtmp1 est également utilisé pour stocker son contenu.

L'aperçu de cet espace de table temporaire partagé est assez limité. Nous pouvons vérifier sa taille et combien d'espace libre est actuellement disponible. Un exemple voir a été pris pendant lourd requête en cours :

```
mysql > select FILE_NAME, FILE_TYPE, TABLESPACE_NAME, ENGINE, TOTAL_EXTENTS, FREE_EXTENTS, EXTENT_SIZE/1024/1024 as 'extent in MB', MAXIMUM_SIZE from INFORMATION_SCHEMA.FILES where file_name like '%ibtmp%'\G
*************************** 1. row ***************************
FILE_NAME: ./ibtmp1
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_temporary
ENGINE: InnoDB
TOTAL_EXTENTS: 588
FREE_EXTENTS: 1
extent in MB: 1.00000000
MAXIMUM_SIZE: NULL
1 row in set (0.00 sec)
```

Et une fois la requête terminée, nous pouvons voir que la majeure partie de l'espace est libérée (FREE\_EXTENTS) :

```
mysql > select FILE_NAME, FILE_TYPE, TABLESPACE_NAME, ENGINE, TOTAL_EXTENTS, FREE_EXTENTS, EXTENT_SIZE/1024/1024 as 'extent in MB', MAXIMUM_SIZE from INFORMATION_SCHEMA.FILES where file_name like '%ibtmp%'\G
*************************** 1. row ***************************
FILE_NAME: ./ibtmp1
FILE_TYPE: TEMPORARY
TABLESPACE_NAME: innodb_temporary
ENGINE: InnoDB
TOTAL_EXTENTS: 780
FREE_EXTENTS: 764
extent in MB: 1.00000000
MAXIMUM_SIZE: NULL
1 row in set (0.00 sec)
```

Cependant, le tablespace ne sera pas tronqué à moins que MySQL ne soit redémarré :

```
$ ls -lh msb_5_7_35/data/ibtmp*
-rw-r----- 1 przemek przemek 780M Sep 30 19:50 msb_5_7_35/data/ibtmp1
```

Pour voir l'activité d'écriture (qui peut s'avérer beaucoup plus élevée pour une seule requête que la croissance totale de la taille réalisée par celle-ci) :

```
mysql > select FILE_NAME, SUM_NUMBER_OF_BYTES_WRITE/1024/1024/1024 as GB_written from performance_schema.file_summary_by_instance where file_name like '%ibtmp%' \G
*************************** 1. row ***************************
FILE_NAME: /data/sandboxes/msb_5_7_35/data/ibtmp1
GB_written: 46.925933837891
1 row in set (0.00 sec)
```

## MySQL 8.0

Pour plus de simplicité, ignorons comment les choses fonctionnaient avant la version 8.0.16 et discutons uniquement de la façon dont cela fonctionne depuis lors, car les changements en la matière sont assez importants :

- [internal_tmp_disk_storage_engine ](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_internal_tmp_mem_storage_engine) a été supprimée et il n'est plus possible d'utiliser le moteur MyISAM pour les tables temporaires internes
- l'espace table partagé ibtmp1 n'est plus utilisé pour aucun des types de table temporaire
- un pool de nouveaux [tablespaces temporaires de session ](https://dev.mysql.com/doc/refman/8.0/en/innodb-temporary-tablespace.html) a été introduit pour gérer à la fois les tables temporaires utilisateur et internes sur le disque et est situé par défaut dans le répertoire de données principal
- le nouveau moteur [TempTable ](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_TEMPTABLE.html) pour les tables en mémoire utilise à la fois l'espace en mémoire et les fichiers mmappés sur le disque

### Tables temporaires créées par l'utilisateur

Pour un exemple de table temporaire :

```
mysql > create temporary table tmp1 (id int, a varchar(10));
Query OK, 0 rows affected (0.00 sec)

mysql > select * from information_schema.INNODB_TEMP_TABLE_INFO;
+----------+----------------+--------+------------+
| TABLE_ID | NAME           | N_COLS | SPACE      |
+----------+----------------+--------+------------+
|     1089 | #sqlbbeb3_a_12 |      5 | 4243767289 |
+----------+----------------+--------+------------+
1 row in set (0.00 sec)
```

Nous pouvons corréler quel fichier a été utilisé à partir de ce pool en regardant le numéro d'espace :

```
mysql > select * from INFORMATION_SCHEMA.INNODB_SESSION_TEMP_TABLESPACES ;
+----+------------+----------------------------+-------+----------+-----------+
| ID | SPACE      | PATH                       | SIZE  | STATE    | PURPOSE   |
+----+------------+----------------------------+-------+----------+-----------+
| 10 | 4243767290 | ./#innodb_temp/temp_10.ibt | 81920 | ACTIVE   | INTRINSIC |
| 10 | 4243767289 | ./#innodb_temp/temp_9.ibt  | 98304 | ACTIVE   | USER      |
|  0 | 4243767281 | ./#innodb_temp/temp_1.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767282 | ./#innodb_temp/temp_2.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767283 | ./#innodb_temp/temp_3.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767284 | ./#innodb_temp/temp_4.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767285 | ./#innodb_temp/temp_5.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767286 | ./#innodb_temp/temp_6.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767287 | ./#innodb_temp/temp_7.ibt  | 81920 | INACTIVE | NONE      |
|  0 | 4243767288 | ./#innodb_temp/temp_8.ibt  | 81920 | INACTIVE | NONE      |
+----+------------+----------------------------+-------+----------+-----------+
10 rows in set (0.00 sec)
```

Mais encore une fois, aucun moyen de rechercher le nom de la table. Heureusement, Percona Server pour MySQL a toujours la table [GLOBAL_TEMPORARY_TABLES ](https://www.percona.com/doc/percona-server/8.0/diagnostics/misc_info_schema_tables.html#GLOBAL_TEMPORARY_TABLES), donc étant donné les trois vues système disponibles, nous pouvons obtenir de meilleures informations sur les tables temporaires créées par l'utilisateur en utilisant divers moteurs, comme ci-dessous :

```
mysql > SELECT SESSION_ID, SPACE, PATH, TABLE_SCHEMA, TABLE_NAME, SIZE, DATA_LENGTH, INDEX_LENGTH, ENGINE, PURPOSE FROM information_schema.GLOBAL_TEMPORARY_TABLES LEFT JOIN information_schema.INNODB_TEMP_TABLE_INFO USING(NAME) LEFT JOIN INFORMATION_SCHEMA.INNODB_SESSION_TEMP_TABLESPACES USING(SPACE)\G
*************************** 1. row ***************************
  SESSION_ID: 10
      SPACE: 4243767290
        PATH: ./#innodb_temp/temp_10.ibt
TABLE_SCHEMA: test
  TABLE_NAME: tmp3
        SIZE: 98304
DATA_LENGTH: 16384
INDEX_LENGTH: 0
      ENGINE: InnoDB
    PURPOSE: USER
*************************** 2. row ***************************
  SESSION_ID: 13
      SPACE: NULL
        PATH: NULL
TABLE_SCHEMA: test
  TABLE_NAME: tmp41
        SIZE: NULL
DATA_LENGTH: 24
INDEX_LENGTH: 1024
      ENGINE: MyISAM
    PURPOSE: NULL
*************************** 3. row ***************************
  SESSION_ID: 13
      SPACE: NULL
        PATH: NULL
TABLE_SCHEMA: test
  TABLE_NAME: tmp40
        SIZE: NULL
DATA_LENGTH: 128256
INDEX_LENGTH: 0
      ENGINE: MEMORY
    PURPOSE: NULL
*************************** 4. row ***************************
  SESSION_ID: 13
      SPACE: 4243767287
        PATH: ./#innodb_temp/temp_7.ibt
TABLE_SCHEMA: test
  TABLE_NAME: tmp33
        SIZE: 98304
DATA_LENGTH: 16384
INDEX_LENGTH: 0
      ENGINE: InnoDB
    PURPOSE: USER
4 rows in set (0.01 sec)
```

Semblable à ibtmp1, ces espaces de table ne sont pas tronqués en dehors du redémarrage de MySQL .

D'après ce qui précède, nous pouvons voir que la connexion utilisateur 10 a une table temporaire InnoDB ouverte et que la connexion 13 a trois tables temporaires utilisant trois moteurs différents.

### Tableaux Interne Temporaires

Lorsqu'une requête lourde est en cours d'exécution dans la connexion 10, nous pouvons obtenir les vues suivantes :

```
mysql > show processlist\G
...
*************************** 2. row ***************************
    Id: 10
  User: msandbox
  Host: localhost
    db: test
Command: Query
  Time: 108
  State: converting HEAP to ondisk
  Info: SELECT pad, COUNT(*) FROM sbtest1 GROUP BY pad

mysql > select * from performance_schema.memory_summary_global_by_event_name where EVENT_NAME like '%temptable%'\G
*************************** 1. row ***************************
                  EVENT_NAME: memory/temptable/physical_disk
                COUNT_ALLOC: 2
                  COUNT_FREE: 0
  SUM_NUMBER_OF_BYTES_ALLOC: 1073741824
    SUM_NUMBER_OF_BYTES_FREE: 0
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 2
            HIGH_COUNT_USED: 2
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 1073741824
  HIGH_NUMBER_OF_BYTES_USED: 1073741824
*************************** 2. row ***************************
                  EVENT_NAME: memory/temptable/physical_ram
                COUNT_ALLOC: 12
                  COUNT_FREE: 1
  SUM_NUMBER_OF_BYTES_ALLOC: 1074790400
    SUM_NUMBER_OF_BYTES_FREE: 1048576
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 11
            HIGH_COUNT_USED: 11
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 1073741824
  HIGH_NUMBER_OF_BYTES_USED: 1073741824
2 rows in set (0.00 sec)

mysql > select * from INFORMATION_SCHEMA.INNODB_SESSION_TEMP_TABLESPACES where id=10\G
*************************** 1. row ***************************
     ID: 10
  SPACE: 4243767290
   PATH: ./#innodb_temp/temp_10.ibt
   SIZE: 2399141888
  STATE: ACTIVE
PURPOSE: INTRINSIC
*************************** 2. row ***************************
     ID: 10
  SPACE: 4243767289
   PATH: ./#innodb_temp/temp_9.ibt
   SIZE: 98304
  STATE: ACTIVE
PURPOSE: USER
2 rows in set (0.00 sec)
```

D'après ce qui précède, nous pouvons voir que la requête a créé une énorme table temporaire, qui a d'abord dépassé la variable [temptable_max_ram ](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_temptable_max_ram)et a continué à croître dans un fichier mmappé (toujours moteur TempTable ), mais comme [temptable_max_mmap a également ](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_temptable_max_mmap) été atteint, la table a dû être convertie en on- table intrinsèque du disque InnoDB . Le même pool de tables InnoDB temporaires est utilisé dans ce cas, mais nous pouvons voir les informations sur l'objectif, selon si la table est externe (créée par l'utilisateur) ou interne.

Le fichier mmappé n'est pas visible dans le système de fichiers car il a un état supprimé, mais peut être surveillé avec lsof :

```
mysqld  862655 przemek   52u      REG              253,3  133644288  52764900 /data/sandboxes/msb_ps8_0_23/tmp/mysql_temptable.8YIGV8 (deleted)
```

Il est important de savoir ici que tant que l'espace mmappé n'a pas dépassé, le compteur [Created_tmp_disk_tables ](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html#statvar_Created_tmp_disk_tables) n'est pas incrémenté même si un fichier est créé sur le disque ici.

De plus, dans Percona Server pour MySQL , le journal lent étendu, la taille des tables temporaires lorsque le moteur TempTable est utilisé, n'est pas pris en compte : [https://jira.percona.com/browse/PS-5168 ](https://jira.percona.com/browse/PS-5168)- il affiche " Tmp\_table\_sizes : 0".

Dans certains cas d'utilisation, des problèmes sont signalés avec TempTable . Il est possible de revenir à l'ancien moteur de mémoire via le variable [internal_tmp_mem_storage_engine ](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_internal_tmp_mem_storage_engine) si nécessaire.

## Les références

- <https://www.percona.com/blog/2017/12/04/internal-temporary-tables-mysql-5-7/>
- <https://dev.mysql.com/worklog/task/?id=7682>
- <https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-temp-table-info.html>
- <https://bugs.mysql.com/bug.php?id=96236>
- <https://www.percona.com/doc/percona-server/8.0/diagnostics/slow_extended.html#other-information>
- <https://dev.mysql.com/doc/refman/8.0/en/internal-temporary-tables.html>

