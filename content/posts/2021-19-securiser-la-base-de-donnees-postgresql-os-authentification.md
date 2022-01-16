+++
title = "Sécurité de la base de données PostgreSQL : OS - Authentification"
description = "Qulques méthodes d'authentification de base impliquant le serveur PostgreSQL, le noyau et le serveur d'identification"
author = "Francis"
date = 2021-10-10T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article14-MyDumper.jpg"
images = ["thumbnail/thumbnailarticle19.jpg"]
slug = "securite-de-la-base-de-donnees-postgresql-os-authentification"
+++

La sécurité est l'affaire de tous lorsqu'on parle de données et d'informations, et elle devient donc le fondement principal de chaque base de données. La sécurité signifie protéger vos données contre les accès non autorisés. Cela signifie que seuls les utilisateurs autorisés peuvent se connecter à un système appelé authentification; un utilisateur ne peut faire que ce qu'il est autorisé à faire (autorisation) et enregistrer l'activité de l'utilisateur (comptabilité). Je les ai expliqués dans mon article principal sur la sécurité, [PostgreSQL Database Security: What You Need To Know](https://www.percona.com/blog/2021/01/04/postgresql-database-security-what-you-need-to-know/) . 

Lorsque nous parlons de sécurité, l'authentification est la première ligne de défense. PostgreSQL fournit diverses méthodes d'authentification, qui sont classées en trois catégories.

- [Authentification interne PostgreSQL ](https://www.percona.com/blog/2021/02/01/postgresql-database-security-authentication/)
- Authentification basée sur le système d'exploitation
- Authentification basée sur un serveur externe 

Dans la plupart des cas, PostgreSQL est configuré pour être utilisé avec une authentification interne. Par conséquent, j'ai discuté de toutes les authentifications internes dans le précédent [article de blog](https://www.percona.com/blog/2021/02/01/postgresql-database-security-authentication/) que j'ai mentionné ci-dessus. Dans ce blog, nous discuterons des méthodes d'authentification basées sur le système d'exploitation pour PostgreSQL. Il existe trois méthodes pour effectuer une authentification basée sur le système d'exploitation.

- [Identifiant](https://www.postgresql.org/docs/current/auth-ident.html)
- [PAM (Pluggable Authentication Modules)](https://www.postgresql.org/docs/current/auth-pam.html)
- [Peer](https://www.postgresql.org/docs/current/auth-ident.html)

## Identifiant - Ident

L'authentification d'identification ne prend en charge que les connexions TCP/IP. Son serveur d'identification fournit un mécanisme pour mapper le nom d'utilisateur du système d'exploitation du client sur le nom d'utilisateur de la base de données. Il a également la possibilité de mapper le nom d'utilisateur.

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
```

```
$ psql postgres -h 127.0.0.1 -U postgres
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL:  Ident authentication failed for user "postgres"
```

Si aucun serveur ident n'est installé, vous devrez installer l' ***ident2*** sur votre box ubuntu ou ***oidentd*** sur CentOS 7. Une fois que vous avez téléchargé et configuré le serveur ident, il est maintenant temps de configurer PostgreSQL. Cela commence par la création d'une carte utilisateur dans le fichier "pg\_ident.conf".

```
# Put your actual configuration here
# ----------------------------------
# MAPNAME       SYSTEM-USERNAME         PG-USERNAME
PG_USER         vagrant                 postgres
```

Ici, nous avons mappé notre utilisateur système « vagrant » avec « postgres » de PostgreSQL. Il est temps de se connecter à l'aide de l'utilisateur vagrant.

```
$ psql postgres -h 127.0.0.1 -U postgres
psql (15devel)
Type "help" for help.

postgres=#
```

[*Remarque : Le protocole d'identification n'est pas conçu comme un protocole d'autorisation ou de contrôle d'accès.*](https://www.postgresql.org/docs/current/auth-ident.html)

## PAM (Pluggable Authentication Modules)
L'authentification PAM fonctionne de la même manière que les « mots de passe ». Vous devrez créer un fichier de service PAM qui devrait activer l'authentification basée sur PAM. Le nom du service doit être défini sur « PostgreSQL ».

Une fois le service créé, [PAM](https://www.kernel.org/pub/linux/libs/pam/) peut désormais valider les pairs noms d'utilisateur/mot de passe et éventuellement le nom d'hôte distant ou l'adresse IP connecté. L'utilisateur doit déjà exister dans la base de données pour que l'authentification PAM fonctionne.
```
$ psql postgres -h 127.0.0.1 -U postgres
Password for user postgres: 
2021-08-11 13:16:38.332 UTC [13828] LOG:  pam_authenticate failed: Authentication failure
2021-08-11 13:16:38.332 UTC [13828] FATAL:  PAM authentication failed for user "postgres"
2021-08-11 13:16:38.332 UTC [13828] DETAIL:  Connection matched pg_hba.conf line 91: "host    all             all             127.0.0.1/32            pam"
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL:  PAM authentication failed for user "postgres"
```

Assurez-vous que le serveur PostgreSQL prend en charge l'authentification PAM. C'est une option de compilation qui doit être définie lorsque les binaires du serveur ont été construits. Vous pouvez vérifier si votre serveur PostgreSQL prend en charge l'authentification PAM à l'aide de la commande suivante.
```
$ pg_config | grep with-pam

CONFIGURE =  '--enable-tap-tests' '--enable-cassert' '--prefix=/usr/local/pgsql/' '--with-pam'
```

S'il n'y a pas de fichier de serveur PAM pour PostgreSQL sous /etc/pam.d, vous devrez le créer manuellement. Vous pouvez choisir n'importe quel nom pour le fichier ; cependant, je préfère l'appeler "postgresql".

```
$ /etc/pam.d/PostgreSQL

@include common-auth
@include common-account
@include common-session
@include common-password
```
Étant donné que l'utilisateur PostgreSQL ne peut pas lire les fichiers de mots de passe, installez sssd ( [SSSD - System Security Services Daemon](https://sssd.io/) ) pour contourner cette limitation.
```
sudo apt-get install sssd
```
Ajoutez postgresql à « ad\_gpo\_map\_remote\_interactive » à « /etc/sssd/sssd.conf »

```
$ cat /etc/sssd/sssd.conf
ad_gpo_map_remote_interactive = +postgresql
```

Démarrez le service sssd et vérifiez qu'il a correctement démarré.

```
$ sudo systemctl start sssd
$ sudo systemctl status sssd
sssd.service - System Security Services Daemon
     Loaded: loaded (/lib/systemd/system/sssd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2021-08-11 16:18:41 UTC; 12min ago
   Main PID: 1393 (sssd)
      Tasks: 2 (limit: 1071)
     Memory: 5.7M
     CGroup: /system.slice/sssd.service
             ├─1393 /usr/sbin/sssd -i --logger=files
             └─1394 /usr/libexec/sssd/sssd_be --domain shadowutils --uid 0 --gid 0 --logger=files
```

Il est temps maintenant de configurer pg\_hba.conf pour utiliser l'authentification PAM. Nous devons spécifier le nom du service PAM (pamservice) dans le cadre des options d'authentification. Cela devrait être le même que le fichier que vous avez créé dans le dossier /etc/pam.d, qui dans mon cas est postgresql.

```
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            pam pamservice=postgresql
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
```

Il faut maintenant recharger (ou redémarrer) le serveur PostgreSQL. Après cela, vous pouvez essayer de vous connecter au serveur PostgreSQL.
```
vagrant@ubuntu-focal:~$ psql postgres -h 127.0.0.1 -U postgres
psql (15devel)
Type "help" for help.
postgres=>
```

Note

[*Si PAM est configuré pour lire /etc/shadow, l'authentification échouera car le serveur PostgreSQL est démarré par un utilisateur non root. Cependant, ce n'est pas un problème lorsque PAM est configuré pour utiliser LDAP ou d'autres méthodes d'authentification.* ](https://www.postgresql.org/docs/current/auth-pam.html)

## Peer

L'authentification par les pairs est « ***identique*** » ; c'est-à-dire, très semblable à l'authentification ident! Les seules différences subtiles sont qu'il n'y a pas de serveurs d'identification, et cette méthode fonctionne sur les connexions locales plutôt que sur TCP/IP.

L'authentification par les pairs fournit un mécanisme pour mapper le nom d'utilisateur du système d'exploitation du client sur le nom d'utilisateur de la base de données. Il a également la possibilité de mapper le nom d'utilisateur. La configuration est très similaire à celle que nous avons configurée pour l'authentification ident, sauf que la méthode d'authentification est spécifiée en tant que « pair » au lieu de « ident ». 

$ cat $PGDATA/pg\_hba.conf
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all             all                                   peer map=PG_USER
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
```

```
$ psql postgres -U postgres
2021-08-12 10:51:11.855 UTC [1976] LOG:  no match in usermap "PG_USER" for user "postgres" authenticated as "vagrant"
2021-08-12 10:51:11.855 UTC [1976] FATAL:  Peer authentication failed for user "postgres"
2021-08-12 10:51:11.855 UTC [1976] DETAIL:  Connection matched pg_hba.conf line 89: "local   all             all                                     peer map=PG_USER"
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "postgres"
```
La configuration de $PGDATA/pg\_hba.conf ressemblera à ceci :
```
$ cat $PGDATA/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only

local   all             all                                     peer map=PG_USER
# IPv4 local connections:
```

$PGDATA/pg\_ident.conf
```
# Put your actual configuration here
# ----------------------------------
# MAPNAME       SYSTEM-USERNAME         PG-USERNAME
PG_USER         vagrant                postgres
```

```
vagrant@ubuntu-focal:~$ psql postgres -U postgres
psql (15devel)
Type "help" for help.
postgres=>
```

## Conclusion

Nous avons couvert plusieurs méthodes d'authentification différentes dans ce blog. Ces méthodes d'authentification de base impliquent le serveur PostgreSQL, le noyau et le serveur d'identification ; les options sont disponibles nativement sans aucune dépendance externe majeure. Il est toutefois important que la base de données soit correctement sécurisée pour empêcher tout accès non autorisé aux données.

## Percona Distribution for PostgreSQL fournit les composants d'entreprise les meilleurs et les plus critiques de la communauté open source dans une seule distribution, conçue et testée pour fonctionner ensemble.

**[Telecharger Percona Distribution pour PostgreSQL](https://www.percona.com/software/postgresql-distribution)**

**Source : [Blog Percona](https://www.percona.com/blog/postgresql-database-security-os-authentication/)**
