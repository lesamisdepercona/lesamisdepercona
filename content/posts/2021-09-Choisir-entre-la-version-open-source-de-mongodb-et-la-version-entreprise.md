+++
title = "Pourquoi payer pour la version entreprise de MongoDB quand la version Open Source est-elle disponible?"
description = "Les raisons de choisir Percona Distribution pour MongoDB au lieu de la version entreprise de MongoDB"
author = "Francis"
date = 2021-07-29T11:43:01+04:00
tags = ['MongoDB']
Categories = ["Article de Percona"]
images = ["posts/article09/img-article09.png"]
featured_image = "posts/article09/img-article09.png"
slug = "mongodb-pourquoi-payer-pour-la-version-entreprise-quand-opensource-est-disponible"
+++


## Gros plan sur la sécurité de votre base de données MongoDB

Lorsque Percona a publié [ce blog](https://www.percona.com/blog/2016/06/17/mongodb-security-pay-enterprise-open-source-covered/) pour la première fois en 2016, il s’agissait d’une comparaison des fonctionnalités importantes d’un déploiement MongoDB au niveau d’entreprise. Pourtant, comparer MongoDB Enterprise à [Percona Server for MongoDB](https://www.percona.com/software/mongodb/percona-server-for-mongodb) (PSMDB) est devenu un sujet de discussion tellement courant avec les clients (et futurs clients) de Percona à tel point que les informations de ce blog sont désormais ancrées dans ma mémoire.

MongoDB et Percona ont tous deux parcouru un long chemin depuis que ce blog a été écrit. Plusieurs des recommandations sont toujours valables quatre ans plus tard, mais certaines choses ont évolué des deux côtés. Le but de cet article est de développer et d’actualiser les comparaisons que nous avons faites il y a quatre ans.

Une chose qui n’a pas changé est que la sécurité est toujours une priorité absolue pour toutes les organisations en ce qui concerne leurs environnements de base de données. Au cours de la dernière année seulement, il y a eu plusieurs fuites de données très médiatisées en raison de [bases de données MongoDB mal configurées](https://www.teiss.co.uk/uk-shoppers-personal-data-exposed/) .

Il est important de noter que ces fuites très médiatisées sont souvent dues à des problèmes de configuration ou à des logiciels obsolètes et non pris en charge. Si vous utilisez des déploiements à jour, bien entretenus et correctement mis en œuvre de MongoDB , il n’y a aucune raison de craindre.

Dans cet article, je vais expliquer les options de sécurité qui existent dans le logiciel open source MongoDB. Ces options vous permettent de déployer au niveau d’entreprise un déploiement MongoDB sécurisé sans vous soucier des frais de licence. Ceci est important car cela permet aux organisations la flexibilité de déployer des modèles cohérents sur l’ensemble de leur infrastructure. Trop souvent, nous voyons des clients exécuter différentes versions de logiciels dans dev/test/ qa en raison de restrictions de licence. Avec les logiciels open source, ce n’est pas un problème.

## L’essor de la base de données en tant que service

Une chose qui a changé depuis la rédaction du blog d’origine est l’explosion de la croissance du secteur de la gestion des bases de données en tant que service ( DBaaS ). DBaaS est une forme plus spécifique de Platform as a Service ( PaaS ).

MongoDB Inc a misé sur l’espace DBaaS. Cela correspond bien à sa mission continue de responsabiliser les développeurs en surmontant le fardeau de la gestion des bases de données.

En 2016, [MongoDB Atlas entra dans sa phase de commercialisation (GA) dans l’offre de MongoDB DBaaS](https://www.mongodb.com/blog/post/announcing-mongodb-atlas-database-as-a-service-for-mongodb). En 2017, MongoDB est devenu [public sur Nasdaq](https://www.cnbc.com/2017/10/18/mongodb-prices-its-ipo-worth-1-point-2b.html) . En 2018, MongoDB a [acquis un concurrent sur la communauté - MLab](https://venturebeat.com/2018/10/10/mongodb-to-acquire-cloud-database-provider-mlab-for-68-million/) . Le changement récent le plus important a été son [passage à une nouvelle licence, SSPL](https://www.percona.com/blog/2018/10/18/percona-statement-on-mongodb-community-server-license-change/) . Cela [impacte les entreprises proposant des solutions « as a service » de MongoDB](https://www.mongodb.com/licensing/server-side-public-license/faq)

[Selon le PDG de MongoDB](https://www.techrepublic.com/article/mongodb-ceo-tells-hard-truths-about-commercial-open-source/) , la stratégie à long terme de MongoDB implique des abonnements logiciels sous licence Enterprise, et DBaaS composants en feront partie.

## S’assurer que votre base de données est sécurisée

Donc, si le verrouillage et l’utilisation d’une technologie véritablement open source sont une priorité pour votre entreprise, quelles sont vos options? C’est un domaine que je développerai dans un blog de Percona axé sur la sécurité où j’expliquerai les options et les chemins qui existent pour les utilisateurs qui ne souhaitent pas utiliser la boîte à outils sous licence d’entreprise de MongoDB .

Il y a de nombreux aspects importants à considérer avant d’exécuter MongoDB au niveau d’une entreprise :

- Sécurité
- Surveillance, diagnostics, analyses et alertes
- Automatisation de la maintenance, du déploiement et des sauvegardes

Dans ce blog, nous examinons spécifiquement la [sécurité](https://www.percona.com/blog/2019/07/12/mongodb-security-vs-five-bad-guys/) . Pour une base de données cela signifie :

- **Authentification :** L’authentification LDAP centralisé selon votre annuaire d’entreprise (pour PCI)
- **Autorisation :** contrôles d’accès basés sur les rôles
- **Chiffrement :** Divisé entre «at-rest» et «in-transit» dans le cadre des exigences PCI régulières
- **Gouvernance :** Validation du schéma et même vérification des données sensibles telles qu’un SSN ou des données de naissance
- **Audit :** La possibilité de voir qui a fait quoi dans la base de données (également requis pour PCI)
- **Rédaction du journal :** Filtrage des informations écrites dans les fichiers journaux

## Authentification

MongoDB a des utilisateurs intégrés. Cependant, il manque des choses, comme la complexité des mots de passe, la rotation basée sur l’âge, la centralisation et l’identification des rôles d’utilisateur par rapport aux fonctions de service. Cela a été [soumis en tant que demande de fonctionnalité](https://jira.mongodb.org/browse/SERVER-7363) et se trouve en attente jusqu’à nouvel ordre.

Ces contrôles d’authentification des utilisateurs sont essentiels pour passer PCI. PCI exige que les gens n’utilisent pas de mots de passe anciens ou faciles à casser et que l’accès des utilisateurs soit révoqué lorsqu’il y a un changement de statut (comme quelqu’un quittant un service ou l’entreprise). La raison pour laquelle cette logique n’a pas été ajoutée au niveau de la base de données est que, pour la plupart, il existe une autre réponse qui s’intègre facilement à MongoDB. LDAP est un projet open source à part entière. De nombreux connecteurs permettent l’utilisation de systèmes Windows Active Directory (AD) pour communiquer avec LDAP.

La version communautaire de MongoDB n’a pas cette fonctionnalité :

- [L’intégration LDAP est une fonctionnalité réservée aux entreprises](https://docs.mongodb.com/manual/core/security-ldap-external/) pour MongoDB
- [L’intégration LDAP est une fonctionnalité gratuite](https://www.percona.com/doc/percona-server-for-mongodb/LATEST/authentication.html%23ext-auth#ext-auth) open source dans Percona Server pour MongoDB

Une dernière remarque concernant l’authentification est l’authentification Kerberos. Percona Server pour MongoDB et MongoDB Enterprise ont tous deux implémenté cette fonctionnalité. Il est disponible dans PSMDB 4.2.6 et les versions plus récentes.

## Autorisation

L’autorisation basée sur les rôles est au cœur de MongoDB Community, MongoDB Enterprise et Percona Server for MongoDB. Dans chaque version depuis Mongo 2.6, vous pouvez utiliser des rôles intégrés, ou même définir les vôtres en fonction des actions que quelqu’un pourrait être capable de faire - en exposant uniquement ce que vous voulez que les utilisateurs puissent faire. Il s’agit d’une fonctionnalité de base de MongoDB qui est présente partout et dans chaque version, quel que soit le fournisseur.

MongoDB Enterprise et Percona Server for MongoDB ont intégré l’autorisation LDAP dans leurs solutions de base de données. Cette fonctionnalité mappe les noms distinctifs de groupe LDAP aux rôles de groupe MongoDB et permet une configuration facile des rôles parmi les groupes d’utilisateurs. Ceci est disponible dans les versions 4.2.5 ou supérieures pour PSMDB.

## Chiffrement

Le chiffrement au niveau de la couche de base de données est devenu de plus en plus populaire et il est obligatoire dans les grandes organisations en raison des récentes compromissions et violations de données. Bien que la plupart des violations très médiatisées soient le résultat de déploiements inappropriés ou extrêmement obsolètes, des mesures de sécurité supplémentaires telles que le cryptage au repos (at rest) et en vol (in flight) sont toujours préférables. Le cryptage est souvent une exigence pour les bases de données fonctionnant sous les normes PCI ou HIPAA.

Le premier aspect du chiffrement est le chiffrement en vol ou par fil (over-the-wire). Cela garantit que le trafic réseau n’est lisible que par le client prévu. Heureusement, il s’agit d’une autre fonctionnalité essentielle du code MongoDB et elle est présente dans chaque version, quel que soit le fournisseur. Percona recommande de standardiser OpenSSL car il est susceptible d’être compatible avec les bases de données open source et les plates-formes de système d’exploitation open source.

Le deuxième aspect du chiffrement est le chiffrement au repos. Cela garantit qu’une fois stockées, les données ne peuvent être lues que par le client prévu. Cette option est disponible pour le moteur de stockage WiredTiger. L’implémentation de MongoDB nécessite une licence d’entreprise MongoDB. Heureusement, Percona vous a couvert avec une implémentation open source de [chiffrement WiredTiger  dans Percona Server pour MongoDB](https://www.percona.com/doc/percona-server-for-mongodb/LATEST/data_at_rest_encryption.html) . La mise en œuvre de Percona intègre Hashicorp Vault pour la gestion des clés et chiffre également les fichiers de restauration, qui pourraient contenir des données sensibles.

## Gouvernance

Le concept de [validation schéma](https://docs.mongodb.com/manual/core/document-validation/) de MongoDB garantit que les normes de stockage des données sont définies et rigoureusement appliquées si nécessaire. Toutes les versions de MongoDB ont des méthodes pour appliquer les normes de format pour les données stockées dans la base de données. Les administrateurs de base de données peuvent rechercher des noms de champ en fonction de certains critères basés sur des chaînes d’expression régulière pour rechercher des numéros SSN ou CC. Le langage JSON natif de MongoDB permet de créer facilement des scripts d’options personnalisées, quelle que soit la version de MongoDB, car il s’agit d’une partie essentielle de la base de données MongoDB. En appliquant des schémas rigides, les organisations peuvent garantir que les développeurs n’insèrent pas de données sensibles dans des documents qui ne leur appartiennent pas.

## Audit

Le concept d’audit de l’activité des utilisateurs n’est étranger à aucun responsable de la sécurité. Il s’agit d’un aspect essentiel de la conformité pour toutes les organisations. C’est un autre domaine où Percona peut offrir une alternative open source à la licence Entreprise de MongoDB :

- Du côté de MongoDB, il s’agit d’une fonctionnalité réservée aux entreprises qui nécessite une licence.

- Du côté de Percona, cela est inclus dans Percona Server for MongoDB.

Ils ont des fonctionnalités très similaires et permettent aux utilisateurs de filtrer la sortie vers des utilisateurs, des bases de données, des collections ou des sources particulières. Cela permet une granularité de ce qui est audité et permet la révision des journaux en cas d’incident de sécurité.

## Rédaction du journal (Log Redaction)

Une autre considération importante en matière de sécurité de MongoDB est la rédaction des journaux. Souvent, il est nécessaire de supprimer les messages d’un événement de journal avant de se connecter. Cela empêchera les journaux de contenir des informations potentiellement sensibles telles que les informations personnelles dans le journal de diagnostic. Les métadonnées telles que les codes d’erreur ou d’opération, les numéros de ligne et les noms de fichiers source restent visibles dans les journaux.

Il s’agit d’une [fonctionnalité réservée aux entreprises du côté de MongoDB, Inc.](https://docs.mongodb.com/manual/reference/parameters/#param.redactClientLogData) , [Percona Server pour MongoDB offre une fonctionnalité de rédaction de journaux](https://www.percona.com/doc/percona-server-for-mongodb/3.4/log-redaction.html) dans sa version open source. Il est important de noter que même si cette fonctionnalité augmente la sécurité, elle peut rendre le dépannage plus difficile.

## Derniers Points

Le but de ce blog était d’informer les personnes intéressées par MongoDB des options alternatives disponibles et de leur fournir des informations pour leur permettre de sélectionner le logiciel de base de données MongoDB le plus approprié pour leur entreprise.

La sécurité est une raison importante pour laquelle de nombreuses entreprises optent pour la version Entreprise de MongoDB . Cependant, Percona Server pour MongoDB peut offrir toutes ces fonctionnalités avec un modèle sans licence. Cela signifie la fin des inquiétudes pour l’achat de licences pour la production, dev, test, qa, sandbox, etc. Vous pouvez assurer un déploiement cohérent dans tous les environnements en utilisant le logiciel, sans licence, open source, tout en veillant à ce que les normes de sécurité requises par votre organisation soient assurées.

Le tableau ci-dessous résume ce que nous avons évoqué :

 ![image 01](/posts/article09/img-article09-01.png)

L’engagement continu de Percona envers MongoDB

En plus de notre logiciel libre open source MongoDB, [Percona Server pour MongoDB](https://www.percona.com/software/mongodb/percona-server-for-mongodb), [Percona Backup pour MongoDB](https://www.percona.com/software/mongodb/percona-backup-for-mongodb), nous avons également lancé [Percona Distribution pour MongoDB](https://www.percona.com/software/mongodb).

Percona Distribution pour MongoDB est une solution MongoDB d’entreprise open source. Il vous aide à garantir la disponibilité des données pour vos applications tout en améliorant la sécurité et en simplifiant le développement de nouvelles applications dans les environnements de cloud public, privé et hybride les plus exigeants.

Nous proposons également des services spécialisés d’assistance, de conseil et de gestion pour les bases de données MongoDB :

- [Percona Support for MongoDB](https://www.percona.com/services/support/mongodb-support) propose un plan complet, réactif et rentable pour réussir le déploiement MongoDB de votre organisation.
- [MongoDB Consulting Services](https://www.percona.com/services/consulting) peuvent vous aider à gérer votre migration vers MongoDB et garantir une performance optimale et continue.
- [MongoDB Managed Services](https://www.percona.com/services/managed-services) fournissent des prestations de gestion de base de données pour votre entreprise.

Source : [Percona](https://www.percona.com/blog/2020/05/28/mongodb-why-pay-for-enterprise-when-open-source-has-you-covered/)
