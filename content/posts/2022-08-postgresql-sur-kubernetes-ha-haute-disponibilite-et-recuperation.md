+++
title = "PostgreSQL sur Kubernetes : Haute disponibilité et Recupération"
description = "Nous allons approfondir la haute disponibilité, la reprise après desastre et la mise à l'échelle des clusters PostgreSQL avec un minimum d'effort manuel en utilisant Percona Distribution for PostgreSQL Operator"
author = "Francis"
date = 2022-02-18
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article08.jpg"
images = ["thumbnail2022/article08.jpg"]
slug = "postgresql-sur-kubernetes-ha-haute-disponibilite-et-recuperation"
+++

![thumbnail](/thumbnail2022/article08.jpg)


[Percona Distribution for PostgreSQL Operator ](https://www.percona.com/doc/kubernetes-operator-for-postgresql/index.html) vous permet de déployer et de gérer des clusters PostgreSQL hautement disponibles et de qualité production sur Kubernetes avec un minimum d'effort manuel. Dans cet article de blog, nous allons approfondir la haute disponibilité, la reprise après desastre et la mise à l'échelle des clusters PostgreSQL.

## La haute disponibilité

Notre manifeste de ressources personnalisé par défaut déploie un cluster PostgreSQL hautement disponible (HA). Les composants clés de la configuration HA sont :

- [Services Kubernetes ](https://kubernetes.io/docs/concepts/services-networking/service/) qui pointent vers pgBouncer et les nœuds de réplique
- [pgBouncer ](https://www.pgbouncer.org/) – un pooler de connexion léger pour PostgreSQL
- [Patroni ](https://patroni.readthedocs.io/en/latest/) - Orchestrator HA pour PostgreSQL
- PostgreSQL - nous avons un nœud principal et 2 nœuds de réplica en secours à chaud par défaut

![image01](/posts/2022/article08/img01.png)

Kubernetes est le moyen d'exposer votre cluster PostgreSQL aux applications ou aux utilisateurs. Nous avons deux services :

- `clusterName-pgbouncer` – Exposition de votre cluster PostgreSQL via le pooler de connexion pgBouncer. Les lectures et les écritures sont envoyées au nœud principal.
- `clusterName-replica` – Expose directement les nœuds de réplique. Il ne doit être utilisé que pour les lectures. N'oubliez pas non plus que les connexions à ce service ne sont pas regroupées. Nous travaillons sur une meilleure solution, où l'utilisateur pourrait tirer parti à la fois de la mise en commun des connexions et de la mise à l'échelle de la lecture via un seul service.

Par défaut, nous utilisons le type de service ClusterIP , mais vous pouvez le modifier respectivement dans `pgBouncer.expose.serviceType` ou `pgReplicas.hotStandby.expose.serviceType`.

Chaque conteneur PostgreSQL exécute Patroni qui surveille l'état du cluster et, en cas de défaillance du nœud principal, transfère le rôle du nœud principal à l'un des nœuds de réplication. PgBouncer sait toujours où se trouve Primary.

Comme vous le voyez, nous distribuons les composants de cluster PostgreSQL sur différents nœuds Kubernetes . Cela se fait avec des règles d'[affinité ](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)et elles sont appliquées par défaut pour garantir qu'une panne de nœud unique n'entraîne pas d'indisponibilité de la base de données.

## Multi-Datacenter avec Multi-AZ

Bonne conception d'architecture consiste à exécuter votre cluster Kubernetes sur plusieurs centres de données. Les clouds publics ont un concept de zones de disponibilité (AZ) qui sont des centres de données dans une région avec une connexion réseau à faible latence entre eux. Habituellement, ces centres de données sont distants d'au moins 100 kilomètres les uns des autres afin de minimiser la probabilité d'une panne régionale. Vous pouvez tirer parti du déploiement Kubernetes multi-AZ pour exécuter des composants de cluster dans différents centres de données pour une meilleure disponibilité.

![image02](/posts/2022/article08/img02.png)

Pour vous assurer que les composants PostgreSQL sont distribués dans les zones de disponibilité, vous devez modifier les règles d'affinité. Désormais, cela n'est possible qu'en modifiant directement les ressources de déploiement :

```
$ kubectl edit deploy cluster1-repl2
…
-            topologyKey: kubernetes.io/hostname
+            topologyKey: topology.kubernetes.io/zone
```

## Mise à l'échelle

La mise à l’échelle de PostgreSQL pour répondre à la demande aux heures de pointe est cruciale pour la haute disponibilité. Notre opérateur vous fournit des outils pour mettre à l'échelle les composants PostgreSQL à la fois horizontalement et verticalement.

### Mise à l'échelle verticale

La mise à l'échelle verticale consiste à ajouter plus de puissance à un nœud PostgreSQL . La méthode recommandée consiste à modifier les ressources dans la ressource personnalisée (au lieu de les modifier directement dans les objets de déploiement). Par exemple, modifiez les éléments suivants dans le `cr.yaml` pour obtenir 256 Mo de RAM pour tous les nœuds PostgreSQL Replica :

```
  pgReplicas:
    hotStandby:
      resources:
        requests:
-         memory: "128Mi"
+         memory: "256Mi"
```


Appliquer `cr.yaml` :

```
$ kubectl apply -f cr.yaml
```


Utilisez la même approche pour régler d'autres composants dans leurs sections correspondantes.

Vous pouvez également tirer parti [de Vertical Pod Autoscaler ](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)(VPA) pour réagir automatiquement aux pics de charge. Nous créons une ressource de déploiement pour le nœud principal et chaque nœud de réplication. Les objets VPA doivent cibler ces déploiements. L'exemple suivant suivra l'une des ressources de déploiement de répliques du cluster1 et la mettra automatiquement à l'échelle :

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: pxc-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:     cluster1-repl1  
    namespace:  pgo
  updatePolicy:
    updateMode: "Auto"
```


Veuillez lire davantage sur VPA et ses fonctionnalités dans la documentation.

### Mise à l'échelle horizontale

L'ajout de nœuds de réplique ou de pgBouncers supplémentaires peut être effectué en modifiant les paramètres de taille dans la ressource personnalisée. Faites le changement suivant dans le `cr.yaml` par défaut :

```
  pgReplicas:
    hotStandby:
-      size: 2
+      size: 3
```


Appliquez la modification pour obtenir un nœud de réplication PostgreSQL supplémentaire :
```
$ kubectl apply -f cr.yaml
```
À partir de la version 1.1.0, il est également possible de mettre à l'échelle notre cluster à l'aide de la commande kubectl scale. Exécutez ce qui suit pour avoir deux nœuds de réplique PostgreSQL dans le cluster 1 :
```
$ kubectl scale --replicas=2 perconapgcluster/cluster1
perconapgcluster.pg.percona.com/cluster1 scaled
```

Dans la dernière version, il n'est pas encore possible d'utiliser [Horizontal Pod Autoscaler ](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)(HPA) et nous le prendrons en charge dans la prochaine. Suivez-nous!

## Reprise après désastre

Il est important de comprendre que la reprise après desastre (DR) n'est pas une haute disponibilité. L'objectif de DR est d'assurer la continuité des activités en cas de catastrophe massive, telle qu'une panne de toute la région. La récupération dans de tels cas peut bien sûr être automatisée, mais pas nécessairement - cela dépend strictement des besoins de l'entreprise.

![image03](/posts/2022/article08/img03.png)

## Sauvegarde et restauration

Je pense que c'est le protocole de récupération après desastre le plus courant - prenez la sauvegarde, stockez-la dans des locaux tiers, restaurez-la dans un autre centre de données si nécessaire.

Cette approche est simple, mais s'accompagne d'un long temps de récupération, surtout si la base de données est volumineuse. Utilisez cette méthode uniquement si elle dépasse vos objectifs de temps de récupération (RTO).

![image04](/posts/2022/article08/img04.png)

Notre opérateur gère la sauvegarde et la restauration des clusters PostgreSQL. La reprise après desastre est construite autour de pgBackrest et ressemble à ceci :

1. Configurez pgBackrest pour télécharger des sauvegardes sur S3 ou GCS (voir notre [documentation ](https://www.percona.com/doc/kubernetes-operator-for-postgresql/backups.html) pour plus de détails).
1. Créez la sauvegarde manuellement ( [via pgTask ](https://www.percona.com/doc/kubernetes-operator-for-postgresql/backups.html#making-on-demand-backup) ou assurez-vous qu'une sauvegarde planifiée a été créée.
1. Une fois le cluster principal défaillant, créez le nouveau cluster dans le centre de données de reprise après desastre. Le cluster doit être exécuté en mode veille et pgBackrest doit pointer vers le même référentiel que le cluster principal :

```
spec:
  standby: true
  backup:
  # same config as on original cluster
```

Une fois les données récupérées, l'utilisateur peut désactiver le mode veille et basculer l'application vers le cluster DR.

## Restauration continue

Cette approche est assez similaire à la précédente : les instances de pgBackrest synchronisent en permanence les données entre deux clusters via le stockage d'objets. Cette approche minimise le RTO et vous permet de basculer le trafic de l'application vers le site DR presque immédiatement.

![image05](/posts/2022/article08/img05.png)

La configuration ici est similaire au cas précédent, mais nous exécutons toujours un deuxième cluster PostgreSQL dans le centre de données Disaster Recovery. En cas de panne du site principal, désactivez simplement le mode veille :
```
spec:
  standby: false
```
Vous pouvez utiliser une configuration similaire pour migrer les données vers et depuis Kubernetes . En savoir plus à ce sujet dans l' article de blog [Migrating PostgreSQL to Kubernetes ](https://www.percona.com/blog/migrating-postgresql-to-kubernetes).

## Conclusion

Kubernetes fournissent un service prêt à l'emploi et, dans le cas de Percona Distribution for PostgreSQL Operator, l'utilisateur obtient un cluster de bases de données hautement disponible de qualité production. De plus, l'opérateur fournit des capacités d'exploitation au jour 2 et automatise la routine quotidienne.

Nous vous encourageons à essayer notre opérateur. Consultez notre [référentiel GitHub](https://github.com/percona/percona-postgresql-operator/) et consultez la [documentation ](https://www.percona.com/doc/kubernetes-operator-for-postgresql/index.html).

Vous avez trouvé un bogue ou avez une idée de fonctionnalité ? N'hésitez pas à le soumettre dans [JIRA ](https://jira.percona.com/browse/K8SPG).

Pour les questions d'ordre général, veuillez soulever le sujet dans le [forum de la communauté ](https://forums.percona.com/c/postgresql/percona-kubernetes-operator-for-postgresql/68).

Vous êtes développeur et souhaitez contribuer ? Veuillez lire notre [CONTRIBUTING.md ](https://github.com/percona/percona-postgresql-operator/blob/main/CONTRIBUTING.md)et envoyer le Pull.

Source: [Percona](https://www.percona.com/blog/postgresql-high-availability-and-disaster-recovery-on-kubernetes)
