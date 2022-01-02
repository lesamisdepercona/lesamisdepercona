+++
title = "Mise à niveau vers MySQL 8 : relever le défi"
description = "Traduit à partir de l'article de Mike Benshoof intitulé, Upgrading to MySQL 8 - Embrace the Challenge"
author = "Francis"
date = 2021-08-08T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/amisdepercona21-00010.jpg"
images = ["thumbnail/amisdepercona21-00010.jpg"]
slug = "un-defi-mise-a-niveau-vers-MySQL8"
+++

![image 10](/thumbnail/amisdepercona21-00010.jpg)

Personne n’aime le changement, surtout quand ce changement peut être difficile. Face à un défi technique, j’essaie de me souvenir de ce commentaire de Theodore Roosevelt : « _Rien au monde ne vaut la peine d’avoir ou de valoir la peine d’être fait sans effort, douleur, difficulté._» Bien que ce soit un peu exagéré, dans ce cas, le concept principal est toujours valable. Nous ne devrions pas nous éloigner d’un chemin de mise à niveau car cela peut être difficile.

**MySQL 8.0 arrive à maturité et se stabilise**. Il y a de nouvelles fonctionnalités (trop nombreuses pour être énumérées ici) et des améliorations de performances. De plus en plus d’organisations mettent à niveau vers MySQL 8 et l’exécutent en production, ce qui accélère la stabilisation. Bien qu’il y ait encore une piste importante sur 5.7 et qu’elle soit définitivement stable (EOL prévue pour octobre 2023), les organisations doivent se préparer à faire le saut si elles ne l’ont pas déjà fait.

## Qu’est ce qui a changé?

Alors, en quoi une mise à niveau majeure vers la version 8.0 est-elle différente de celle des années précédentes? Honnêtement, ce n’est vraiment pas si différent. Le même processus général s’applique :

1. Mise à niveau dans un environnement inférieur
2. Testez, testez, puis testez encore
3. Mettre à niveau un réplica et commencer à envoyer du trafic de lecture
4. Promouvoir un réplica en principal
5. Soyez prêt pour un retour en arrière si nécessaire

Le dernier point est le plus gros changement, surtout une fois que vous avez terminé la mise à niveau vers MySQL 8. Historiquement, les mises à niveau de version mineures étaient assez triviales. Un simple arrêt d’instance, un échange binaire et un démarrage d’instance suffisaient pour revenir à une version précédente.

Dans la version 8.0, ce processus n’est plus pris en charge (comme indiqué dans la [documentation officielle](https://dev.mysql.com/doc/refman/8.0/en/downgrading.html) )

« La mise à niveau de MySQL 8.0 vers MySQL 5.7, ou d’une version **MySQL 8.0 vers une version précédente MySQL 8.0** , n’est pas prise en charge. »

C’est un changement définitif dans le paradigme des versions, et cela a montré de réels problèmes dans les versions mineures. Un bon exemple de l’impact que cela peut avoir sur un système en direct a été bien capturé dans l’article de blog [MySQL 8 Minor Version Upgrades Are ONE-WAY Only](https://www.percona.com/blog/2020/01/10/mysql-8-minor-version-upgrades-are-one-way-only/) à partir du début de 2020.

## Comment aborder la mise à niveau vers MySQL 8

Avec ce nouveau paradigme, il peut sembler effrayant de faire avancer la mise à niveau. Je pense qu’à certains égards, cela peut être un changement positif. Comme mentionné ci-dessus, une préparation et des tests appropriés devraient constituer la majorité du processus. La mise à niveau/le basculement réel devrait essentiellement être un non-événement. Il n’y a rien qu’un DBA aime plus que de finaliser une mise à niveau sans que personne ne le remarque (à part les améliorations potentielles).

Malheureusement, dans la pratique, les tests et la préparation appropriés sont généralement une réflexion après coup. Avec la facilité des mises à niveau (et en particulier les annulations), il était généralement plus facile de simplement «essayer et annuler si nécessaire». Comme les déclassements ne sont plus triviaux, cela devrait être considéré comme une opportunité en or d’améliorer la phase de préparation d’une mise à niveau.

Une attention supplémentaire devrait être accordée à :

- Examiner les notes de version en détail pour tout changement potentiel (les nouvelles fonctionnalités sont parfois activées par défaut dans la version 8.0)
- Tester le système avec un trafic applicatif RÉEL (les benchmarks sont sympas, mais ne signifient rien s’ils sont génériques)
- Solidifier le processus de sauvegarde et de restauration (c’est déjà parfait, non ?)
- Examen de l’automatisation (cela signifie qu’il n’y a plus de mises à niveau automatisées «silencieuses»)
- Le processus de mise à niveau réel (en commençant au bas de la chaîne de réplication et en maintenant les réplicas pour les annulations si nécessaire)

Avec autant d’emphase sur la préparation, nous devrions, espérons-le, commencer à voir la mise à niveau réelle devenir moins impactante. Cela devrait également inspirer plus de confiance dans l’ensemble de l’organisation.

## Qu’en est-il de «la mise en échelle (at scale)»?

Ayant travaillé avec un large éventail de clients en tant que TAM, j’ai vu des environnements allant d’une seule paire primaire/réplique à des milliers de serveurs. J’admettrai librement qu’effectuer une mise à niveau de version majeure sur 10 000 serveurs n’est pas trivial. Rien n’est plus pénible que de devoir restaurer 5 000 serveurs lorsque quelque chose survient à mi-chemin du processus de mise à niveau. Bien qu’aucune quantité de tests ne puisse éliminer complètement cette possibilité, nous pouvons nous efforcer de minimiser le risque.

À cette échelle, tester les modèles de trafic réels est tellement plus critique. Lorsque vous examinez des environnements et des charges de travail complexes, la probabilité de rencontrer un cas limite augmente définitivement. L’identification de ces cas limites dans les environnements inférieurs est essentielle pour un processus de production fluide. De même, il est essentiel de s’assurer que des processus et des playbooks existent pour le retour en arrière (dans le cas où un problème apparaîtrait).

Enfin, le déploiement des mises à niveau par phases est également essentiel. En supposant que vous ayez mis en place une surveillance telle que [Percona Monitoring and Management](https://www.percona.com/software/database-tools/percona-monitoring-and-management) , les tests et comparaisons A/B peuvent être inestimables. Voir les versions X et Y sur le même tableau de bord tout en servant le même trafic permet une bonne comparaison. Comparer X dans la mise en scène à Y dans la production est important, mais peut parfois être trompeur.

### Conclusion

Dans l’ensemble, la mise à niveau vers MySQL 8 n’est pas si différente des versions précédentes. Des précautions supplémentaires doivent être prises pendant la phase de préparation, mais cela doit être considéré comme un élément positif dans l’ensemble. Nous ne devrions certainement pas fuir le changement, mais plutôt l’embrasser car il doit éventuellement se produire. La pire chose qui puisse arriver est de continuer à avancer lentement, puis d’être pressé par le temps à l’approche de 5.7 EOL.

Pour solidifier la phase de préparation et de test, quels outils manquent selon vous ? Qu’est-ce qui faciliterait la relecture précise du trafic par rapport aux instances de test ? Bien qu’il existe des outils disponibles, y a-t-il quelque chose qui permettrait de s’assurer que ces meilleures pratiques sont suivies ?

Si votre organisation a besoin d’aide pour préparer ou terminer une mise à niveau, notre [équipe de services professionnels](https://www.percona.com/services/consulting) peut être un atout précieux. De même, nos ingénieurs de support peuvent aider votre équipe lorsque vous rencontrez des cas extrêmes lors des tests.

Et, enfin, la stratégie la plus importante en matière de mises à niveau : le **read-only Friday** devrait être pris en consideration!

Source : [Percona Blog](https://www.percona.com/blog/2021/04/02/upgrading-to-mysql-8/)
