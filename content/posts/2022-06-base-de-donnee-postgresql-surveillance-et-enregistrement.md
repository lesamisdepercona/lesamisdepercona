+++
title = "Base de données PostgreSQL 14 : Surveillance et Enregistrement"
description = "La surveillance est une fonctionnalité clé de tout système de SGBD, et PostgreSQL continue de mettre à niveau ses capacités pour améliorer ses capacités de journalisation et de surveillance. Les fonctionnalités nouvellement ajoutées offre plus d'informations sur les connexions permettant de suivre les requêtes, observer les performances, et identifier le temps passé par le processus de vide dans les opérations de lecture/écriture."
author = "Francis"
date = 2022-02-11
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article06.jpg"
images = ["thumbnail2022/article06.jpg"]
slug = "base-de-donnee-postgresql-surveillance-et-enregistrement"
+++

![thumbnail](/thumbnail2022/article06.jpg)


PostgreSQL-14 a été publié en septembre 2021 et contenait de nombreuses améliorations des performances et des fonctionnalités, y compris certaines fonctionnalités du point de vue surveillance. Comme nous le savons, la surveillance est l'élément clé de tout système de gestion de base de données, et PostgreSQL continue de mettre à jour et d'améliorer les capacités de surveillance. Voici quelques clés dans PostgreSQL-14.

## Identificateur de requête

L'identificateur de requête est utilisé pour identifier la requête, qui peut être référencée entre les extensions. Avant PostgreSQL-14, les extensions utilisaient un algorithme pour calculer le query_id . Habituellement, le même algorithme est utilisé pour calculer le query_id , mais toute extension peut utiliser son propre algorithme. Désormais, PostgreSQL-14 fournit éventuellement un query_id à calculer dans le noyau. Maintenant, les extensions de surveillance et les utilitaires de PostgreSQL-14 comme pg_stat_activity , explain, et pg_stat_statments utilisent ce query_id au lieu de calculer le sien. Ce query_id peut être vu dans csvlog , après avoir spécifié dans le log_line_prefix . Du point de vue de l'utilisateur, cette fonctionnalité présente deux avantages:

- Tous les utilitaires/extensions utiliseront le même query_id calculé par core, ce qui permet de référencer facilement ce query_id . Auparavant, tous les utilitaires/extensions devaient utiliser le même algorithme dans leur code pour obtenir cette capacité.
- Le deuxième avantage est que l'extension/les utilitaires peuvent utiliser le query_id calculé et n'ont plus besoin de le faire, ce qui est un avantage en termes de performances.

PostgreSQL introduit un nouveau paramètre de configuration GUC compute_query_id pour activer/désactiver cette fonctionnalité. La valeur par défaut est automatique; cela peut être activé/désactivé dans le fichier postgresql.conf ou à l'aide de la commande SET.

- **pg_stat_activity**

SET compute_query_id = OFF;
```
SELECT datname, query, query_id FROM pg_stat_activity;
 datname  |                                 query                                 | query_id 
----------+-----------------------------------------------------------------------+----------
 postgres | select datname, query, query_id from pg_stat_activity;                |         
 postgres | UPDATE pgbench_branches SET bbalance = bbalance + 2361 WHERE bid = 1; |
```

SET compute_query_id = on;
```
SELECT datname, query, query_id FROM pg_stat_activity;
 datname  |                                 query                                 |      query_id       
----------+-----------------------------------------------------------------------+---------------------
 postgres | select datname, query, query_id from pg_stat_activity;                |  846165942585941982
 postgres | UPDATE pgbench_tellers SET tbalance = tbalance + 3001 WHERE tid = 44; | 3354982309855590749
```

- **Log**

Dans les versions précédentes, il n'y avait aucun mécanisme pour calculer le query_id dans le noyau du serveur. Le query_id est particulièrement utile dans les fichiers journaux. Pour activer cela, nous devons configurer le paramètre de configuration log_line_prefix . L'option "%Q" est ajoutée pour afficher le query_id; voici un exemple.
```
log_line_prefix = 'query_id = [%Q] -> '
```
```
query_id = [0] -> LOG:  statement: CREATE PROCEDURE ptestx(OUT a int) LANGUAGE SQL AS $$ INSERT INTO cp_test VALUES (1, 'a') $$;
query_id = [-6788509697256188685] -> ERROR:  return type mismatch in function declared to return record
query_id = [-6788509697256188685] -> DETAIL:  Function's final statement must be SELECT or INSERT/UPDATE/DELETE RETURNING.
query_id = [-6788509697256188685] -> CONTEXT:  SQL function "ptestx"
query_id = [-6788509697256188685] -> STATEMENT:  CREATE PROCEDURE ptestx(OUT a int) LANGUAGE SQL AS $$ INSERT INTO cp_test VALUES (1, 'a') $$;
```
- **Explain**

EXPLAIN VERBOSE affichera le query_id si compute_query_id est vrai.

SET compute_query_id = OFF;
```
EXPLAIN VERBOSE SELECT * FROM foo;
                          QUERY PLAN                          
--------------------------------------------------------------

 Seq Scan on public.foo  (cost=0.00..15.01 rows=1001 width=4)
   Output: a
(2 rows)
```
SET compute_query_id = on;
```
EXPLAIN VERBOSE SELECT * FROM foo;
                          QUERY PLAN                          
--------------------------------------------------------------
 Seq Scan on public.foo  (cost=0.00..15.01 rows=1001 width=4)
   Output: a
 Query Identifier: 3480779799680626233
(3 rows)
```
## Améliorations de l’Autovacuum et enregistrement auto-analyse

PostgreSQL-14 améliore l’enregistrement de l’auto-vacuum et auto-analyze. Nous pouvons maintenant voir les délais I/O dans le journal, indiquant la lecture et écriture.
```
automatic vacuum of table "postgres.pg_catalog.pg_depend": index scans: 1
pages: 0 removed, 67 remain, 0 skipped due to pins, 0 skipped frozen
tuples: 89 removed, 8873 remain, 0 are dead but not yet removable, oldest xmin: 210871
index scan needed: 2 pages from table (2.99% of total) had 341 dead item identifiers removed
index "pg_depend_depender_index": pages: 39 in total, 0 newly deleted, 0 currently deleted, 0 reusable
index "pg_depend_reference_index": pages: 41 in total, 0 newly deleted, 0 currently deleted, 0 reusable

I/O timings: read: 44.254 ms, write: 0.531 ms

avg read rate: 13.191 MB/s, avg write rate: 8.794 MB/s
buffer usage: 167 hits, 126 misses, 84 dirtied
WAL usage: 85 records, 15 full page images, 78064 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.07 s
```
Ces journaux ne sont disponibles que si track_io_timing est activé.

## Connecter le journal

PostgreSQL enregistre déjà la connexion/déconnexion si log_connections / log_disconnections est activé. Par conséquent, PostgreSQL-14 enregistre désormais également le nom d'utilisateur réel fourni par l'utilisateur. Dans le cas où une authentification externe est utilisée et que le mappage est défini dans pg_ident.conf, il deviendra difficile d'identifier le nom d'utilisateur réel. Avant PostgreSQL-14, vous ne voyez que l'utilisateur mappé au lieu de l'utilisateur réel.

**pg_ident.conf**
```
# MAPNAME       SYSTEM-USERNAME         PG-USERNAME

pg              vagrant                 postgres
```
**pg_hba.conf**
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     peer map=pg
```
**Avant PostgreSQL-14**
```
LOG:  database system was shut down at 2021-11-19 11:24:30 UTC
LOG:  database system is ready to accept connections
LOG:  connection received: host=[local]
LOG:  connection authorized: user=postgres database=postgres application_name=psql
```

**PostgreSQL -14**
```
LOG:  database system is ready to accept connections
LOG:  connection received: host=[local]
LOG:  connection authenticated: identity="vagrant" method=peer (/usr/local/pgsql.14/bin/data/pg_hba.conf:89)
LOG:  connection authorized: user=postgres database=postgres application_name=psql
```
## Conclusion

Chaque version majeure de PostgreSQL comporte des améliorations significatives, et PostgreSQL-14 n'était pas différent.

La surveillance est une fonctionnalité clé de tout système de SGBD, et PostgreSQL continue de mettre à niveau ses capacités pour améliorer ses capacités de journalisation et de surveillance. Avec ces fonctionnalités nouvellement ajoutées, vous avez plus d'informations sur les connexions ; on peut facilement suivre les requêtes et observer les performances, et identifier le temps passé par le processus de vide dans les opérations de lecture/écriture. Cela peut vous être très utile pour mieux configurer les paramètres de vide.

Source: [Percona](https://www.percona.com/blog/postgresql-14-database-monitoring-and-logging-enhancements/)
