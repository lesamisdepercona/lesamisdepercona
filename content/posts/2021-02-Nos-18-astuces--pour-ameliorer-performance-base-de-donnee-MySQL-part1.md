+++
title = "Améliorer la performance de  votre base de donnée MySQL (Partie 01)"
description = "Astuces et conseils pour améliorer la performance de votre base de donnée MySQL en heure de pointe"
author = "Francis"
date = 2021-04-10T11:43:01+04:00
tags = ['mysql']
Categories = ["Article de Percona"]
featured_image = "posts/article02/18astuces.png"
images = ["thumbnail/amisdepercona21-002.jpg"]
slug = "ameliorer-performance-de-base-de-donnee-MySQL-part01"
+++

### Les 18 choses que vous pouvez faire pour supprimer les goulots d’étranglement MySQL causés par un trafic élevé (Partie 01)

*Cet article est divisé en trois parties. La deuxième partie se trouve [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part2/), et la troisième partie se trouve [ici](/posts/article02-mysql-18astuces-heure-de-pointe-part3/)*

Il n’y avait aucune raison de le planifier, mais la charge sur votre système a augmenté de 100%, 300%, 500% et votre base de données MySQL doit la supporter. C’est une réalité à laquelle de nombreux systèmes en ligne doivent faire face ces jours-ci. Cette série se concentre sur la gestion de l’événement inattendu à fort trafic au fur et à mesure qu’il se produit.

Il y a aussi beaucoup de choses que vous pouvez faire de manière proactive, que nous avons abordées dans «[Préparez vos bases de données pour un trafic élevé du Black Friday](https://www.percona.com/blog/2019/11/11/prepare-your-databases-for-high-traffic-on-black-friday/)».

Tout d’abord, voyons quel impact un pic de trafic peut-il avoir sur la base de données ? Quels problèmes votre équipe d’ingénieurs d’application verra-t-elle probablement?

- Mauvais temps de réponse aux requêtes

- Taux d’erreur élevé (connexion à la base de données et exécution de requêtes)

- La base de données se plante (indisponible)

- Données incorrectes (obsolètes) en raison d’un retard de réplication ou de travaux par lots incapables de se terminer

L’objectif immédiat de la gestion de pic de trafic est d’éliminer ces problèmes le plus rapidement possibles, ce qui, pour la plupart des équipes, signifie se concentrer sur les «outils à portée de main» - des solutions qui peuvent être déployées en quelques heures ou quelques jours, et qui ne nécessitent pas une application massive ou des changements d’infrastructure.

La bonne nouvelle est que pour la majorité des applications, vous pouvez améliorer de façon significative la capacité de la base de données en effectuant quelques actions très simples:

#### 1. Ajustez la taille de votre instance cloud

_Complexité: faible_

_Impact potentiel: élevé_

Si vous exécutez dans le cloud (ou dans un environnement virtualisé), passer à une taille d’instance plus grande (également appelée «réglage par carte de crédit») est souvent la chose la plus simple que vous puissiez faire. C’est l’une des choses les plus coûteuses que vous puissiez faire, mais c’est une action à court terme que vous pouvez entreprendre pendant que vous procédez à la mise en œuvre d’autres actions d’optimisation des performances.

Remarque: les bases de données n’ont pas tendance à évoluer de manière linéaire, alors ne vous laissez pas convaincre par un faux sentiment de sécurité - si votre fournisseur de cloud dispose d’instances 10 fois plus grandes, ne vous attendez pas à ce qu’il gère un trafic 10x de plus. Cela peut être beaucoup moins en fonction du volume de travail.

#### 2. Déployez plus de MySQL Réplicas

_Complexité: moyenne_

_Impact potentiel: élevé_

Si votre volume de travail est gourmande en lecture, déployer davantage de réplicas peut être un excellent moyen d’améliorer les performances. Si vous ne savez pas quel type de volume de travail vous avez, notre article «[Est-ce un volume de travail intensif en lecture ou en écriture](https://www.percona.com/blog/2018/08/30/read-intensive-or-write-intensive-workload/)» pourrait vous aider à le comprendre.

Le déploiement de Replicas n’est cependant pas suffisant; vous devez vous assurer que votre application soit capable de leur fournir suffisamment assez de trafic. Dans certains cas, l’activation de cette fonctionnalité pourrait être facile au niveau de l’application. Pour d’autres, déployer ProxySQL et utiliser sa [fonctionnalité de fractionnement en lecture-écriture](https://proxysql.com/blog/configure-read-write-split/) pourrait être beaucoup plus adequat.

Dans de nombreux cas, vous pouvez même déplacer des applications complètes vers des Replicas : il faudrait alors recourir à des applications de reporting ou des applications utilisant la recherche en texte intégral MySQL.

Gardez à l’esprit que la réplication MySQL est asynchrone, ce qui signifie que les replicas auront des données propagées avec un retard (parfois important), par conséquent, acheminez uniquement les requêtes vers des replicas qui peuvent tolérer des données non à jour et assurez-vous de surveiller le retard de réplication et l’état général.

#### 3. Déployer ProxySQL pour la gestion des connexions et de mise en cache

_Complexité: moyenne_

_Impact potentiel: élevé_

ProxySQL est un excellent outil pour aider à gérer le trafic MySQL, en particulier lors des pics de trafic. ProxySQL peut aider par exemple grâce au [pool de connexions](https://www.percona.com/resources/webinars/utilizing-proxysql-connection-pooling-php) afin que l’application ne soit pas à court de connexions et ne surcharge pas MySQL en ayant trop de connexions simultanées.

Si vous exécutez [Percona Server pour MySQL](https://www.percona.com/software/mysql-database/percona-server) ou MariaDB, vous pouvez également activer [ThreadPool](https://www.percona.com/doc/percona-server/LATEST/performance/threadpool.html) qui peut permettre à une [instance MySQL de gérer directement plus de 100 000 connexions](https://www.percona.com/blog/2019/02/25/mysql-challenge-100k-connections/).

Une autre fonctionnalité de ProxySQL qui peut être encore plus utile lors des pics de trafic est [ProxySQL Query Cache](https://www.percona.com/blog/2018/02/07/proxysql-query-cache/), qui vous permet de mettre en cache les résultats des requêtes pendant un certain temps.

Lorsque vous trouvez des requêtes qui n’ont pas absolument besoin de fournir des résultats complètement à jour, dirigez-les vers des répliques MySQL où vous pouvez mettre en cache les mêmes requêtes pour un avantage supplémentaire.

**4. Désactiver les fonctionnalités des applications à charge lourde**

_Complexité: moyenne_

_Impact potentiel: moyen_

Les équipes de gestion et de développement détesteront souvent de telles idées, mais c’est un excellent outil. Toutes les fonctionnalités de l’application n’offrent pas la même valeur ou ne sont pas utilisées à la même fréquence, mais ce sont des fonctionnalités avancées, rarement appliquées, qui peuvent souvent être les plus coûteuses, car peu de temps a été passé à les optimiser. Les désactiver, au moins temporairement, pendant que vous traversez des pics de trafic ou que vous trouvez un moment pour les optimiser, est souvent une bonne chose à faire.

Il ne s’agit forcement pas des fonctionnalités destinées aux utilisateurs - pensez à identifier s’il existe des rapports internes dont vous pouvez vous passer?

#### 5. Vérifiez les goulots d’étranglement des ressources

_Complexité: faible_

_Impact potentiel: élevé_

Les bases de données de niveau matériel sont susceptibles d’être bloquées par une (ou plusieurs) ressources principales - CPU, mémoire, disque ou réseau. Si vous exécutez [Percona Monitoring and Management (PMM](https://www.percona.com/software/database-tools/percona-monitoring-and-management)), vous pouvez les voir dans la section Récapitulatif des nœuds de votre tableau de bord récapitulatif des instances MySQL.

 ![image 01](/posts/article02/p01_image01.png)

Si une ressource particulière est saturée, vous pouvez généralement obtenir de meilleures performances en augmentant cette ressource, bien que se concentrer sur la réduction de l’utilisation de cette ressource soit une autre alternative. Par exemple, un problème d’utilisation élevée du processeur est souvent mieux résolu en optimisant vos requêtes plutôt qu’en obtenant un processeur plus rapide.

#### 6. Obtenez plus de cœurs ou des cœurs plus rapides

_Complexité: faible_

_Impact potentiel: moyen_

Une chose importante à savoir à propos de MySQL est qu’il ne peut utiliser qu’un seul cœur de processeur pour effectuer la plupart du travail d’exécution d’une seule requête, ce qui signifie que l’obtention de plus de cœurs de processeur ne rend souvent pas vos requêtes lentes ou vos travaux par lots exécutant de nombreuses requêtes séquentiellement plus rapide. Si tel est votre problème, vous devez vous concentrer sur l’obtention de cœurs de processeur plus rapides ou vous devriez peut-être vous concentrer sur l’obtention de plus de cœurs.

Mais comment savoir quel type de charge de travail vous exécutez?

Jetez un œil à votre utilisation du processeur, à la saturation du processeur et à l’utilisation maximale du cœur dans le résumé des nœuds de Percona Monitoring and Management (ou graphique comparable dans votre système de surveillance préféré).

 ![image 02](/posts/article02/p01_image02.png)

Si l’utilisation du processeur est élevée (à l’exclusion de IO wait) et si la charge de processeur normalisée est supérieure ou égale à 2, votre système pourrait avoir besoin plus de cœurs de processeur disponibles pour exécuter un volume de travail.

Si, cependant, l’utilisation maximale du cœur du processeur est plus proche de 100% et que votre utilisation du processeur n’est pas élevée, vous devez vous concentrer sur des cœurs plus rapides.

Par exemple, si vous exécutez sur AWS, le [type d’instance](https://aws.amazon.com/ec2/instance-types/) Cloud C5 offre des performances de processeur supérieures par rapport au type d’instance M5 à usage général.

Une autre chose à surveiller en ce qui concerne le processeur, en particulier dans le cloud et les environnements virtualisés, est le «vol de processeur» - cela peut laisser votre instance MySQL avec beaucoup moins de ressources processeur que la fréquence du processeur et le nombre de cœurs indiqué. Lisez «[Choisissez judicieusement votre type d’instance EC2 sur AWS](https://www.percona.com/blog/2019/11/04/choose-your-ec2-instance-type-wisely-on-aws/)» pour plus de détails.


Page source : [Percona Blog](https://www.percona.com/blog/2020/04/03/18-things-you-can-do-to-remove-mysql-bottlenecks-caused-by-high-traffic-part-one/)
