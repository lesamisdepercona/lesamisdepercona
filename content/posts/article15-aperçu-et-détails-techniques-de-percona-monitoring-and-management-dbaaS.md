+++
title = "Aperçu et détails techniques de Percona Monitoring and Management DBaaS"
description = "Traduit à partir de l'article de Denys Kondratenko intitulé, Percona Monitoring and Management DBaaS Overview and Technical Details"
author = "Francis"
date = 2021-09-10T11:43:01+04:00
tags = ['PMM']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article13.jpg"
images = ["thumbnail/article15-aperçu-et-details-techniques-pmm-dbaaS.jpg"]
slug = "article15-aperçu-et-détails-techniques-de-percona-monitoring-and-management-dbaaS"
+++

Database-as-a-Service ( DBaaS ) est une gestion de base de donnée qui n'a pas besoin d'être installée et maintenue, mais qui est plutôt fournie en tant que service à l'utilisateur. Le composant de [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management) (PMM) DBaaS permet aux utilisateurs de CRUD (Create, Read, Update, Delete) [Percona XtraDB Cluster](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html) (PXC) et [Percona Server pour MongoDB](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html) (PSMDB) de gérer les bases de données dans Kubernetes clusters.

PXC et PSMDB implémentent DBaaS au-dessus de Kubernetes (k8s), et PMM DBaaS fournit une interface et une API agréables pour les gérer.

Déployer Playground avec minikube

Le moyen le plus simple de jouer et de tester PMM DBaaS est d'utiliser minikube . Veuillez suivre  [les guides d'installation de](https://minikube.sigs.k8s.io/docs/start/) minikube . Il est possible que votre distribution de système d'exploitation fournisse des packages natifs pour cela, alors vérifiez également cela avec votre gestionnaire de packages.

Dans les exemples ci-dessous, Linux est utilisé avec le [pilote kvm2](https://minikube.sigs.k8s.io/docs/drivers/kvm2/)  , donc en plus kvm et libvirt doivent être installés. D'autres systèmes d'exploitation et pilotes peuvent également être utilisés. Installez également l'outil kubectl , il serait plus pratique de l'utiliser et minikube configurera kubeconfig afin que le cluster k8s puisse être facilement accessible depuis l'hôte.

Créons un cluster k8s et ajustons les ressources selon les besoins. Les exigences minimales peuvent être trouvées dans la [documentation](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/System-Requirements.html) .

- Démarrer le cluster minikube
```
$ minikube start --cpus 12 --memory 32G --driver=kvm2
```

- Téléchargez le déploiement du serveur PMM pour minikube et déployez-le dans le cluster k8s
```
$ curl -sSf -m 30 https://raw.githubusercontent.com/percona-platform/dbaas-controller/main/deploy/pmm-server-minikube.yaml \
| kubectl apply -f -
```

- Pour la première fois, le serveur PMM peut mettre un certain temps à initialiser le volume, mais il finira par démarrer
- Voici comment vérifier que le déploiement du serveur PMM est en cours d'exécution :
```
$ kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
pmm-deployment   1/1     1            1           3m40s

$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
pmm-deployment-d688fb846-mtc62   1/1     Running   0          3m42s

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON   AGE
pmm-data                                   10Gi       RWO            Retain           Available                                              3m44s
pvc-cb3a0a18-b6dd-4b2e-92a5-dfc0bc79d880   10Gi       RWO            Delete           Bound       default/pmm-data   standard                3m44s

$ kubectl get pvc
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pmm-data   Bound    pvc-cb3a0a18-b6dd-4b2e-92a5-dfc0bc79d880   10Gi       RWO            standard       3m45s

$ kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP                      6m10s
pmm          NodePort    10.102.228.150   <none>        80:30080/TCP,443:30443/TCP   3m5
```


- Exposez les ports du serveur PMM sur l'hôte, car cela ouvre également des liens vers l'interface utilisateur PMM ainsi que vers le point de terminaison de l'API dans le navigateur par défaut.

```
$ minikube service pmm
```

*REMARQUE* :

Pour ne pas trop entrer dans les détails : PV ( kubectl get pv ) et PVC( kubectl get pvc ) sont essentiellement le stockage des données PMM ( répertoire /srv ). Service est un réseau pour PMM.

***Attention :*** ce déploiement de serveur PMM n'est pas censé être utilisé en production, mais simplement comme un bac à sable pour tester et jouer, car il commence toujours avec la dernière version de PMM et k8s n'est pas encore un environnement pris en charge.

Configurer PMM DBaaS

Désormais, le [tableau de bord PMM DBaaS](https://www.percona.com/doc/percona-monitoring-and-management/2.x/using/platform/dbaas.html) peut être utilisé et un cluster k8s peut être ajouté, DB ajoutée et configurée.

![image01](/posts/article15/img01.png)



*REMARQUE:*

Pour activer la fonctionnalité PMM DBaaS, vous devez soit transmettre un environnement spécial ( [ENABLE_DBAAS=1](https://github.com/percona-platform/dbaas-controller/blob/main/deploy/pmm-server-minikube.yaml#L73) au conteneur, soit l'activer dans les paramètres (capture d'écran suivante).

Pour permettre à PMM de gérer le cluster k8s, il doit être configuré. Consultez la [documentation](https://www.percona.com/doc/percona-monitoring-and-management/2.x/setting-up/server/dbaas.html) , mais voici quelques étapes courtes 

- Définir Public Address adresse à PMM sur la page Configuration -> Settings -> Advanced Settings

![image02](/posts/article15/img02.png)

- Obtenez la configuration k8s ( kubeconfig ) et copiez-la pour l'enregistrement :
```
kubectl config view --flatten --minify

```

- Enregistrez la configuration qui a été copiée sur le tableau de bord du cluster DBaaS Kubernetes : 


![image03](/posts/article15/img03.png)

Entrons dans les détails sur ce que tout cela signifie.

L'adresse publique est propagée aux conteneurs pmm -client qui sont exécutés dans le cadre des déploiements PXC et PSMDB pour surveiller les services de base de données. Les conteneurs pmm -client exécutent pmm -agent, qui doit se connecter au serveur PMM. Il utilise l'adresse publique. Le nom DNS pmm est défini par Service dans le fichier pmm -server- minikube.yaml pour notre déploiement de serveur PMM.

Jusqu'à présent, PMM DBaaS utilise [kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#merging-kubeconfig-files) pour accéder à l'API k8s afin de pouvoir gérer les opérateurs PXC et PSMDB. Le fichier kubeconfig et les informations du cluster k8s sont [stockés ](https://github.com/percona/pmm-managed/blob/PMM-2.0/models/kubernetes_helpers.go#L110) en toute sécurité dans la base de données interne du serveur PMM.

PMM DBaaS n'a pas pu déployer les opérateurs dans le cluster k8s pour le moment, mais cette [fonctionnalité](https://jira.percona.com/browse/PMM-7915) sera implémentée très prochainement. Et c'est pourquoi le statut de l'opérateur sur le tableau de bord du cluster Kubernetes affiche des conseils sur la façon de les installer.

Quels sont les opérateurs et pourquoi sont-ils nécessaires ? Ceci est très bien défini dans la [documentation](https://www.percona.com/software/percona-kubernetes-operators) . Pour faire court, ils sont au cœur de DBaaS qui déploient et configurent des bases de données à l'intérieur du cluster k8s.

Les opérateurs eux-mêmes sont des logiciels complexes qui doivent être correctement démarrés et configurés pour déployer des bases de données. C'est là que PMM DBaaS est utile, pour configurer beaucoup de choses pour l'utilisateur final et fournir une interface utilisateur pour choisir quelle base de données doit être créée, configurée ou supprimée.

Déployer PSMDB avec DBaaS

Allons déployer l' [opérateur PSMDB](https://www.percona.com/doc/percona-monitoring-and-management/2.x/setting-up/server/dbaas.html#install-percona-operators-in-minikube) et BDs étape par étape pour les vérifier en détail.

- Déployer l’opérateur PSMDB

```
curl -sSf -m 30 https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v1.7.0/deploy/bundle.yaml \
| kubectl apply -f -
```


- Voici comment vérifier que l'opérateur a été créé :

```
$ kubectl get deployment
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
percona-server-mongodb-operator   1/1     1            1           46h
pmm-deployment                    1/1     1            1           24h


$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
percona-server-mongodb-operator-586b769b44-hr7mg   1/1     Running   2          46h
pmm-deployment-7fcb579576-hwf76                    1/1     Running   1          24h
```

Maintenant, on voit sur le tableau de bord du cluster PMM DBaaS Kubernetes que l' opérateur MongoDB est installé. 

![image04](/posts/article15/img04.png)

API PMM

Toutes les REST APIs peuvent être découvertes via [Swagger](https://www.percona.com/doc/percona-monitoring-and-management/2.x/details/api.html) ; il est exposé sur les deux ports (30080 et 30443 dans le cas de minikube ) et accessible en ajoutant /swagger à l'adresse du serveur PMM. Il est recommandé d'utiliser https (30443 port ), et par exemple, l'URL pourrait ressembler à ceci : https://192.168.39.202:30443/swagger

Comme DBaaS est une fonctionnalité en cours de développement, remplacez /swagger.json par /swagger-dev.json et appuyez sur le bouton Explorer

![image05](/posts/article15/img05.png)

Désormais, toutes les API peuvent être vues et même exécutées.

Essayons-le! D’abord Authorize, puis recherchez /v1/management/DBaaS/Kubernetes/List et appuyez sur Try it out et Execute. Il y aura un exemple de curl ainsi qu'une réponse à la requête REST API POST. L'exemple curl peut également être utilisé à partir de la ligne de commande :

```
$ curl -kX POST "https://192.168.39.202:30443/v1/management/DBaaS/Kubernetes/List" -H  "accept: application/json" -H  "authorization: Basic YWRtaW46YWRtaW4=" -H  "Content-Type: application/json" -d "{}"
{
  "kubernetes_clusters": [
    {
      "kubernetes_cluster_name": "minikube",
      "operators": {
        "xtradb": {
          "status": "OPERATORS_STATUS_NOT_INSTALLED"
        },
        "psmdb": {
          "status": "OPERATORS_STATUS_OK"
        }
      },
      "status": "KUBERNETES_CLUSTER_STATUS_OK"
    }
  ]
}
```

![image06](/posts/article15/img06.png)

Créer DB et une étude approfondie

PMM Server se compose de différents composants, et pour la fonctionnalité DBaaS, voici les principaux :

- Grafana UI avec les tableaux de bord DBaaS communique avec pmm-managed via REST API pour afficher l'état actuel et fournir une interface utilisateur
- pmm -managed agit comme une passerelle REST et détient kubeconfig et communique avec dbaas -controller via gRPC
- dbaas-controller met en œuvre DBaaS caractéristiques, communique avec K8s, et expose gRPC interface pour pmm-managed

Grafana UI est ce que les utilisateurs voient, et maintenant, lorsque les opérateurs sont installés, l'utilisateur peut créer l'instance MongoDB. Procédons comme suit .

- Accédez à la page DBaaS -> DB Cluster et appuyez sur le lien Create DB Cluster
- Choisissez vos options et appuyez sur le bouton Create Cluster

![image07](/posts/article15/img07.png)

Il dispose d'options plus avancées pour configurer les ressources allouées au cluster :

![image08](/posts/article15/img08.png)

Comme on le voit, le cluster a été créé et peut être manipulé. Voyons maintenant en détail ce qui s'est passé en dessous.

![image09](/posts/article15/img09.png)

Lorsque l'utilisateur appuie sur le bouton Create Cluster, Grafana UI POSTS /v1/management/DBaaS/PSMDBCluster/Create request to pmm-managed. pmm-managed reçoit la demande et l’envoie via gRPC aux dbaas-Controller avec kubeconfig.

dbaas -controller reçoit les requêtes, et avec la connaissance de la structure de l'opérateur (Custom Resources/CRD), il prépare [CR](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/operator.html) avec tous les paramètres nécessaires pour créer un MongoDB cluster. Après avoir rempli toutes [les structures nécessaires](https://github.com/percona-platform/dbaas-controller/blob/main/service/k8sclient/k8sclient.go#L927) , dbaas -controller convertit CR en fichier yaml et l'applique avec la commande kubectl apply -f . kubectl est préconfiguré avec le fichier *kubeconf* (qui a été transmis par pmm -managed à partir de sa DB) pour communiquer avec le cluster approprié, et le fichier *kubeconf* est temporairement créé et supprimé immédiatement après la requête.

La même chose se produit lorsque certains paramètres changent ou que dbaas-controller obtient des paramètres du cluster k8s.

Essentiellement, le dbaas-controller automatise toutes les étapes pour remplir les CR avec les paramètres corrects, vérifier que tout fonctionne correctement et renvoie des détails sur les clusters créés. L'interface kubectl est utilisée pour plus de simplicité mais elle est sujette AU changement avant GA, très probablement en fonction de l'[API k8s Go](https://github.com/kubernetes/client-go) .   

Resumé

Dans l'ensemble, PMM Server DBaaS offre une expérience transparente à l'utilisateur pour déployer  DB clusters sur Kubernetes avec une interface utilisateur simple et agréable sans avoir besoin de connaître les éléments internes des opérateurs. En déployant les clusters PXC et PSMDB, il configure également les agents et les exportateurs PMM, ainsi toutes les données de surveillance sont immédiatement présentes dans le serveur PMM.

![image10](/posts/article15/img10.png)

Accédez au PMM Dashboard -> MongoDB -> MongoDB Overview et consultez également les données de surveillance MongoDB, les nœuds d'exploration et la surveillance des services, qui sont préconfigurés à l'aide de la fonctionnalité DBaaS.

Essayez-le, soumettez vos commentaires et [discutez avec nous](https://discord.gg/JDedgsgM) , nous serions heureux de vous entendre!

PS

N'oubliez pas d'arrêter et/ou de supprimer votre cluster minikube s'il n'est pas utilisé :

- Arrêter le cluster minikube , pour ne pas utiliser de ressources (pourrait être démarré avec start)

```
$ minikube stop
```

- Si un cluster n'est plus nécessaire, supprimez le cluster minikube
```
$ minikube delete
```

Source : [Percona Blog](https://www.percona.com/blog/2021/05/19/percona-monitoring-and-management-dbaas-overview-and-technical-details/)
