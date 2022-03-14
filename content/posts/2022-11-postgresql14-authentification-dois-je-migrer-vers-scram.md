+++
title = "PostgreSQL 14 et Authentification SCRAM. Dois-je migrer vers SCRAM?"
description = "L'authentification SCRAM n'est pas quelque chose de nouveau dans PostgreSQL. Nous traitons dans cet article les questions fréquemment posées pour une prise de conscience rapide"
author = "Francis"
date = 2022-03-10
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article11.jpg"
images = ["thumbnail2022/article11.jpg"]
slug = "postgresql14-authentification-dois-je-migrer-vers-scram"
+++

![thumbnail](/thumbnail2022/article11.jpg)


Récemment, quelques utilisateurs de PostgreSQL ont signalé avoir eu des échecs de connexion après avoir passés à PostgreSQL 14.

“ *Pourquoi est-ce que j'obtiens l'erreur **FATAL : l'authentification du mot de passe a échoué pour un utilisateur** dans le nouveau serveur ?* » est devenue l'une des questions les plus intrigantes.

Au moins dans un cas, c'était un peu surprenant que le message d'application soit le suivant :

**FATAL : La connexion à la base de données a échoué : la connexion au serveur sur «localhost» (::1 ), le port 5432 a échoué : fe_sendauth : aucun mot de passe fourni**

La raison de ces erreurs est que les valeurs par défaut pour le chiffrement du mot de passe sont modifiées dans les nouvelles versions de PostgreSQL vers l'authentification SCRAM. Même si le dernier ne semble rien directement lié à SCRAM, oh oui, un script de post-installation a échoué qui cherchait "md5".

L'authentification SCRAM n'est pas quelque chose de nouveau dans PostgreSQL . Il était là depuis PostgreSQL 10 mais n'a jamais affecté DBA en général car cela n'a jamais été la valeur par défaut. Il s'agissait d'une fonctionnalité opt-in en modifiant explicitement les paramètres par défaut. Ceux qui font un opt-in comprennent généralement et le font intentionnellement, et cela n'a jamais causé de problème. La communauté PostgreSQL était réticente à en faire une méthode de choix pendant des années car de nombreuses bibliothèques client/application n'étaient pas prêtes pour l'authentification SCRAM.

Mais cela change dans PostgreSQL 14. Avec PostgreSQL 9.6 qui n'est plus pris en charge, le paysage change. Maintenant, nous nous attendons à ce que toutes les anciennes bibliothèques clientes soient mises à niveau et l'authentification SCRAM devient la principale méthode d'authentification par mot de passe. Mais, ceux qui l'ignorent complètement vont être accueillis par une surprise un jour ou l'autre. Le but de cet article est de créer une prise de conscience rapide pour ceux qui ne le sont pas encore et de répondre à certaines des questions fréquemment posées.

## Qu'est-ce que l'authentification SCRAM ?

En termes simples, la base de **données** **le client et le serveur se prouvent et se convainquent qu'ils connaissent le mot de passe sans échanger le mot de passe ou le hashage du mot de passe** . Oui, c'est possible en faisant un défi et des réponses salés, SCRAM-SHA-256, comme spécifié par [RFC 7677 ](https://datatracker.ietf.org/doc/html/rfc7677). Cette façon de stocker, de communiquer et de vérifier les mots de passe rend très difficile la rupture d'un mot de passe.

Cette méthode est plus résistante à :

- dictionnaire attaques
- Rejouer attaques
- Hashes volés

Dans l'ensemble, il devient très difficile de casser une authentification par mot de passe.

## Qu'est-ce qui a changé avec le temps ?

### Liaison de canal

L'authentification n'est qu'une partie de la communication sécurisée. Après l'authentification, un serveur malveillant au milieu peut potentiellement prendre le relais et tromper la connexion client. PostgreSQL 11 a introduit SCRAM-SHA-256-PLUS qui prend en charge la liaison de canal. C'est pour s'assurer qu'il n'y a pas de serveur malveillant agissant comme un vrai serveur OU faisant une attaque de l'homme du milieu.

À partir de PostgreSQL 13, un client peut demander et même insister sur la liaison de canal.

Par exemple :

```
psql -U postgres -h c76pri channel_binding=prefer
or
psql -U postgres -h c76pri channel_binding=require
```

La liaison de canal fonctionne sur SSL/TLS, la configuration SSL/TLS est donc obligatoire pour que la liaison de canal fonctionne.

### Définition du mot de passe de cryptage

Le **md5** était la seule option disponible pour le chiffrement de mot de passe avant PostgreSQL 10, donc PostgreSQL permet aux paramètres d'indiquer que "le chiffrement de mot de passe est requis" qui est par défaut à md5.

```
–Upto PG 13
postgres=# set password_encryption TO ON;
SET
```

Pour la même raison, la déclaration ci-dessus était effectivement la même que :

```
postgres=# set password_encryption TO MD5;
SET
```

Nous pourrions même utiliser "vrai", "1", "oui" au lieu de "on" comme valeur équivalente.

Mais maintenant, nous avons plusieurs méthodes de cryptage et "ON" ne transmet pas vraiment ce que nous voulons vraiment. Ainsi, à partir de PostgreSQL 14, le système s'attend à ce que nous spécifions la méthode de chiffrement.

```
postgres=# set password_encryption TO 'scram-sha-256';
SET
```


```
postgres=# set password_encryption TO 'md5';
SET
```

Toute tentative d'utiliser "on"/"true", "yes" sera rejetée avec une erreur.

```
–-From PG 14
postgres=# set password_encryption TO 'on';
ERROR:  invalid value for parameter "password_encryption": "on"
HINT:  Available values: md5, scram-sha-256.
```

Veuillez donc vérifier vos scripts et vous assurer qu'ils n'ont pas l'ancienne méthode "d'activation" du chiffrement.

## Quelque Question Fréquemment posées

1. **Ma sauvegarde et ma restauration logiques sont-elles affectées ?** 
   Sauvegarde et restauration logique de PostgreSQL globals ( pg_dumpall ) n'affectera pas l'authentification SCRAM, le même mot de passe devrait fonctionner après la restauration. En fait, il sera intéressant de rappeler que l'authentification SCRAM est plus résistante aux changements. Par exemple, si nous renommons un USER, l'ancien mot de passe MD5 ne fonctionnera plus, car la façon dont PostgreSQL génère le MD5 utilise également le nom d'utilisateur. 

   `postgres =# ALTER USER jobin1 RENOMMER EN jobin ; 
   AVIS : Le mot de passe MD5 a été effacé en raison du rôle renameALTER ROLE
   `

   Comme le NOTICE l'indique, le hashage du mot de passe dans pg_authid sera effacé car l'ancien n'est plus valide. Mais ce ne sera pas le cas avec l'authentification SCRAM, car nous pouvons renommer les utilisateurs sans affecter le mot de passe. 

   `postgres =# ALTER USER jobin RENOMMER EN jobin1 ; 
   MODIFIER LE RÔLE
   `

1. **La méthode de cryptage existante/ancienne (md5) était une grande vulnérabilité. Y avait-il un gros risque ?** 
   Cette inquiétude vient principalement du nom "MD5" qui est bien trop idiot pour le matériel moderne. La façon dont PostgreSQL utilise md5 est différente, ce n'est pas seulement le hashage du mot de passe, mais il considère également le nom d'utilisateur. De plus, il est communiqué sur le réseau après avoir préparé un hashage avec un salt aléatoire fourni par le serveur. Effectivement ce qui est communiqué sera différent du hashage du mot de passe, il n'est donc pas trop vulnérable. Mais sujet aux attaques par dictionnaire et aux fuites de problèmes de hashage du mot de passe du nom d'utilisateur.

1. **Les nouvelles annonces d'authentification scram sont-elles complexes à authentifier ? Est-ce que ma demande de connexion va prendre plus de temps ?** 
   Le protocole filaire SCRAM est très efficace et n'est pas connu pour provoquer de dégradation du temps de connexion. De plus, par rapport aux autres surcoûts de la gestion des connexions côté serveur, le surcoût créé par SCRAM devient très négligeable


1. **Est-il obligatoire d'utiliser l'authentification SCRAM depuis PostgreSQL 14 et de forcer tous mes comptes utilisateurs à y basculer ?** 
   Absolument pas, seules les valeurs par défaut sont modifiées. L'ancienne méthode md5 est toujours une méthode valide qui fonctionne très bien, et si l'accès à l'environnement PostgreSQL est restreint par des règles de pare-feu/ hba , il y a déjà moins de risques à utiliser md5.

1. **Pourquoi est-ce que j'obtiens l'erreur « : FATAL : l'authentification du mot de passe a échoué pour l'utilisateur » lorsque je suis passé à PostgreSQL 14 ?** 
   La raison la plus probable est les entrées pg_hba.conf . Si nous spécifions "md5" comme méthode d'authentification, PostgreSQL autorisera également l'authentification SCRAM. Mais l'inverse ne fonctionnera pas. Lorsque vous avez créé l'environnement PostgreSQL 14, il peut très probablement avoir "scram-sha-256" comme méthode d'authentification. Dans certains des packages PostgreSQL, le script d'installation le fait automatiquement pour vous. Dans le cas où l'authentification fonctionne à partir des outils client PostgreSQL et non à partir de l'application, veuillez [vérifier la version du pilote](https://wiki.postgresql.org/wiki/List_of_drivers) et vérifier la portée de la mise à niveau

1. **Pourquoi ai-je d'autres types d'erreurs d'authentification ?** 
   Les raisons les plus probables sont les scripts de post-installation. Il est courant dans de nombreuses organisations d'utiliser des outils DevOps ( Ansible /Chef) ou même des scripts shell pour effectuer les personnalisations post-installation. Beaucoup d'entre eux feront une gamme de choses qui impliquent des étapes telles que set password_encryption TO ON; ou même la modification de pg_hba.conf en utilisant sed , qui devrait échouer s'il essaie de modifier une entrée qui n'y est plus.

## Pourquoi devrais-je m'en soucier et que faire

Tout ce qui commence par les scripts d'automatisation/déploiement, les outils, les connexions d'application et les poolers de connexion peut potentiellement se casser. L'un des principaux arguments pour retarder ce changement jusqu'à PostgreSQL 14 est que la version la plus ancienne prise en charge (9.6) ne sera bientôt plus prise en charge. C'est donc le bon moment pour inspecter vos environnements pour voir si l'un de ces environnements possède d'anciennes bibliothèques PostgreSQL (9.6 ou antérieures) et avoir un plan pour la mise à niveau, car les bibliothèques PostgreSQL de l'ancienne version ne peuvent pas gérer les négociations SCRAM.

En résumé, avoir un bon plan pour migrer aidera, même si ce n'est pas urgent.

1. Inspectez les environnements et les pilotes d'application pour voir si l'un d'entre eux utilise encore d'anciennes versions des bibliothèques client PostgreSQL et mettez-les à niveau si nécessaire. 
   Veuillez vous référer à : [https://wiki.postgresql.org/wiki/List_of_drivers ](https://wiki.postgresql.org/wiki/List_of_drivers)
   Encourager / Piloter la mise à niveau des bibliothèques clientes avec un calendrier
1. Si l'environnement existant utilise md5, encouragez les utilisateurs à passer à l'authentification 
   SCRAM . N'oubliez pas que la méthode d'authentification mentionnée comme "md5" dans pg_hba.conf continuera à fonctionner pour l'authentification SCRAM et MD5 dans PostgreSQL 14 également.
1. Profitez de chaque occasion pour tester et migrer l'automatisation, les pooleurs de connexions et d'autres infrastructures vers l'authentification SCRAM.

En modifiant l'authentification par défaut, la communauté PostgreSQL montre une direction claire pour l'avenir.

**Percona Distribution for PostgreSQL fournit les meilleurs composants entreprise de la communauté open source, dans une distribution unique, conçue et testée pour fonctionner ensemble.**

[Téléchargez Percona Distribution pour PostgreSQL](https://www.percona.com/software/postgresql-distribution)

Source : [Percona](https://www.percona.com/blog/postgresql-14-and-recent-scram-authentication-changes-should-i-migrate-to-scram/)
