+++
title = "Dois-je créer un index sur les clés étrangères dans PostgreSQL?"
description = "Il est generalement vrai que les index améliorent les performances de lecture, mais nous savons également que cela aura toujours un impact sur les écritures. Dans certains cas, cela peut ne donner aucune amélioration des performances. Les clés étrangères (FK) en sont un excellent exemple."
author = "Francis"
date = 2022-02-15
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article07.jpg"
images = ["thumbnail2022/article07.jpg"]
slug = "index-sur-les-cles-etrangeres-dans-postgresql"
+++

![thumbnail](/thumbnail2022/article07.jpg)

*Bienvenue sur un blog hebdomadaire où je peux répondre (comme, vraiment répondre) à certaines des questions que j'ai vues dans les webinaires que j'ai présentés récemment. Si vous avez manqué le dernier, [PostgreSQL Performance Tuning Secrets ](https://www.brighttalk.com/webcast/18708/513655?utm_source=Percona&utm_medium=brighttalk&utm_campaign=513655), il peut être utile d'en écouter une partie avant ou après avoir lu cet article. Chaque semaine, je vais plonger profondément dans une question. Faites-moi savoir ce que vous pensez dans les commentaires.*

Nous entendons constamment dire que les index améliorent les performances de lecture et c'est généralement vrai, mais nous savons également que cela aura toujours un impact sur les écritures. Ce dont nous n'entendons pas parler trop souvent, c'est que dans certains cas, cela peut ne donner aucune amélioration des performances. Cela se produit plus que nous ne le souhaitons et peut se produire plus que nous ne le remarquons, et les clés étrangères (FK) en sont un excellent exemple. Je ne dis pas que tous les index de FK sont mauvais, mais la plupart de ceux que j'ai vus sont tout simplement inutiles, ne faisant qu'ajouter de la charge au système.

Par exemple, la relation ci-dessous où nous avons une relation 1 :N entre la table « Supplier» et la table « Product» :

![image01](/posts/2022/article07/img01.png)

Si nous prêtons une attention particulière aux FK dans cet exemple, il n'y aura pas un nombre élevé de recherches sur la table enfant en utilisant la colonne FK, " *SupplierID* " dans cet exemple, si nous comparons avec le nombre de recherches en utilisant " *ProductID* " et probablement " *ProductName* ". L'utilisation principale sera de maintenir la cohérence de la relation et de rechercher dans l'autre sens, en trouvant le fournisseur d'un certain produit. Dans ce cas, l'ajout d'un index à l'enfant FK sans s'assurer que le modèle d'accès l'exige ajoutera le coût supplémentaire de la mise à jour de l'index chaque fois que nous mettons à jour la table « *Product ».*

## Un autre point auquel nous devons prêter attention est la cardinalité de l'index
Si la cardinalité de l'index est trop faible, Postgres ne l'utilisera pas et l'index sera simplement ignoré. On peut se demander pourquoi cela se produit et si cela ne serait pas encore moins cher pour la base de données de parcourir, par exemple, la moitié des index au lieu de faire une analyse complète de la table? La réponse est non, en particulier pour les bases de données qui utilisent des tables de tas comme Postgres. L'accès à la table dans Postgres est principalement séquentiel, ce qui est plus rapide que l'accès aléatoire dans les disques durs en rotation et encore un peu plus rapide sur les disques SSD, tandis que l'accès à l'index b+-tree est aléatoire par nature.

Lorsque Postgres utilise un index, il doit ouvrir le fichier d'index, trouver les enregistrements dont il a besoin, puis ouvrir le fichier de table, effectuer une recherche à l'aide des adresses de page obtenues à partir des index, en modifiant le modèle d'accès de séquentiel à aléatoire et en fonction des données distribution, il accédera probablement à la majorité des pages de la table, se terminant par une analyse complète de la table mais utilisant maintenant un accès aléatoire, ce qui est beaucoup plus coûteux. Si nous avons des colonnes avec une faible cardinalité et que nous avons vraiment besoin de les indexer, nous devons utiliser une alternative aux index b-tree, par exemple, un index GIN, mais c'est un sujet pour une autre discussion.

## Avec tout cela, on peut penser que les index FK sont toujours mauvais et ne jamais les utiliser sur une table enfant, n'est-ce pas?
Eh bien, ce n'est pas vrai non plus et il y a de nombreuses circonstances où ils sont utiles et nécessaires, par exemple, l'image ci-dessous a deux autres tableaux, " Customer" et " Order" :

![image02](/posts/2022/article07/img02.png)

Il peut être pratique d'avoir un index sur la table enfant " *Order-> CustomerId* " car il est courant d'afficher toutes les commandes d'un certain utilisateur et la colonne "*CustomerId* " de la table "Order" sera utilisée assez fréquemment comme clé de recherche .

Un autre bon exemple est de fournir une méthode plus rapide pour valider l'intégrité référentielle. Si l'on a besoin de changer la table parent (mettre à jour ou supprimer une clé parent), les enfants doivent être vérifiés pour s'assurer que la relation n'est pas rompue. Dans ce cas, avoir un index du côté de l'enfant aiderait à améliorer les performances. Dans ce cas, l’index est utile mais "dépendant de la charge". Si la clé parent a de nombreuses suppressions, cela peut être envisageable, cependant, s'il s'agit d'une table principalement statique ou ayant principalement des insertions ou des mises à jour dans les autres colonnes autres que la colonne de clé parent, ce n'est pas un bon candidat pour avoir un index sur les tables enfants.

Il existe de nombreux autres exemples qui peuvent être donnés pour expliquer pourquoi un index sur la table enfant peut être utile et vaut le coût/la pénalité d'écriture supplémentaire.

## Conclusion

**Le point à retenir ici est que nous ne devons pas créer d'index sans distinction sur tous les FK, car beaucoup d'entre eux ne seront tout simplement pas utilisés ou si rarement utilisés qu'ils n'en valent pas la peine.** Il est préférable de concevoir initialement la base de données avec les FK mais pas les index et de les ajouter pendant que la base de données se développe et que nous comprenons la charge de travail. Il est possible qu'à un moment donné, nous trouvions que nous ayons besoin d'un index sur « *Product-> SupplierId* » en raison de notre charge de travail et que l'index sur « *Order-> CustomerId* » ne soit plus nécessaire. Les charges changent et la distribution des données également, l'index doit les suivre et ne pas être traité comme des entités immuables.

Source : [Percona](https://www.percona.com/blog/should-i-create-an-index-on-foreign-keys-in-postgresql/)
