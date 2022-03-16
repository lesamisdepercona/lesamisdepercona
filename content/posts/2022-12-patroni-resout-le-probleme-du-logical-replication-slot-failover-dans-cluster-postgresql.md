+++
title = "Comment Patroni résout le problème du Logical Replication Slot Failover dans un cluster PostgreSQL"
description = "Le basculement de l'emplacement de réplication logique a toujours été le problème lors de l'utilisation de la réplication logique dans PostgreSQL. La bonne nouvelle est que les développeurs et les mainteneurs de Patroni ont résolu ce problème à partir de la version 2.1.0"
author = "Francis"
date = 2022-03-15
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail2022/article12.jpg"
images = ["thumbnail2022/article12.jpg"]
slug = "patroni-resout-le-probleme-du-logical-replication-slot-failover-dans-cluster-postgresql"
+++

![thumbnail](/thumbnail2022/article12.jpg)


Le basculement de l'**emplacement de réplication logique,** ou en anglais Logical Replication Slot Failover, a toujours été le problème lors de l'utilisation de la réplication logique dans PostgreSQL. Ce manque de fonctionnalité a sapé l'utilisation de la réplication logique et a agi comme l'un des plus grands moyens de dissuasion. L'enjeu et l'impact étaient si importants que de nombreuses organisations ont dû abandonner leurs projets de réplication logique, et cela a affecté de nombreux projets de migration vers PostgreSQL . Il était pénible de voir que beaucoup devaient opter pour des solutions propriétaires/spécifiques à un fournisseur à la place.

Chez Percona, nous avons écrit à ce sujet dans le passé : [Pièce manquante : Basculement de l'emplacement de réplication logique ](https://www.percona.com/blog/2020/05/21/failover-of-logical-replication-slots-in-postgresql/). Dans cet article, nous avons discuté de l'une des approches possibles pour résoudre ce problème, mais il n'existait aucun mécanisme fiable pour copier les informations d'emplacement sur une réserve physique et les maintenir.

Le problème, en résumé, est le suivant : le slot de réplication sera toujours maintenu sur le nœud principal. S'il y a un basculement pour promouvoir l'un des nœuds de secours, le nouveau nœud principal n'aura aucune idée de l'emplacement de réplication maintenu par le nœud principal précédent. Cela interrompt la réplication logique à partir des systèmes en aval ou si un nouvel emplacement est créé, il devient dangereux à utiliser.

La bonne nouvelle est que les développeurs et les mainteneurs de Patroni ont résolu ce problème à partir de la version 2.1.0 et ont fourni une solution de travail **sans aucune méthode/extension invasive**. C'est un travail qui mérite une grande salve d'applaudissements de la part de la communauté Patroni et c'est l'intention de cet article afin de s'assurer qu'une plus grande foule en soit consciente.

## Comment le configurer

Patroni prêt à l'emploi est disponible dans le [référentiel Percona](https://www.percona.com/software/postgresql-distribution). Mais vous êtes libre d'utiliser Patroni à partir de n'importe quelle source.

### Configuration de base

Si cela vous intéresse et que vous souhaitez l'essayer, les étapes suivantes peuvent être utiles.

Toute la discussion porte sur la réplication logique. Ainsi, l'exigence minimale est d'avoir un wal_level défini sur "logique". Si la configuration Patroni existante a wal_level défini sur "réplica" et si vous souhaitez utiliser cette fonctionnalité, vous pouvez simplement modifier la configuration Patroni .

![image01](/posts/2022/article12/img01.png)

Cependant, cette modification nécessite le redémarrage de PostgreSQL :

![image02](/posts/2022/article12/img02.png)


"En attente de redémarrage" et le marquage * indiquent la même chose.

Vous pouvez utiliser la fonction de "basculement" de Patroni pour redémarrer le nœud afin que les modifications prennent effet, car le nœud rétrogradé va redémarrer.

![image03](/posts/2022/article12/img03.png)

S'il reste des nœuds, ils peuvent être redémarrés ultérieurement.

![image04](/posts/2022/article12/img04.png)

## Création d'emplacements logiques

Nous pouvons maintenant ajouter un emplacement de réplication logique permanent à PostgreSQL qui sera maintenu par Patroni.

Modifiez la configuration des Patroni:

```
$ patronictl -c /etc/patroni/patroni.yml edit-config
```

Une spécification d'emplacement peut être ajoutée comme suit :

```
…
slots:
  logicreplia:
    database: postgres
    plugin: pgoutput
    type: logical
…
```

La section « **slots : »** définit les slots de réplication permanents. Ces emplacements seront conservés pendant le basculement. « pgoutput » est le plugin de décodage pour la réplication logique PostgreSQL .

Une fois la modification appliquée, l'emplacement de réplication logique sera créé sur le nœud principal et cela peut être vérifié en interrogeant :

```
select * from pg_replication_slots;
```

Voici un exemple de sortie :

![image05](/posts/2022/article12/img05.png)

**Voici maintenant le premier niveau de magie! Le même** emplacement de réplication sera également créé sur les standbys. Oui, Patroni le fait. Patroni copie en interne les informations d'emplacement de réplication du nœud principal vers tous les nœuds de secours éligibles !

Nous pouvons utiliser la même requête sur les **pg_replication_slots** en veille et voir des informations similaires.

Voici un exemple illustrant le même emplacement de réplication se reflétant du côté veille :

![image06](/posts/2022/article12/img06.png)

Cet emplacement peut être utilisé par l'abonnement en spécifiant explicitement le nom de l'emplacement lors de la création de l'abonnement.

```
CREATE SUBSCRIPTION sub2 CONNECTION '<connection_string' PUBLICATION <publication_name> WITH (copy_data = true, create_slot=false, enabled=true, slot_name=logicreplia);
```

Alternativement, un abonnement existant peut être modifié pour utiliser le nouveau créneau, ce que je préfère généralement faire.

Par exemple :

```
ALTER SUBSCRIPTION name SET (slot_name=logicreplia);
```

Le journal PostgreSQL  log entries peuvent confirmer le changement de nom d'emplacement :

```
2021-12-27 15:56:58.294 UTC [20319] LOG:  logical replication apply worker for subscription "sub2" will restart because the replication slot name was changed
2021-12-27 15:56:58.304 UTC [20353] LOG:  logical replication apply worker for subscription "sub2" has started
```

Du côté de l'éditeur, nous pouvons confirmer l'utilisation des créneaux en vérifiant le **active_pid** et en avançant le LSN pour les créneaux.

![image07](/posts/2022/article12/img07.png)

**Le deuxième niveau de Surprise!** **Les informations sur l'emplacement de réplication dans tous les nœuds de secours du cluster Patroni sont également avancées au fur et à mesure que la réplication logique progresse du côté principal.**

![image08](/posts/2022/article12/img08.png)

À un niveau supérieur, c'est exactement ce que fait cette fonctionnalité :

1. Crée/copie automatiquement les informations d'emplacement de réplication du nœud principal du cluster Patroni vers tous les nœuds de secours éligibles.
1. Avance automatiquement les numéros LSN sur les emplacements des nœuds de secours au fur et à mesure que le numéro LSN avance sur l'emplacement correspondant sur le nœud principal.

## Après un basculement

En cas de basculement ou de basculement, nous ne perdons aucune information sur les slots car ils sont déjà conservés sur les nœuds de secours.

![image09](/posts/2022/article12/img09.png)

Après le basculement, la topologie ressemble à ceci :

![image10](/posts/2022/article12/img10.png)

Désormais, toute réplique logique en aval peut être redirigée vers le nouveau primaire.

```
postgres=# ALTER SUBSCRIPTION sub2 CONNECTION 'host=192.168.50.10 port=5432 dbname=postgres user=postgres password=vagrant';                                                                                                                                                  
ALTER SUBSCRIPTION
```

Cela continue la réplication, et les informations de pg_replication_slot peuvent le confirmer.

![image11](/posts/2022/article12/img11.png)

## Résumé + points clés

L'emplacement de réplication logique n'est conceptuellement possible que sur l'instance principale, car c'est là que se produit le décodage logique. Désormais, avec cette amélioration, Patroni s'assure que les informations sur les créneaux horaires sont également disponibles en veille et qu'il sera prêt à prendre en charge la connexion de l'abonné.

- Cette solution nécessite **PostgreSQL 11 ou supérieur** car elle utilise le  **pg_replication_slot_advance ()** disponible à partir de PostgreSQL 11, pour faire avancer le slot.
- La connexion en aval peut utiliser HAProxy afin que la connexion soit automatiquement acheminée vers le principal (non couvert dans cet article). Aucune modification du code PostgreSQL ni création d'extension n'est requise.
- La copie de l'emplacement se produit via le **protocole PostgreSQL ( libpq )** plutôt que via des outils/méthodes spécifiques au système d'exploitation. Patroni utilise des informations `rewind` ou `superuser`. Patroni utilise la fonction **pg_read_binary_file ( )** pour lire les informations de slot. [Référence du code source ](https://github.com/zalando/patroni/blob/a015e0e2717caf170baa679aceb0900479ce9fda/patroni/postgresql/slots.py#L262).
- Une fois l'emplacement logique créé du côté de réplica, Patroni utilise **pg_replication_slot_ advance ( )** pour déplacer le slot vers l'avant.
- Les informations sur les créneaux horaires permanents seront **ajoutées au DCS** et **maintenues en permanence par l'instance principale** du Patroni . Une nouvelle clé DCS avec le nom ["status" est introduite et prise en charge dans toutes les options DCS ](https://github.com/zalando/patroni/pull/1820/commits/c46ee7db268c08145a1b8ecb7ba3d58235b2b980)(zookeeper, etcd , consul, etc.).
- **hot_standby_feedback** doit être activé sur tous les nœuds de secours où l'emplacement de réplication logique doit être maintenu.
- paramètre Patroni **postgresql.use_slots** doit être activé pour s'assurer que chaque nœud de secours utilise un emplacement sur le nœud principal.

