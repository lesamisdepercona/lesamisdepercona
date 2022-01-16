+++
title = "Utilisation de deux mots de passe MySQL 8"
description = "Exemple d'utilisation de deux mots de passe en MySQL 8"
author = "Francis"
date = 2021-09-30T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/thumbnailarticle18.jpg"
images = ["thumbnail/thumbnailarticle18.jpg"]
slug = "utilisation-de-deux-mots-de-passe-MySQL8"
+++

MySQL 8 a apporté de nombreuses fonctionnalités très attendues, avec la prise en charge des rôles d'utilisateur, un nouveau shell, un dictionnaire de données plus robuste et une meilleure prise en charge de SQL, pour n'en citer que quelques-uns. Il existe cependant de nouvelles fonctionnalités moins connues qui visent à réduire la charge de travail globale de l'administrateur de base de données et à rationaliser les processus de gestion. Les comptes d'utilisateurs sont désormais autorisés à avoir des mots de passe doubles, avec un primaire et un secondaire. Cela permet d'effectuer de manière transparente les modifications des informations d'identification des utilisateurs, même avec un grand nombre de serveurs ou avec plusieurs applications connectées à différents serveurs MySQL.

Historiquement, une modification des informations d'identification MySQL devait être programmée de sorte que lorsque la modification du mot de passe était effectuée et propagée dans les nœuds de la base de données, toutes les applications utilisant ce compte pour les connexions devaient être mises à jour en même temps. Ceci est problématique pour de nombreuses raisons, mais avec un nombre de bases de données et de serveurs d'applications aussi élevé qu'aujourd'hui, cela devient particulièrement lourd à l'échelle de l'entreprise moderne.

**Aperçu**

Avec les doubles mots de passe, les modifications des informations d'identification peuvent être effectuées facilement sans nécessiter de coordination ni de temps d'arrêt. Le processus fonctionnerait comme suit:

- Pour chaque compte à mettre à jour, établissez un nouveau mot de passe principal sur le(s) serveur(s) tout en conservant le mot de passe actuel comme mot de passe secondaire.
  - À ce stade, les serveurs reconnaîtront les deux mots de passe (principal et secondaire) et toutes les applications pourront continuer à se connecter avec l'ancien mot de passe comme avant.
- Une fois le changement de mot de passe effectué côté base de données, les applications peuvent être mises à jour pour se connecter à l'aide du nouveau mot de passe principal.
  - Notez que cela peut être fait sur un certains temps, en utilisant les fenêtres de maintenance des temps d'arrêt existantes si nécessaire.
- Une fois que toutes les applications ont été migrées vers le nouveau mot de passe principal, le mot de passe secondaire n'est plus nécessaire du côté de la base de données et peut être supprimé.
  - Une fois supprimé, seul le nouveau mot de passe principal peut être utilisé et la modification des informations d'identification est désormais terminée.

**Utilisation et exemple**

Pour accomplir cette capacité de double mot de passe, la nouvelle syntaxe suivante enregistrera et supprimera les mots de passe secondaires :

- **RETAIN CURRENT PASSWORD** pour **ALTER USER** et **SET PASSWORD** enregistre le mot de passe actuel en tant que mot de passe secondaire lorsqu'un nouveau mot de passe principal est affecté.
- **DISCARD OLD PASSWORD** pour **ALTER USER** supprimera un mot de passe secondaire, ne laissant que le mot de passe principal en jeu.

À titre d'exemple, utilisons la fonction de double mot de passe pour mettre à jour le mot de passe d'un utilisateur théorique ('appuser'@'percona.com'). Pour cet exemple, supposons que les applications se connecteront à la base de données avec cet utilisateur et que nous changerons le mot de passe de 'oldpass' à 'newpass'.

1. Sur chaque serveur (qui n'est pas un replica), nous allons d'abord définir 'newpass' comme nouveau mot de passe principal pour 'appuser'@'percona.com' tout en conservant le mot de passe actuel comme secondaire :
```
mysql> ALTER USER ‘appuser’@’percona.com’ IDENTIFIED BY ‘newpass’ RETAIN CURRENT PASSWORD;
```

2. Une fois que le changement de mot de passe s'est propagé à travers tous les nœuds de réplique, nous pouvons maintenant commencer à changer la ou les applications qui utilisent le compte 'appuser'@'percona.com' pour commencer à se connecter avec le nouveau mot de passe (newpass) plutôt que l'original mot de passe (oldpass). Cela peut être fait sur certain temps, pendant les fenêtres de maintenance des temps d'arrêt existants si nécessaire pour minimiser l'impact. N'oubliez pas qu'à ce stade, les deux mots de passe sont valides.
2. Avec les modifications apportées à l'application, le mot de passe secondaire (ancien) n'est plus nécessaire. Sur chaque serveur (qui n'est pas une réplique), supprimez le mot de passe secondaire :

```
mysql> ALTER USER ‘appuser’@’percona.com’ DISCARD OLD PASSWORD;
```
4. Une fois que cela s'est propagé dans tous les réplicas, la modification des informations d'identification est maintenant terminée.

Il y a quelques mises en garde que vous voudrez peut-être prendre en compte lors de l'utilisation de la fonction de double mot de passe :

- Si vous utilisez **RETAIN CURRENT PASSWORD** pour un compte dont le mot de passe principal est vide, l'instruction échouera.
- Si un compte a un mot de passe secondaire et que vous modifiez le mot de passe principal sans spécifier **RETAIN CURRENT PASSWORD**, le mot de passe secondaire reste inchangé.
- Si vous utilisez **ALTER USER** et modifiez le plug-in d'authentification attribué au compte, le mot de passe secondaire est supprimé.
  - Si vous modifiez le plug-in d'authentification et spécifiez également **RETAIN CURRENT PASSWORD** , l'instruction échoue.

**Privilèges requis**

Pour modifier vos propres comptes utilisateur, le privilège **APPLICATION PASSWORD ADMIN** est requis afin d'utiliser **RETAIN CURRENT PASSWORD** ou **DISCARD OLD PASSWORD** pour les instructions **ALTER USER** et **SET PASSWORD.**

Pour modifier le mot de passe secondaire pour un (ou tous) comptes au niveau administratif, le privilège **CREATE USER** est nécessaire plutôt que le privilège **APPLICATION PASSWORD ADMIN** comme ci-dessus.

**En conclusion**

Bien qu'il s'agisse d'une nouvelle fonctionnalité très simple, elle peut avoir un impact assez important sur la façon dont votre entreprise gère les aspects de sécurité des changements fréquents de mot de passe, minimisant voire éliminant complètement les temps d'arrêt des mises à jour de mot de passe.

**Percona Distribution for MySQL est la solution MySQL open source la plus complète, stable, évolutive et sécurisée disponible, offrant des environnements de base de données de niveau entreprise pour vos applications métier les plus critiques… et son utilisation est gratuite !**

[**Télécharger Percona Distribution pour MySQL**](https://www.percona.com/software/mysql-database)



Source : [Blog Percona](https://www.percona.com/blog/using-mysql-8-dual-passwords/)
