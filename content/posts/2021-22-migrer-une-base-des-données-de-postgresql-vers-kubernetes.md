+++
title = "Migrer une base de données PostgreSQL vers Kubernetes"
description = "Migration des bases de données pour les entreprises. Cas d'une base de données PostgreSQL vers Kubernetes avec Percona Distribution for PostgreSQL Operator"
author = "Francis"
date = 2021-11-01T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle22.jpg"
images = ["thumbnail/thumbnailarticle22.jpg"]
slug = "migration-des-bases-de-données-postgresql-vers-kubernetes"
+++

De plus en plus d'entreprises adoptent Kubernetes. Pour certains, il s'agit d'être à la pointe de la technologie, pour d’autres, il s'agit d'une stratégie bien définie et d'une transformation de l'entreprise. Les développeurs et les équipes d'exploitation du monde entier ont du mal à déplacer des applications qui ne sont pas compatibles avec le cloud natif pour les conteneurs et Kubernetes.

La migration des bases de données est toujours un défi, qui comporte des risques et des temps d'arrêt pour les entreprises. Aujourd'hui, je vais montrer à quel point il est facile de migrer une base de données PostgreSQL vers Kubernetes avec un temps d'arrêt minimal avec [Percona Distribution for PostgreSQL Operator](https://www.percona.com/doc/kubernetes-operator-for-postgresql/index.html) . 

## Objectif

Pour effectuer la migration, je vais utiliser la configuration suivante :

![image01](/posts/article22/img01.png)

1. Base de données PostgreSQL déployée sur site ou quelque part dans le cloud. Ce sera la Source.
1. [Cluster Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine) (GKE) où Percona Operator déploie et gère le cluster PostgreSQL (la cible) et le pod [pgBackRest](https://pgbackrest.org/)   
1. Les sauvegardes PostgreSQL et les journaux d'écriture anticipée sont téléchargés dans un compartiment de stockage d'objets (GCS dans mon cas)
1. Pod  pgBackRest lit les données du bucket
1. Pod pgBackRest restaure les données en continu vers le cluster PostgreSQL dans Kubernetes

Les données doivent être synchronisées en permanence. En fin de compte, je souhaite arrêter PostgreSQL exécuté sur site et conserver uniquement le cluster dans GKE.

## Migration

Conditions préalables

Pour reproduire la configuration, vous aurez besoin des éléments suivants :

- PostgreSQL (v12 ou 13) exécuté quelque part
- pgBackRest installé
- Google Cloud Storage ou n'importe quel bucket S3. Mes exemples concerneront GCS.
- cluster Kubernetes

Configurer la source

J'ai [Percona Distribution pour PostgreSQL](https://www.percona.com/software/postgresql-distribution) version 13 qui s'exécute sur certaines machines Linux.  

\1. Configurer [pgBackrest](https://github.com/spron-in/blog-data/blob/master/postgres-to-kubernetes/pgbackrest.conf) 

```
# cat /etc/pgbackrest.conf
[global]
log-level-console=info
log-level-file=debug
start-fast=y

[db]
pg1-path=/var/lib/postgresql/13/main
repo1-type=gcs
repo1-gcs-bucket=sp-test-1
repo1-gcs-key=/tmp/gcs.key
repo1-path=/on-prem-pg
```

- pg1-path doit pointer vers le répertoire de données PostgreSQL
- repo1-type est défini sur GCS car nous voulons que nos sauvegardes y soient
- La clé se trouve dans le fichier /tmp/gcs.key. La clé peut être obtenue via l'interface utilisateur de Google Cloud. En savoir plus à ce sujet [ici](https://cloud.google.com/iam/docs/creating-managing-service-account-keys) . 
- Les sauvegardes vont être stockées dans le dossier on-prem-pg dans le compartiment sp-test-1

\2. Modifiez la configuration `postgresql.conf` pour activer l'archivage via pgBackrest   

```
archive_mode = on   
archive_command = 'pgbackrest --stanza=db archive-push %p'
```

Un redémarrage est nécessaire après avoir modifié la configuration.

\3. L'opérateur nécessite d'avoir un fichier `postgresql.conf` dans le répertoire de données. Il suffit d'avoir un fichier vide :  
```
touch /var/lib/postgresql/13/main/postgresql.conf
```

\4. `primaryuser` doit être créé sur la source pour s'assurer que la réplication soit correctement configurée par l'opérateur.   

```
# create user primaryuser with encrypted password '<PRIMARYUSER PASSWORD>' replication;
```

## Configurer la cible

\1. Déployez Percona Distribution for PostgreSQL Operator sur Kubernetes. En savoir plus à ce sujet [ici](https://www.percona.com/doc/kubernetes-operator-for-postgresql/kubernetes.html) . 

```
# create the namespace
kubectl create namespace pgo

# clone the git repository
git clone -b v0.2.0 https://github.com/percona/percona-postgresql-operator/
cd percona-postgresql-operator

# deploy the operator
kubectl apply -f deploy/operator.yaml
```

\2. Modifiez le manifeste de ressource personnalisé principal - deploy/cr.yaml.

- Je ne vais pas changer le nom du cluster et le garder cluster1
- le cluster va fonctionner en mode veille, ce qui signifie qu'il va synchroniser les données du bucket GCS. Définissez `spec.standby` sur `true` .   
- configurer GCS lui-même. La section `spec.backup` ressemblerait à ceci ( `bucket` et `repoPath` sont les mêmes que dans la configuration pgBackrest ci-dessus)      

```
 backup:
...
    repoPath: "/on-prem-pg"
...
    storages:
      my-s3:
        type: gcs
        endpointUrl: https://storage.googleapis.com
        region: us-central1-a
        uriStyle: path
        verifyTLS: false
        bucket: sp-test-1
    storageTypes: [
      "gcs"
    ]
```

- J'aimerais avoir au moins une réplique dans mon cluster PostgreSQL. Définissez `spec.pgReplicas.hotStandby.size` sur 1.  

\3. L'opérateur doit pouvoir s'authentifier auprès de GCS. Pour ce faire, nous devons créer un objet secret appelé `<CLUSTERNAME> - dossier - repo - config` avec `gcs-key` dans les données. Ce devrait être la même clé que nous avons utilisée sur la source. Voir l'exemple de ce secret [ici](https://github.com/spron-in/blog-data/blob/master/postgres-to-kubernetes/gcs.yaml) .     

```
kubectl apply -f gcs.yaml
```

\4. Créez des utilisateurs en créant des objets Secrets : `postgres` et `primaryuser` (celui que nous avons créé sur la Source). Voir les exemples d'utilisateurs Secrets [ici](https://github.com/spron-in/blog-data/blob/master/postgres-to-kubernetes/users.yaml) . Les mots de passe doivent être les mêmes que sur la Source.     

```
kubectl apply -f users.yaml
```

\5. Déployons maintenant notre cluster sur Kubernetes en appliquant le `cr.yaml` : 

```
kubectl apply -f deploy/cr.yaml
```


## Vérifier et dépanner

Si tout est fait correctement, vous devriez voir ce qui suit dans les journaux du pod principal :

```
kubectl -n pgo logs -f --tail=20 cluster1-5dfb96f77d-7m2rs
2021-07-30 10:41:08,286 INFO: Reaped pid=548, exit status=0
2021-07-30 10:41:08,298 INFO: establishing a new patroni connection to the postgres cluster
2021-07-30 10:41:08,359 INFO: initialized a new cluster
Fri Jul 30 10:41:09 UTC 2021 INFO: PGHA_INIT is 'true', waiting to initialize as primary
Fri Jul 30 10:41:09 UTC 2021 INFO: Node cluster1-5dfb96f77d-7m2rs fully initialized for cluster cluster1 and is ready for use
2021-07-30 10:41:18,781 INFO: Lock owner: cluster1-5dfb96f77d-7m2rs; I am cluster1-5dfb96f77d-7m2rs                                 2021-07-30 10:41:18,810 INFO: no action.  i am the standby leader with the lock                                                     2021-07-30 10:41:28,781 INFO: Lock owner: cluster1-5dfb96f77d-7m2rs; I am cluster1-5dfb96f77d-7m2rs                                 2021-07-30 10:41:28,832 INFO: no action.  i am the standby leader with the lock
```

Modifiez certaines données sur la source et assurez-vous qu'elles sont correctement synchronisées avec le cluster cible.

## Problèmes courants

Le message d'erreur suivant indique que vous avez oublié de créer le fichier `postgresql.conf` dans le répertoire de données :  

```
FileNotFoundError: [Errno 2] No such file or directory: '/pgdata/cluster1/postgresql.conf' -> '/pgdata/cluster1/postgresql.base.conf'
```

Parfois, il est facile d'oublier de créer `primaryuser` et de voir ce qui suit dans les journaux :  

```
psycopg2.OperationalError: FATAL:  password authentication failed for user "primaryuser"
```

Des informations d'identification de magasin d'objets erronées ou manquantes déclencheront l'erreur suivante :

```
WARN: repo1: [CryptoError] unable to load info file '/on-prem-pg/backup/db/backup.info' or '/on-prem-pg/backup/db/backup.info.copy':      CryptoError: raised from remote-0 protocol on 'cluster1-backrest-shared-repo': unable to read PEM: [218529960] wrong tag            HINT: is or was the repo encrypted?                                                                                                 CryptoError: raised from remote-0 protocol on 'cluster1-backrest-shared-repo': unable to read PEM: [218595386] nested asn1 error
      HINT: is or was the repo encrypted?
      HINT: backup.info cannot be opened and is required to perform a backup.
      HINT: has a stanza-create been performed?
ERROR: [075]: no backup set found to restore
Fri Jul 30 10:54:00 UTC 2021 ERROR: pgBackRest standby Creation: pgBackRest restore failed when creating standby
```


## Basculement ou Cutover
Tout semble bon et il est temps d'effectuer la transition. Dans cet article de blog, je ne couvre que le côté base de données mais n'oubliez pas que votre application doit être reconfigurée pour pointer vers le bon cluster PostgreSQL. Il peut être judicieux d'arrêter l'application avant le basculement.

\1. Arrêtez le cluster PostgreSQL source pour vous assurer qu'aucune donnée n'est écrite

```
systemctl stop postgresql
```

\2. Faites la promotion du cluster cible en tant que cluster principal. Pour ce faire, supprimez `spec.backup.repoPath` , remplacez `spec.standby` par false dans `deploy/cr.yaml` et appliquez les modifications : 
     
```
kubectl apply -f deploy/cr.yaml
```

PostgreSQL sera redémarré automatiquement et vous verrez ce qui suit dans les journaux :
```
2021-07-30 11:16:20,020 INFO: updated leader lock during promote
2021-07-30 11:16:20,025 INFO: Changed archive_mode from on to True (restart might be required)
2021-07-30 11:16:20,025 INFO: Changed max_wal_senders from 10 to 6 (restart might be required)
2021-07-30 11:16:20,027 INFO: Reloading PostgreSQL configuration.
server signaled
2021-07-30 11:16:21,037 INFO: Lock owner: cluster1-5dfb96f77d-n4c79; I am cluster1-5dfb96f77d-n4c79
2021-07-30 11:16:21,132 INFO: no action.  i am the leader with the lock
```

## Conclusion

Le déploiement et la gestion de clusters de bases de données n'est pas une tâche facile. Percona Distribution for PostgreSQL Operator, récemment publié, automatise les opérations du jour 1 et du jour 2 et transforme l'exécution de PostgreSQL sur Kubernetes en un parcours fluide et agréable.

Kubernetes devenant le plan de contrôle par défaut, la tâche la plus courante pour les développeurs et les équipes d'exploitation consiste à effectuer la migration, qui se transforme généralement en un projet complexe. Ce billet de blog montre que la migration de base de données peut être une tâche facile avec un temps d'arrêt minimal.

Nous vous encourageons à essayer notre opérateur. Consultez notre [référentiel github](https://github.com/percona/percona-postgresql-operator/) et consultez la [documentation](https://www.percona.com/doc/kubernetes-operator-for-postgresql/index.html) .   

Vous avez trouvé un bug ou vous avez une idée de fonctionnalité ? N'hésitez pas à le soumettre dans [JIRA](https://jira.percona.com/browse/K8SPG) . 

Pour les questions générales, veuillez soulever le sujet dans le [forum de la communauté](https://forums.percona.com/c/postgresql/percona-kubernetes-operator-for-postgresql/68) . 

Vous êtes développeur et vous souhaitez contribuer ? Veuillez lire notre [CONTRIBUTING.md](https://github.com/percona/percona-postgresql-operator/blob/main/CONTRIBUTING.md) et envoyer un Pull Request.  

**Percona Distribution for PostgreSQL fournit dans une seule distribution, les meilleurs et les plus critiques des composants entreprise de la communauté open source, conçue et testée pour fonctionner ensemble.**

[**Télécharger Percona Distribution pour PostgreSQL**](https://www.percona.com/software/postgresql-distribution)


Source : [Blog Percona](https://www.percona.com/blog/migrating-postgresql-to-kubernetes)
