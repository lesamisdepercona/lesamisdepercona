+++
title = "Améliorer la performance de  votre base de donnée MySQL (Partie 03)"
description = "Guide pour améliorer la performance de votre base de donnée MySQL en heure de pointe"
author = "Francis"
date = 2021-04-30T11:43:01+04:00
tags = ['mysql']
Categories = ["Article de Percona"]
featured_image = "posts/article02/18astuces.png"
images = ["thumbnail/amisdepercona21-002.jpg"]
slug = "ameliorer-performance-de-base-de-donnee-MySQL-part03"
+++



*Il s’agit d’une série de blogs en trois parties qui se concentre sur la gestion d’un événement inattendu à fort trafic au fur et à mesure qu’il se produit. La première partie peut être trouvée [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part1/) , et la deuxième partie peut être trouvée [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part2/).*


**13. Configurer correctement le serveur MySQL**

_Complexité: moyenne_

_Impact potentiel: élevé_

Un serveur MySQL mal configuré peut causer de graves problèmes, en particulier sous une charge élevée lors des heures de pointes, mais il n’est pas si difficile de bien comprendre les bases. Bien que MySQL Server dispose de plus de 400 variables que vous pouvez régler, vous devez rarement en modifier plus de 10 à 20 pour obtenir 95% des performances possibles pour votre volume de travail.

[Cet article](https://www.percona.com/blog/2016/10/12/mysql-5-7-performance-tuning-immediately-after-installation/) de blog couvre les bases les plus importantes.

**14. Purger les données inutiles**

_Complexité: moyenne_

_Impact potentiel: moyen_

Toutes choses étant égales par ailleurs, plus vous avez de données, plus la base de données fonctionnera lentement. La suppression (ou l’archivage) des données dont vous n’avez pas besoin en ligne est un excellent moyen d’améliorer les performances.

Dans de nombreux cas, vous constatez que les applications conservent divers journaux dans la base de données remontant à plusieurs années, où ils ne sont guère utilisés au-delà de quelques semaines ou de quelques mois.

[Pt-archiver](https://www.percona.com/doc/percona-toolkit/LATEST/pt-archiver.html) de Percona Toolkit est un excellent outil pour archiver les anciennes données.

Remarque: Bien qu’une fois la purge terminée, vous disposerez d’un archivage de base de données plus simple et plus rapide, le processus lui-même nécessite des ressources supplémentaires et ce n’est pas quelque chose que vous devriez faire tant que votre base de données est déjà surchargée.

**15. Terminer la maintenance de la base de données**

_Complexité: moyenne_

_Impact potentiel: moyen_

Lorsque les choses étaient calmes, vous pouvez vous en tirer sans maintenir votre base de données. Cependant, en tant que telles, les statistiques de la base de données peuvent être obsolètes et vos tables peuvent être fragmentées et autrement pas dans l’état le plus optimal.

Exécutez OPTIMIZE TABLE sur vos tables pour les reconstruire, les rendre plus efficaces et mettre à jour les statistiques.

Pour exécuter OPTIMIZE pour toutes les tables, vous pouvez exécuter mysqlcheck –optimize -A.

Gardez à l’esprit que l’optimisation aura probablement un impact encore plus important sur votre système que la purge d’anciennes données, vous ne voudrez peut-être pas le faire en cas de volume élevée. Une bonne approche peut être de supprimer vos réplicas du service du trafic et d’exécuter le processus un à un lors de la promotion de l’un de ces réplicas à une source.

**16. Vérifiez vos travaux d’arrière-plan**

_Complexité: moyenne_

_Impact potentiel: moyen_

Les tâches en arrière-plan telles que la sauvegarde, la maintenance, la génération de rapports et les charges de données importantes ne sont souvent pas très bien optimisées- elles peuvent être exécutées dans une période plus lente où le serveur MySQL peut gérer la charge supplémentaire. Pendant les pics de trafic, ils peuvent entraîner une surcharge de la base de données et des temps d’arrêt.

Un autre problème avec les travaux d’arrière-plan exécutés pendant les heures de pointe est le chevauchement ou la boule de neige. Si votre travail en arrière-plan s’exécute normalement 15 minutes et que vous en avez planifié deux tâche à 2 heures du matin et à 3 heures du matin, un seul d’entre eux s’exécute généralement à la fois. Cependant, en raison de la charge supplémentaire, son exécution peut prendre deux heures et vous pouvez avoir plusieurs travaux d’arrière-plan en cours d’exécution en même temps, entraînant une charge supplémentaire et une possible corruption des données.

Vérifiez vos emplois en arrière-plan et posez les questions suivantes:

- Ai-je besoin de ce travail d’arrière-plan ou peut-il être reporté?

- Ce travail peut-il être exécuté sur un réplica? Exécuter différentes tâches sur différentes réplicas peut être une excellente solution!

- Avez-vous planifié vos travaux par lots pour vous assurer qu’ils ne se chevauchent pas?

- Est-il possible d’optimiser un travail d’arrière-plan? Optimisez les requêtes qu’il utilise ou si vous faites une sauvegarde avec mysqldump, vous devriez utiliser [Percona Xtrabackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) à la place, ce qui est beaucoup plus efficace.

- Pouvez-vous limiter les ressources utilisées par ce travail? Par exemple, limiter la concurrence (nombre de connexions parallèles) qu’un travail par lots utilise. Ou, si vous exécutez Percona Xtrabackup et que cela a un impact sur les performances de votre serveur, vous pouvez limiter les sauvegardes ou [Throttle Backups.](https://www.percona.com/doc/percona-xtrabackup/LATEST/advanced/throttling_backups.html)

**17. Rechercher les hotspots de données**

_Complexité: élevée_

_Impact potentiel: élevé_

Certaines applications évoluent très bien avec la mise à l’échelle matérielle, et d’autres un peu moins. La différence se produit généralement lorsque les applications s’appuient sur des «hotspots» - des données qui doivent être mises à jour si fréquemment qu’elles deviennent un goulot d’étranglement. Par exemple, si vous créez un compteur unique dans la base de données où chaque transaction devra le mettre à jour, il ne sera pas bien mis à l’échelle.

Il existe de nombreux types de points d’accès, et certains d’entre eux sont difficiles à trouver et à diagnostiquer. Le plus courant est similaire à ce qui est décrit ci-dessus et se manifeste par des attentes de verrouillage au niveau des lignes élevées ou row-level locks (et un taux d’interblocage élevé ou high deadlock rate).

Avec [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management), vous pouvez consulter le tableau de bord MySQL Innodb Details pour voir quelle partie du temps global a-t-on servi pour attendre les row-level locks:
 ![image 01](/posts/article02/p03_image01.png)

Ou consultez les taux de retour ou rollback rates :

 ![image 02](/posts/article02/p03_image02.png)

Notez que différentes applications peuvent avoir des valeurs normales différentes pour celles-ci si vous les avez vues en dehors de la norme lors des heures de pointe.

Vous pouvez également examiner quelles requêtes spécifiques ont de longues attentes de verrouillage au niveau des lignes:

 ![image 03](/posts/article02/p03_image03.png)

La réduction des hotspots peut être aussi simple que l’ajout d’un meilleur index, tout comme cela peut se révéler plus difficile, nécessitant une refonte de l’application. Quoi qu’il en soit, je l’ai inclus ici car si vous avez conçu une application avec de très mauvais hotspots de données, les techniques d’optimisation plus simples mentionnées peuvent ne pas fonctionner pour vous.

**18. Configurez correctement votre serveur d’applications**

_Complexité: moyenne_

_Impact potentiel: moyen_

Lors de la configuration de MySQL Server, il est extrêmement important d’utiliser les paramètres appropriés du côté de votre serveur d’applications. Vous voulez vous assurer que vous utilisez des connexions persistantes et que vous ne vous reconnectez pas pour chaque petite transaction, surtout si vous utilisez TLS / SSL pour les connexions à la base de données. Si vous utilisez un pool de connexions, assurez-vous qu’il est correctement configuré, en particulier si vous n’utilisez pas ProxySQL ou Threadpool. Les recommandations spécifiques d’optimisation des performances varient en fonction du langage de programmation, du cadre ORM ou du pool de connexions utilisé – Vous pouvez faire un peu de documentation la dessus!

### Résumé

C’est toute une liste de recommandations, et en effet, il y a beaucoup de choses que vous pouvez faire pendant les heures de pointe pour mettre les choses sous contrôle. La bonne nouvelle est que vous n’aurez pas besoin de suivre chacune de ces suggestions pour obtenir des gains de performances et, en fin de compte, ravir vos clients avec des performances d’application fantastiques (ou du moins rendre votre équipe de développement heureuse lorsque la base de données n’est pas le problème). Regardez ces recommandations sous forme de menu - voyez ce qui est le plus facile à appliquer dans votre environnement et ce qui est susceptible de fournir le plus grand resultat, et utilisez-les pour guider vos actions!

Il s’agit d’une série de blogs en trois parties. La première partie peut être trouvée ici, et la deuxième partie peut être trouvée ici.

Page source : [Percona Blog](https://www.percona.com/blog/2020/04/07/18-things-you-can-do-to-remove-mysql-bottlenecks-caused-by-high-traffic-part-three/)
