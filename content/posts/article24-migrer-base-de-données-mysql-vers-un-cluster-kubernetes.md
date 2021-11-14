+++
title = "Migration d'une base de données MySQL vers un cluster Kubernetes"
description = "Migrer une base de données MySQL vers un cluster Kubernetes à l'aide de la réplication asynchrone s'avère de plus en plus facile en considérant les derniers changements importants : migrez la base de données vers le cluster PXC qui présente certaines particularités et, bien sûr, Kubernetes lui-même."
author = "Francis"
date = 2021-11-14T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle24.jpg"
images = ["thumbnail/thumbnailarticle24.jpg"]
slug = "migrer-base-de-données-mysql-vers-un-cluster-kubernetes"
+++

De nos jours, de plus en plus d'entreprises réfléchissent à la migration de leur infrastructure vers Kubernetes. Les bases de données ne font pas exception. De nombreux opérateurs k8s ont été créés pour simplifier les différents types de déploiements et également pour effectuer des tâches quotidiennes de routine telles que les sauvegardes, le renouvellement des certificats, etc. Si, il y a quelques années, personne ne voulait même entendre parler de l'exécution de bases de données dans Kubernetes, tout a changé maintenant.

Chez Percona, nous avons créé quelques opérateurs k8s très fonctionnels pour Percona Server pour les bases de données MongoDB, PostgreSQL et MySQL. Aujourd'hui, nous allons parler de l'utilisation de la [réplication inter-sites](https://www.percona.com/doc/kubernetes-operator-for-pxc/replication.html#set-up-percona-xtradb-cluster-cross-site-replication) - une nouvelle fonctionnalité qui a été ajoutée à la dernière version de [Percona Distribution for MySQL Operator](https://www.percona.com/software/percona-kubernetes-operators) . Cette fonctionnalité est basée sur [un mécanisme de basculement de connexion synchrone](https://dev.mysql.com/doc/refman/8.0/en/replication-asynchronous-connection-failover.html) .

La réplication inter-sites implique la configuration d'un cluster Percona XtraDB ou d'un/plusieurs serveurs MySQL en tant que source, et d'un autre cluster Percona XtraDB (PXC) en tant que réplique pour permettre une réplication asynchrone entre eux. Si un opérateur dispose de plusieurs sources en ressource personnalisée ( [CR](https://github.com/percona/percona-xtradb-cluster-operator/blob/v1.9.0/deploy/cr.yaml#L53-L55) ), il gérera automatiquement les échecs de connexion de la base de données source.

Cette fonctionnalité de réplication intersites n'est prise en charge que depuis MySQL 8.0.23, mais vous pouvez en savoir plus sur la migration de MySQL de versions antérieures dans cet [article de blog](https://www.percona.com/blog/2021/06/14/migrating-into-kubernetes-running-the-percona-distribution-for-mysql-operator/https://www.percona.com/blog/2021/06/14/migrating-into-kubernetes-running-the-percona-distribution-for-mysql-operator/). 

## Le but

Migrer la base de données MySQL, qui est déployée sur site ou dans le cloud, vers l'opérateur Percona Distribution for MySQL à l'aide de la réplication asynchrone. Cette approche vous aide à réduire les temps d'arrêt et la perte de données pour votre application.

Nous avons donc la configuration suivante :

![image01](/posts/article24/img01.png)

## Les composants suivants sont utilisés :

\1. Base de données MySQL 8.0.23 (dans mon cas, il s'agit de [Percona Server for MySQL](https://www.percona.com/software/mysql-database/percona-server) ) qui est déployée dans DO (en tant que source) et [Percona XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) pour la sauvegarde. Dans mon déploiement de test, j'utilise un seul serveur comme source pour simplifier le déploiement. Selon votre topologie de déploiement de base de données, vous pouvez utiliser plusieurs serveurs pour utiliser le mécanisme de basculement de connexion synchrone du côté de l'opérateur.

\2. Le cluster Google Kubernetes Engine (GKE) où [Percona Distribution for MySQL Operator](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html) est déployé avec le cluster PXC (en tant que cible).

\3. Le compartiment AWS S3 est utilisé pour enregistrer la sauvegarde à partir de la base de données MySQL, puis pour restaurer le cluster PXC dans k8s.

## Les étapes suivantes doivent être effectuées pour effectuer la procédure de migration

\1. Effectuez la sauvegarde de la base de données MySQL à l'aide de [Percona XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup) et chargez-la dans le compartiment S3 à l'aide de [xbcloud](https://www.percona.com/doc/percona-xtrabackup/LATEST/xbcloud/xbcloud.html).

\2. Effectuez la restauration de la base de données MySQL depuis le compartiment S3 dans le cluster PXC qui est déployé dans k8s.

\3. Configurez la réplication asynchrone entre le serveur MySQL et le cluster PXC géré par l'opérateur k8s

En conséquence, nous avons une réplication asynchrone entre le serveur MySQL et le cluster PXC dans k8s qui est en mode lecture seule.

## Migration

## Configurez le cluster PXC cible géré par l'opérateur k8s

\1. Déployez Percona Distribution for MySQL Operator sur Kubernetes (j'ai utilisé GKE 1.20).

```
# clone the git repository
git clone -b v1.9.0 https://github.com/percona/percona-xtradb-cluster-operator
cd percona-xtradb-cluster-operator

# deploy the operator
kubectl apply -f deploy/bundle.yaml
```

\2. Créez un cluster PXC à l'aide du manifeste de ressource personnalisé (CR) par défaut.

```
# create my-cluster-secrets secret (do no use default passwords for production systems)
kubectl apply -f deploy/secrets.yaml

# create cluster by default it will be PXC 8.0.23
kubectl apply -f deploy/cr.yaml
```

\3. Créez le secret avec les informations d'identification pour le compartiment AWS S3 qui sera utilisé pour accéder au compartiment S3 pendant la procédure de restauration.

```
# create S3-secret.yaml file with following content, and use correct credentials instead of XXXXXX

apiVersion: v1
kind: Secret
metadata:
  name: aws-s3-secret
type: Opaque
data:
  AWS_ACCESS_KEY_ID: XXXXXX
  AWS_SECRET_ACCESS_KEY: XXXXXX
```

```
# create secret
kubectl apply -f S3-secret.yaml
```

## Configurer le serveur MySQL source

\1. Installez Percona Server pour MySQL 8.0.23 et Percona XtraBackup pour la sauvegarde. Reportez-vous aux chapitres [Installation de Percona Server pour MySQL](https://www.percona.com/doc/percona-server/LATEST/installation/yum_repo.html) et [Installation de Percona XtraBackup](https://www.percona.com/doc/percona-xtrabackup/8.0/installation.html#installing-percona-xtrabackup-from-repositories) dans la documentation pour les instructions d'installation.

*REMARQUE :* 

*vous devez ajouter les options suivantes à my.cnf pour activer la prise en charge de [GTID](https://dev.mysql.com/doc/refman/8.0/en/replication-options-gtids.html#sysvar_gtid_mode); sinon, la réplication ne fonctionnera pas car elle est utilisée par défaut par le cluster PXC*

```
[mysqld]
enforce_gtid_consistency=ON
gtid_mode=ON
```

\2. Créez tous les utilisateurs nécessaires qui seront utilisés par l'opérateur k8s, le mot de passe doit être le même que dans deploy/secrets.yaml . Veuillez également noter que le mot de passe de l'utilisateur root doit être le même que dans le fichier `deploy/secrets.yaml` pour k8s the secret. Dans mon cas, j'ai utilisé nos mots de passe par défaut du fichier `deploy/secrets.yaml`
```
CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitory' WITH MAX_USER_CONNECTIONS 100;
GRANT SELECT, PROCESS, SUPER, REPLICATION CLIENT, RELOAD ON *.* TO 'monitor'@'%';
GRANT SERVICE_CONNECTION_ADMIN ON *.* TO 'monitor'@'%';

CREATE USER 'operator'@'%' IDENTIFIED BY 'operatoradmin';
GRANT ALL ON *.* TO 'operator'@'%' WITH GRANT OPTION;

CREATE USER 'xtrabackup'@'%' IDENTIFIED BY 'backup_password';
GRANT ALL ON *.* TO 'xtrabackup'@'%';

CREATE USER 'replication'@'%' IDENTIFIED BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* to 'replication'@'%';
FLUSH PRIVILEGES;
```
\2. Effectuez la sauvegarde de la base de données MySQL à l'aide de l'outil XtraBackup et chargez-la dans le compartiment S3.
```
# export aws credentials
export AWS_ACCESS_KEY_ID=XXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXX

#make the backup
xtrabackup --backup --stream=xbstream --target-dir=/tmp/backups/ --extra-lsndirk=/tmp/backups/  --password=root_password | xbcloud put --storage=s3 --parallel=10 --md5 --s3-bucket="mysql-testing-bucket" "db-test-1"
```
Maintenant, tout est prêt pour effectuer la restauration de la sauvegarde sur la base de données cible. Revenons donc à notre cluster k8s.

## Configurer la réplication asynchrone sur le cluster PXC cible

Si vous avez une base de données source complètement propre (sans aucune donnée), vous pouvez ignorer les points liés à la sauvegarde et à la restauration de la base de données. Sinon, procédez comme suit :

\1. Restaurez la sauvegarde à partir du compartiment S3 à l'aide du manifeste suivant :
```
# create restore.yml file with following content

apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBClusterRestore
metadata:
  name: restore1
spec:
  pxcCluster: cluster1
  backupSource:
    destination: s3://mysql-testing-bucket/db-test-1
    s3:
      credentialsSecret: aws-s3-secret
      region: us-east-1
```
```
# trigger the restoration procedure
kubectl apply -f restore.yml
```
En conséquence, vous aurez un cluster PXC avec des données de la base de données source. Maintenant, tout est prêt pour configurer la réplication.

\2. Modifiez le manifeste de ressource personnalisé `deploy/cr.yaml` pour configurer la section section `spec.pxc.replicationChannels`
```
spec:
  ...
  pxc:
    ...
    replicationChannels:
    - name: ps_to_pxc1
      isSource: false
      sourcesList:
        - host: <source_ip>
          port: 3306
          weight: 100
```
```
# apply CR
kubectl apply -f deploy/cr.yaml
```
## Vérifier la réplication

Afin de vérifier la réplication, vous devez vous connecter au nœud PXC et exécuter les commandes suivantes:
```
kubectl exec -it cluster1-pxc-0 -c pxc -- mysql -uroot -proot_password -e "show replica status \G"
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: <ip-of-source-db>
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000004
          Read_Master_Log_Pos: 529
               Relay_Log_File: cluster1-pxc-0-relay-bin-ps_to_pxc1.000002
                Relay_Log_Pos: 738
        Relay_Master_Log_File: binlog.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 529
              Relay_Log_Space: 969
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 9741945e-148d-11ec-89e9-5ee1a3cf433f
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 3
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: 9741945e-148d-11ec-89e9-5ee1a3cf433f:1-2
            Executed_Gtid_Set: 93f1e7bf-1495-11ec-80b2-06e6016a7c3d:1,
9647dc03-1495-11ec-a385-7e3b2511dacb:1-7,
9741945e-148d-11ec-89e9-5ee1a3cf433f:1-2
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name: ps_to_pxc1
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:
```
Vous pouvez également vérifier la réplication en vérifiant que les données changent.

## Promouvoir le cluster PXC en tant que principal

Dès que vous êtes prêt (votre application a été reconfigurée et prête à fonctionner avec la nouvelle base de données) pour arrêter la réplication et promouvoir le cluster PXC dans k8s en tant que base de données principale, vous devez effectuer les actions simples suivantes :

\1. Modifiez le `deploy/cr.yaml` et commentez les replicationChannels
```
spec:
  ...
  pxc:
    ...
    #replicationChannels:
    #- name: ps_to_pxc1
    #  isSource: false
    #  sourcesList:
    #    - host: <source_ip>
    #      port: 3306
    #      weight: 100
```
\2. Arrêtez le service mysqld sur le serveur source pour vous assurer qu'aucune nouvelle donnée n'est écrite.
```
 systemctl stop mysqld
```
\3. Appliquez un nouvel opérateur CR pour k8s.
```
# apply CR
kubectl apply -f deploy/cr.yaml
```
Par conséquent, la réplication est arrêtée et le mode lecture seule est désactivé pour le cluster PXC.

## Conclusion

Les technologies évoluent si vite qu'une procédure de migration vers le cluster k8s, en apparence très complexe à première vue, s'avère être ni difficile ni chronophage. Mais vous devez garder à l'esprit que des changements importants ont été apportés. Tout d'abord, vous migrez la base de données vers le cluster PXC qui présente certaines particularités et, bien sûr, Kubernetes lui-même. Si vous êtes prêt, vous pouvez commencer votre voyage vers Kubernetes dès maintenant.

L'équipe Percona est prête à vous guider tout au long de cette expérience. Si vous avez des questions, veuillez soulever le sujet dans le [forum de communauté](https://forums.percona.com/c/mysql-mariadb/percona-kubernetes-operator-for-mysql/28) .

**Les opérateurs Percona Kubernetes automatisent la création, la modification ou la suppression de membres dans votre environnement Percona Distribution for MySQL, MongoDB ou PostgreSQL.**

[En savoir plus sur les opérateurs Percona Kubernetes](https://www.percona.com/software/percona-kubernetes-operators)

