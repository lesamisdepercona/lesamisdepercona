+++
title = "Impact du réseau et du curseur sur les performances des requêtes de PostgreSQL"
description = "Traduit à partir de l'article de Jobin Augustine intitulé, Impact of Network and Cursor on Query Performance of PostgreSQL"
author = "Francis"
date = 2021-10-17T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article20-PostgreSQL.jpg"
images = ["thumbnail/article20-PostgreSQL.jpg"]
slug = "impact-du-reseau-et-du-curseur-sur-les-performances-des-requetes-de-postgresql"
+++

Plusieurs fois, nous voyons les utilisateurs de PostgreSQL se perdre sur la durée des requêtes/instructions signalée dans les journaux PostgreSQL. D'autres outils PostgreSQL comme pgBadger présentent les mêmes données basées sur le fichier journal, ce qui augmente encore la confusion. Connaître l'impact total de la surcharge et des curseurs liés au réseau est important non seulement pour atténuer la confusion, mais également pour obtenir les meilleures performances.

On peut se demander «Pourquoi discuter spécifiquement de la surcharge du réseau et des curseurs?». Eh bien, ce sont les coûts cachés après l'exécution de la requête. Une fois que l'exécution de la requête commence et qu'il y a des données à fournir au client, ce sont les facteurs qui affectent principalement les performances. Donc, le point important que nous pouvons vouloir garder à l'esprit est que puisque tout cela se produit en dehors de l'exécution de la requête, ***les informations correspondantes ne seront pas disponibles via EXPLAIN (ANALYZE).***

## Impact du réseau

Pour la démonstration, j'utilise une requête basée sur les tables pgBench.
```
select a.bid, b.cnt from pgbench_accounts a,
   (select bid,count(*) cnt from pgbench_accounts group by bid) b
where b.bid > a.bid;
```
Il n'y a aucune signification pour cette requête. Une requête aléatoire est sélectionnée, ce qui prend un certain temps dans la base de données pour s'exécuter.

Afin de capturer la sortie Explain Analyze, [auto_explain](https://www.postgresql.org/docs/current/auto-explain.html) est utilisé. Les paramètres sont définis pour capturer toutes les déclarations qui prennent plus de 250 ms.
```
ALTER SYSTEM SET auto_explain.log_min_duration = 250
```
Voici quelques autres paramètres pour capturer des détails supplémentaires avec EXPLAIN ANALYZE:
```
ALTER SYSTEM SET auto_explain.log_analyze = on;
ALTER SYSTEM SET auto_explain.log_buffers=on;
ALTER SYSTEM SET auto_explain.log_timing=on;
ALTER SYSTEM SET auto_explain.log_verbose=on;
ALTER SYSTEM SET auto_explain.log_triggers=on;
ALTER SYSTEM SET auto_explain.log_wal=on;
```
Afin d'illustrer la différence, la même requête sera exécutée à partir de :

\1. Le serveur hôte de la base de données

\2. Le serveur hôte de l'application qui est connecté sur un réseau

## Requêtes renvoyant un grand nombre de lignes

**Cas 1. Exécution sur l'hôte de base de données lui-même**

Voici les quelques lignes des journaux PostgreSQL générés par auto\_explain :
```
2021-08-02 03:27:56.347 UTC [25537] LOG:  duration: 1591.784 ms  plan:
        Query Text: select a.bid, b.cnt from pgbench_accounts a,
           (select bid,count(*) cnt from pgbench_accounts group by bid) b
        where b.bid > a.bid;
        Nested Loop  (cost=12322.13..63020.46 rows=833333 width=12) (actual time=119.739..1069.924 rows=1000000 loops=1)
          Output: a.bid, b.cnt
          Join Filter: (b.bid > a.bid)
          Rows Removed by Join Filter: 1500000
          ...
```

Comme nous pouvons le voir, la boucle imbriquée externe de la requête a été achevée en **1069,924 ms,** renvoyant 1 million d'enregistrements, mais la durée totale de la requête est de **1591,784 ms.** Quelle pourrait être la différence ?

Une analyse directe EXPLAIN ANALYZE montre que le **temps de planification est inférieur à la milliseconde** pour cette requête simple où les données proviennent d'une seule table sans aucun index. Donc, le temps de planification ne devrait pas être la raison.

**Cas 2. Exécution à partir d'un hôte d'application distant**

Encore une fois, les informations du journal PostgreSQL sont différentes :
```
2021-08-02 04:08:58.955 UTC [25617] LOG:  duration: 6568.659 ms  plan:
        Query Text: select a.bid, b.cnt from pgbench_accounts a,
           (select bid,count(*) cnt from pgbench_accounts group by bid) b
        where b.bid > a.bid;
        Nested Loop  (cost=12322.13..63020.46 rows=833333 width=12) (actual time=140.644..1069.153 rows=1000000 loops=1)
          Output: a.bid, b.cnt
          Join Filter: (b.bid > a.bid)
          Rows Removed by Join Filter: 1500000
          ...
```         

Comme on peut le voir, la durée de l'instruction a bondi à **6568,659 ms !** même si l'exécution réelle de la requête est restée à peu près la même **1069,153 ms** . C'est une énorme différence. Quelle pourrait être la raison?

## Requêtes renvoyant un nombre de lignes inférieur

La requête mentionnée ci-dessus peut être légèrement modifiée pour ne renvoyer que les valeurs max. La requête de test modifiée peut ressembler à ceci :
``` 
select max(a.bid), max(b.cnt) from pgbench_accounts a,
   (select bid,count(*) cnt from pgbench_accounts group by bid) b
where b.bid > a.bid ;
``` 

Le plan ou l'heure de la requête ne change pas grand-chose à part qu'il y a un agrégat supplémentaire. Même s'il y a un changement de plan qui n'est pas pertinent pour le sujet, nous en discutons car nous ne considérons que la différence de temps entre le nœud externe de l'exécution de la requête et la durée rapportée par PostgreSQL.

**Cas 1 : exécution sur l'hôte de base de données lui-même**
``` 
2021-08-03 06:58:14.364 UTC [28129] LOG:  duration: 1011.143 ms  plan:
        Query Text: select max(a.bid), max(b.cnt) from pgbench_accounts a,
           (select bid,count(*) cnt from pgbench_accounts group by bid) b
        where b.bid > a.bid ;
        Aggregate  (cost=67187.12..67187.13 rows=1 width=12) (actual time=1010.914..1011.109 rows=1 loops=1)
          Output: max(a.bid), max(b.cnt)
          Buffers: shared hit=12622 read=3786
          ->  Nested Loop  (cost=12322.13..63020.46 rows=833333 width=12) (actual time=135.635..908.315 rows=1000000 loops=1)
                Output: a.bid, b.cnt
                Join Filter: (b.bid > a.bid)
                Rows Removed by Join Filter: 1500000
                Buffers: shared hit=12622 read=3786
``` 
Comme nous pouvons le voir, il n'y a pas beaucoup de différence entre l'achèvement de la requête 1011.109 et la durée rapportée 1011.143 ms. L'observation jusqu'à présent indique qu'il y a du temps supplémentaire consommé lorsqu'il y a beaucoup de lignes renvoyées.

**Cas 2 : Exécution de la déclaration à partir de l'hôte distant**
``` 
2021-08-03 06:55:37.221 UTC [28111] LOG:  duration: 1193.387 ms  plan:
        Query Text: select max(a.bid), max(b.cnt) from pgbench_accounts a,
           (select bid,count(*) cnt from pgbench_accounts group by bid) b
        where b.bid > a.bid ;
        Aggregate  (cost=67187.12..67187.13 rows=1 width=12) (actual time=1193.139..1193.340 rows=1 loops=1)
          Output: max(a.bid), max(b.cnt)
          Buffers: shared hit=11598 read=4810
          ->  Nested Loop  (cost=12322.13..63020.46 rows=833333 width=12) (actual time=124.508..1067.409 rows=1000000 loops=1)
                Output: a.bid, b.cnt
                Join Filter: (b.bid > a.bid)
                Rows Removed by Join Filter: 1500000
                Buffers: shared hit=11598 read=4810
``` 
Encore une fois, il n'y a pas beaucoup de différence 1193.340 vs 1193.387 ms. Dans l'ensemble, je suis sûr de supposer à partir des résultats que si le transfert de données est minime, le serveur d'applications sur une machine hôte différente ne fait pas beaucoup de différence ; en attendant, l'impact est énorme s'il y a beaucoup de transfert de résultats.

**Analyse des événements d'attente**

Heureusement, les versions les plus récentes de PostgreSQL nous fournissent un excellent moyen de surveiller les informations [« wait event » à partir de la vue pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW) de la session/connexion.

Au support Percona, nous utilisons un [script de pg_gather](https://raw.githubusercontent.com/percona/support-snippets/master/postgresql/pg_gather/gather.sql) snippet pour collecter des informations sur les performances, y compris les événements d'attente, en collectant plusieurs échantillons. Le script collecte des échantillons d'événements d'attente toutes les 10 ms, il y aura donc 2 000 échantillons en 20 secondes. Outre l'interface utilisateur, les informations recueillies peuvent également être analysées à l'aide de requêtes backend.

Voici ce que j'ai pu voir à propos du PID : 25617 (le cas du renvoi d'un grand nombre de lignes à l'hôte distant).
``` 
postgres=# select pid,wait_event,count(*) from pg_pid_wait where pid=25617 group by 1,2 order by 3 desc;
  pid  |  wait_event  | count 
-------+--------------+-------
 25617 | ClientWrite  |   286
 25617 |              |    75
 25617 | DataFileRead |     3
(3 rows)
``` 

Les sessions passent plus de temps sur "ClientWrite" selon la [documentation PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html) . 

|ClientEcrire|En attente d'écriture des données sur le client.|
| - | - |

C'est le temps passé à écrire les données au client. Le wait\_event NULL indique l'utilisation du processeur.

## Impact des curseurs

En règle générale, après l'exécution d'une requête, les données de résultat doivent être traitées par l'application. Les curseurs sont utilisés pour conserver le résultat des requêtes et les traiter. L'impact sur les performances des requêtes est principalement déterminé par l'emplacement du curseur, que ce soit côté serveur PostgreSQL ou côté client. L'emplacement du curseur devrait affecter le moment où la requête est émise à partir d'un hôte d'application distinct, je ne teste donc que ce cas.

**Curseurs côté client**

En général, c'est le cas avec la plupart des clients et applications PostgreSQL. Les données sont récupérées intégralement vers le client de la base de données, puis traitées une par une.

Voici un simple extrait de code python (pour imiter la connexion de l'application) pour le tester. (Seules les lignes pertinentes sont copiées.)
``` 
   conn =  psycopg2.connect(connectionString)
   cur = conn.cursor()
   cur.itersize = 2000
   cur.execute("select a.bid, b.cnt from pgbench_accounts a, (select bid,count(*) cnt from pgbench_accounts group by bid) b where b.bid > a.bid")
   row = cur.fetchone()
   while row is not None:
     print(row)
     time.sleep(0.001)
     row = cur.fetchone()

   conn.close()
``` 
Comme nous pouvons le voir, `itersize` est spécifié de sorte que seuls ces nombreux enregistrements doivent être récupérés à la fois pour le traitement et il y a un délai de 1 milliseconde dans une boucle utilisant `fetchone()` dans chaque boucle

Mais *aucun de ces éléments n'affecte les performances des requêtes côté serveur,* car le curseur est déjà mis en cache côté client. L'heure et la durée de la requête signalées sont similaires à l'exécution à partir d'un hôte d'application distant pour un grand nombre de lignes. Comme attendu, l'impact du réseau est clairement visible :
``` 
2021-08-03 17:39:17.119 UTC [29180] LOG:  duration: 5447.793 ms  plan:
        Query Text: select a.bid, b.cnt from pgbench_accounts a, (select bid,count(*) cnt from pgbench_accounts group by bid) b where b.bid > a.bid
        Nested Loop  (cost=12322.13..63020.46 rows=833333 width=12) (actual time=130.919..1240.727 rows=1000000 loops=1)
          Output: a.bid, b.cnt
          Join Filter: (b.bid > a.bid)
          Rows Removed by Join Filter: 1500000
          Buffers: shared hit=1647 read=14761
          ->  Seq Scan on public.pgbench_accounts a  (cost=0.00..13197.00 rows=500000 width=4) (actual time=0.086..183.503 rows=500000 loops=1)
                Output: a.aid, a.bid, a.abalance, a.filler
                Buffers: shared hit=864 read=7333
``` 
## Curseurs côté serveur

La façon dont le curseur est créé et utilisé changera totalement si nous avons un curseur nommé qui reste côté serveur et que l'instruction `cur = conn.cursor()` est modifiée pour inclure un nom comme `cur = conn.cursor('nom')`

Comme le dit la documentation [psycopg2: (Le connecteur Python pour PostgreSQL)](https://www.psycopg.org/docs/usage.html#server-side-cursors) : 

*« Psycopg encapsule le curseur côté serveur de la base de données dans des curseurs nommés. Un curseur nommé est créé à l'aide de la méthode [cursor()](https://www.psycopg.org/docs/connection.html#connection.cursor) en spécifiant le paramètre name"*

Étonnamment, le journal PostgreSQL ne donne pas plus d'informations sur la requête même si auto\_explain est configuré. Aucune information sur la durée non plus. Il n'y a qu'une seule ligne d'infos :
``` 
2021-08-03 18:02:45.184 UTC [29249] LOG:  duration: 224.501 ms  statement: FETCH FORWARD 1 FROM "curname"
``` 

Le curseur PostgreSQL prend en charge diverses options FETCH avec des spécifications de taille personnalisées. Veuillez vous référer à [la documentation de FETCH](https://www.postgresql.org/docs/current/plpgsql-cursors.html#id-1.8.8.9.6.5) pour plus de détails. Il appartient au pilote/connecteur de langage d'encapsuler cette fonctionnalité dans les fonctions correspondantes.

Le pilote python pour PostgreSQL - psychopg2 - encapsule la fonctionnalité pour exécuter les lignes FETCH avec la taille de lot personnalisée spécifiée comme suit :
``` 
cur.fetchmany(size=20000)
``` 
Ce qui produit une entrée de journal PostgreSQL comme:
```
2021-08-05 05:13:30.728 UTC [32374] LOG:  duration: 262.931 ms  statement: FETCH FORWARD 20000 FROM "curname"
```
Comme prévu, la taille du fetch est augmentée.

Mais le point important à noter ici est : même si l'itération de l'application sur un curseur côté serveur a pris énormément de temps (plusieurs minutes), il n'y a presque aucune information sur la requête ou la session dans les journaux PostgreSQL.

Ahhh! Cela pourrait être la chose la plus étrange à laquelle quelqu'un pourrait s'attendre.

## Analyse des événements d'attente

Encore une fois, l'analyse des événements d'attente est utile pour comprendre ce qui se passe. Sur les 2000 échantillons d'événements d'attente collectés par le script pg\_gather, les événements d'attente ressemblent à ceci:
```
  pid  |  wait_event  | count 
-------+--------------+-------
 30115 | ClientRead   |  1754
 30115 |   (CPU)      |   245
 30115 | DataFileRead |     1
```
Le temps est en attente de l'événement de poids « ClientRead ». Cela signifie que le côté serveur attend que le client envoie la prochaine requête. Ainsi, un réseau lent entre le serveur d'applications et le serveur de base de données peut affecter négativement les curseurs côté serveur. Mais aucune information ne sera disponible dans les journaux PostgreSQL sur la déclaration. 

## Conclusion

Dans cet article de blog, j'ai essayé d'évaluer l'impact du transfert d'une grande quantité de données d'un hôte de base de données vers un hôte d'application.

VEUILLEZ NOTER : Les chiffres discutés dans l'article de blog en termes de nombre de lignes et de temps enregistré n'ont aucune pertinence absolue et peuvent changer d'un système à l'autre avec de nombreux facteurs environnementaux.

La discussion porte davantage sur les domaines d'impact et ce à quoi nous devons nous attendre et comment analyser en tant qu'utilisateur final plutôt que sur des chiffres absolus.

1. Essayez toujours de spécifier le nombre minimum de colonnes dans la requête. Évitez « SELECT \* » ou les colonnes qui ne sont pas utilisées du côté de l'application pour éviter un transfert de données inutile. 
1. Évitez d'extraire un grand nombre de lignes à la fois dans l'application. Si nécessaire, utilisez une pagination appropriée avec LIMIT et OFFSET. 
1. Évitez autant que possible les curseurs côté serveur. PostgreSQL ne signale que la durée de la première récupération et les performances réelles de la requête peuvent passer inaperçues. Des requêtes peu performantes pourraient se cacher derrière.
1. Les curseurs côté client et le temps de traitement des données n'affecteront pas les performances des requêtes côté serveur PostgreSQL.
1. L'analyse des événements d'attente est très pratique pour comprendre où le serveur passe du temps.

**Percona Distribution for PostgreSQL fournit les meilleurs et les plus critiques des composants d'entreprise de la communauté open source, dans une seule distribution, conçue et testée pour fonctionner ensemble.**

[**Téléchargez Percona Distribution pour PostgreSQL, C'est gratuit !**](https://www.percona.com/software/postgresql-distribution)

Source : [Blog Percona](https://www.percona.com/blog/impact-of-network-and-cursor-on-query-performance-of-postgresql/)

