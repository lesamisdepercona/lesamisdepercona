+++
title = "Relire l'exécution de MySQL avec Record and Replay"
description = "Traduit à partir de l'article de Marcelo Altmann intitulé, Replay the Execution of MySQL With Record and Replay"
author = "Francis"
date = 2021-08-20T11:43:01+04:00
tags = ['MySQL']
Categories = ["Article de Percona"]
featured_image = "thumbnail/amisdepercona21-00012.jpg"
images = ["thumbnail/amisdepercona21-00012.jpg"]
slug = "relire-l-execution-de-MySQL-avec-rr-record-and-replay"
+++



**La chasse aux bugs peut être une tâche fastidieuse, et les logiciels multi-threads ne facilitent pas la tâche.** Les threads seront programmés à des moments différents, les instructions n'auront pas de résultats déterministes, et pour reproduire un problème particulier, cela peut nécessiter exactement les mêmes threads, effectuant exactement le même travail, exactement au même moment. Comme vous pouvez l'imaginer, ce n'est pas si simple.

Disons que votre base de données tombe en panne ou même qu'elle a un décrochage transitoire. Au moment où vous y arrivez, le crash s'est produit et vous êtes bloqué en restaurant le service rapidement et en faisant des investigations après coup. Ne serait-il pas agréable de rejouer le travail juste avant ou pendant le crash et de voir exactement ce qui se passait ?

[Record and Replay](https://rr-project.org/) est une technique où nous enregistrons l'exécution d'un programme lui permettant d'être rejoué encore et encore produisant le même résultat. Les ingénieurs de Mozilla ont créé RR, et fondamentalement, cet outil open source vous permet d'enregistrer l'exécution du logiciel et de le rejouer sous le célèbre GDB. 

**Un problème de sauvegarde**

Pour démontrer à quel point l'outil est puissant, nous expliquerons comment nous l'avons utilisé pour réduire le problème de [PXB-2180](https://jira.percona.com/browse/PXB-2180) (remerciement spécial à [Satya Bodapati](https://www.percona.com/blog/author/satya-bodapati/) , qui a aidé à toutes les recherches internes d'InnoDB pour ce bug).

En résumé, nous avions vu [Percona XtraBackup ](https://www.percona.com/software/mysql-database/percona-xtrabackup) planter au stade de la préparation (rappelez-vous de toujours tester votre sauvegarde!). Le crash se produisait de manière aléatoire, parfois après le deuxième incrémentiel, parfois après le 10ème incrémentiel, sans motif visible.

La trace de la pile n'était pas non plus toujours la même. Il plantait sur différentes parties d'InnoDB , mais ici, nous avions un point commun entre tous les plantages - cela se produisait toujours en essayant d'appliquer un enregistrement de journalisation à la même page de blocage et au même espace id :
```
#12 0x00000000015ad05f in recv_parse_or_apply_log_rec_body (type=MLOG_COMP_REC_INSERT, ptr=0x7f2849150556 "\003K4G", '\377' <repeats 13 times>, end_ptr=0x7f2849150573 "", space_id=<optimized out>, page_no=<optimized out>, block=0x7f2847d7da00, mtr=0x7f286857b4f0, parsed_bytes=18446744073709551615) at /home/marcelo.altmann/percona-xtrabackup/storage/innobase/log/log0recv.cc:2002
2002         ptr = page_cur_parse_insert_rec(FALSE, ptr, end_ptr, block, index, mtr);
(gdb) p block->page->id
+p block->page->id
$3 = {
  m_space = 4294967294,
  m_page_no = 5
}
```

Notre soupçon était que la mise en page de ce bloc divergeait entre MySQL et XtraBackup. Lorsque vous travaillez avec ces types de bug, le plantage est toujours la conséquence de quelque chose qui s'est passé plus tôt, par exemple : un plantage sur la sixième sauvegarde incrémentielle pourrait être la conséquence d'un problème survenu lors de la quatrième sauvegarde incrémentielle.

L'objectif principal à cette étape est de prouver et d'identifier où la mise en page s'est détournée.

Avec ces informations, nous avons exécuté MySQL sous RR et réexécuté la sauvegarde jusqu'à ce que nous voyions le même problème lors de la préparation. Nous pouvons maintenant rejouer l'exécution de MySQL et vérifier comment ça se présente. Notre idée est de : 

1. Lire les [LSN](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_lsn) pour cette même page avant/après chaque préparation de sauvegarde
1. Identifier toutes les modifications apportées à m\_space = 4294967294 & m\_page\_no = 5 sur mysqld

Avant d'aller plus loin, nous tenons à expliquer quelques points :

1. m\_space = 4294967294 correspond au dictionnaire de données MySQL (mysql.ibd) – [dict0dict.h:1146](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/dict0dict.h#L1146)
1. Sur la page du disque, LSN est stocké au 16e octet de la page avec une taille de 8 octets – [fil0types.h:66](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/fil0types.h#L66) 
1. Les pages sont écrites séquentiellement sur le disque, par exemple, pour la taille de page par défaut de 16 ko, à partir d’octet 1 à 16384 auront les données de la page 0. Celle de l'octet entre 16385 et 32768, les données seront dans la page 1, et ainsi de suite.
1. Le Frame est une donnée brute d'une page – [buf0buf.h:1358](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/buf0buf.h#L1358)

**Rejouer l'exécution**

Pour commencer, lisons le LSN que nous avons sur mysql.ibd pour la page cinq avant la sauvegarde. Nous utiliserons od (consultez man od pour plus d'informations) et les informations expliquées ci-dessus.
```
$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 10 21 85
0240030
```
Vérifions s'il correspond à un tampon LSN de mysqld. Pour cela nous allons ajouter un point d'arrêt conditionnel sur le replay de l'exécution de MySQL à la fonction buf\_flush\_note\_modification : 
```
$ rr replay .
. . .
(rr) b buf_flush_note_modification if block->page->id->m_space == 4294967294 && block->page->id->m_page_no == 5
+b buf_flush_note_modification if block->page->id->m_space == 4294967294 && block->page->id->m_page_no == 5
Breakpoint 1 at 0x495beb1: file /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic, line 69.
(rr) c
[Switching to Thread 18839.18868]

Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=17892965, end_lsn=17893015, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$1 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x10,
  [0x6] = 0x21,
  [0x7] = 0x85}
(rr)
```

Nous pouvons voir le tampon LSN d'avant la préparation de la sauvegarde complète et le premier tampon de relecture de la session match. Il est temps de préparer la sauvegarde, d'avancer l'exécution de la relecture et de revérifier :
```
xtrabackup --prepare --apply-log-only --target-dir=full/
. . .
Shutdown completed; log sequence number 17897577
Number of pools: 1
210402 17:46:29 completed OK!


$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 11 07 06
0240030


(rr) c
+c
Continuing.
[Switching to Thread 18839.18868]

Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=19077332, end_lsn=19077382, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$16 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x11,
  [0x6] = 0x7,
  [0x7] = 0x6}
(rr)
```
Même tampon LSN à tous les deux, serveur et sauvegarde. Il est temps de passer à autre chose et de commencer à appliquer les incrémentiels :
```
xtrabackup --prepare --apply-log-only --target-dir=full/ --incremental-dir=inc1/
. . .
Shutdown completed; log sequence number 19082430
. . .
210402 18:12:20 completed OK!


$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 23 19 06
0240030


(rr) c
+c
Continuing.
Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=20262758, end_lsn=20262808, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$17 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x23,
  [0x6] = 0x19,
  [0x7] = 0x6}
(rr)
```
Encore une fois, nous avons le même tampon LSN des deux côtés. Passage à l'incrémentiel suivant :
```
xtrabackup --prepare --apply-log-only --target-dir=full/ --incremental-dir=inc2/
. . .
Shutdown completed; log sequence number 20269669
. . .
210402 18:15:04 completed OK!


$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 35 2f 98
0240030


(rr) c
+c
Continuing.

Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=21449997, end_lsn=21450047, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$18 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x35,
  [0x6] = 0x2f,
  [0x7] = 0x98}
(rr)
```
L’incrémentiel 02 s’appliquent et correspondent au LSN à partir de mysqld. Continuons jusqu'à ce que nous trouvions une incompatibilité :
```
xtrabackup --prepare --apply-log-only --target-dir=full/ --incremental-dir=inc3/
. . .
Shutdown completed; log sequence number 21455916
. . .
210402 18:18:25 completed OK!


$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 47 4d 3f
0240030


(rr) c
+c
Continuing.

Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=25529471, end_lsn=25529521, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$19 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x47,
  [0x6] = 0x4d,
  [0x7] = 0x3f}
(rr)


xtrabackup --prepare --apply-log-only --target-dir=full/ --incremental-dir=inc4/
. . .
Shutdown completed; log sequence number 23044902
. . .
210402 18:24:00 completed OK!

$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 5f a3 26
0240030


(rr) c
+c
Continuing.

Breakpoint 1, buf_flush_note_modification (block=0x7fd2df4ad750, start_lsn=27218464, end_lsn=27218532, observer=0x0) at /home/marcelo.altmann/percona-server/storage/innobase/include/buf0flu.ic:69
69     ut_ad(!srv_read_only_mode ||
++rr-set-suppress-run-hook 1
(rr) p/x block->frame[16]@8
+p/x block->frame[16]@8
$242 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x9f,
  [0x6] = 0x3f,
  [0x7] = 0xc9}
(rr)
```
Ok, ici nous avons quelque chose. Les fichiers de sauvegarde passent de 0x01474d3f à 0x015fa326 lors de l'application de l’incrémentiel quatre tandis que le serveur est passé de 0x01474d3f à 0x019f3fc9. Peut-être avons-nous manqué un autre endroit où nous pouvons mettre à jour le tampon LSN d'une page? Maintenant, nous sommes à un point d’avance avec notre exécution de relecture du serveur MySQL

**Rejouer l'exécution à l'envers**

Voici (encore) une autre fonctionnalité très intéressante de RR, elle vous permet de rejouer l'exécution à l'envers. Pour éliminer la possibilité de manquer un endroit qui met également à jour le LSN de ce bloc, ajoutons un point de surveillance matériel sur l’adresse mémoire block- >frame et inversons l'exécution :
```
(rr) p block->frame
+p block->frame
$243 = (unsigned char *) 0x7fd2e0758000 "\327\064X["
(rr) watch *(unsigned char *) 0x7fd2e0758000
+watch *(unsigned char *) 0x7fd2e0758000
Hardware watchpoint 2: *(unsigned char *) 0x7fd2e0758000
(rr) disa 1
+disa 1
(rr) reverse-cont
+reverse-cont
+continue
Continuing.
Hardware watchpoint 2: *(unsigned char *) 0x7fd2e0758000

Old value = 215 '\327'
New value = 80 'P'

0x0000000004c13903 in mach_write_to_4 (b=0x7fd2e0758000 "P\257\"\347", n=3610531931) at /home/marcelo.altmann/percona-server/storage/innobase/include/mach0data.ic:135
135   b[0] = static_cast<byte>(n >> 24);
++rr-set-suppress-run-hook 1
++rr-set-suppress-run-hook 1
(rr) p/x buf_flush_init_for_writing::block->frame[16]@8
+p/x buf_flush_init_for_writing::block->frame[16]@8
$11 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x9f,
  [0x6] = 0x3f,
  [0x7] = 0xc9}
(rr) reverse-cont
+reverse-cont
+continue
Continuing.
Hardware watchpoint 2: *(unsigned char *) 0x7fd2e0758000

Old value = 80 'P'
New value = 43 '+'
0x0000000004c13903 in mach_write_to_4 (b=0x7fd2e0758000 "+k*\304", n=1353655015) at /home/marcelo.altmann/percona-server/storage/innobase/include/mach0data.ic:135
135   b[0] = static_cast<byte>(n >> 24);
++rr-set-suppress-run-hook 1
++rr-set-suppress-run-hook 1
(rr) p/x buf_flush_init_for_writing::block->frame[16]@8
+p/x buf_flush_init_for_writing::block->frame[16]@8
$12 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x1,
  [0x5] = 0x47,
  [0x6] = 0x4d,
  [0x7] = 0x3f}
(rr)
```
En rejouant l'exécution à l'envers, nous pouvons voir qu'en effet le serveur a changé le LSN de 0x01474d3f à 0x019f3fc9 . Cela confirme que le problème se situe au niveau de la sauvegarde incrémentielle quatre, car le LSN 0x015fa326 que nous voyons à la fin de la sauvegarde incrémentielle quatre n'a jamais été un LSN valide lors de l'exécution du serveur.

**Cause Principale**

Maintenant que nous avons limité la portée de six sauvegardes à une seule, les choses vont devenir plus faciles.

Si nous examinons de près les messages de journal de -prepare de la sauvegarde, nous pouvons voir que le LSN de mysql.ibd correspond au tampon LSN à la fin de la sauvegarde :
```
xtrabackup --prepare --apply-log-only --target-dir=full/ --incremental-dir=inc4/
. . .
Shutdown completed; log sequence number 23044902
. . .
210402 18:24:00 completed OK!

$ od -j $((16384 * 5 + 16)) -N 8 -t x1 full/mysql.ibd
0240020 00 00 00 00 01 5f a3 26
0240030


$ echo $(( 16#015fa326 ))
23044902
```
En vérifiant la trace de la pile du problème et en examinant plus en détail le bloc que nous avons analysé, nous pouvons voir qu'il s'agit de l'index innodb\_dynamic\_metadata:
```
(gdb) f 13
+f 13
#13 0x00000000015af3dd in recv_recover_page_func (just_read_in=just_read_in@entry=true, block=block@entry=0x7f59efd7da00) at /home/marcelo.altmann/percona-xtrabackup/storage/innobase/log/log0recv.cc:2624
2624       recv_parse_or_apply_log_rec_body(recv->type, buf, buf + recv->len,
(gdb) p/x block->frame[66]@8
+p/x block->frame[66]@8
$4 =   {[0x0] = 0x0,
  [0x1] = 0x0,
  [0x2] = 0x0,
  [0x3] = 0x0,
  [0x4] = 0x0,
  [0x5] = 0x0,
  [0x6] = 0x0,
  [0x7] = 0x2}
```
Vous vous demandez peut-être d'où vient le 66? Cela vient de l'examen de la position [FIL_PAGE_DATA](https://www.percona.com/blog/2021/04/12/replay-the-execution-of-mysql-with-rr-record-and-replay/) + [PAGE_INDEX_ID](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/page0types.h#L83) . Cela nous a donné l'ID d'index 2. C'est en dessous de 1024, qui est réservé aux tables du [dictionnaire de données](https://github.com/percona/percona-xtrabackup/blob/8.0/storage/innobase/include/dict0dd.h#L82). En vérifiant la deuxième table de cette liste, nous pouvons voir qu'il s'agit de [innodb_dynamic_metadata](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/include/dict0dd.h#L276) . Avec toutes ces informations résumées, nous pouvons regarder ce que fait le serveur à l'arrêt, et le problème devient beaucoup plus évident : [srv0start.cc:3965](https://github.com/percona/percona-server/blob/Percona-Server-8.0.22-13/storage/innobase/srv/srv0start.cc#L3965)

```
/** Shut down the InnoDB database. */
void srv_shutdown() {
  . . .

  /* Write dynamic metadata to DD buffer table. */
  dict_persist_to_dd_table_buffer();
. . .
}
```
Dans le cadre du processus d'arrêt, nous conservons les métadonnées modifiées dans la table DD Buffer ( innodb\_dynamic\_metadata ), ce qui est faux. Ces modifications seront probablement conservées par le serveur et enregistrées à nouveau une fois que le serveur aura effectué un point de contrôle. En outre, beaucoup de données peuvent être fusionnées selon le moment où la sauvegarde a été effectuée et lorsque le serveur lui-même conserve ces données dans les tables DD. Ceci est le résultat de la mise en œuvre de [WL#7816](https://dev.mysql.com/worklog/task/?id=7816) et [WL#6204](https://dev.mysql.com/worklog/task/?id=6204) qui a obligé Percona XtraBackup à modifier la façon dont il gère ces types d'enregistrements de rétablissement.

**Résumé**

Dans ce blog, nous avons parcouru le processus d'analyse d'un véritable bug sur Percona XtraBackup . Ce bug expose un défi auquel nous sommes confrontés dans divers types de bug, où le crash/dysfonctionnement est une conséquence de quelque chose qui s'est passé bien avant, et au moment où nous avons une trace de pile/ coredump, il est trop tard pour effectuer une analyse appropriée. L'enregistrement et la relecture nous ont permis de rejouer de manière cohérente l'exécution du serveur source, ce qui a permis de réduire le problème permettant d’identifier la cause principale.

[Percona XtraBackup est une solution de sauvegarde de base de données complète, gratuite et open source pour toutes les versions de Percona Server pour MySQL](https://www.percona.com/software/mysql-database/percona-xtrabackup?utm_source=blog) 

Page Source : [Percona Blog](https://www.percona.com/blog/2021/04/12/replay-the-execution-of-mysql-with-rr-record-and-replay/)

