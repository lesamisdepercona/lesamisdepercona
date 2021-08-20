+++
title = "Rejouer l'exécution de MySQL avec Record and Replay"
description = "Traduit à partir de l'article de Marcelo Altmann intitulé, Replay the Execution of MySQL With Record and Replay"
author = "Francis"
date = 2021-08-20T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/amisdepercona21-00012.jpg"
images = ["thumbnail/amisdepercona21-00012.jpg"]
slug = "rejouer-l-execution-de-MySQL-avec-rr-record-and-replay"
+++



**La chasse aux bugs peut être une tâche fastidieuse, et les logiciels multi-threads ne facilitent pas la tâche.** Les threads seront programmés à des moments différents, les instructions n'auront pas de résultats déterministes, et pour reproduire un problème particulier, cela peut nécessiter exactement les mêmes threads, effectuant exactement le même travail, exactement au même moment. Comme vous pouvez l'imaginer, ce n'est pas si simple.

Disons que votre base de données tombe en panne ou même qu'elle a un décrochage transitoire. Au moment où vous y arrivez, le crash s'est produit et vous êtes bloqué en restaurant le service rapidement et en faisant des investigations après coup. Ne serait-il pas agréable de rejouer le travail juste avant ou pendant le crash et de voir exactement ce qui se passait ?

[Record and Replay](https://rr-project.org/) est une technique où nous enregistrons l'exécution d'un programme lui permettant d'être rejoué encore et encore produisant le même résultat. Les ingénieurs de Mozilla ont créé RR, et fondamentalement, cet outil open source vous permet d'enregistrer l'exécution du logiciel et de le rejouer sous le célèbre GDB. 

**Un problème de sauvegarde**

Pour démontrer à quel point l'outil est puissant, nous expliquerons comment nous l'avons utilisé pour réduire le problème de [PXB-2180](https://jira.percona.com/browse/PXB-2180) (remerciement spécial à [Satya Bodapati](https://www.percona.com/blog/author/satya-bodapati/) , qui a aidé à toutes les recherches internes d'InnoDB pour ce bug).

En résumé, nous avions vu [Percona XtraBackup ](https://www.percona.com/software/mysql-database/percona-xtrabackup) planter au stade de la préparation (rappelez-vous de toujours tester votre sauvegarde!). Le crash se produisait de manière aléatoire, parfois après le deuxième incrémentiel, parfois après le 10ème incrémentiel, sans motif visible.

La trace de la pile n'était pas non plus toujours la même. Il plantait sur différentes parties d'InnoDB , mais ici, nous avions un point commun entre tous les plantages - cela se produisait toujours en essayant d'appliquer un enregistrement de journalisation à la même page de blocage et au même identifiant d'espace :

Notre soupçon était que la mise en page de ce bloc divergeait entre MySQL et XtraBackup. Lorsque vous travaillez avec ces types de bug, le plantage est toujours la conséquence de quelque chose qui s'est passé plus tôt, par exemple : un plantage sur la sixième sauvegarde incrémentielle pourrait être la conséquence d'un problème survenu lors de la quatrième sauvegarde incrémentielle.

L'objectif principal à cette étape est de prouver et d'identifier où la mise en page s'est détournée.

Avec ces informations, nous avons exécuté MySQL sous RR et réexécuté la sauvegarde jusqu'à ce que nous voyions le même problème lors de la préparation. Nous pouvons maintenant rejouer l'exécution de MySQL et vérifier comment ça se présente. Notre idée est de : 

1. Lire les [LSN](https://dev.mysql.com/doc/refman/8.0/en/glossary.html%23glos_lsn#glos_lsn) pour cette même page avant/après chaque préparation de sauvegarde
1. Identifier toutes les modifications apportées à m\_space = 4294967294 & m\_page\_no = 5 sur mysqld

Avant d'aller plus loin, nous tenons à expliquer quelques points :

1. m\_space = 4294967294 correspond au dictionnaire de données MySQL (mysql.ibd) – [dict0dict.h:1146](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/dict0dict.h%23L1146#L1146)
1. Sur la page du disque, LSN est stocké au 16e octet de la page avec une taille de 8 octets – [fil0types.h:66](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/fil0types.h%23L66#L66) 
1. Les pages sont écrites séquentiellement sur le disque, par exemple, pour la taille de page par défaut de 16 ko, à partir d’octet 1 à 16384 auront les données de la page 0. Celle de l'octet entre 16385 et 32768, les données seront dans la page 1, et ainsi de suite.
1. Le Frame est une donnée brute d'une page – [buf0buf.h:1358](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/buf0buf.h%23L1358#L1358)

**Rejouer l'exécution**

Pour commencer, lisons le LSN que nous avons sur mysql.ibd pour la page cinq avant la sauvegarde. Nous utiliserons od (consultez man od pour plus d'informations) et les informations expliquées ci-dessus.

Vérifions s'il correspond à un tampon LSN de mysqld. Pour cela nous allons ajouter un point d'arrêt conditionnel sur le replay de l'exécution de MySQL à la fonction buf\_flush\_note\_modification : 

Nous pouvons voir le tampon LSN d'avant la préparation de la sauvegarde complète et le premier tampon de relecture de la session match. Il est temps de préparer la sauvegarde, d'avancer l'exécution de la relecture et de revérifier :

Même tampon LSN à tous les deux, serveur et sauvegarde. Il est temps de passer à autre chose et de commencer à appliquer les incrémentiels :

Encore une fois, nous avons le même tampon LSN des deux côtés. Passage à l'incrémentiel suivant :

L’incrémentiel 02 s’appliquent et correspondent au LSN à partir de mysqld. Continuons jusqu'à ce que nous trouvions une incompatibilité :

Ok, ici nous avons quelque chose. Les fichiers de sauvegarde passent de 0x01474d3f à 0x015fa326 lors de l'application de l’incrémentiel quatre tandis que le serveur est passé de 0x01474d3f à 0x019f3fc9. Peut-être avons-nous manqué un autre endroit où nous pouvons mettre à jour le tampon LSN d'une page? Maintenant, nous sommes à un point d’avance avec notre exécution de relecture du serveur MySQL

**Rejouer l'exécution à l'envers**

Voici (encore) une autre fonctionnalité très intéressante de RR, elle vous permet de rejouer l'exécution à l'envers. Pour éliminer la possibilité de manquer un endroit qui met également à jour le LSN de ce bloc, ajoutons un point de surveillance matériel sur l’adresse mémoire block- >frame et inversons l'exécution :

En rejouant l'exécution à l'envers, nous pouvons voir qu'en effet le serveur a changé le LSN de 0x01474d3f à 0x019f3fc9 . Cela confirme que le problème se situe au niveau de la sauvegarde incrémentielle quatre, car le LSN 0x015fa326 que nous voyons à la fin de la sauvegarde incrémentielle quatre n'a jamais été un LSN valide lors de l'exécution du serveur.

**Cause première**

Maintenant que nous avons limité la portée de six sauvegardes à une seule, les choses vont devenir plus faciles.

Si nous examinons de près les messages de journal de -prepare de la sauvegarde, nous pouvons voir que le LSN de mysql.ibd correspond au tampon LSN à la fin de la sauvegarde :

En vérifiant la trace de la pile du problème et en examinant plus en détail le bloc que nous avons analysé, nous pouvons voir qu'il s'agit de l'index innodb\_dynamic\_metadata:

Vous vous demandez peut-être d'où vient le 66? Cela vient de l'examen de la position [FIL_PAGE_DATA](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/fil0types.h%23L114#L114) + [PAGE_INDEX_ID](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/page0types.h%23L83#L83) . Cela nous a donné l'ID d'index 2. C'est en dessous de 1024, qui est réservé aux tables du [dictionnaire de données](https://github.com/percona/percona-xtrabackup/blob/8.0/storage/innobase/include/dict0dd.h%23L82#L82). En vérifiant la deuxième table de cette liste, nous pouvons voir qu'il s'agit de [innodb_dynamic_metadata](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/dict0dd.h%23L276#L276) . Avec toutes ces informations résumées, nous pouvons regarder ce que fait le serveur à l'arrêt, et le problème devient beaucoup plus évident : [srv0start.cc:3965](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/srv/srv0start.cc%23L3965#L3965)

Dans le cadre du processus d'arrêt, nous conservons les métadonnées modifiées dans la table DD Buffer ( innodb\_dynamic\_metadata ), ce qui est faux. Ces modifications seront probablement conservées par le serveur et enregistrées à nouveau une fois que le serveur aura effectué un point de contrôle. En outre, beaucoup de données peuvent être fusionnées selon le moment où la sauvegarde a été effectuée et lorsque le serveur lui-même conserve ces données dans les tables DD. Ceci est le résultat de la mise en œuvre de [WL#7816](https://dev.mysql.com/worklog/task/%3Fid%3D7816) et [WL#6204](https://dev.mysql.com/worklog/task/%3Fid%3D6204) qui a obligé Percona XtraBackup à modifier la façon dont il gère ces types d'enregistrements de rétablissement.

**Résumé**

Dans ce blog, nous avons parcouru le processus d'analyse d'un véritable bug sur Percona XtraBackup . Ce bug expose un défi auquel nous sommes confrontés dans divers types de bug, où le crash/dysfonctionnement est une conséquence de quelque chose qui s'est passé bien avant, et au moment où nous avons une trace de pile/ coredump, il est trop tard pour effectuer une analyse appropriée. L'enregistrement et la relecture nous ont permis de rejouer de manière cohérente l'exécution du serveur source, ce qui a permis de réduire le problème permettant d’identifier la cause principale.

Percona XtraBackup est une solution de sauvegarde de base de données complète, gratuite et open source pour toutes les versions de Percona Server pour MySQL 

