+++
title = "MyDumper 0.10.5 est maintenant disponible"
description = "Nouvelles fonctionnalités, Documentation, Correction des bugs et refactorisation"
author = "Francis"
date = 2021-09-02T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/article14-MyDumper.jpg"
images = ["thumbnail/article14-MyDumper.jpg"]
slug = "mydumper-0.10.5-est-maintenant-disponible"
+++

La nouvelle version [MyDumper 0.10.5](https://github.com/maxbube/mydumper/releases/tag/v0.10.5) , qui inclut de nombreuses nouvelles fonctionnalités et corrections de bugs, est désormais disponible. Vous pouvez télécharger le code [ici](https://github.com/maxbube/mydumper/archive/refs/tags/v0.10.5.tar.gz)

Pour cette version, nous nous sommes concentrés sur la résolution d'anciens problèmes et sur le test d'anciennes demandes d'extraction pour obtenir un code de meilleure qualité. Sur les versions 0.10.1, 0.10.3 et 0.10.5, nous avons publié les packages compilés avec les bibliothèques MySQL 5.7, mais à partir de maintenant, nous compilons également avec les bibliothèques MySQL 8 à guise de test, et non pas de publication, car nous pensons que beaucoup des membres de la communauté commenceront à compiler avec la dernière version, et nous devrions être prêts.

**Nouvelles fonctionnalités** :

- Obfuscation du mot de passe [#312](https://github.com/maxbube/mydumper/pull/312)  
- Utilisation du paramètre dynamique pour SET NAMES [#154](https://github.com/maxbube/mydumper/pull/154) 
- Refactoriser le logging et activer –logfile dans myloader [#233](https://github.com/maxbube/mydumper/pull/233) 
- Ajouter l'option de mode purge (NONE/TRUNCATE/DROP/DELETE) pour décider quel est le mode préférable [#91 ](https://github.com/maxbube/mydumper/pull/91)[#25](https://github.com/maxbube/mydumper/issues/25)  
- Éviter l’envoi de COMMIT lorsque commit\_count est égal à 1 [#234](https://github.com/maxbube/mydumper/pull/234) 
- Vérifier si le répertoire existe [#294](https://github.com/maxbube/mydumper/pull/294) 

**Corrections de bugs** :

- Ajout du support de construction homebrew [#313](https://github.com/maxbube/mydumper/pull/313)  
- Suppression de la dépendance MyISAM dans les tables temporaires pour VIEWS [#261](https://github.com/maxbube/mydumper/pull/261) 
- Correction des avertissements dans la génération de document sphinx [#287](https://github.com/maxbube/mydumper/pull/287) 
- Correction d'une boucle sans fin lorsque la liste de processus ne pouvait pas être vérifiée [#295](https://github.com/maxbube/mydumper/pull/295) 
- Correction des problèmes lorsque daemon est utilisé sur glist [#317](https://github.com/maxbube/mydumper/pull/317) 

**Documentation** :

- Correction des dépendances des paquets ubuntu / debian [#310](https://github.com/maxbube/mydumper/pull/310) 
- Fournir de meilleures instructions de construction CentOS 7 [#232](https://github.com/maxbube/mydumper/pull/232) 
- utilisation du fichier –defaults-file pour la section mydumper et le client [#306](https://github.com/maxbube/mydumper/pull/306) 
- Ajoutée la documentation INSERT IGNORE [#195](https://github.com/maxbube/mydumper/pull/195) 

**Refactorisation** :

- Ajout de la fonction new\_table\_job [#315](https://github.com/maxbube/mydumper/pull/315) 
- Ajout de IF NOT EXISTS à SHOW CREATE DATABASE [#314](https://github.com/maxbube/mydumper/pull/314) 
- Mettre à jour FindMySQL.cmake [#149](https://github.com/maxbube/mydumper/pull/149) 

Source : [Percona Blog](https://www.percona.com/blog/2021/05/07/mydumper-0-10-5-is-now-available/)
