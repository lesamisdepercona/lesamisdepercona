+++
title = "Solutions de réplication multi-master pour PostgreSQL"
description = "Traduit à partir de l'article de Ibrar Ahmed intitulé, Multi-Master Replication Solutions for PostgreSQL, disponible sur le blog de Percona"
author = "Francis"
date = 2021-07-22T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "posts/article08/image01.png"
slug = "gestion-des-valeurs-null-dans-postgresql"
+++

**Solutions de réplication multi-master pour PostgreSQL**

Par [Ibrar Ahmed](https://www.percona.com/blog/author/ibrar-ahmed/)

En raison de l'immense génération de données, l’évolutivité est devenue l’un des sujets les plus brûlants dans le domaine des bases de données. L’évolutivité peut être réalisée horizontalement ou verticalement. L’évolutivité verticale signifie ajouter plus de ressources/matériel aux nœuds existants pour améliorer la capacité de la base de données à stocker et traiter plus de données, par exemple, ajouter un nouveau processus, mémoire ou disque à un nœud existant. Chaque moteur de SGBD améliore la capacité d’évolutivité verticale en améliorant les mécanismes de verrouillage/mutex et la concurrence par laquelle il peut utiliser plus efficacement les ressources nouvellement ajoutées. Les moteurs de base de données fournissent des paramètres de configuration, ce qui permet d’utiliser plus efficacement les ressources matérielles disponibles.

En raison du coût du matériel et de la limite d’ajout du nouveau matériel dans le nœud existant, il n’est pas toujours possible d’ajouter du nouveau matériel. Par conséquent, une évolutivité horizontale est requise, ce qui signifie ajouter plus de nœuds aux nœuds de réseau existants au lieu d’améliorer la capacité du nœud existant.

Contrairement à l’évolutivité verticale, l’évolutivité horizontale est difficile à mettre en œuvre. Il demande plus d’efforts de développement et demande plus de travail de mise en place. PostgreSQL fournit un ensemble de fonctionnalités assez riche pour l’évolutivité verticale et l’évolutivité horizontale. Il prend en charge les machines avec plusieurs processeurs et beaucoup de mémoire et fournit des paramètres de configuration pour gérer l’utilisation de ces ressources. Les nouvelles fonctionnalités de parallélisme de PostgreSQL rendent l’évolutivité verticale plus importante, mais elle ne manque pas non plus d’évolutivité horizontale. La réplication est le pilier crucial de l’évolutivité horizontale, et PostgreSQL prend en charge la réplication maître-esclave unidirectionnelle, ce qui est suffisant pour de nombreux cas d’utilisation.

Concepts clés

**Réplication de base de données**

La réplication de base de données réplique les données sur d’autres serveurs et les stocke sur plusieurs nœuds. Dans ce processus, l’instance de base de données est transférée d’un nœud à un autre et une copie exacte est effectuée. La réplication des données est utilisée pour améliorer la disponibilité des données, une caractéristique clé de la haute disponibilité. Il peut y avoir une instance de base de données complète, ou certains objets fréquemment utilisés ou souhaités sont répliqués sur un autre serveur. Comme la réplication fournit les multiples copies cohérentes de la base de données, elle offre non seulement une haute disponibilité, mais améliore également les performances.

**Réplication synchrone**

Il existe deux stratégies lors de l’écriture des données sur le disque, **synchrone** et **asynchrone.** La réplication synchrone signifie que les données sont écrites simultanément sur le maître et l’esclave, en d’autres termes, la « réplication synchrone » signifie que la validation attend l’écriture/le vidage du côté distant. La réplication synchrone est utilisée dans l’environnement transactionnel haut de gamme avec des exigences de basculement instantané.

**Réplication asynchrone**

Asynchrone signifie que les données sont d’abord écrites sur le maître, puis répliquées sur l’esclave. En cas de panne, une perte de données peut se produire, mais la réplication asynchrone fournit très peu de surcharge et est donc acceptable dans la plupart des cas. Cela ne surcharge pas le maître. Le basculement du primaire au secondaire prend plus de temps qu’avec la réplication synchrone.

En un mot, la principale différence entre synchrone et asynchrone réside dans le moment où les données sont écrites sur le maître et l’esclave.

**Réplication maître (Master) unique**

La réplication maître unique signifie que les données ne peuvent être modifiées que sur un seul nœud, et ces modifications sont répliquées sur un ou plusieurs nœuds. La mise à jour et l’insertion des données ne sont possibles que sur le nœud maître. Dans ce cas, les applications nécessitent d’acheminer le trafic vers le maître, ce qui ajoute une certaine complexité à l’application. En raison des maîtres uniques, il n’y a aucune chance de conflits. La plupart du temps, la réplication à maître unique est suffisante pour l’application car elle est moins compliquée à configurer et à gérer. Mais dans certains cas, la réplication à maître unique ne suffit pas et vous avez besoin d’une réplication à maîtres multiples.

**Réplication multimaître (multimaster)**

Les réplications multi-maîtres signifient qu’il y a plus d’un nœud qui fait office de nœuds maîtres. Les données sont répliquées entre les nœuds et les mises à jour et l’insertion peut être possible sur un groupe de nœuds maîtres. Dans ce cas, il existe plusieurs copies des données. Le système est également responsable de la résolution de tout conflit survenant entre des modifications simultanées. Il y a deux raisons principales d’avoir une réplication à plusieurs maîtres : l’un est HA, et le second est la performance. Dans la plupart des cas, certains nœuds sont dédiés à l’opération d’écriture intensive et certains nœuds sont dédiés à certains nœuds ou pour le basculement.

Voici les avantages et les inconvénients de la réplication multi-maîtres

**Avantages:**

- En cas de défaillance d’un maître, l’autre maître est là pour effectuer la mise à jour et l’insertion.
- Les nœuds maîtres sont situés à plusieurs endroits différents, de sorte que les risques de défaillance de tous les maîtres sont très minimes.
- Les mises à jour des données sont possibles sur plusieurs serveurs.
- L’application ne nécessite pas de router le trafic vers un seul maître.

**Les inconvénients:**

- Le principal inconvénient de la réplication multi-maîtres est sa complexité.
- La résolution des conflits est très difficile car des écritures simultanées sur plusieurs nœuds sont possibles.
- Parfois, une intervention manuelle est nécessaire en cas de conflit.
- Risque d’incohérence des données .

Comme nous l’avons déjà évoqué, la réplication Single-Master est suffisante dans la plupart des cas et fortement recommandée, mais il existe néanmoins des cas où la réplication Multi-Master est requise. PostgreSQL a une réplication à maître unique intégrée, mais malheureusement, il n’y a pas de réplication à maîtres multiples dans PostgreSQL principal. Certaines solutions de réplication multimaître sont disponibles, certaines d’entre elles sont sous forme d’applications et d’autres sont des forks PostgreSQL. Ces forks ont leurs propres petites communautés et sont principalement gérés par une seule entreprise, mais pas par la communauté PostgreSQL principale.

 ![image 01](/posts/article08/image01.png)

Il existe plusieurs catégories de ces solutions, open-source/closed source, prioritaires, gratuites et payantes.

1. BDR (bi- directionnel Replication )
2. xDB
3. PostgreSQL -XL
4. PostgreSQL -XC / PostgreSQL -XC2
5. Rubyrep
6. Bucardo

Voici quelques caractéristiques clés de toutes les solutions de réplication

 ![image 02](/posts/article08/image02.png)

1- BDR (Réplication Bidirectionnelle)

[BDR](https://www.2ndquadrant.com/en/resources/postgres-bdr-2ndquadrant/%3Ffbclid%3DIwAR2BUPuFPFAeZX_G_VrI1zsUc6qCWR-sPCwd1uoCThKJgrUfQUblYXON3m0) est une solution de réplication multi-maîtres, et il existe différentes versions. Les premières versions de BDR sont open source mais sa dernière version est fermée. Cette solution est [développée par 2ndQuadrant](https://www.2ndquadrant.com/en/) et l’une des solutions Multimaster les plus élégantes à ce jour. BDR fournit une réplication logique multimaître asynchrone. Ceci est basé sur la fonctionnalité de décodage logique de PostgreSQL . Étant donné que BDR applique essentiellement rejoue la transaction sur les autres nœuds, l’opération de relecture peut échouer s’il y a un conflit entre une transaction appliquée et une transaction qui a été validée sur le nœud de réception.

 ![image 03](/posts/article08/image03.png)

2 – xDB

[EnterpriseDB](https://www.enterprisedb.com/) a développé sa propre solution de réplications bidirectionnelles en Java appelée [xDB](https://www.enterprisedb.com/edb-docs/d/edb-postgres-replication-server/user-guides/user-guide/6.0/EDB_Postgres_Replication_Server_Users_Guide.1.10.html) . Il est basé sur leur propre protocole. Parce qu’il s’agit d’une solution à source fermée, aucune information de conception n’est connue du monde.

- Développé et maintenu par EnterpriseDB
- Développé en Java.
- Le code source est fermé.
- xDB Replication Server contenait plusieurs programmes exécutables
- Il s’agit d’un logiciel propriétaire entièrement fermé
- Développé en Java, les gens se sont plaints de ses performances
- Le temps de basculement n’est pas acceptable.
- L’interface utilisateur est disponible pour la configuration et la maintenance du système de réplication

 ![image 04](/posts/article08/image04.png)

3 – PostgreSQL XC/XC2

PostgreSQL -XC est développé par EnterpriseDB et NTT. C’est une solution de réplication synchrone. Postgres -XC est un projet open source pour fournir une solution de cluster PostgreSQL évolutive en écriture, synchrone, symétrique et transparente. Je n’ai pas vu beaucoup de développement dans PostgreSQL -XC depuis de nombreuses années à partir d’EnterpriseDB et de NTT. Actuellement, Huawei travaille là-dessus. Certains gains de performances ont été signalés dans le cas d’OLAP, mais ne conviennent pas pour TPS.

4 – PostgreSQL XL

C’est un fork de PostgreSQL -XC et actuellement pris en charge par 2ndQuadrant. Il est bien derrière Community PostgreSQL. A ce que je sache, il est basé sur PostgreSQL 10.6, qui n’est pas aligné avec PostgreSQL dernière version PostgreSQL-12. Comme nous savons qu’il est basé sur PostgreSQL -XC, il est très bon quand on parle d’OLAP, mais pas très adapté pour Hight TPS.

 ![image 05](/posts/article08/image05.png)

_Remarque : Tous les logiciels PostgreSQL XC/XC2/XL sont considérés comme des « logiciels dérivés de PostgreSQL » qui ne sont pas synchronisés avec le développement actuel de PostgreSQL._

5 – Rubyrep

Il s’agit d’une réplication maître-maître (master-master) asynchrone développée par Arndt Lehmann. Il revendique la configuration et l’installation les plus simples et fonctionne sur toutes les plates-formes, y compris Windows. Il fonctionne toujours sur deux serveurs, qui sont référencés comme « gauche » et « droite » dans la terminologie Rubyrep. Il est donc logique de l’appeler une configuration « 2 maîtres » plutôt que « multi-maîtres ».

- Le rubyrep peut reproduire en continu les changements entre les bases de données de gauche et de droite.
- Configure automatiquement les déclencheurs nécessaires, les tables de journal, etc.
- Détecte automatiquement les tables nouvellement ajoutées et synchronise le contenu de la table
- Reconfigure automatiquement les séquences pour éviter les conflits de clés en double
- Suit les modifications apportées aux colonnes de clé primaire
- Peut mettre en œuvre à la fois la réplication maître-esclave et maître-maître
- Méthodes de résolution des conflits pré-construites disponibles : victoires gauche/droite ; le changement plus tôt / plus tard gagne
- Résolution de conflit personnalisée spécifiable via des extraits de code ruby
- Les décisions de réplication peuvent éventuellement être enregistrées dans la table du journal des événements rubyrep

_Remarque : – Ce projet n’a pas été actif depuis trois ans, en termes de développement._

6 – Bucardo

[Bucardo](https://github.com/bucardo/bucardo) est une solution de réplication basée sur Trigger développée par Jon Jensen et Greg Sabino Mullane de [End Point Corporation](https://www.endpoint.com/) . La solution de Bucardo existe depuis près de 20 ans maintenant et a été conçue à l’origine comme une solution asynchrone «paresseuse» qui _finit par_ répliquer toutes les modifications. Il existe un Perl daemon qui écoute les requêtes NOTIFY et agit sur celles-ci. Les changements qui se produisent sur les tables sont enregistrés dans une table ( bucardo\_delta ) et notifie le daemon. Daemon avertit le contrôleur qui lance un «kid» pour synchroniser les changements de table. En cas de conflit, des gestionnaires de conflit standard ou personnalisés sont utilisés pour le résoudre.

- Réplication basée sur des déclencheurs
- Politique de résolution des conflits
- Dépendance sur Perl 5, DBI, DBD::Pg, DBIx ::Safe.
- L’installation et la configuration sont complexes.
- La réplication s’interrompt souvent et est boguée.

**Conclusion**

La plupart des cas de réplication à maître unique suffisent, et il a été observé que les gens configurent la réplication multi-maître (multi-master) et compliquent excessivement leur conception. Il est fortement recommandé de concevoir le système et d’essayer d’éviter la réplication multi-maître et de ne l’utiliser que là où il n’y a aucun moyen de concevoir le système sans cela. Il y a deux raisons : la première est que cela rend le système trop complexe et difficile à déboguer, et deuxièmement, vous n’obtiendrez aucun support de la communauté PostgreSQL car il n’y a pas de réplications multi-maîtres gérées par la communauté.

Source : [Blog Percona](https://www.percona.com/blog/2020/06/09/multi-master-replication-solutions-for-postgresql/)