+++
title = "PostgreSQL 14 – Performances, sécurité, convivialité et observabilité"
description = "PostgreSQL 14: index B-tree, libpq, compression LZ4 à TOAST, parallélisme SQL, et Commodité pour les développeurs d'applications"
author = "Francis"
date = 2021-11-19
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle25.jpg"
images = ["thumbnail/thumbnailarticle25.jpg"]
slug = "postgresql14-performances-securite-convivialite-et-observabilite"
+++

Aujourd'hui a vu le lancement de PostgreSQL14. Pour moi, le domaine important sur lequel se concentrer est ce que la communauté a fait pour améliorer les performances des charges de travail transactionnelles lourdes et une prise en charge supplémentaire des données distribuées.

La quantité de données que les entreprises doivent traiter continue de croître de façon exponentielle. Être capable de monter en charge et de prendre en charge ces charges de travail sur PostgreSQL 14 grâce à une meilleure gestion des connexions simultanées et des fonctionnalités supplémentaires pour le parallélisme des requêtes est très judicieux pour les performances, tandis que l'expansion de la réplication logique sera également utile.

## Améliorations des performances

### Améliorations côté serveur

#### Réduire la météorisation de l'index B-tree

Les index fréquemment mis à jour ont tendance à avoir des tuples morts qui provoquent un gonflement des index. En règle générale, ces tuples ne sont supprimés que lorsqu'un vide est exécuté. Entre les vides, au fur et à mesure que la page se remplit, une mise à jour ou une insertion provoquera une division de la page - quelque chose qui n'est pas réversible. Cette division se produirait même si les tuples morts à l'intérieur de la page existante auraient pu être supprimés, laissant de la place pour des tuples supplémentaires.

PostgreSQL 14 implémente l'amélioration où les tuples morts sont détectés et supprimés même entre les vides, permettant un nombre réduit de fractionnements de page, réduisant ainsi le gonflement de l'index.

#### Supprimer avec empressement les pages B-tree

En réduisant les frais généraux des B-tree, le système d'aspiration a été amélioré pour supprimer rapidement les pages supprimées. Auparavant, il fallait 2 cycles de vide pour le faire, le premier marquant la page comme supprimée et le second libérant réellement cet espace.

### Améliorations pour les cas d'utilisation spécialisés

#### Mode pipeline pour libpq

Les connexions à latence élevée avec des opérations d'écriture fréquentes peuvent ralentir les performances du client car libpq attend que chaque transaction soit réussie avant d'envoyer la suivante. Avec PostgreSQL 14, le « mode pipeline » a été introduit dans libpq, permettant au client d'envoyer plusieurs transactions en même temps, ce qui donne potentiellement un énorme coup de pouce aux performances. De plus, puisqu'il s'agit d'une fonctionnalité côté client, la libpq de PostgreSQL 14 peut même être utilisée avec les anciennes versions du serveur PostgreSQL.

#### Réplication des transactions en cours

La réplication logique a été étendue pour permettre le streaming des transactions en cours aux abonnés. Les transactions volumineuses étaient auparavant écrites sur le disque jusqu'à ce que la transaction soit terminée avant d'être répliquées vers l'abonné. En permettant la diffusion en continu des transactions en cours, les utilisateurs bénéficient d'importants avantages en termes de performances et d'une plus grande confiance dans leurs charges de travail distribuées.

#### Ajouter la compression LZ4 à TOAST

PostgreSQL 14 ajoute la prise en charge de la compression LZ4 pour TOAST, un système utilisé pour stocker efficacement des données volumineuses. LZ4 est un algorithme de compression sans perte qui se concentre sur la vitesse de compression et de décompression. LZ4 peut être configuré au niveau de la colonne ainsi qu'au niveau du système. Auparavant, la seule option était la compression pglz - qui est rapide mais a un faible taux de compression.

### Améliorations pour les charges de travail distribuées

Le wrapper de données étrangères PostgreSQL - postgres\_fdw - a contribué à alléger le fardeau de la gestion des charges de travail distribuées. Avec PostgreSQL 14, il y a 2 améliorations majeures dans le FDW pour améliorer les performances de ces transactions : le parallélisme des requêtes peut désormais être exploité pour [les analyses de tables parallèles sur les tables étrangères](https://www.percona.com/blog/2021/06/02/postgres_fdw-enhancement-in-postgresql-14/) , et l' [insertion en bloc de données est désormais autorisée sur les tables étrangères](https://www.percona.com/blog/2021/05/27/new-features-in-postgresql-14-bulk-inserts-for-foreign-data-wrappers/) .  

Ces deux améliorations renforcent la capacité croissante de PostgreSQL à évoluer horizontalement et à gérer nativement les bases de données distribuées.

#### Amélioration du parallélisme SQL

La prise en charge par PostgreSQL du parallélisme des requêtes permet au système d'engager plusieurs cœurs de processeur dans plusieurs threads pour exécuter des requêtes en parallèle, améliorant ainsi considérablement les performances. PostgreSQL 14 apporte plus de raffinement à ce système en ajoutant la prise en charge de RETURN QUERY et REFRESH MATERIALIZED VIEW pour exécuter des requêtes en parallèle. Des améliorations ont également été apportées aux performances des analyses séquentielles parallèles et des jointures en boucle imbriquées.

## Sécurité

### SCRAM comme authentification par défaut

L'authentification SCRAM-SHA-256 a été introduite dans PostgreSQL 10 et est maintenant devenue l'authentification par défaut dans PostgreSQL 14. L'authentification par défaut MD5 précédente avait quelques faiblesses qui ont été exploitées dans le passé. SCRAM est beaucoup plus puissant et permet une conformité réglementaire plus facile pour la sécurité des données.

### Rôles prédéfinis

Deux rôles prédéfinis ont été ajoutés dans PostgreSQL 14 – pg\_read\_all\_data et pg\_write\_all\_data. Le premier permet d'accorder à un utilisateur un accès en lecture seule à toutes les tables, vues et schémas de la base de données. Ce rôle aura un accès en lecture par défaut à toutes les nouvelles tables créées. Ce dernier permet de créer des privilèges de style super-utilisateur, ce qui signifie qu'il faut être très prudent lors de son utilisation !

## Commodité pour les développeurs d'applications

### Accéder à JSON à l'aide d'indices

Du point de vue du développement d'applications, j'ai toujours trouvé la prise en charge de JSON dans PostgreSQL très intéressante. PostgreSQL prend en charge ce formulaire de données non structuré depuis la version 9.2, mais il a une syntaxe unique pour récupérer les données. Dans la version 14, la prise en charge des indices a été ajoutée, ce qui permet aux développeurs de récupérer plus facilement les données JSON à l'aide d'une syntaxe communément reconnue. Pour plus d'informations, Matt Yonkovit a récemment rédigé le post [Utilisation de la nouvelle syntaxe JSON dans PostgreSQL 14 que](https://www.percona.com/blog/using-new-json-syntax-in-postgresql-14-update/) vous devriez consulter.

### Types multigamme

PostgreSQL a des types Range depuis la version 9.2. PostgreSQL 14 introduit désormais la prise en charge « multiplage » qui permet des plages non contiguës, aidant les développeurs à écrire des requêtes plus simples pour des séquences complexes. Un exemple simple d'utilisation pratique serait de spécifier les plages horaires pendant lesquelles une salle de réunion est réservée au cours de la journée. Voir le post de Matt Yonkovit sur [les types de données multiplages dans PostgreSQL 14](https://www.percona.com/blog/using-the-multirange-data-type-in-postgresql-14/) pour plus d'informations.  

### Paramètres OUT dans les procédures stockées

Des procédures stockées ont été ajoutées dans PostgreSQL 11, donnant aux développeurs un contrôle transactionnel dans un bloc de code. PostgreSQL 14 implémente le paramètre OUT, permettant aux développeurs de renvoyer des données en utilisant plusieurs paramètres dans leurs procédures stockées. Cette fonctionnalité sera familière aux développeurs Oracle et bienvenue pour les personnes essayant de migrer d'Oracle vers PostgreSQL.

## Observabilité

L'observabilité est l'un des plus grands mots à la mode de 2021, car les développeurs veulent plus d'informations sur les performances de leurs applications au fil du temps. PostgreSQL 14 ajoute plus de fonctionnalités pour aider à la surveillance, l'un des changements les plus importants étant le déplacement du système de hachage des requêtes de pg\_stat\_statement vers la base de données principale. Cela permet de surveiller une requête à l'aide d'un seul ID sur plusieurs systèmes PostgreSQL et fonctions de journalisation. Cette version ajoute également de nouvelles fonctionnalités pour suivre la progression des statistiques de COPY, d'activité WAL et de slots de réplication.

Ce qui précède est un petit sous-ensemble des plus de 200 fonctionnalités et améliorations qui composent la version PostgreSQL 14. Dans l'ensemble, cette mise à jour de version aidera PostgreSQL à continuer de dominer le secteur des bases de données open source.

Alors que de plus en plus d'entreprises envisagent de migrer loin d'Oracle ou de mettre en œuvre de nouvelles bases de données avec leurs applications, PostgreSQL est souvent la meilleure option pour ceux qui souhaitent s'exécuter sur des bases de données open source.

**Source : [Blog Percona](https://www.percona.com/blog/postgresql-14-performance-security-usability-and-observability/)**

