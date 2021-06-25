+++
title = "MySQL - Quelques problèmes de conception"
description = "Quelques problèmes liés à la conception de MySQL"
author = "Jamko"
date = 2021-04-08T11:43:01+04:00
tags = ['mysql']
Categories = ["Article de Percona"]
featured_image = "posts/article01/Bad-Design-MySQL-367x205.png"
slug = "article01-quelques-problemes-de-conception-mysql"
toc = true
+++

MySQL dispose certainement plusieurs atouts particuliers, sinon, ce ne serait pas la base de données Open Source la plus populaire au monde (selon [DB-Engines](https://db-engines.com/en/ranking)). Cependant, je vois des décisions ou des comportements qui sont dus à des mauvaises conceptions. Beaucoup d’entre eux reposent sur de nombreux raisonnements historiques et sont peut-être toujours là parce que les ressources allouées au nettoyage du code sont insuffisantes.

Je suis passionné par l'observabilité, en particulier lorsqu'il s'agit de comprendre les performances du système. L'une des données les plus importantes pour comprendre les performances de MySQL est de comprendre sa discorde de verrous (mutexes, rwlocks, etc.).

La «meilleure» façon de comprendre les verrous dans MySQL est le schéma de performance. Malheureusement, le profilage de verrouillage est désactivé par défaut dans le schéma de performances car il entraîne une surcharge assez importante à tel point que vous ne l’utilisez probablement pas en production à tout moment.

Si vous cherchez à obtenir des informations mutex qui sont toujours disponibles en MySQL, vous pouvez les obtenir à partir du moteur de stockage InnoDB (ce qui est souvent assez pertinent, car c'est là que la plupart des conflits se produisent).

Vous pouvez regarder dans le résultat de **SHOW ENGINE INNODB STATUS** - en particulier dans la section SEMAPHORES:

```
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 490020789
--Thread 140407582807808 has waited at row0ins.cc line 2412 for 0 seconds the semaphore:
S-lock on RW-latch at 0x7fa0159bd6f0 created in file buf0buf.cc line 785
a writer (thread id 140407910762240) has reserved it in mode  exclusive
number of readers 0, waiters flag 1, lock_word: 0
Last time read locked in file row0ins.cc line 2412
Last time write locked in file /mnt/workspace/percona-server-8.0-debian-binary/label_exp/min-bionic-x64/test/percona-server-8.0.18-9/storage/innobase/include/mtr0mtr.ic line 142
--Thread 140386577712896 has waited at row0ins.cc line 2412 for 0 seconds the semaphore:
S-lock on RW-latch at 0x7fa0159bd6f0 created in file buf0buf.cc line 785
a writer (thread id 140407910762240) has reserved it in mode  exclusive
number of readers 0, waiters flag 1, lock_word: 0
Last time read locked in file row0ins.cc line 2412
Last time write locked in file /mnt/workspace/percona-server-8.0-debian-binary/label_exp/min-bionic-x64/test/percona-server-8.0.18-9/storage/innobase/include/mtr0mtr.ic line 142
```

Cette section fournit des informations sur les mutex attendus ainsi que des informations sur les temps d'attente qui peuvent être très utiles. Malheureusement, ces informations sont fournies sous une forme qui n'est pas facilement analysable et ne peut être récupérée qu'avec l’intégralité du résultat de **SHOW ENGINE INNODB STATUS** entraînant ainsi une charge supplémentaire qui le rend inadéquat à l'échantillonnage répétitif.

Savoir pourquoi cette information n'a jamais été rendue accessible avec une table `INFORMATION_SCHEMA` est un casse-tête. Est-ce parce que l’idée était que `PERFORMANCE_SCHEMA` devrait être le seul et unique outil d’observabilité, même lorsque l’équipe d’ingénierie MySQL ne peut pas le faire fonctionner avec une surcharge acceptable?

Mais attendez, dites-vous ... il existe un meilleur moyen - vous pouvez utiliser **SHOW ENGINE INNODB MUTEX**  pour obtenir un résumé des statistiques:

```
mysql> SHOW ENGINE INNODB MUTEX;
+--------+----------------------------+-----------------+
| Type   | Name                       | Status          |
+--------+----------------------------+-----------------+
| InnoDB | rwlock: dict0dict.cc:2454  | waits=138       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=545       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=124       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=110       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=134       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=132       |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=5317      |
| InnoDB | rwlock: dict0dict.cc:2454  | waits=538       |

…
| InnoDB | rwlock: hash0hash.cc:171   | waits=219       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=291       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=290       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=312       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=281       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=226       |
| InnoDB | rwlock: hash0hash.cc:171   | waits=327       |
| InnoDB | sum rwlock: buf0buf.cc:785 | waits=138699546 |
+--------+----------------------------+-----------------+
332 rows in set (0.59 sec)
```

Cette commande ne fournit pas les mêmes informations (elle indique le nombre d'attentes, et non pas l'attente existante) mais elle est utile. Le problème avec cette commande est qu'elle semble avoir été spécialement conçue pour être la moins utile possible. Regardez qu’il y a beaucoup de doublons dans "Name". En effet, il existe plusieurs instances du même type de mutex. En general, lorsque vous souhaitez comprendre le type de conflit auquel vous êtes confronté, vous souhaitez totaliser la somme des nombre d’attentes en le regroupant par nom, et malheureusement, vous ne pouvez pas le faire avec les commandes SHOW.

Encore plus étrange est le choix de la syntaxe waits = N et l'attribution d'un nom à une colonne «Statut» là où la conception de la base de données relationnelle l’aurait appelée «Waits».

J'aurais également préféré voir le nom de l'objet de synchronisation ici au lieu de la ligne de code source car il est généralement beaucoup plus descriptif. MariaDB le fait, et il le rend également disponible sous [forme de table de d'informations innodb_mutex](https://mariadb.com/kb/en/information-schema-innodb_mutexes-table/).

Enfin, notez à quel point cette commande est lente (lire: coûteuse): 0,6 seconde sur un pool de mémoire tampon de 80 Go. La raison en est qu'il capture les conflits de mutex sur les pages du pool de mémoire tampon. C'es indispensable pour identifier les conflits relatifs à chaque page, mais cela nécessite également l'agrégation des informations pour des millions d'objets potentiels, ce qui prend du temps.

Je pense que la table INFORMATION_SCHEMA pourrait également être plus adaptée pour afficher ces informations.

Alors, nous ne pouvons donc pas exécuter SELECT sur la table INFORMATION_SCHEMA pour obtenir les données sous une forme facilement assimilable, mais peut-être devrions-nous écrire une procédure stockée à la place? 

Cela nous amène à un autre problème de conception. Bien que vous puissiez facilement itérer SELECT dans une procédure stockée, cela ne fonctionne pas pour les commandes SHOW. Il y avait probablement une bonne raison pratique à cette limitation, mais elle n'est pas du tout conviviale.

Quand rien ne va, nous pouvons toujours recourir aux scripts Shell pour résoudre le problème:

```
root@rocky:~# mysql -BNe "SHOW ENGINE INNODB MUTEX" | awk -F'\t' '{split($3, waits, "="); out[$2]+=waits[2];} END { for(el in out) printf "%s\t%d\n", el, out[el] } '
sum rwlock: buf0buf.cc:785      138984320
rwlock: btr0sea.cc:202  226767645
rwlock: trx0purge.cc:222        13
rwlock: ibuf0ibuf.cc:543        345822
rwlock: dict0dict.cc:2454       1958468
rwlock: dict0dict.cc:1042       66610
rwlock: fil0fil.cc:3150 1064444
rwlock: hash0hash.cc:171        37536
rwlock: dict0dict.cc:330        131
```

Cependant, si votre base de données vous oblige à proceder ainsi pour regrouper les informations ordinaires c'est qu'il y a quelquechose qui ne va pas!

**Ce dont j'aimerais voir?** Je pense que toutes les instructions SHOW doivent être revisées et si elles ne sont pas prévues pour la dépréciation, des informations similaires à ce qu'elles fournissent doivent être mises à disposition à partir de tables ou de vues. En fait, ce travail a déjà été effectué pour la plupart des commandes habituelles mais il semble qu'il ne soit jamais terminé.


Page source : [Percona Blog](https://www.percona.com/blog/2020/01/08/mysql-a-series-of-bad-design-decisions/)
