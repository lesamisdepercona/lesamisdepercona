+++
title = "Migrer une base de donnée open source de MongoDB vers Kubernetes"
description = "Cet article vient en complement des deux précedents: Migration d'une base de données PostgreSQL vers Kubernetes et Migration d'une base de données MySQL vers un cluster Kubernetes à l'aide de la réplication asynchrone"
author = "Francis"
date = 2021-12-06
tags = ['MongoDB']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle27.jpg"
images = ["thumbnail/thumbnailarticle27.jpg"]
slug = "article27-migration-de-mongodb-vers-kubernetes"
+++

Cet article de blog est le dernier d'une série d'articles sur la migration de bases de données vers Kubernetes avec les opérateurs Percona. Deux articles précédents peuvent être trouvés ici:

- [Migration de PostgreSQL vers Kubernetes](https://www.percona.com/blog/migrating-postgresql-to-kubernetes/)
- [Migration d'une base de données MySQL vers un cluster Kubernetes à l'aide de la réplication asynchrone](https://www.percona.com/blog/migration-of-a-mysql-database-to-a-kubernetes-cluster-using-asynchronous-replication/)

Comme vous l'avez peut-être déjà deviné, cette fois, nous allons couvrir la migration de MongoDB vers Kubernetes. Dans la version 1.10.0 de [Percona Distribution pour MongoDB Operator](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html) , nous avons introduit une [nouvelle fonctionnalité](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/replication.html) (en aperçu technique) qui permet aux utilisateurs d'exécuter de telles migrations via les capacités de réplication MongoDB régulières. Nous avons déjà montré comment il peut être utilisé pour fournir [une reprise après sinistre interrégionale pour MongoDB](https://www.percona.com/blog/disaster-recovery-for-mongodb-on-kubernetes) , nous vous encourageons à le lire

## Le but

Il existe deux manières de migrer la base de données :

1. Prenez la sauvegarde et restaurez-la. 

 - Cette option est la plus simple, mais s'accompagne malheureusement d'un temps d'arrêt. Plus la base de données est volumineuse, plus le temps de récupération est long.

2. Répliquez les données sur le nouveau site et basculez l'application une fois les réplicas synchronisés. 
   – Cela permet à l'utilisateur d'effectuer la migration et de basculer l'application avec zéro ou peu de temps d'arrêt.

Cet article de blog explique comment migrer le jeu de réplicas MongoDB vers Kubernetes avec des capacités de réplication. 

![image01](/posts/article27/blog-mongo-migrate-1024x582.png)


1. Nous avons un cluster MongoDB quelque part (la Source). Cela peut être sur site ou sur une machine virtuelle. À des fins de démonstration, je vais utiliser un nœud de jeu de réplicas autonome. La procédure de migration d'un jeu de réplicas avec plusieurs nœuds ou cluster partitionné est presque identique.
2. Nous avons un cluster Kubernetes avec Percona Operator (la cible). L'opérateur déploie 3 nœuds MongoDB autonomes en mode non managé (nous en parlerons ci-dessous).
3. Chaque nœud est exposé afin que les nœuds de la Source puissent les atteindre.
4. Nous allons répliquer les données sur les nœuds cibles en les ajoutant au jeu de réplicas.

Comme toujours, tous les scripts d'articles de blog et les fichiers de configuration sont disponibles publiquement dans [ce référentiel Github](https://github.com/spron-in/blog-data/tree/master/mongodb-to-kubernetes). 

## Conditions préalables

- Cluster MongoDB - sur site ou VM. Il est important de pouvoir configurer mongod dans une certaine mesure et d'ajouter des nœuds externes au jeu de réplicas.
- Cluster Kubernetes pour la cible.
- kubectl pour déployer et gérer l'opérateur et la base de données sur Kubernetes.

## Préparer la source

Cette section explique quelles préparations doivent être effectuées sur la source pour configurer la réplication.

## Exposer

Tous les nœuds du jeu de répliques doivent former un maillage et pouvoir se joindre. La communication entre les nœuds peut passer par l'Internet public ou un VPN. À des fins de démonstration, nous allons exposer la Source à l'Internet public en éditant mongod.conf :

```
net:
  bindIp: <PUBLIC IP>
```

Si vous avez plusieurs jeux de réplicas, vous devez exposer tous les nœuds de chacun d'eux, y compris les serveurs de configuration.

## TLS
Nous prenons la sécurité au sérieux chez Percona, et c'est pourquoi, par défaut, notre opérateur déploie des clusters MongoDB avec le cryptage activé. J'ai préparé un [script](https://github.com/spron-in/blog-data/blob/master/mongodb-to-kubernetes/gen_certs.sh) qui génère des certificats et des clés auto-signés avec l'outil openssl. Si vous disposez déjà d'une autorité de certification (CA) dans votre organisation, générez les certificats et signez-les par votre autorité de certification.

La liste des noms alternatifs se trouve soit dans [ce](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/TLS.html) document, soit dans [ce](https://github.com/spron-in/blog-data/blob/master/mongodb-to-kubernetes/mongo.conf) fichier de configuration openssl. Remarque entrée DNS.20 :

```
DNS.20      = *.mongo.spron.in
```

Je vais utiliser cette entrée générique pour configurer la réplication entre les nœuds. Le script génère également un fichier `ssl-secret.yaml` , que nous allons utiliser du côté cible.

Vous devez télécharger l'autorité de certification et le certificat avec une clé privée sur chaque nœud du jeu de réplicas source, puis les définir dans le `mongod.conf`

```
# network interfaces
net:
  ...
  tls:
    mode: preferTLS
    CAFile: /path/to/ca.pem
    certificateKeyFile: /path/to/mongod.pem

security:
  clusterAuthMode: x509
  authorization: enabled
```

Notez que j'ai également défini `clusterAuthMode` sur x509. Il impose l'utilisation de l'authentification x509. Testez-le soigneusement sur un environnement de non-production d'abord, car cela pourrait casser votre réplication existante.

## Créer des utilisateurs système

Notre opérateur a besoin d'utilisateurs du système pour gérer le cluster et effectuer des contrôles de santé. Les noms d'utilisateur et les mots de passe des utilisateurs du système doivent être les mêmes sur la source et la cible. [Ce](https://github.com/spron-in/blog-data/blob/master/mongodb-to-kubernetes/gen_users.sh) script va générer l'`user-secret.yaml` à utiliser sur la cible et le code shell mongo pour ajouter les utilisateurs sur la source (c'est un exemple, ne l'utilisez pas en production).

Connectez-vous au nœud principal sur la source et exécutez les commandes mongo shell générées par le script.

## Préparer la cible

### Appliquer les utilisateurs et les secrets TLS

Les informations d'identification des utilisateurs du système et les certificats TLS doivent être similaires des deux côtés. Les scripts que nous avons utilisés ci-dessus génèrent des manifestes d'objets secrets à utiliser sur la cible. Appliquez-les :

```
$ kubectl apply -f ssl-secret.yaml
$ kubectl apply -f user-secret.yam

```

### Déployer l'opérateur et les nœuds MongoDB

Veuillez suivre l'un des [guides d'installation](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html) pour déployer l'opérateur. Habituellement, il s'agit d'une opération en une seule étape via kubectl :

```
$ kubectl apply -f operator.yaml

```

Les nœuds MongoDB sont déployés avec un manifeste de ressource personnalisé - [cr.yaml](https://github.com/spron-in/blog-data/blob/master/mongodb-to-kubernetes/cr.yaml) . Il contient les éléments de configuration importants suivants :

```
spec:
  unmanaged: true
```

Cet indicateur indique à l'opérateur de déployer les nœuds en mode non géré, ce qui signifie qu'ils ne sont pas configurés pour former le cluster. De plus, l'opérateur ne génère pas de certificats TLS et d'utilisateurs système.

```
spec:
…
  updateStrategy: Never
```

Désactivez la fonctionnalité Smart Update car le cluster n'est pas géré.

```
spec:
…
  secrets:
    users: my-new-cluster-secrets
    ssl: my-custom-ssl
    sslInternal: my-custom-ssl-internal
```

Cette section pointe vers les objets Secret que nous avons créés à l'étape précédente.

```
spec:
…
  replsets:
  - name: rs0
    size: 3
    expose:
      enabled: true
      exposeType: LoadBalancer
```
N'oubliez pas que les nœuds doivent être exposés et accessibles. Pour ce faire, nous créons un service par Pod. Dans notre cas, il s'agit d'un objet `LoadBalancer` , mais il peut s'agir de n'importe quel autre [type Service](https://kubernetes.io/docs/concepts/services-networking/service/).

```
spec:
...
  backup:
    enabled: false
```

Si le cluster et les nœuds ne sont pas gérés, l'opérateur ne doit pas effectuer de sauvegardes. 

Déployez des nœuds non gérés avec la commande suivante :

```
$ kubectl apply -f cr.yaml
```

Une fois les nœuds opérationnels, vérifiez également les services. Nous aurons besoin des adresses IP des nouvelles répliques pour les ajouter ultérieurement à l'ensemble de répliques sur la source.

```
$ kubectl get services
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)           AGE
…
my-new-cluster-rs0-0    LoadBalancer   10.3.252.134   35.223.104.224   27017:31332/TCP   2m11s
my-new-cluster- rs0-1   LoadBalancer   10.3.244.166   34.134.210.223   27017:32647/TCP   81s
my-new-cluster-rs0-2    LoadBalancer   10.3.248.119   34.135.233.58    27017:32534/TCP   45s
```

## Configurer les domaines

L'authentification X509 est stricte et requiert que le nom commun ou le nom alternatif du certificat corresponde au nom de domaine du nœud. Comme vous vous en souvenez, nous avions un joker `*.mongo.spron.in` inclus dans notre certificat. Il peut s'agir de n'importe quel domaine que vous utilisez, mais assurez-vous qu'un certificat est émis pour ce domaine.

Je vais créer des enregistrements A pour pointer vers les adresses IP publiques des nœuds MongoDB :

```
k8s-1.mongo.spron-in -> 35.223.104.224
k8s-2.mongo.spron.in -> 34.134.210.223
k8s-3.mongo.spron-in -> 34.135.233.58
```

## Répliquer les données vers la cible

Il est temps d'ajouter nos nœuds du cluster Kubernetes au jeu de réplicas. Connectez-vous au shell mongo sur la source et exécutez ce qui suit :

```
rs.add({ host: "k8s-1.mongo.spron.in", priority: 0, votes: 0} )
rs.add({ host: "k8s-2.mongo.spron.in", priority: 0, votes: 0} )
rs.add({ host: "k8s-3.mongo.spron.in", priority: 0, votes: 0} )
```

Si tout est fait correctement, ces nœuds seront ajoutés en tant que secondaires. Vous pouvez vérifier l'état avec la commande `rs.status()` .

## Basculement

Vérifiez que le nœud nouvellement ajouté est synchronisé. Plus vous avez de données, plus le processus de synchronisation prendra du temps. Pour comprendre si les nœuds sont synchronisés, vous devez comparer les valeurs de optime et optimeDate du nœud principal avec les valeurs du nœud secondaire dans la sortie `rs.status()`:

```
{
        "_id" : 0,
        "name" : "147.182.213.59:27017",
        "stateStr" : "PRIMARY",
...
        "optime" : {
                "ts" : Timestamp(1633697030, 1),
                "t" : NumberLong(2)
        },
        "optimeDate" : ISODate("2021-10-08T12:43:50Z"),
...
},
{
        "_id" : 1,
        "name" : "k8s-1.mongo.spron.in:27017",
        "stateStr" : "SECONDARY",
...
        "optime" : {
                "ts" : Timestamp(1633697030, 1),
                "t" : NumberLong(2)
        },
        "optimeDurable" : {
                "ts" : Timestamp(1633697030, 1),
                "t" : NumberLong(2)
        },
        "optimeDate" : ISODate("2021-10-08T12:43:50Z"),
...
},
```

Lorsque les nœuds sont synchronisés, nous sommes prêts à effectuer le basculement. Veuillez vous assurer que votre application est correctement configurée pour minimiser les temps d'arrêt pendant le basculement.

Le basculement va comporter deux étapes :

1. L'un des nœuds de la cible devient le nœud principal.
1. L'opérateur commence à gérer le cluster et les nœuds de la source ne sont plus présents dans le jeu de réplicas.

## Changer le primaire

Connectez-vous avec mongo shell au primaire du côté source et créez l'un des nœuds du primaire cible. Cela peut être fait en modifiant la configuration du jeu de réplicas :

```
cfg = rs.config()
cfg.members[1].priority = 2
cfg.members[1].votes = 1
rs.reconfig(cfg)
```

Nous activons le vote et définissons la priorité à deux sur l'un des nœuds du cluster Kubernetes. L'identifiant du membre peut être différent pour vous, veuillez donc examiner attentivement la sortie de la commande `rs.config()` .  

## Commencer à gérer le cluster
Une fois que le primaire s'exécute dans Kubernetes, nous allons dire à l'opérateur de commencer à gérer le cluster. Remplacez `spec.unmanaged` par `false` dans la commande Custom Resource with patch :

```
$ kubectl patch psmdb my-cluster-name --type=merge -p '{"spec":{"unmanaged": true}}'
```

Vous pouvez également le faire en modifiant `cr.yaml` et en l'appliquant. Ça y est, vous avez maintenant le cluster dans Kubernetes qui est géré par l'opérateur.

## Conclusion
Vous commencez vraiment à apprécier les opérateurs une fois que vous vous y habituez. Lorsque j'écrivais cet article de blog, je trouvais extrêmement ennuyeux de déployer et de configurer un seul nœud MongoDB sur une machine Linux ; Je ne veux pas penser au cluster. Les opérateurs résument les primitives Kubernetes et la configuration de la base de données et vous fournissent un service de base de données entièrement opérationnel au lieu d'un groupe de nœuds. La migration de MongoDB vers Kubernetes est une tâche difficile, mais elle est beaucoup plus simple avec Operator. Et une fois que vous êtes sur Kubernetes, Operator s'occupe également de toutes les opérations du jour 2.

Nous vous encourageons à essayer notre opérateur. Consultez notre [référentiel GitHub](https://github.com/percona/percona-server-mongodb-operator/) et consultez la [documentation](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html)

Vous avez trouvé un bug ou vous avez une idée de fonctionnalité ? N'hésitez pas à le soumettre dans [JIRA](https://translate.google.com/translate?hl=en&prev=_t&sl=en&tl=fr&u=https://jira.percona.com/browse/K8SPSMDB)

Pour les questions générales, veuillez soulever le sujet dans le [forum de la communauté](https://forums.percona.com/c/mongodb/percona-kubernetes-operator-for-mongodb/29)

Vous êtes développeur et souhaitez contribuer ? Veuillez lire notre [CONTRIBUTING.md](https://github.com/percona/percona-server-mongodb-operator/blob/main/CONTRIBUTING.md) et envoyer la Pull Request.

## Distribution Percona pour l'opérateur MongoDB

L'[opérateur Percona Distribution for MongoDB](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html) simplifie l'exécution de Percona Server for MongoDB sur Kubernetes et fournit une automatisation pour les opérations du jour 1 et du jour 2. Il est basé sur l'API Kubernetes et permet des environnements hautement disponibles. Quel que soit l'endroit où il est utilisé, l'opérateur crée un membre identique aux autres membres créés avec le même opérateur. Cela offre un niveau de stabilité assuré pour créer facilement des environnements de test ou déployer un environnement de base de données reproductible et cohérent qui répond aux meilleures pratiques recommandées par les experts de Percona.

Source : [blog Percona](https://www.percona.com/blog/migrating-mongodb-to-kubernetes)
