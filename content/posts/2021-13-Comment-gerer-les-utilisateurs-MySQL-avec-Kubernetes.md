+++
title = "Gérer les utilisateurs MySQL avec Kubernetes"
description = "Traduit à partir de l'article de Sergey Pronin intitulé, Manage MySQL Users with Kubernetes"
author = "Francis"
date = 2021-08-26T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article13.jpg"
images = ["thumbnail/article13.jpg"]
slug = "gerer-les-utilisateurs-mysql-avec-Kubernetes"
+++


Une demande assez courante que nous recevons de la communauté et des clients est de fournir un moyen de gérer les utilisateurs de la base de données avec des opérateurs - à la fois MongoDB et MySQL. Même si nous le voyons comme une tâche intéressante, nos opérateurs sont principalement un outil pour simplifier le déploiement et la gestion de nos logiciels sur Kubernetes. Notre objectif est de fournir un cluster de bases de données prêt à héberger des applications critiques et déployé avec les meilleures pratiques.

## Pourquoi gérer les utilisateurs avec des opérateurs ?
Il y a peu de cas d'utilisation :

1. Simplifiez le pipeline CICD. Il est plus simple d'appliquer un seul manifeste que d'exécuter plusieurs commandes pour créer l'utilisateur une fois la base de données prête.
1. Effectuer le contrôle des utilisateurs de la base de données aux développeurs ou aux applications, mais sans fournir un accès direct à la base de données.
1. Il existe une opinion selon laquelle Kubernetes passera d'un orchestrateur de conteneurs à un plan de contrôle pour tout gérer. C’est une stratégie pour certaines entreprises.

Nous voulons avoir la fonctionnalité permettant de fournir les utilisateurs des opérateurs. Bien que faisable, il semble que de le faire séparément pour chaque opérateur n’est pas la bonne solution. Il pourrait être unifié.

Et si nous passions à un autre niveau et nous créions un moyen de provisionner des utilisateurs sur n'importe quelle base de données via le plan de contrôle Kubernetes? L'utilisateur a créé l'instance MySQL sur un cloud public via le plan de contrôle, alors pourquoi ne pas créer l'utilisateur DB de la même manière ?

[Crossplane.io](http://crossplane.io/) - Un module complémentaire Kubernetes qui permet aux utilisateurs de décrire et de provisionner de manière déclarative l'infrastructure via le plan de contrôle k8s. De par sa conception, il est extensible via des « fournisseurs ». L'un d'eux - [provider- sql](https://github.com/crossplane-contrib/provider-sql/) - permet à la fonctionnalité de gérer les utilisateurs MySQL et PostgreSQL (et même les bases de données) via les CRD. Voyons comment le faire fonctionner avec [Percona XtraDB Cluster Operator](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html)

## Action
Pré - requis :

- cluster Kubernetes
- Percona XtraDB Cluster déployé avec Operator (voir la documentation [ici](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html)

L'objectif est de créer un objet de ressource personnalisée (CR) via l'API Kubernetes pour déclencher crossplane.io afin de créer l'utilisateur sur le cluster PXC. En résumé, cela ressemble comme suit : 

 ![image01](/posts/article13/crossplane-provider-sql-1024x489.png)

1. Un utilisateur crée le CR avec utilisateur souhaité et rôle
1. Crossplane le detecte
1. provider-sql (Crossplane provider) est configuré via un objet secret qui a le point de terminaison PXC et les informations d'identification racine
1. provider-sql se connecte à PXC et crée l'utilisateur

J'ai placé tous les fichiers de cet article de blog dans le référentiel github public avec le runbook condensé qui répertorie simplement toutes les étapes. Vous pouvez tout trouver [ici](https://github.com/spron-in/blog-data/tree/master/crossplane-provider-sql) . 

## **Installer le crossplane**
Le moyen le plus simple est de le faire via helm:
```
kubectl create namespace crossplane
helm repo add crossplane-alpha https://charts.crossplane.io/alpha
helm install crossplane --namespace crossplane crossplane-alpha/crossplane
```
D'autres méthodes d'installation peuvent être trouvées dans la [documentation crossplane.io](https://crossplane.io/docs/v0.1/install-crossplane.html) . 

## **Installer le fournisseur - sql**
[Le Provider dans Crossplane](https://crossplane.io/docs/v0.12/introduction/providers.html) est un concept similaire au [Provider de Terraform](https://www.terraform.io/docs/providers/index.html). Tout dans le crossplane se fait via l' API Kubernetes et cela inclut l'installation des fournisseurs.

```
$ cat crossplane-provider-sql.yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Provider
metadata:
  name: provider-sql
spec:
  package: "crossplane/provider-sql:master"

$ kubectl apply -f crossplane-provider-sql.yaml
```

Cela va installer des définitions de ressources personnalisées pour provider-sql qui va être utilisé pour gérer les utilisateurs MySQL sur notre cluster PXC. Documentation complète pour provider-sql peut être trouvée [ici](https://github.com/crossplane-contrib/provider-sql) , mais ils ne sont pas très détaillés.

## **Nous y sommes presque**
Tout est installé et nécessite une dernière touche de configuration.

1. Créez un secret que provider-sql va utiliser pour se connecter à la base de données MySQL. J'ai le cluster1 Percona XtraDB cluster déployé dans un espace de noms pxc et le secret correspondant ressemblera à ceci : 
```
$ cat crossplane-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: crossplane-secret
  namespace: pxc
stringData:
  username: root
  password: <root password>
  endpoint: cluster1-haproxy.pxc.svc.cluster.local
  port: "3306"

type: Opaque
```
Vous pouvez obtenir le mot de passe root à partir du secret qui est créé lorsque PXC est déployé via l'opérateur. Un moyen rapide de l'obtenir est comme ceci:
```
$ kubectl get secret -n pxc my-cluster-secrets -o yaml | awk '/root:/ {print $2}' | base64 --decode && echo
<root password>
```

Crossplane utilisera le point de terminaison et le port comme chaîne de connexion MySQL, ainsi que le nom d'utilisateur et le mot de passe pour s'y connecter. Configurez provider-sql pour obtenir les informations à partir du secret:
```
$ cat crossplane-mysql-config.yaml
apiVersion: mysql.sql.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: cluster1-pxc
spec:
  credentials:
    source: MySQLConnectionSecret
    connectionSecretRef:
      namespace: pxc
      name: crossplane-secret

$ kubectl apply -f crossplane-mysql-config.yaml
```
Vérifions que la configuration est en place :
```
$ kubectl get providerconfig.mysql.sql.crossplane.io
NAME           AGE
cluster1-pxc   14s
```

## **Faisons-le!**
Tout est prêt. Crossplane peut maintenant se connecter à la base de données et créer les utilisateurs. Du point de vue de Kubernetes et de l'utilisateur, il s'agit simplement de créer les ressources personnalisées via l'API du plan de contrôle.

### **Création de base de données** 
```
$ cat crossplane-db.yaml
apiVersion: mysql.sql.crossplane.io/v1alpha1
kind: Database
metadata:
  name: my-db
spec:
  providerConfigRef:
    name: cluster1-pxc

$ kubectl apply -f crossplane-db.yaml
database.mysql.sql.crossplane.io/my-db created

$ kubectl get database.mysql.sql.crossplane.io
NAME    READY   SYNCED   AGE
my-db   True    True     14s
```
Cela a créé la base de données sur mon Percona XtraDB cluster: 
```
$ mysql -u root -p -h cluster1-haproxy
Server version: 8.0.22-13.1 Percona XtraDB Cluster (GPL), Release rel13, Revision a48e6d5, WSREP version 26.4.3
...
mysql> show databases like 'my-db';
+------------------+
| Database (my-db) |
+------------------+
| my-db            |
+------------------+
1 row in set (0.01 sec)
```
La base de données peut également être supprimée via l'API Kubernetes - supprimez simplement l'objet correspondant `database.mysql.sql.crossplane.io.` 

### **Création d'utilisateur**
L'utilisateur a besoin d'un mot de passe. Le mot de passe ne doit jamais être stocké sous forme de texte brut, alors mettons-le dans un secret :
```
$ cat user-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-user-secret
stringData:
  password: mysuperpass
type: Opaque
```
Nous pouvons créer l'utilisateur maintenant :
```
$ cat crossplane-user.yaml
apiVersion: mysql.sql.crossplane.io/v1alpha1
kind: User
metadata:
  name: my-user
spec:
  providerConfigRef:
    name: cluster1-pxc
  forProvider:
    passwordSecretRef:
      name: my-user-secret
      namespace: default
      key: password
  writeConnectionSecretToRef:
    name: connection-secret
    namespace: default

$ kubectl apply -f crossplane-user.yaml
user.mysql.sql.crossplane.io/my-user created

$ kubectl get user.mysql.sql.crossplane.io
NAME      READY   SYNCED   AGE
my-user   True    True     11s
```
Et ajoutez quelques «grant » :
```
$ cat crossplane-grants.yaml
apiVersion: mysql.sql.crossplane.io/v1alpha1
kind: Grant
metadata:
  name: my-grant
spec:
  providerConfigRef:
    name: cluster1-pxc
  forProvider:
    privileges:
      - DROP
      - CREATE ROUTINE
      - EVENT
    userRef:
      name: my-user
    databaseRef:
      name: my-db

$ kubectl apply -f crossplane-grants.yaml
grant.mysql.sql.crossplane.io/my-grant created

$ kubectl get grant.mysql.sql.crossplane.io
NAME       READY   SYNCED   AGE   ROLE      DATABASE   PRIVILEGES
my-grant   True    True     7s    my-user   my-db      [DROP CREATE ROUTINE EVENT]
```
Vérifiez que l'utilisateur est là :
```
mysql> show grants for 'my-user';
+-----------------------------------------------------------------+
| Grants for my-user@%                                            |
+-----------------------------------------------------------------+
| GRANT USAGE ON *.* TO `my-user`@`%`                             |
| GRANT DROP, CREATE ROUTINE, EVENT ON `my-db`.* TO `my-user`@`%` |
+-----------------------------------------------------------------+
2 rows in set (0.00 sec)
```

## **Garder l'État**
Kubernetes est déclaratif et ses contrôleurs font toujours de leur mieux pour synchroniser la configuration déclarée et l’état réel. Cela signifie que si vous supprimez l'utilisateur manuellement de la base de données (pas via l'API Kubernetes), lors du prochain passage d'une boucle de réconciliation, Crossplane synchronisera l'état et recréera l'utilisateur et les « grant » à nouveau.

### **Conclusion**
Certaines fonctionnalités d'un moteur de base de données diffèrent beaucoup de l'autre, mais il existe parfois un modèle. La création d'utilisateurs est l'un de ces modèles qui peuvent être unifiés sur plusieurs moteurs de base de données. Heureusement, le paysage de la Cloud Native Foundation est immense et se compose de nombreux blocs de construction qui, lorsqu'ils sont utilisés ensemble, peuvent fournir de merveilleuses infrastructures ou applications.

Cet article montre que la communauté a peut-être déjà trouvé une meilleure solution au problème et que la réinventer pourrait être une perte de temps.

L'extension des fournisseurs crossplane.io pour prendre en charge d'autres moteurs de base de données (comme MongoDB ) est un défi mais peut être résolu. Nous rédigeons une proposition et travaillerons avec nos équipes et notre communauté pour y parvenir.

Source : [Percona Blog](https://www.percona.com/blog/2021/05/20/manage-mysql-users-with-kubernetes/)
