+++
title = "Base de donnée open source : Gestion des mots de passe dans MySQL 8 - partie 1"
description = "Gestion de mots de passe livré avec MySQL 8 en particulier la politique de réutilisation des mots de passe et de la génération de mot de passe aléatoire"
author = "Francis"
date = 2021-11-28T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle26.jpg"
images = ["thumbnail/thumbnailarticle26.jpg"]
slug = "systemes-ameliores-de-gestion-des-mots-de-passe-dans-MySQL8"
+++


MySQL 8 est livré avec de nombreuses fonctionnalités intéressantes et j'ai récemment exploré ses systèmes de gestion de mots de passe. Je voulais monter une série de blogs à ce sujet, et c'est la première partie. Dans cet article, je vais expliquer en détail les sujets suivants.

- Politique de réutilisation des mots de passe
- Génération de mot de passe aléatoire

## Politique de réutilisation des mots de passe

MySQL a mis en place des restrictions sur la réutilisation des mots de passe. La restriction peut être établie de deux manières :

* Nombre de changements de mot de passe
* Temps écoulé

## Nombre de changements de mot de passe

À partir des documents MySQL :

*Si un compte est restreint sur la base du nombre de changements de mot de passe, un nouveau mot de passe ne peut pas être choisi parmi un nombre spécifié des mots de passe les plus récents.*

Pour tester cela, dans mon environnement local, j'ai créé l'utilisateur avec "nombre de changements de mot de passe = 2".

```
mysql> create user 'herc'@'localhost' identified by 'Percona@321' password history 2;
Query OK, 0 rows affected (0.02 sec)

mysql> select user, host, password_reuse_history from mysql.user where user='herc'\G
*************************** 1. row ***************************
                  user: herc
                  host: localhost
password_reuse_history: 2
1 row in set (0.00 sec)
```


Ici, « historique des mots de passe 2» définira le nombre de changements de mot de passe. MySQL suivra les changements de mot de passe sur la table « mysql.password\_history ».

```
mysql> select * from mysql.password_history;
+-----------+------+----------------------------+------------------------------------------------------------------------+
| Host      | User | Password_timestamp         | Password                                                               |
+-----------+------+----------------------------+------------------------------------------------------------------------+
| localhost | herc | 2021-09-20 15:44:42.295778 | $A$005$=R:q'M(Kh#D];c~SdCLyluq2UVHFobjWOFTwn2JYVFDyI042sl56B7DCPSK5 |
+-----------+------+----------------------------+------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

Maintenant, je vais changer le mot de passe du compte « herc@localhost ».

```
mysql> alter user 'herc'@'localhost' identified by 'MySQL@321';
Query OK, 0 rows affected (0.02 sec)

mysql> select * from mysql.password_history\G
*************************** 1. row ***************************
              Host: localhost
              User: herc
Password_timestamp: 2021-09-20 15:49:15.459018
CGeRQT31UUwtw194KOKGdNbgj3558VUB.dxcoS8r4IKpG8
*************************** 2. row ***************************
              Host: localhost
              User: herc
Password_timestamp: 2021-09-20 15:44:42.295778
          Password: $A$005$=R:q'M(Kh#D];c~SdCLyluq2UVHFobjWOFTwn2JYVFDyI042sl56B7DCPSK5
2 rows in set (0.00 sec)
```


Ça a marché. Après avoir changé le mot de passe, j'ai vérifié la table "mysql.password\_history". Maintenant, la table a la trace des deux derniers mots de passe.

Maintenant, je vais à nouveau changer le mot de passe du compte « herc@localhost ». Cette fois, je vais attribuer le même mot de passe qui a été attribué lors de la création de l'utilisateur « Percona@321 ».
```
mysql> alter user 'herc'@'localhost' identified by 'Percona@321';
ERROR 3638 (HY000): Cannot use these credentials for 'herc@localhost' because they contradict the password history policy
```


ça ne marche pas; Je n'arrive pas à réutiliser le premier mot de passe. Parce que selon ma politique de réutilisation, je ne peux pas réutiliser les deux derniers mots de passe et ils sont suivis dans la table « mysql.password\_policy ». Donc, dans mon cas, si je veux réutiliser mon premier mot de passe, il ne peut pas être dans cette liste.

J'ai donc attribué un mot de passe différent. Maintenant, mon premier mot de passe est supprimé de la liste des deux derniers mots de passe) et j'ai essayé d'attribuer le premier mot de passe.
```
mysql> alter user 'herc'@'localhost' identified by 'Herc@321';
Query OK, 0 rows affected (0.01 sec)

mysql> alter user 'herc'@'localhost' identified by 'Percona@321';
Query OK, 0 rows affected (0.02 sec)
```
Cela fonctionne maintenant. C'est ainsi que vous pouvez restreindre la réutilisation des mots de passe en fonction du nombre de changements de mot de passe.

Cela peut être implémenté globalement et au démarrage pour tous les utilisateurs à l'aide de la variable « password\_history ».

```
#vi my.cnf
[mysqld]
password_history=6

#set global
mysql> set global password_history=5;
Query OK, 0 rows affected (0.00 sec)
```

## Politique de réutilisation des mots de passe basée sur le temps écoulé

À partir du document MySQL :

*Si un compte est restreint en fonction du temps écoulé, un nouveau mot de passe ne peut pas être choisi parmi les mots de passe de l'historique qui sont plus récents qu'un nombre de jours spécifié.*

Pour tester cela dans mon environnement local, j'ai créé l'utilisateur « sri@localhost » avec un intervalle de réutilisation du mot de passe de cinq jours.

```
mysql> create user 'sri'@'localhost' identified by 'Percona@321' password reuse interval 5 day;
Query OK, 0 rows affected (0.01 sec)

mysql> select user, host, password_reuse_time from mysql.user where user='sri'\G
*************************** 1. row ***************************
               user: sri
               host: localhost
password_reuse_time: 5
1 row in set (0.00 sec)
```

Donc, cela signifie que pendant cinq jours, je ne peux pas réutiliser le mot de passe du compte « sri@localhost ».

```
mysql> select * from mysql.password_history where user='sri'\G
*************************** 1. row ***************************
              Host: localhost
              User: sri
Password_timestamp: 2021-09-20 16:09:27.918585
          Password: $A$005$+B   e3!C9&8m
                                         eFRG~IqRWX4b6PtzLA8I4VsdYvWU3qRs/nip/QRhXXR5phT6
1 row in set (0.00 sec)
```

Maintenant, je vais faire l'ALTER pour changer le mot de passe.

```
mysql> alter user 'sri'@'localhost' identified by 'Herc@321';
Query OK, 0 rows affected (0.02 sec)

mysql> select * from mysql.password_history where user='sri'\G
*************************** 1. row ***************************
              Host: localhost
              User: sri
Password_timestamp: 2021-09-20 16:17:51.840483
          Password: $A$005$~k7qp8.OP=^#e79qwtiYd7/cmCFLvHM7MHFbvfX2WlhXqzjmrN03gGZ4
*************************** 2. row ***************************
              Host: localhost
              User: sri
Password_timestamp: 2021-09-20 16:09:27.918585
          Password: $A$005$+B   e3!C9&8m
                                         eFRG~IqRWX4b6PtzLA8I4VsdYvWU3qRs/nip/QRhXXR5phT6
2 rows in set (0.00 sec)
```

Ça fonctionne. Mais, si je réutilise l'un de ces mots de passe, sur la base de la politique de réutilisation, cela ne sera pas autorisé pendant cinq jours. Laissez-moi essayer avec le premier mot de passe maintenant.

```
mysql> alter user 'sri'@'localhost' identified by 'Percona@321';
ERROR 3638 (HY000): Cannot use these credentials for 'sri@localhost' because they contradict the password history policy
```

Il donne l'erreur comme prévu. Cette restriction peut être implémentée globalement et au démarrage pour tous les utilisateurs à l'aide de la variable « password\_reuse\_interval ».

```
#vi my.cnf
[mysqld]
password_reuse_interval=365

#set global
mysql> set global password_reuse_interval=365;
Query OK, 0 rows affected (0.00 sec)
```

## Génération de mot de passe aléatoire

Depuis MySQL 8.0.18, MySQL a la capacité de créer des mots de passe aléatoires pour les comptes d'utilisateurs. Cela signifie que nous n'avons pas besoin d'attribuer les mots de passe et MySQL s'en chargera. Il a le support des déclarations suivantes :

- CRÉER UN UTILISATEUR
- MODIFIER L'UTILISATEUR
- FIXER LE MOT DE PASSE

Nous devons utiliser le « MOT DE PASSE ALÉATOIRE » au lieu de fournir le texte du mot de passe, et le mot de passe sera affiché à l'écran lors de la création.

Par exemple:
```
mysql> create user 'sakthi'@'localhost' identified by random password;
+--------+-----------+----------------------+
| user   | host      | generated password   |
+--------+-----------+----------------------+
| sakthi | localhost | .vZYy+<<BO7l1;vtIufH |
+--------+-----------+----------------------+
1 row in set (0.01 sec)

mysql> alter user 'sri'@'localhost' identified by random password;
+------+-----------+----------------------+
| user | host      | generated password   |
+------+-----------+----------------------+
| sri  | localhost | 5wb>2[]q*jbDsFvlN-i_ |
+------+-----------+----------------------+
1 row in set (0.02 sec)
```

Les hachages de mot de passe seront stockés dans la table « mysql.user ».
```
mysql> select user, authentication_string from mysql.user where user in ('sakthi','sri')\G
*************************** 1. row ***************************
                 user: sakthi
authentication_string: $A$005$L`PYcedj%3tz*J>ioBP1.Rsrj7H8wtelqijvV0CFnXVnWLNIc/RZL0C06l4oA
*************************** 2. row ***************************
                 user: sri
authentication_string: $A$005$/k?aO&ap.#b=
                                          ^zt[E|x9q3w9uHn1oEumXUgnqNMH8xWo4xd/s26hTPKs1AbC2
2 rows in set (0.00 sec)
```


*** Par défaut, la longueur du mot de passe est de 20 caractères sur la base de la variable "generated\_random\_password\_length". Nous pouvons définir la longueur du mot de passe en utilisant cette variable et la longueur autorisée est de 5 à 255. ***
```
mysql> select @@generated_random_password_length;
+------------------------------------+
| @@generated_random_password_length |
+------------------------------------+
|                                 20 |
+------------------------------------+
1 row in set (0.00 sec)
```

Les mots de passe aléatoires ne dérangeront pas la politique "validate\_password" si le composant est implémenté dans MySQL.

Espérons que ce blog vous sera utile pour en savoir plus sur la politique de réutilisation des mots de passe et les mots de passe aléatoires dans MySQL 8. Il y a quelques fonctionnalités supplémentaires à parcourir, qui seront couvertes dans la prochaine partie de la série de blogs. Restez à l'écoute!

**Percona Distribution for MySQL est la solution MySQL open source la plus complète, stable, évolutive et sécurisée disponible, offrant des environnements de base de données de niveau entreprise pour vos applications métier les plus critiques… et son utilisation est gratuite !**

Source : [Percona Blog](https://www.percona.com/blog/enhanced-password-management-systems-in-mysql-8-part-1/)
