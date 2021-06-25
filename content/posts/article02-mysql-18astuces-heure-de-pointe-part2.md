+++
title = "Les 18 astuces pour mettre au point votre base de données MySQL en heure de pointe (Partie 02)"
description = "Astuces pour améliorer la performance de votre base de donnée MySQL en heure de pointe"
author = "Francis"
date = 2021-04-20T11:43:01+04:00
tags = ['mysql']
Categories = ["Article de Percona"]
featured_image = "posts/article02/18astuces.png"
slug = "article02-mysql-18astuces-heure-de-pointe-part2"
+++

### Les 18 choses que vous pouvez faire pour supprimer les goulots d&#39;étranglement MySQL causés par un trafic élevé (Partie 02)

*Il s&#39;agit d&#39;une série de blogs en trois parties qui se concentre sur la gestion d&#39;un événement inattendu à fort trafic au fur et à mesure qu&#39;il se produit. La première partie peut être consultée [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part1/) et la troisième partie se trouve [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part3/).*

**7. Obtenez plus de mémoire**

_Complexité: faible_

_Impact potentiel: élevé_

Si vos données ne bénéficient pas suffisamment assez de mémoire, vos performances MySQL seront très limitées. Si vos données s&#39;intègrent déjà bien, l&#39;ajout d&#39;encore plus de mémoire n&#39;améliorera pas ses performances.

Même si vous utilisez un stockage très rapide, tel qu&#39;Intel Optane ou un stockage NVMe directement connecté, l&#39;accès aux données en mémoire est un atout majeur.

Comment savoir si vous avez suffisamment de mémoire? Regardez l&#39;utilisation de la mémoire et l&#39;I/O activity.

 ![image 01](/posts/article02/p02_image01.png)

L&#39;I/O activity est en fait le premier élément que je voudrais examiner. Si, comme dans le cas ci-dessus, vous n&#39;avez pas de « Read I/O » à proprement parler, toutes vos données sont dans le cache - que ce soit les caches de données de MySQL ou le cache de fichiers du système d&#39;exploitation. Cependant, le « write activity » ne sera pas complètement éliminée même si toutes les données tiennent dans le cache car les modifications de la base de données doivent être enregistrées sur le disque.

En règle générale, vous ne chercherez pas à éliminer complètement Read IO - Dans la plupart des cas, cela nécessitera trop de mémoire et inutile. Cependant, vous voulez vous assurer que Read IO n&#39;affecte pas considérablement vos performances. Vous pouvez le faire en vous assurant que votre volume de disque est gérable. Si vous avez [Percona Monitoring and Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management), vous pouvez vérifier dans quelle mesure les lectures de disque affectent les performances de vos requêtes spécifiques dans Query Analytics.

 ![image 02](/posts/article02/p02_image02.png)

Remarque: Bien que vous puissiez obtenir une certaine valeur en ajoutant simplement plus de mémoire, elle sera utilisée par le système d&#39;exploitation comme cache. Pour obtenir la plupart de la mémoire nouvellement disponible, vous devez configurer MySQL pour pouvoir l&#39;utiliser. Innodb\_buffer\_pool\_size est la variable la plus importante à considérer. 80% de la mémoire est souvent utilisée en règle générale, mais [il y en a d&#39;autres](https://www.percona.com/blog/2015/06/02/80-ram-tune-innodb_buffer_pool_size/).

Une chose à laquelle vous devez prêter attention, lorsque vous configurez MySQL pour tirer profit de toute votre mémoire, est de vous assurer que vous ne surchargez pas la mémoire et que MySQL ne manque pas de mémoire virtuelle (car il peut planter ou tomber hors service à cause du « Out of Memory ( OOM) Killer »).

 ![image 03](/posts/article02/p02_image03.png)

Vous devriez également vous assurer qu&#39;il n&#39;y a pas d&#39;activité d&#39;échange significative (1 Mo/s ou plus) même si une certaine utilisation de l&#39;espace d&#39;échange est normale. Consultez &quot;[In defense of swap: common misconceptions](https://chrisdown.name/2018/01/02/in-defence-of-swap.html)&quot; pour plus de détails.

 ![image 04](/posts/article02/p02_image04.png)

**8. Passer à un stockage plus rapide**

_Complexité: moyenne_

_Impact potentiel: élevé_

Lorsque la taille de vos données est petite, les mettre en mémoire est le meilleur moyen de lecture. Si votre base de données est volumineuse, cela peut devenir peu pratique et un disque plus rapide pourrait être indispensable. En plus, il y aura des écritures qui doivent être gérées même si vous avez beaucoup de mémoire. Cet article ancien mais toujours valable entre dans [les détails sur ce sujet](https://www.percona.com/blog/2010/04/08/fast-ssd-or-more-memory/).

Avec les processeurs, vous devez savoir qu&#39;à chaque fois que vous avez besoin de plus de cœurs ou de cœurs plus rapides, le stockage devient beaucoup plus compliqué. Vous devez comprendre la différence entre le débit (IOPS) et la latence ([consultez cet article fantastique sur le sujet](https://louwrentius.com/understanding-storage-performance-iops-and-latency.html)) ainsi que la différence entre les performances de lecture et d&#39;écriture.

Une façon d&#39;examiner les performances IO consiste à examiner le nombre de stockage IOPS servi ou la bande passante de l&#39;activité d&#39;IO.

 ![image 05](/posts/article02/p02_image05.png)

Il est très utile de connaitre les limites de votre stockage et de savoir si vous en êtes proche ou que vous y êtes confronté. Vous ne connaissez peut-être pas les performances exactes du stockage. Dans ce cas, il est recommandé de jeter un œil sur **Disk IO Load** qui vous montre approximativement combien d&#39;opérations IO sont en cours.

 ![image 06](/posts/article02/p02_image06.png)

Si vous voyez ce nombre en dizaines ou en centaines, il est probable que votre disque soit surchargé. Le problème avec le stockage, contrairement au CPU, est que nous n&#39;avons aucun moyen de savoir quel est le «niveau naturel de concurrence», quand-est-ce-que les requêtes peuvent se dérouler en parallèle ou quand-est-ce-que la mise en file d&#39;attente doit avoir lieu.

 ![image 07](/posts/article02/p02_image07.png)

Jetez un œil à la latence des demandes pour les lectures et les écritures pour voir s&#39;il y a une différence par rapport à la situation avant le pic de trafic. En outre, les latences de lecture et d&#39;écriture peuvent être affectées indépendamment et doivent être examinées séparément.

Dans quelle mesure un disque plus rapide peut-il avoir un impact sur les performances de vos requêtes? Du point de vue des lectures, vous pouvez vérifier l&#39;analyse des Query Analytics PMM comme je l&#39;ai expliqué dans la **section 7. Obtenir plus de mémoire** , mais pour les écritures, c&#39;est plus compliqué.

Écrire dans InnoDB Redo Log, ou plus précisément, le conserver sur le disque via fsync () est un goulot d&#39;étranglement très courant. Vous verrez si cela se produit dans votre système en regardant le nombre de fsyncs en attente (tableau de bord MySQL Innodb Details, Section Innodb Disk IO).

 ![image 08](/posts/article02/p02_image08.png)

S&#39;il est proche de 1 tout le temps, vous avez probablement un goulot d&#39;étranglement avec le débordement du disque. Pour améliorer la situation, vous aurez besoin d&#39;un stockage avec une meilleure latence d&#39;écriture (fsync ()). Vous pouvez ajuster votre configuration MySQL pour réduire la garantie de durabilité ou ajuster votre volume de travail pour regrouper les requêtes en un plus petit nombre de transactions.

Quelles options de stockage plus rapides sont disponibles? Intel Optane SSD ou NVMe storage a tendance à offrir les meilleures performances et une latence aussi rapide que prévisible. Cependant, si vous utilisez ces solutions, en particulier dans le cloud, assurez-vous d&#39;utiliser une forme de réplication pour la redondance des données.

Si vous devez utiliser le stockage réseau, recherchez des options optimisées pour le débit, telles que le [type de volume EBS io1](https://louwrentius.com/understanding-storage-performance-iops-and-latency.html) AWS. Les gp2 volumes traditionnels «à usage général» peuvent être beaucoup plus rentables, mais ils ont une pointe de performances relativement inférieures.

**9. Vérifiez votre réseau**

_Complexité: faible_

_Impact potentiel: élevé_

Pour vérifier si un réseau est un goulot d&#39;étranglement dans votre trafic en heure de pointe, vous devez examiner la bande passante, la latence et les erreurs.

Les réseaux ont tendance à être plus compliqués que les autres ressources, car toutes ces ressources doivent être mesurées séparément pour différents clients. Par exemple, les clients qui opèrent sur «localhost» ont tendance à ne pas avoir de problème, cependant, les clients situés dans d&#39;autres parties du monde et communiquant avec votre base de données auront des problèmes.

Il est rare que la bande passante du réseau, du moins en ce qui concerne le nœud local, soit un problème.

 ![image 09](/posts/article02/p02_image09.png)

Rarement, les applications récupèrent des ensembles de résultats volumineux et saturent le réseau. Les sauvegardes réseau et autres transferts de données volumineux peuvent saturer le réseau, entraînant ainsi la lenteur des autres transactions des utilisateurs.

La latence entre votre client et le serveur de base de données peut être approximativement mesurée par l&#39;outil «ping» ou «mtr». Si vous avez un réseau de 10 Go, vous pouvez vous attendre à 0,2 ms dans le même centre de données. Il est généralement légèrement plus élevé chez les fournisseurs de cloud au sein de la même zone de disponibilité. Différentes zones à haute disponibilité ont une latence plus élevée et la latence entre les régions distantes peut atteindre 100 ms avec une variance beaucoup plus élevée que le réseau local.

 ![image 10](/posts/article02/p02_image10.png)

Dans ce cas, nous voyons que le chemin entre le client et le serveur passe par un seul routeur (et peut-être quelques commutateurs) avec une latence moyenne de 1,5 ms et aucun paquet perdu.

Vous devez garder votre serveur d&#39;applications et votre base de données aussi proches que possible - dans la même zone de disponibilité si possible, mais sûrement dans la même région pour les applications sensibles à la latence.

En ce qui concerne les erreurs, la retransmission TCP est votre pire ennemi car elle peut ajouter une latence très importante.
 ![image 11](/posts/article02/p02_image11.png)

Si vous constatez une augmentation des taux de retransmission pendant les heures de pointe, il y a des chances qu&#39;il y ait des problèmes au niveau du réseau qui doivent être résolus.

**10. Localisez et optimisez les requêtes à l&#39;origine de la charge**

_Complexité: moyenne_

_Impact potentiel: élevé_

La localisation et l&#39;optimisation des mauvaises requêtes est l&#39;une des activités les plus rentables que vous puissiez faire car elle offre des avantages à long terme. Contrairement au renforcement de votre matériel, il ne nécessite aucun investissement supplémentaire (autre que du temps).

Si vous exécutez [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management), vous devez jeter un œil à l&#39;outil d&#39;analyse des requêtes, qui trie par défaut les requêtes en fonction de la charge qu&#39;elles génèrent.

 ![image 12](/posts/article02/p02_image12.png)

Examiner et optimiser les requêtes dans cet ordre est un moyen fantastique de rendre votre système plus rapide. Dans certains cas, comme la requête de validation, vous ne pouvez pas vraiment optimiser la requête elle-même, mais vous pouvez l&#39;accélérer grâce à des modifications matérielles ou à la configuration de MySQL.

Consultez les détails de l&#39;exécution de la requête:

 ![image 13](/posts/article02/p02_image13.png)

Et lisez l&#39;explication pour voir si et comment cette requête peut être optimisée:
 ![image 14](/posts/article02/p02_image14.png)

L&#39;optimisation des requêtes MySQL est un sujet trop complexe pour être traité dans un seul article de blog. J&#39;envisagerais d&#39;apprendre à lire [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) et d&#39;assister à un [webinar](https://www.percona.com/resources/webinars/how-analyze-and-tune-mysql-queries-better-performance).

**11. Ajouter des index manquants**

_Complexité: faible_

_Impact potentiel: élevé_

L&#39;optimisation complète des requêtes peut nécessiter des modifications de la façon dont une requête est écrite, ce qui nécessite du temps de développement et de test qui peut être difficile à obtenir. C&#39;est pourquoi, lors du premier passage, vous souhaiterez peut-être vous concentrer uniquement sur l&#39;ajout d&#39;index manquants. Cela ne nécessite pas de modifications d&#39;application et c&#39;est raisonnablement sûr (à de rares exceptions près), et ne devrait pas modifier les résultats de la requête.

Consultez ce [webinar](https://www.percona.com/resources/mysql-videos/mysql-indexing-best-practices-mysql-56) pour plus de détails.

**12. Supprimer les index inutiles**

_Complexité: moyenne_

_Impact potentiel: moyen_

Au fil du temps, il est très courant que le schéma de base de données accumule des index dupliqués, redondants ou inutilisés. Certains sont ajoutés par erreur ou par malentendu, d&#39;autres ont été utiles dans le passé, mais ne le sont plus lorsque l&#39;application a changé.

Vous pouvez en savoir plus sur les index redondants et dupliqués dans cet [article de blog](https://www.percona.com/blog/2012/06/20/find-and-remove-duplicate-indexes/). Le vérificateur de clé [pt-duplicate-key](https://www.percona.com/doc/percona-toolkit/LATEST/pt-duplicate-key-checker.html) de Percona Toolkit est également un excellent outil pour les trouver.

Un index inutilisé est à la fois un peu plus compliqué et risqué - ce n&#39;est pas parce qu&#39;il n&#39;y a pas eu de requête nécessitant cet index la semaine dernière qu&#39;il n&#39;y a pas de rapport mensuel ou trimestriel qui en a besoin.

L&#39;article de blog, [Basic Housekeeping for MySQL Indexes](https://www.percona.com/blog/2016/09/09/basic-housekeeping-for-mysql-indexes/), fournit une recette sur la façon de trouver de tels index. Si vous exécutez MySQL 8, vous pouvez envisager de [rendre un tel index invisible](https://dev.mysql.com/doc/refman/8.0/en/invisible-indexes.html) pendant un certain temps avant de le supprimer.

Il s&#39;agit d&#39;une série de blogs en trois parties. La première partie peut être trouvée ici, et la troisième partie peut être trouvée ici.


Page source : [Percona Blog](https://www.percona.com/blog/2020/04/06/18-things-you-can-do-to-remove-mysql-bottlenecks-caused-by-high-traffic-part-two/)