+++
title = "Améliorer PostgreSQL Query Performance Insights avec pg_stat_monitor"
description = "Traduit à partir de l'article de Ibrar Ahmed intitulé, Improve PostgreSQL Query Performance Insights with pg_stat_monitor"
author = "Francis"
date = 2021-09-19T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article16.jpg"
images = ["thumbnail/article16.jpg"]
slug = "ameliorer-postgreSQL-query-performance-insights-avec-pg_stat_monitor"
+++

**Améliorer PostgreSQL Query Performance Insights avec pg\_stat\_monitor**

Par [Ibrar Ahmed](https://www.percona.com/blog/author/ibrar-ahmed/)

La compréhension des modèles de requêtes performance est essentiellement la base de l’optimisation de requête performance. Il dicte, à bien des égards, l'évolution d'un cluster de bases de données. Et puis il y a évidemment aussi des connotations de coûts directs et indirects.

PostgreSQL fournit des statistiques très détaillées via un certain nombre de catalogue et d'extensions qui peuvent être facilement ajoutées pour fournir des statistiques de requête plus détaillées. Chaque vue étant axée sur un aspect particulier, l'image doit presque toujours être assemblée en combinant différents ensembles de données. Cela demande des efforts et pourtant, le tableau d'ensemble peut ne pas être complet.

L'extension pg\_stat\_monitor tente de fournir une image plus globale en fournissant des informations sur les performances des requêtes indispensables dans une seule vue. L'extension a évolué au cours de l'année écoulée et se rapproche maintenant de la version GA.

**Quelques extensions utiles**

Actuellement, vous pouvez compter sur un certain nombre d'extensions pour comprendre le comportement d'une requête, le temps pris dans les phases de planification et d'exécution, les valeurs min/max/meantime, les accès à l'index, le plan de requête et les détails de l'application cliente. Voici quelques extensions que vous connaissez peut-être déjà.

**pg\_stat\_activity**

Cette vue est disponible par défaut avec PostgreSQL. Il fournit une ligne par processus serveur avec l'activité actuelle et le texte de la requête.

Si vous souhaitez en savoir plus, [consultez la documentation officielle de PostgreSQL ici](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW) . 

**pg\_stat\_statements**

Cette extension fait partie des packages contrib fournis avec le serveur PostgreSQL. Cependant, vous devrez créer l'extension manuellement. Il s'agit d'une agrégation de données statistiques par requête avec un écart min/max/moyenne/type pour les temps d'exécution et de planification et diverses informations utiles et texte de requête.

Vous pouvez en savoir plus sur pg\_stat\_statements sur le site officiel de [documentation de PostgreSQL](https://www.postgresql.org/docs/13/pgstatstatements.html)

**auto\_explain**

Une autre extension utile est fournie par le serveur PostgreSQL. Il vide les plans de requête dans le journal du serveur pour toute requête dépassant un seuil de temps spécifié par un GUC (Grand Unified Configuration).

Vous pouvez en savoir plus sur auto\_explain [ici](https://www.postgresql.org/docs/current/auto-explain.html) . 

**pg\_stat\_monitor**

Bien que toutes les vues/extensions mentionnées précédemment soient excellentes en elles-mêmes, il faut combiner manuellement les informations client/connexion de pg\_stat\_activity , les données statistiques de pg\_stat\_statements et le plan de requête de auto\_analyze pour compléter l'ensemble de données afin de comprendre les modèles de performances des requêtes.

Et c'est précisément la peine que **pg\_stat\_monitor** soulage.

L'ensemble de fonctionnalités a augmenté au cours de la dernière année, fournissant, dans une seule vue, toutes les informations liées aux performances dont vous pourriez avoir besoin pour déboguer une requête peu performante. Pour plus d'informations sur l'extension, consultez notre [référentiel GitHub repository](https://github.com/percona/pg_stat_monitor) , ou pour une documentation spécifique à l'utilisateur, consultez notre [guide de l'utilisateur](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md).

**Paramètre de Fonctionnalité**

Certaines fonctionnalités qui faisaient partie des versions précédentes sont déjà abordées dans [ce blog](https://www.percona.com/blog/2020/10/14/announcing-pg_stat_monitor-tech-preview-get-better-insights-into-query-performance-in-postgresql/) , cependant, pour être complet, je vais également les aborder ici. 

- ***Groupement d'intervalles de temps [Time Interval Grouping](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#function-execution-tracking)*** au lieu de fournir un ensemble de nombres toujours croissants, pg\_stat\_monitor calcule les statistiques pour un nombre configuré d'intervalles de temps ; time buckets. Cela permet une meilleure précision des données, en particulier dans le cas de réseaux haute résolution ou peu fiables.
- ***Regroupement multidimensionnel (Multi-Dimensional Grouping)*** : tandis que pg\_stat\_statements regroupe les compteurs par ( userid, dbid , queryid), pg\_stat\_monitor utilise un groupe plus détaillé pour une plus grande précision.
  - *Bucket ID (bucket),*
  - *User ID (userid),*
  - *Database ID (dbid),*
  - *Query ID (queryid),*
  - *Client IP Address (client\_ip),*
  - *Plan ID (planid),*
  - *Application Name (application\_name).*

Cela vous permet d'explorer les performances des requêtes provenant d'adresses et d'applications client particulières, ce que nous, chez Percona, avons trouvé très utile dans un certain nombre de cas.

- ***Capturer les paramètres réels dans les requêtes (Capture Actual Parameters in the Queries)*** : pg\_stat\_monitor vous permet de choisir si vous souhaitez voir les requêtes avec des espaces réservés pour les paramètres ou des exemples de requêtes réelles.

- ***[Plan de requête (Query Plan)](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#function-execution-tracking)*** : Chaque SQL est désormais accompagné de son plan réel qui a été construit pour son exécution. De plus, nous avons trouvé que les valeurs des paramètres de requête sont très utiles, car vous pouvez exécuter EXPLAIN, ou jouer facilement avec la modification de la requête pour l'améliorer, ainsi que pour rendre la communication sur la requête plus claire lors des discussions avec d'autres administrateurs de bases de données et développeurs d'applications.

- ***[Statistiques d'accès aux tables pour une instruction Tables Access Statistics for a Statement](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#function-execution-tracking)*** : Cela nous permet d'identifier facilement toutes les requêtes ayant accédé à une table donnée. Cet ensemble est à égalité avec les informations fournies par les pg\_stat\_statements.

- ***[Histogramme](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#function-execution-tracking)*** : La représentation visuelle est très utile lorsqu'elle peut aider à identifier les problèmes. Avec l'aide de la fonction d'histogramme, vous pouvez désormais afficher un histogramme de données de synchronisation/appel en réponse à une requête SQL. Et oui, cela fonctionne même en psql.

```
SELECT * FROM histogram(0, 'F44CD1B4B33A47AF') AS a(range TEXT, freq INT, bar TEXT);
       range        | freq |              bar
--------------------+------+--------------------------------
  (0 - 3)}          |    2 | ■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  (3 - 10)}         |    0 |
  (10 - 31)}        |    1 | ■■■■■■■■■■■■■■■
  (31 - 100)}       |    0 |
  (100 - 316)}      |    0 |
  (316 - 1000)}     |    0 |
  (1000 - 3162)}    |    0 |
  (3162 - 10000)}   |    0 |
  (10000 - 31622)}  |    0 |
  (31622 - 100000)} |    0 |
(10 rows)
```

- ***[Fonctions](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#function-execution-tracking)*** : Cela peut surprendre, mais nous comprenons que les fonctions peuvent exécuter des instructions en interne !!! Pour faciliter le suivi et l'analyse, pg\_stat\_monitor fournit désormais une colonne qui aide spécifiquement à garder une trace de la requête principale pour une instruction afin que vous puissiez revenir à la fonction d'origine.

- ***[Noms des relations (Relation Names)](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#object-information)*** : Les relations utilisées dans une requête sont disponibles dans la colonne « relations » de la vue pg\_stat\_monitor . Cela réduit le travail chez vous et rend l'analyse plus simple et plus rapide.

- ***[Types de requêtes (Query Types)](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#query-command-type-select-insert-update-or-delete)***: Avec la classification des requêtes comme SELECT, INSERT, UPDATE ou DELETE, l'analyse devient plus simple. C'est un autre effort réduit de votre côté, et une autre simplification par pg\_stat\_monitor .

```
SELECT bucket, substr(query,0, 50) AS query, cmd_type FROM pg_stat_monitor WHERE elevel = 0;
 bucket |                       query                       | cmd_type 
--------+---------------------------------------------------+----------
      4 | END                                               | 
      4 | SELECT abalance FROM pgbench_accounts WHERE aid = | SELECT
      4 | vacuum pgbench_branches                           | 
      4 | select count(*) from pgbench_branches             | SELECT
      4 | UPDATE pgbench_accounts SET abalance = abalance + | UPDATE
      4 | truncate pgbench_history                          | 
      4 | INSERT INTO pgbench_history (tid, bid, aid, delta | INSERT
```

- [***Query Metadata***](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#sql-commenter---tags) : [Google Sqlcommenter ](https://google.github.io/sqlcommenter/) est un outil tile qui comble en quelque sorte l'écart entre les bibliothèques ORM et la compréhension des performances de la base de données. Et nous le soutenons. Ainsi, vous pouvez maintenant mettre n'importe quelle donnée de valeur clé dans les commentaires dans la syntaxe /\* … \*/ (voir la [Sqlcommenter documentation](https://google.github.io/sqlcommenter/) pour plus de détails) dans vos instructions SQL , et les informations seront analysées par pg\_stat\_monitor et mises à disposition dans la colonne des commentaires de la vue pg\_stat\_monitor.

```
CREATE EXTENSION hstore;
CREATE FUNCTION text_to_hstore(s text) RETURNS hstore AS $$
BEGIN
    RETURN hstore(s::text[]);
EXCEPTION WHEN OTHERS THEN
    RETURN NULL;
END; $$ LANGUAGE plpgsql STRICT;


SELECT 1 AS num /* { "application", java_app, "real_ip", 192.168.1.1} */;
 num 
-----
   1
(1 row)
```

```
SELECT query, text_to_hstore(comments)->'real_ip' AS real_ip from pg_stat_monitor;
query                                                                       |  real_ip 
----------------------------------------------------------------------------+-------------
 SELECT $1 AS num /* { "application", psql_app, "real_ip", 192.168.1.3) */  | 192.168.1.1
```


- [***Logging Error and Warning***](https://github.com/percona/pg_stat_monitor/blob/master/docs/USER_GUIDE.md#error-messages--error-codes-and-error-level): comme on le voit dans différents outils de surveillance/collecteur de statique, la plupart des outils/extensions ne surveillent que les requêtes réussies. Mais dans de nombreux cas, la surveillance d'ERROR, d'AVERTISSEMENT et de LOG fournit des informations utiles pour déboguer le problème. pg\_stat\_monitor non seulement surveille les ERREURS/AVERTISSEMENTS/LOG mais collecte également les statistiques sur ces requêtes. Dans les requêtes PostgreSQL avec ERROR/WARNING, il existe un niveau d'erreur ( elevel ), un code SQL ( sqlcode ) et un message d'erreur est joint. Pg\_stat\_monitor collecte toutes ces informations ainsi que ses agrégats.

```
SELECT substr(query,0,50) AS query, decode_error_level(elevel) AS elevel,sqlcode, calls, substr(message,0,50) message 
FROM pg_stat_monitor;
                       query                       | elevel | sqlcode | calls |                      message                      
---------------------------------------------------+--------+---------+-------+---------------------------------------------------
 select substr(query,$1,$2) as query, decode_error |        |       0 |     1 | 
 select bucket,substr(query,$1,$2),decode_error_le |        |       0 |     3 | 
 select 1/0;                                       | ERROR  |     130 |     1 | division by zero
```

**Nous avons parcouru un long chemin**

Ce qui a commencé comme un concept approche maintenant de son approche finale. L'extension pg\_stat\_monitor a évolué et est devenue très riche en fonctionnalités. Nous n'avons aucun doute sur son utilité pour les administrateurs de bases de données, les ingénieurs de performances, les développeurs d'applications et tous ceux qui ont besoin d'examiner les performances des requêtes. Nous pensons que cela peut aider à gagner de nombreuses heures et à identifier les comportements de requête inattendus. 

[pg_stat_monitor](https://github.com/percona/pg_stat_monitor) est disponible sur Github . Nous le publions pour obtenir des commentaires de la communauté sur ce que nous faisons correctement et ce que nous devrions faire différemment avant de publier pg\_stat\_monitor en tant que version disponible pour le grand public à prendre en charge pour les années à venir. Veuillez le vérifier, [envoyez-nous une note](https://www.percona.com/about-percona/contact) , [signalez un problème](https://jira.percona.com/projects/PG) ou faites un [pull request](https://github.com/percona/pg_stat_monitor/pulls) !

[Essayez Percona Distribution pour PostgreSQL : C'est gratuit !](https://www.percona.com/software/postgresql-distribution) 

Source : [Percona Blog](https://www.percona.com/blog/improve-postgresql-query-performance-insights-with-pg_stat_monitor/)

