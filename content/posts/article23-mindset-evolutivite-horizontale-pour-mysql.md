+++
title = "MySQL Traitement de donnée volumineux et Partage horizontal "
description = "Astuces MySQL pour traiter des données volumineux. Face à un ensemble de données qui explose, l' équipe des services professionnels de Percona peut vous aider à concevoir une solution plus flexible et évolutive."
author = "Francis"
date = 2021-11-07T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle23.jpg"
images = ["thumbnail/thumbnailarticle23.jpg"]
slug = "mysql-donnee-volumineux-et-partage-horizontal"
+++

En tant que Responsable Technique chez Percona, je travaille avec plusieurs de nos plus gros clients. Bien que les secteurs verticaux varient, un défi principal reste généralement le même : que dois-je faire avec toutes ces données ? Traiter des ensembles de données volumineux dans MySQL n'est pas un nouveau défi, mais la meilleure approche n'est toujours pas triviale. Chaque application est évidemment différente, mais je voulais discuter de certaines des meilleures pratiques principales concernant le traitement des lacs de données.

## Gardez les instances MySQL petites

Tout d'abord, l'architecture doit être conçue pour garder chaque instance MySQL relativement petite. Une question très courante que me posent les équipes novices dans l'utilisation de MySQL est : "Alors, quelle est la plus grande taille d'instance prise en charge par MySQL ?". Ma réponse remonte à mon temps de consultant : « Cela dépend ». Mon instance MySQL peut-elle prendre en charge un ensemble de données de 20 To ? Peut-être, mais cela dépend du modèle de charge de travail. Dois-je stocker 20 To de données dans une seule instance MySQL? Dans la plupart des cas, absolument pas.

MySQL peut certainement stocker des quantités massives de données. Mais les SGBDR sont conçus pour stocker, écrire et lire ces données. Lorsque les données deviennent aussi volumineuses, les performances de lecture commencent souvent à souffrir. Mais que se passe-t-il si mon jeu de données de travail tient toujours dans la RAM ? C'est souvent la considération critique lorsqu'il s'agit de dimensionner une instance. Dans ce cas, les opérations de lecture/écriture actives peuvent rester rapides, mais que se passe-t-il lorsque vous devez effectuer une sauvegarde ou MODIFIER une table ? Vous lisez (et écrivez) 20 To qui seront toujours limités par les E/S.

Alors, quel est le nombre magique pour le dimensionnement ? De nombreux magasins à grande échelle essaient de maintenir la taille des instances individuelles sous la barre des 2-3 To. Cela se traduit par quelques avantages majeurs :

- Temps de fonctionnement prévisibles (sauvegardes, modifications, etc.)
- Permet un matériel optimisé et standardisé
- Potentiel de parallélisme dans le chargement des données

Si je sais que mon instance ne dépassera jamais quelques téraoctets, je peux optimiser pleinement mes systèmes pour cette taille de données. Les résultats sont des actions opérationnelles prévisibles et reproductibles. Désormais, lorsqu'une sauvegarde est « lente », elle est presque assurément due au matériel et n'est pas une instance aberrante dont la taille est le double. Il s'agit d'une énorme victoire pour l'équipe d'exploitation dans la gestion de l'infrastructure globale. En plus des sauvegardes, vous avez la considération supplémentaire du temps de restauration. Les sauvegardes massives ralentiront la restauration et auront un impact négatif sur le RTO.

## Stocker moins de données

Maintenant que l'impact négatif des grandes instances individuelles est connu, voyons comment nous réduisons les tailles. Bien qu'apparemment évident, le meilleur moyen de réduire la taille des données est de stocker moins de données. Il y a plusieurs façons d'aborder cela :

- Optimiser les types de données
  - Si les types de données sont plus gros que nécessaire, cela entraîne un encombrement disque excessif (c'est-à-dire utiliser bigint quand int suffira)
- Examiner les index pour le ballonnement
  - Limiter les clés primaires composites (PK)
  - Rechercher et supprimer les index redondants (à l'aide de [pt-duplicate-key-checker](https://www.percona.com/doc/percona-toolkit/LATEST/pt-duplicate-key-checker.html)
  - Évitez les PK en utilisant varchar
- Purger les anciennes données
  - Dans la mesure du possible, supprimez les enregistrements qui ne sont pas lus
  - Des outils comme [pt-archiver](https://www.percona.com/doc/percona-toolkit/LATEST/pt-archiver.html) peuvent vraiment aider dans ce processus

Ces techniques peuvent vous aider à retarder le besoin de techniques plus avancées. Cependant, dans certains cas (en raison de la conformité, d'une flexibilité limitée, etc.), les options ci-dessus ne sont pas possibles. Dans d'autres cas, vous les faites peut-être déjà et atteignez toujours les limites de taille.

## Partage horizontal

Alors, quelle est une autre façon de gérer des ensembles de données volumineux dans MySQL? Lorsque toutes les autres options sont épuisées, vous devez envisager de diviser les données horizontalement et de les répartir sur plusieurs instances de taille égale. Malheureusement, c'est beaucoup plus facile à dire qu'à faire. Bien qu'il existe des outils et des options pour MySQL (tels que Vitess), la meilleure approche et la plus flexible consiste souvent à intégrer directement cette logique de partitionnement dans votre application. Le partitionnement peut être effectué de manière statique (module de clé par exemple) ou plus dynamique (via une recherche dans un dictionnaire) ou une approche hybride des deux :

![image01](/posts/article23/img01.png)

## Considérations relatives au partage

Lorsque vous devez enfin mordre la balle et diviser les données horizontalement, il y a certainement certaines choses à garder à l'esprit. Tout d'abord, il est impératif de choisir la bonne clé de partitionnement. Avec la mauvaise clé, les fragments ne seront pas équilibrés et vous vous retrouverez avec des tailles partout. Cela devient alors le même problème lorsqu'un fragment peut devenir trop gros.

Une fois que vous avez la bonne clé, vous devez comprendre que différentes charges de travail seront affectées différemment par le sharding. Lorsque les données sont réparties sur plusieurs partitions, les recherches individuelles sont généralement les plus faciles à mettre en œuvre. Vous prenez la clé, mappez sur un fragment et récupérez les résultats. Cependant, si la charge de travail nécessite un accès agrégé (pensez aux rapports, aux totaux, etc.), vous devez maintenant combiner plusieurs fragments. Il s'agit d'un défi principal et majeur lorsque l'on examine le sharding horizontal. Comme c'est le cas dans la plupart des architectures, les exigences de l'entreprise et la charge de travail dicteront la conception.

Si votre équipe est aux prises avec un ensemble de données qui explose, l' [équipe des services professionnels de Percona](https://www.percona.com/services/consulting) peut vous aider à concevoir une solution plus flexible et évolutive. Chaque cas est unique et notre équipe peut travailler avec votre cas d'utilisation spécifique et vos exigences commerciales pour vous guider dans la bonne direction. La chose la plus importante à retenir : ne vous contentez pas d'ajouter de l'espace disque dur à vos instances en vous attendant à ce qu'elles évoluent. Une conception appropriée et un partitionnement horizontal sont des facteurs critiques à mesure que vos données se développent ! Consultez le prochain article de la série pour [plus de théorie et de considérations sur le sharding](https://www.percona.com/blog/horizontal-scaling-in-mysql-sharding-followup/) , ainsi qu'un exemple d'implémentation ProxySQL léger.

**Percona Distribution for MySQL est la solution MySQL open source la plus complète, stable, évolutive et sécurisée disponible, offrant des environnements de base de données de niveau entreprise pour les applications les plus critiques… et son utilisation est gratuite !**

## Télécharger Percona Distribution for MySQL

Source : [Blog Percona](https://www.percona.com/software/mysql-database)

