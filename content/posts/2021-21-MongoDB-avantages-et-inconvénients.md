+++
title = "Fonctionnalité et limite de MongoDB dans la gestion de votre base de données"
description = "Avantages et incovénients de l'utilisation de MongoDB pour gérer vos base de données"
author = "Francis"
date = 2021-10-22T11:43:01+04:00
tags = ['MongoDB']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle21.jpg"
images = ["thumbnail/thumbnailarticle21.jpg"]
slug = "avantages-et-inconvenients-quand-vous-devriez-et-ne-devriez-pas-utiliser-mongoDB"
+++

Assez souvent, nous voyons que le stockage opérationnel principal est utilisé en conjonction avec certains services supplémentaires, par exemple, pour la mise en cache ou la recherche en texte intégral.

Une autre approche d'architecture utilisant plusieurs bases de données est celle des microservices, où chaque microservice a sa propre base de données mieux optimisée pour les tâches de ce service particulier. Par exemple, vous pouvez utiliser MySQL pour le stockage principal, Redis et Memcache pour la mise en cache, Elastic Search ou Sphinx natif pour la recherche. Vous pouvez appliquer quelque chose comme Kafka pour transférer des données vers le système d'analyse, ce qui était souvent fait sur quelque chose comme Hadoop.

Si nous parlons du stockage opérationnel principal, il peut y avoir deux options. Nous pouvons choisir des bases de données relationnelles avec le langage SQL. Alternativement, nous pouvons opter pour une base de données non relationnelle puis pour l'un de ses types disponibles dans ce cas.

Si nous parlons de modèles de données NoSQL, il y a aussi beaucoup de choix. Les plus courantes sont les bases de données clé-valeur, de document ou à colonnes étendues.

Les exemples sont Memcache, MongoDB et Cassandra, respectivement.

En regardant [DB-Engines Ranking](http://db-engines.com/en/ranking) , nous verrons que la popularité des bases de données open source a augmenté au fil des années, tandis que les bases de données commerciales ont progressivement diminué. 

![image01](/posts/article21/img01.png)

Ce qui est encore plus intéressant, la même tendance a été observée pour différents types de bases de données : les bases de données open source sont les plus populaires pour de nombreux types tels que les bases de données en colonnes, les séries chronologiques et les histoires de documents. Les licences commerciales ne prévalent que pour les technologies classiques telles que les données de bases de données relationnelles ou encore plus anciennes comme les bases de données multivaleurs.

![image02](/posts/article21/img02.png)



Chez Percona, nous travaillons en étroite collaboration avec les bases de données open source relationnelles et non relationnelles les plus populaires (MySQL, PostgreSQL et MongoDB), traitant de nombreux clients ; nous les aidons à faire des choix et leur fournissons les meilleurs conseils pour chaque cas.

Dans cet esprit, cet article a pour but de montrer des scénarios qui valent la peine d'être envisagés avant de déployer MongoDB, vous amenant à penser à quand vous devriez et ne devriez pas l'utiliser. De plus, si vous avez déjà votre configuration, cet article peut également être intéressant, car lors de l'évaluation d'un produit, certains des sujets suivants peuvent être passés inaperçus.

## Table des matières

Voici une liste de sujets dont je parlerai plus loin dans cet article :

1. ***Expérience et préférences de l'équipe***
1. ***Approche de développement et cycle de vie de l'application***
1. ***Modèle de données***
1. ***Transactions et cohérence (ACID)***
1. ***Évolutivité***
1. ***Administration***

## 1. Expérience et préférences de l'équipe

Avant de plonger dans MongoDB, le plus important est de prendre en compte l'expérience et les préférences de l'équipe.

Du point de vue de MongoDB, l'avantage est que nous avons des documents au format JSON flexibles, et pour certaines tâches et certains développeurs, c'est pratique. Pour certaines équipes, c'est difficile, surtout si elles travaillent depuis longtemps avec des bases de données SQL et comprennent très bien l'algèbre relationnelle et le langage SQL.

Dans MongoDB, vous pouvez facilement vous familiariser avec les opérations [CRUD](https://docs.mongodb.com/manual/crud/) telles que :

- [find()](https://docs.mongodb.com/manual/tutorial/query-documents/)
- [insert()](https://docs.mongodb.com/manual/tutorial/insert-documents/)
- [update()](https://docs.mongodb.com/manual/tutorial/update-documents/)
- [delete()](https://docs.mongodb.com/manual/tutorial/remove-documents/)

Les requêtes simples sont moins susceptibles de causer des problèmes. Néanmoins, dès que la tâche quotidienne nécessite un traitement de données plus approfondi, vous avez certainement **besoin d'un outil puissant pour gérer cela** , comme [le pipeline d'agrégation](https://docs.mongodb.com/manual/core/aggregation-pipeline/) MongoDB et [map-reduce](https://docs.mongodb.com/manual/core/map-reduce/) , dont nous discuterons plus en détail dans cet article.

Il existe d'excellents cours gratuits disponibles à l'[Université MongoDB](https://university.mongodb.com/) qui peuvent sans aucun doute aider à développer les connaissances de l'équipe. Néanmoins, il est important de garder à l'esprit que le sommet de la courbe d'apprentissage peut prendre un certain temps à atteindre si l'équipe ne le connaît pas entièrement.

## 2. Approche de développement et cycle de vie des applications

Si nous parlons d'applications où MongoDB est utilisé, elles se concentrent principalement sur un développement rapide car vous pouvez tout changer à tout moment. Vous n'avez pas à vous soucier du format strict du document.

Le deuxième point est un schéma de données. Ici, vous devez comprendre que les données ont toujours le schéma ; la seule question est de savoir où il est mis en œuvre. Vous pouvez implémenter le schéma de données dans votre application car, d'une manière ou d'une autre, ce sont les données que vous utilisez. Ou ce schéma est implémenté au niveau de la base de données.

C'est assez souvent quand vous avez une application, et seule cette application traite les données de la base de données. Par exemple, si nous sauvegardons les données de l'application dans une base de données, le schéma au niveau de l'application fonctionne bien. Mais, si nous avons les mêmes données utilisées par de nombreuses applications, cela devient très gênant et difficile à contrôler.

Un point de vue du cycle de développement d'applications peut être représenté comme suit :

- *Vitesse de développement*
- *Pas besoin de synchroniser le schéma dans la base de données et l'application*
- *Il est clair comment évoluer davantage*
- *Solutions prédéterminées simples*

## 3. Modèle de données

Comme mentionné dans le premier sujet, le modèle de données dépend fortement de l'application et de l'expérience de l'équipe.

Les données de nombreuses applications Web sont généralement faciles à afficher. Parce que si nous stockons la structure, quelque chose comme un tableau associé de l'application, il est simple et clair pour le développeur de la sérialiser dans un document JSON.

Prenons un exemple. Nous voulons enregistrer une liste de contacts à partir du téléphone. Il y a des données qui rentrent bien dans une table relationnelle : prénom, nom, etc. Mais si vous regardez les numéros de téléphone ou les adresses e-mail, une personne peut en avoir plusieurs. Si nous voulons le stocker sous une bonne forme relationnelle, ce serait bien de l'avoir dans des tables séparées, puis de le collecter à l'aide de JOIN, ce qui est moins pratique que de le stocker dans une collection avec des documents hiérarchiques.

Modèle de données – Exemple de liste de contacts

***Base de données relationnelle***

- *Nom, prénom, date de naissance*
- *Une personne peut avoir plusieurs numéros de téléphone et e-mails*
- *Vous devez créer des tables séparées pour eux*
- *Les tableaux JSON sont des extensions non traditionnelles*

***Base de données orientée document***

- *Tout est stocké dans une « collection ».*
- *Tableaux et documents intégrés*

**Cependant**, il est crucial de considérer qu'une solution plus flexible résulte en une liste de documents qui peuvent avoir des structures complètement différentes. Comme quelqu'un l'a déjà dit, « *Avec un grand pouvoir vient une grande responsabilité*.»

**Malheureusement, il est assez courant** de voir des opérations échouer dans la gestion de documents de plus de [16 Mo](https://docs.mongodb.com/manual/reference/limits/#mongodb-limit-BSON-Document-Size) ou d'une seule collection contenant des téraoctets de données ; Ou, dans un pire scénario, les clés de partition ont été mal conçues.

Ces anomalies pourraient être une bonne indication que vous transformez votre base de données en un [Data Swamp](https://developer.ibm.com/articles/ba-data-becomes-knowledge-2/) . C'est un terme couramment utilisé dans le déploiement de Big Data pour des données mal conçues, insuffisamment documentées ou mal entretenues.

![image03](/posts/article21/img03.png)

Vous n'avez pas besoin de normaliser strictement vos données, mais il est essentiel de prendre le temps et d'analyser comment vous allez structurer vos données pour avoir le meilleur des mondes une fois que vous utilisez MongoDB et éviter ces pièges.

Vous pouvez consulter l'article de blog « [Schema Design in MongoDB vs Schema Design in MySQL](https://www.percona.com/blog/2013/08/01/schema-design-in-mongodb-vs-schema-design-in-mysql/) » pour obtenir une meilleure clarification sur la modélisation des données et en quoi elle diffère. Il convient de mentionner la **fonction de [**validation de schéma](https://www.percona.com/blog/2018/08/16/mongodb-how-to-use-json-schema-validator/)** que vous pouvez utiliser **lors des** mises **à** jour et des insertions. Vous pouvez **définir des règles de validation** par collection, en restreignant le type de contenu stocké.

**Termes**

Fait intéressant, lors de la modélisation et de l'interrogation, il existe de nombreux points communs entre les SGBD relationnels et non relationnels. Nous parlons de bases de données dans les deux cas, mais ce que nous appelons une table dans une base de données relationnelle est souvent appelé une collection dans une base de données non relationnelle. Qu'est-ce qu'une colonne dans SQL, c'est un champ dans MongoDB, et la liste continue.

En termes d'utilisation de JOIN, dont nous avons parlé, MongoDB n'a pas un tel concept. Cependant, vous pouvez utiliser [$lookup](https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/) sur votre pipeline d'agrégation. Il effectue uniquement une **jointure externe gauche** sur votre recherche ; L'utilisation intensive de [$lookup](https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/) peut indiquer une erreur dans la modélisation de vos données.

Quant à l'accès : nous appliquons SQL pour les données relationnelles. Pour MongoDB et de nombreuses autres bases de données NoSQL, nous utilisons un standard tel que CRUD. Cette norme dit qu'il y a des opérations pour créer, lire, supprimer et mettre à jour des documents.

Vous trouverez ci-dessous quelques exemples des tâches les plus courantes pour traiter des documents et leur équivalent dans le monde SQL :

- CRÉER:
```
db.customers.insert({
  "CustomerName":"Cardinal",
  "ContactName":"Tom B. Erichsen",
  "Address":"Skagen 21",
  "City":"Stavanger",
  "PostalCode":4006,
  "Country":"Norway"
})
```

```
INSERT INTO customers
            (customername,
            contactname,
            address,
            city,
            postalcode,
            country)
VALUES      ('Cardinal',
            'Tom B. Erichsen',
            'Skagen 21',
            'Stavanger',
            '4006',
            'Norway');
```

- LIRE:
```
db.customers.find({
  "PostalCode":{
      "$eq":4006}},
{
  "_id":0,
  "customername":1,
  "City":1
})
```

```
SELECT customername,
      city
FROM   customers
WHERE  PostalCode = 12346;
```


- **METTRE À JOUR:**
```
db.customers.update({
  "CustomerName":"Cardinal"},
{"$set":{"City":"Oslo"}})
```

```
UPDATE customers
SET    city = 'Oslo'
WHERE  customername = 'Cardinal';
```


- **EFFACER:**
```
db.customers.remove({
  "customername":"John"
})
```

```
DELETE FROM customers
WHERE  customername = 'John';
```

Si vous êtes un développeur familiarisé avec le langage JavaScript, cette syntaxe fournie par CRUD (MongoDB) vous sera plus naturelle que la syntaxe SQL.

À mon avis, lorsque nous avons les opérations les plus simples, telles que la recherche ou l'insertion, elles fonctionnent toutes assez bien. Lorsqu'il s'agit d'opérations d'échantillonnage plus délicates, le langage SQL est beaucoup plus lisible.

- **COMPTER:**
```
db.customers.find({
  "PostalCode":{
      "$eq":4006
  }
}).count()
```

```
SELECT Count(1)
FROM   customers
WHERE  postalcode = 4006;
```


Avec l'interface, il est assez facile de faire des choses comme compter le nombre de lignes dans un tableau ou une collection.

- **Agrégation**

```
db.customers.aggregate([
  {
      "$group":{
        "_id":"$City",
        "total":{
            "$sum":1
        }
      }
  }
])
```

```
SELECT city,
      Count(1)
FROM   customers
GROUP  BY city;
```

Mais si nous faisons des choses plus complexes comme GROUP BY dans MongoDB, le framework d'agrégation sera nécessaire. Il s'agit d'une interface plus complexe qui montre comment nous voulons filtrer, comment nous voulons regrouper, etc.

## 4. Transactions et cohérence (ACID)

La raison pour laquelle ce sujet est abordé est que, **selon les besoins de l'entreprise** , la solution de base de données **peut devoir être compatible avec ACID** . Dans ce jeu, les bases de données relationnelles **sont loin devant.** Un excellent exemple d'exigences ACID est les opérations impliquant de l'argent.

**Imaginez que** vous construisiez une fonction pour transférer de l'argent d'un compte à un autre. Si vous retirez avec succès de l'argent du compte source **mais ne le créditez jamais** à la destination ; **Ou** si vous avez plutôt crédité la destination mais n'avez jamais retiré d'argent de la source pour la couvrir. Ces deux écritures doivent se produire ou les deux ne parviennent pas à maintenir notre système sain, savent également « *tout ou rien* ».

**Avant la version 4.0, MongoDB ne prenait pas en charge les transactions**, mais il prenait en charge les opérations atomiques dans un seul document. 

Cela signifie que, du point de vue d'**un document, l'opération sera atomique**. Si le processus modifie plusieurs documents et qu'un échec se produit pendant la modification, certains de ces documents seront modifiés et d'autres non. 

**Cette restriction pour MongoDB a été levée avec la version 4.0 et au-delà** . Pour les situations qui nécessitent l'atomicité des lectures et des écritures sur plusieurs documents (dans une ou plusieurs collections), **MongoDB prend en charge les transactions multi-documents**. Il peut être utilisé dans plusieurs opérations, collections, bases de données, documents et fragments avec des transactions distribuées.

- **Dans la version 4.0** , MongoDB prend en charge les transactions multi-documents sur les jeux de réplicas.
- **Dans la version 4.2** , MongoDB introduit les transactions distribuées, qui ajoute la prise en charge des transactions multi-documents sur les clusters partitionnés et intègre la prise en charge existante des transactions multi-documents sur une Replica Set

## 5. Évolutivité

Qu'est-ce que l'évolutivité dans ce contexte? C'est la facilité avec laquelle vous pouvez prendre une petite application et la faire évoluer vers des millions, voire des milliards d'utilisateurs.

Si l'on parle de l'évolutivité d'un cluster où nos applications sont déjà suffisamment volumineuses, il est clair qu'une machine ne s'en sortira pas, même si c'est la plus puissante.

Il est également logique de déterminer si nous mettons à l'échelle les lectures, les écritures ou le volume de données. Les priorités peuvent différer selon les applications, mais en général, si l'application est très volumineuse, elles doivent généralement gérer toutes ces choses.

Dans MongoDB, l'accent était initialement mis sur l'évolutivité sur plusieurs nœuds. Même dans le cas d'une petite application. Nous pouvons le remarquer sur la fonctionnalité Sharding publiée au début, qui a été développée et est devenue plus mature depuis lors.

Si vous recherchez **une évolutivité verticale** , elle peut être réalisée dans MongoDB via la configuration de l' [ensemble de répliques](https://docs.mongodb.com/manual/replication/) . Vous pouvez augmenter et réduire votre base de données en très peu d'étapes, mais le fait est que seules **votre disponibilité** et **vos lectures** sont mises à l'échelle. Vos écritures sont toujours liées à un seul point, le principal.

Cependant, nous savons que l'application demandera plus de capacité d'écriture à un moment donné, ou que l'ensemble de données deviendra trop volumineux pour le [Replica Set](https://docs.mongodb.com/manual/replication/) ; Il est donc recommandé de procéder à une mise à l'échelle horizontale en utilisant [Sharding](https://docs.mongodb.com/manual/sharding/) , en divisant l'ensemble de données et en écrivant sur plusieurs fragments.

Le sharding MongoDB a quelques limitations : toutes les opérations ne fonctionnent pas avec, **et une mauvaise conception** sur les clés de shard peut [diminuer les performances des requêtes](https://docs.mongodb.com/manual/core/sharding-troubleshooting-shard-keys/#std-label-sharding-troubleshooting-scatter-gather) , créer [des données inégalement réparties](https://docs.mongodb.com/manual/core/sharding-troubleshooting-shard-keys/#uneven-load-distribution) et avoir un impact sur le fonctionnement interne du cluster en tant que [fractionnement automatique des données](https://docs.mongodb.com/manual/core/sharding-data-partitioning/#indivisible-jumbo-chunks) et, dans le pire des cas, **exigeant [un re- sharding**](https://docs.mongodb.com/v4.4/faq/sharding/#can-i-select-a-different-shard-key-after-sharding-a-collection-)** , qui est une opération **étendue et sujette aux erreurs**.

Avec la sortie de **MongoDB 5.0** , une fonctionnalité de [repartitionnement ou resharding](https://docs.mongodb.com/v5.0/core/sharding-reshard-a-collection/#std-label-sharding-resharding) a été récemment introduite. Comme pour toute nouvelle fonctionnalité, ma recommandation est de tester de manière approfondie avant toute utilisation en production. Si, à un moment donné, vous envisagez des approches **pour affiner votre clé de partition,** puis **repartir** avec la nouvelle fonctionnalité, l'article [**Affiner les clés de partition dans MongoDB 4.4 et plus](https://www.percona.com/blog/refining-shard-keys-in-mongodb-4-4-and-above/)** peut vous guider pour un meilleur choix.

## 6. Adminsitration

L'administration est toutes ces choses auxquelles les développeurs ne pensent pas. Du moins, ce n'est pas leur première priorité. L'administration est tout au sujet de la nécessité de sauvegarder, mettre à jour, surveiller, restaurer une application en cas de panne.

MongoDB est davantage axé sur la méthode standard – l'administration est minimisée. Mais il est clair que cela se fait au détriment de la flexibilité. **Une communauté de solutions open source pour MongoDB est nettement plus petite.** Vous pouvez le remarquer dans le [DB-Engines Ranking](https://db-engines.com/en/ranking) mis en évidence au début de cet article et l'enquête annuelle de [StackOverflow](https://insights.stackoverflow.com/survey/2020#technology-databases) ; sans aucun doute, MongoDB est la base de données NoSQL la plus populaire, mais malheureusement, elle manque d'une forte communauté.

De plus, de nombreuses choses recommandées dans MongoDB sont **assez étroitement liées** aux services Ops Manager et Atlas, **qui sont des plates-formes commerciales de MongoDB**.

Jusqu'à récemment, l'exécution de routines de **sauvegarde/restauration** n'était pas des opérations triviales pour le [cluster partagé](https://docs.mongodb.com/manual/sharding/) ou le [ReplicaSet](https://docs.mongodb.com/manual/replication/) . Les administrateurs de bases de données devaient s'appuyer sur des méthodes autour de l'outil [mongodump](https://docs.mongodb.com/database-tools/mongodump/) / [mongorestore](https://docs.mongodb.com/database-tools/mongorestore/#mongodb-binary-bin.mongorestore) ou sur l'utilisation de File System Snapshot.

Ce **scénario a commencé à s'améliorer** avec des fonctionnalités telles que [Percona Hot-Backup](https://www.percona.com/doc/percona-server-for-mongodb/LATEST/hot-backup.html) et l' [outil Percona Backup for MongoDB](https://www.percona.com/doc/percona-backup-mongodb/index.html).

Si nous vérifions la base de données relationnelle la plus populaire comme MySQL, elle est suffisamment flexible et a de nombreuses approches différentes. Il existe de bonnes implémentations open source pour tout, qui sont des faiblesses qui existent toujours dans MongoDB.

## Conclusion

J'ai discuté de quelques sujets qui pourraient vous aider dans votre routine quotidienne, en vous offrant une vision large des avantages de MongoDB. Il est important de considérer que cet article est écrit au-dessus de la dernière version disponible [MongoDB 5.0](https://docs.mongodb.com/manual/release-notes/5.0/) ; Si vous disposez déjà d'un déploiement, mais qu'il utilise des versions plus anciennes ou [obsolètes](https://www.mongodb.com/support-policy/legacy) , certaines observations et fonctionnalités peuvent ne pas être valides.

Si vous rencontrez un problème ou avez une question à un niveau granulaire, veuillez consulter notre blog ; nous avons peut-être écrit un article à ce sujet ; Nous vous invitons également à consulter notre livre blanc [***ici***](https://www.percona.com/resources/white-papers/why-choose-mongodb) , dans lequel nous détaillons plus de scénarios et de cas où MongoDB est un bon choix, et où ce n'est pas le cas.

J'espère que ceci vous aide!

**Percona Distribution for MongoDB est une alternative de base de données MongoDB disponible gratuitement, vous offrant une solution unique qui combine les meilleurs et les plus importants composants d'entreprise de la communauté open source, conçus et testés pour fonctionner ensemble.**

## [Télécharger Percona Distribution pour MongoDB](https://www.percona.com/software/mongodb)
