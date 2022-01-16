+++
title = "Mis au point des paramètres de configuration MySQL 8"
description = "Apprendre à optimiser la configuration de MySQL 8"
author = "Francis"
date = 2021-08-13T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/amisdepercona21-00011.jpg"
images = ["thumbnail/amisdepercona21-00011.jpg"]
slug = "parametrer-et-configurer-mysql8"
+++


Dans les versions précédentes de MySQL, il y avait souvent une "valse de mise à niveau" qui devait être exécutée lors du démarrage d'une instance MySQL nouvellement mise à niveau avec le fichier de configuration de la version précédente. Dans certains cas, quelques options obsolètes peuvent ne plus être prises en charge dans la nouvelle version du serveur, déclenchant ainsi une erreur et un arrêt quelques instants après le démarrage. La même chose peut se produire même en dehors des scénarios de mise à niveau si une modification de configuration a été apportée avec une erreur ou une faute de frappe dans le nom ou la valeur d’une variable.

Depuis MySQL 8.0.16 et versions ultérieures, il existe désormais une option ' **validate- config '** pour tester et valider rapidement les options de configuration du serveur sans avoir à démarrer le serveur. Une fois utilisé, si aucun problème n'est trouvé avec le fichier de configuration, le serveur se fermera avec un code de sortie zéro (0). Si un problème est détecté, le serveur se fermera avec un code d'erreur un (1) pour la première circonstance de tout ce qui est déterminé comme non valide.

## **Validation Single Option**
Par exemple, considérons l'option de serveur suivante ( old\_passwords ) qui a été supprimée dans MySQL 8 :

```
./mysqld --old_passwords=1 --validate-config
2021-03-25T17:56:49.932782Z 0 [ERROR] [MY-000067] [Server] unknown variable 'old_passwords=1'.
2021-03-25T17:56:49.932862Z 0 [ERROR] [MY-010119] [Server] Aborting
```

Notez que le serveur s'est terminé avec un code d'erreur un (1) et affiche l'erreur pour l'option invalide.

## **Validation d’un fichier de configuration**
Il est également possible de valider un fichier de configuration my.cnf entier pour vérifier toutes les options :

```
./mysqld --defaults-file=/home/sandbox/my.cnf --validate-config
2021-03-25T18:03:41.938734Z 0 [ERROR] [MY-000067] [Server] unknown variable 'old_passwords=1'.
2021-03-25T18:03:41.938865Z 0 [ERROR] [MY-010119] [Server] Aborting
```

Notez que le serveur s'est arrêté à la **première occurrence** d'une valeur non valide. Toutes les erreurs restantes dans le fichier de configuration devront être trouvées après avoir corrigé la première erreur et exécuté à nouveau l'option **validate-config** . Donc, dans cet exemple, j'ai maintenant supprimé l' option ' **old\_passwords =1** ' dans le fichier de configuration, et je dois réexécuter **validate-config** pour voir s'il reste d'autres erreurs.

```
./mysqld --defaults-file=/home/sandbox/my.cnf --validate-config
2021-03-25T18:08:28.612912Z 0 [ERROR] [MY-000067] [Server] unknown variable 'query_cache_size=41984'.
2021-03-25T18:08:28.612980Z 0 [ERROR] [MY-010119] [Server] Aborting
```

En effet, il y a encore une autre option qui a été supprimée de MySQL 8 dans le fichier de configuration, donc après avoir ré-exécuté la validation (après avoir corrigé le premier problème), nous avons maintenant identifié un deuxième problème. Après avoir supprimé l'option **query\_cache\_size** du fichier my.cnf et exécuté à nouveau **validate- config** , nous obtenons enfin une configuration soignée.
```
./mysqld --defaults-file=/home/sandbox/my.cnf --validate-config
```

## **Modifier la verbosité de la validation**
Par défaut, l’option **validate-config** ne rapportera que les messages d'erreur. Si vous souhaitez également voir des avertissements ou des messages d'information, vous pouvez modifier **log\_error\_verbosity** avec une valeur supérieure à un (1) :

```
./mysqld --defaults-file=/home/sandbox/my.cnf --log-error-verbosity=2 --validate-config
2021-03-25T18:20:01.380727Z 0 [Warning] [MY-000076] [Server] option 'read_only': boolean value 'y' was not recognized. Set to OFF.
```

Maintenant, nous voyons des messages de niveau d'avertissement, et le serveur renvoi un code d'erreur zéro (0) car il n'y a techniquement aucune erreur, seulement des avertissements. Pour aller plus loin, j'ai ajouté l' option **query\_cache\_size** d'en haut dans le fichier my.cnf, et si nous exécutons à nouveau la validation, nous voyons à la fois des erreurs et des avertissements cette fois, tandis que le serveur se ferme avec un code d'erreur un (1) car il y a vraiment eu une erreur :

```
./mysqld --defaults-file=/home/sandbox/my.cnf --log-error-verbosity=2 --validate-config
2021-03-25T18:23:11.364288Z 0 [Warning] [MY-000076] [Server] option 'read_only': boolean value 'y' was not recognized. Set to OFF.
2021-03-25T18:23:11.372729Z 0 [ERROR] [MY-000067] [Server] unknown variable 'old_passwords=1'.
2021-03-25T18:23:11.372795Z 0 [ERROR] [MY-010119] [Server] Aborting
```

## **Ancienne version alternative**
Lors de mes tests de la fonctionnalité **validate\_config,** un collègue a souligné qu'il existe un moyen de répliquer cette validation sur les anciennes versions de MySQL en utilisant une combinaison d'options « help » et « verbose » comme ci-dessous :

```
$ mysqld --defaults-file=/tmp/my.cnf --verbose --help 1>/dev/null; echo $?
0
 
$ echo default_table_encryption=OFF >> /tmp/my.cnf
 
$ mysqld --defaults-file=/tmp/my.cnf --verbose --help 1>/dev/null; echo $?
2021-03-26T08:43:04.987265Z 0 [ERROR] unknown variable 'default_table_encryption=OFF'
2021-03-26T08:43:04.990898Z 0 [ERROR] Aborting
1
 
$ mysqld --version
mysqld  Ver 5.7.33-36 for Linux on x86_64 (Percona Server (GPL), Release 36, Revision 7e403c5)
```

## **Pour Conclure**
Bien qu'elle ne soit pas parfaite, la fonctionnalité **validate-config** contribue grandement à faciliter la gestion des mises à niveau et des modifications de configuration. Il est maintenant possible de savoir avec une certaine certitude que vos fichiers de configuration ou vos options sont valides avant de redémarrer le serveur et de trouver le problème qui finit par empêcher votre nœud de démarrer normalement, ce qui entraîne un temps d'arrêt inattendu.

**Percona Distribution for MySQL est la solution MySQL open source la plus complète, stable, évolutive et sécurisée disponible, offrant des environnements de base de données de niveau entreprise pour vos applications et métier les plus exigeant … son utilisation est gratuite !**

**Télécharger [Percona Distribution MySQL**](https://www.percona.com/software/mysql-database)**

Article original : [Percona](https://www.percona.com/blog/2021/04/01/easily-validate-configuration-settings-in-mysql-8/)
