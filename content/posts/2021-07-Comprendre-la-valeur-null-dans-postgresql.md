+++
title = "Gestion des valeurs NULL dans PostgreSQL"
description = "Traduit à partir de l'article de Ibrar Ahmed intitulé, Handling NULL Values in PostgreSQL, disponible sur le blog de Percona"
author = "Francis"
date = 2021-07-15T11:43:01+04:00
tags = ['PostgreSQL']
Categories = ["Article de Percona"]
featured_image = "posts/article07/postgresql.png"
images = ["posts/article07/postgresql.png"]
slug = "gestion-des-valeurs-null-dans-postgresql"
+++


**Qu'est-ce que NULL ?**

Il y a souvent une certaine confusion à propos de la valeur NULL, car elle est traitée différemment dans différents langages. Il y a donc un besoin évident de clarifier ce qu'est NULL, comment cela fonctionne dans différentes langages et quelle est la valeur réelle derrière le NULL. Avant d'entrer dans les détails, il est également nécessaire de comprendre le concept de logique à trois valeurs [1] et de logique à deux valeurs dite bivalente[2]. Le bivalent est un concept de valeur booléenne où la valeur peut être vraie ou fausse, mais contrairement au bivalent, la logique à trois valeurs peut être vraie, fausse ou (valeur intermédiaire) inconnue. Maintenant, revenons à NULL. Dans certains langages, NULL agit comme bivalent, et dans d'autres, la logique à trois valeurs (en particulier dans les bases de données).

**C/C++**

Dans « C/C++ », le NULL est défini comme 0 dans le « stddef.h » qui est inclus `<cstddef>` dans le cas de C++ et stdlib.h dans le cas de C.
```
#if defined (_STDDEF_H) || defined (__need_NULL)
#undef NULL     /* in case <stdio.h> has defined it. */
#ifdef __GNUG__
#define NULL __null
#else   /* G++ */
#ifndef __cplusplus
#define NULL ((void *)0)
#else   /* C++ */
#define NULL 0
#endif  /* C++ */
#endif  /* G++ */
#endif  /* NULL not defined and <stddef.h> or need NULL.  */
#undef  __need_NULL
```

La valeur peut être testée par rapport à NULL en utilisant directement les opérateurs d'égalité « == » ou ! =. Prenons maintenant un exemple de programme et essayez de vérifier les valeurs par rapport à NULL en C.
```
#include <stddef.h>
#include <stdio.h>
void main()
{
    if ('0' == NULL)
        printf("NULL is '0' \n");
    if ("" == NULL)
        printf("NULL is empty string \n");
    if (' ' == NULL)
        printf("NULL is space \n");
    if (0 == NULL)
        printf("NULL is 0 \n");
}
```
Le output du programme ci-dessus sera "NULL est 0", il est donc tout à fait évident que NULL est défini comme "0" en langage C.

**Java**

Contrairement à C où NULL vaut 0, en Java, NULL signifie que les références de variables ont une valeur. La valeur peut être testée par rapport à NULL par des opérateurs d'égalité. Lorsque nous imprimons la valeur nulle, il imprimera la valeur nulle. En Java, null est sensible à la casse et il doit être en minuscules comme "null".

```
public class Test
{ 
    public static void main (String[] args) throws java.lang.Exception
    {
        System.out.println("Null is: " + null);
    }
}
Null is: null
```

**PostgreSQL**

Dans PostgreSQL , NULL signifie aucune valeur. En d'autres termes, la colonne NULL n'a aucune valeur. Il n'est pas égal à 0, à une chaîne vide ou à des espaces. La valeur NULL ne peut pas être testée à l'aide d'un opérateur d'égalité tel que "=" "!=" etc. Il existe des instructions spéciales pour tester la valeur par rapport à NULL, mais à part cela, aucune instruction ne peut être utilisée pour tester la valeur NULL.

Faisons quelques comparaisons intéressantes, qui éclairciront le concept de NULL dans PostgreSQL. Dans l'extrait de code suivant, nous comparons 1 avec 1 et le résultat évident est « t » (TRUE). Cela nous amène à comprendre que l'opérateur d'égalité PostgreSQL nous donne vrai lorsque deux valeurs correspondent. De même, l'opérateur d'égalité fonctionne pour la valeur textuelle.

Comparaison numérique normale
```
postgres=# SELECT 1 = 1 result;
 result 
--------
 t
(1 row)
```

Comparaison textuelle normale
```
postgres=# SELECT 'foo' = 'foo' result;
 result 
--------
 t
(1 row)
```

Faisons d'autres expériences, en comparant NULL avec NULL. Si NULL est une valeur normale, le résultat doit être « t ». Mais NULL n'est pas une valeur normale, il n'y a donc aucun résultat.

Comparaison NULL normale
```
postgres=# SELECT NULL = NULL result;
 result 
--------


(1 row)
```

Comparons NULL avec NULL en utilisant un opérateur d'inégalité. Le résultat est le même que ce que nous avons obtenu précédemment. Cela prouve que nous ne pouvons pas comparer NULL avec NULL en utilisant les opérateurs d'égalité et d'inégalité.

Comparaison d'inégalité normale NULL
```
postgres=# SELECT NULL != NULL result;
 result 
--------
 

(1 row)
```
De même, aucune opération mathématique ne peut être effectuée sur NULL. PostgreSQL ne produit rien lorsqu'un NULL est utilisé comme opérande.

Valeur numérique multiple avec NULL
```
postgres=# SELECT NULL * 10 is NULL result;
 result 
--------
 t
(1 row)
```
**Comment utiliser NULL**

Par conséquent, il est prouvé que NULL ne peut être comparé à aucune valeur utilisant des opérateurs d'égalité. Alors comment pouvons-nous utiliser le NULL si nous ne pouvons pas utiliser d'opérateur ou d'opération mathématique ? PostgreSQL fournit des instructions et des fonctions spéciales pour vérifier et tester les valeurs par rapport à NULL. Il existe un seul moyen d'utiliser le NULL dans PostgreSQL .

**EST NULL/N'EST PAS NULL**

Utilisation de Comparaison NULL
```
postgres=# SELECT NULL is NULL result;
 result 
--------
 t
(1 row)
```

Utilisation de Comparaison NULL
```
postgres=# SELECT NULL is NOT NULL result;
 result 
--------
 f
(1 row)
```

**COALESCE**

PostgreSQL a un nom de fonction « COALESCE » [3]. La fonction prend n nombre d'arguments et renvoie les premiers arguments, non nuls. Vous pouvez tester votre expression par rapport à NULL en utilisant la fonction.
```
 COALESCE (NULL, 2 , 1);
```

**NULLIF**

Il existe une autre fonction appelée « NULLIF »[3], renvoie NULL si le premier et le deuxième arguments sont égaux, sinon renvoie le premier argument. Voici l'exemple où nous comparons 10 avec 10 et nous savons déjà qu'ils sont égaux, donc cela renvoie NULL. Dans le deuxième exemple, nous comparons 10 avec 100 et dans ce cas, il renverra 10 la première valeur.

```
postgres=# SELECT NULLIF (10, 10);
nullif
--------
      

(1 row)

postgres=# SELECT NULLIF (10, 100);
nullif
--------
     10
(1 row)
```

**Utilisation de NULL**

Si NULL n'a aucune valeur, alors quel est l'avantage de NULL ? Voici quelques exemples de son utilisation:

Dans le cas où un champ n'a aucune valeur, et par exemple, nous avons des champs de base de données avec le prénom/le deuxième prénom et le nom. Est-ce que, en réalité, tout le monde a un prénom/un deuxième prénom et un nom de famille? La réponse est non, il ne devrait pas y avoir de champ qui ne puisse avoir aucune valeur.
```
postgres=# CREATE TABLE student(id INTEGER, fname TEXT, sname TEXT, lname TEXT, age INTEGER);

postgres=# SELECT * FROM STUDENT;
 id | fname | sname | lname | age 
----+-------+-------+-------+-----
  1 | Adams | Baker | Clark |  21
  2 | Davis |       | Evans | 22
  3 | Ghosh | Hills |       | 24
(3 rows)

```
Sélectionnons les étudiants qui ont un deuxième prénom. Cette requête fonctionne-t-elle ici? Non, et la raison derrière cela est la même que nous avons discutée dans les requêtes précédentes.

```
postgres=# SELECT * FROM STUDENT WHERE sname = '';
 id | fname | sname | lname | age 
----+-------+-------+-------+-----
(0 rows)
```

Sélectionnons-les en utilisant les instructions appropriées pour obtenir les résultats souhaités.

```
postgres=# SELECT * FROM STUDENT WHERE sname IS NULL;
 id | fname | sname | lname | age 
----+-------+-------+-------+-----
  2 | Davis |       | Evans | 22
(1 row)
```

Le champ n'a pas de sens, par exemple, parce que le nom du conjoint d'une personne seule ou les détails des enfants ne sont pas "KID". Voici l'exemple où KID dans le champ divorcé n'a pas de sens. Nous ne pouvons pas mettre vrai ou faux, donc NULL est la bonne valeur ici.
```
postgres=# CREATE TABLE person(id INTEGER, name TEXT, type TEXT, divorced bool);
postgres=# SELECT * FROM person;
 id | name  | type | divorced 
----+-------+-------+---------
  1 | Alice | WOMAN | f
  3 | Davis | KID   | 
  2 | Bob   | MAN | t
(3 rows)
```

L'autre utilisation de NULL est de représenter une chaîne vide et une valeur numérique vide. Le chiffre 0 a une signification, il ne peut donc pas être utilisé pour représenter le champ numérique vide, la valeur inconnue à un certain moment.

Dans cet exemple, il y a trois élèves : Alice a 90 points, Bob a 0 points et Davis n'a pas encore de points. Dans le cas de Bob, nous avons inséré 0, et pour Davis, nous avons inséré NULL. En faisant cela, nous pouvons facilement distinguer qui a 0 points et qui n'a pas encore de résultats.

```
postgres=# SELECT * FROM students_mark;
 id | name  | marks 
----+-------+-------
  1 | Alex  | 90
  2 | Bob   | 0
  2 | Davis |      
(3 rows)
```

```
postgres=# SELECT * FROM students_mark WHERE marks IS NULL;
 id | name  | marks 
----+-------+-------
  2 | Davis |      
(1 row)
```

```
postgres=# SELECT * FROM students_mark WHERE marks = 0;
 id | name | marks 
----+------+-------
  2 | Bob  | 0
(1 row)
```

**Conclusion**

Le but de ce blog est d'être clair sur le fait que chaque langue a sa propre signification de NULL. Par conséquent, soyez prudent lorsque vous utilisez NULL, sinon vous obtiendrez des résultats erronés. Dans les bases de données ( PostgreSQL ), NULL a des concepts différents, alors soyez prudent lorsque vous écrivez des requêtes impliquant NULL.

Source : [Blog Percona](https://www.percona.com/blog/2020/03/05/handling-null-values-in-postgresql/)

[1] – [https://en.wikipedia.org/wiki/Three-valued\_logic](https://en.wikipedia.org/wiki/Three-valued_logic)

[2] – [https://en.wikipedia.org/wiki/Principle\_of\_bivalence](https://en.wikipedia.org/wiki/Principle_of_bivalence)

[3] – [https://www.postgresql.org/docs/current/functions-conditional.html](https://www.postgresql.org/docs/current/functions-conditional.html)
