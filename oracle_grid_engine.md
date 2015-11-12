# Oracle Grid Engine
*Oracle Grid Engine* (anciennement *Sun Grid Engine*) est un système de gestion de grille informatique. Il permet de gérer le plus efficacement possibles les ressources d'un grand nombre de serveurs (jusqu'à plusieurs centaines de milliers).
Le système de file d'attente de *Sun Grid Engine* peut s'avérer utile lorsque l'on a de nombreuses tâches à effectuer et que l'on souhaite distribuer ces tâches sur un cluster de machines. Tout l'intérêt d'un cluster est de pouvoir effectuer certaines tâches en quelques heures là où un seul serveur mettraient des jours voir des semaine, si la tâche est parallélisable. Ces tâches peuvent être très variées : il peut s'agir de la conversion fichiers d'un format vers un autre, ou encore d'une simulation informatique acceptant différents paramètres d'entrées, l'utilisateur souhaitant savoir le résultat en fonction de plusieurs jeux de paramètres.

Le cluster est formé de plusieurs machines, aussi appelées **noeuds**. Ces machines peuvent être de plusieurs types.
C'est sur la **machine maître** (*master machine*) que tourne en arrière plan le démon `qmaster`, qui est central dans le fonctionnement du cluster. Il permet d'accepter les tâches soumises par les utilisateurs, d'assigner ces tâches à des noeuds du cluster, et de surveiller son utilisation. C'est via cette machine que l'utilisateur communique, l'utilisateur n'a pas d'interaction avec les autres machines du cluster
On peut également prévoir une ou plusieurs *shadow master machine*, dont le rôle est de prendre le relai de la *master machine* dans le cas d'une défaillance de celle-ci ou du démon `qmaster` qu'elle héberge.
Les autres machines du cluster sont des **machines d'exécution**, sur lesquelles tournent des démons d'exécution (*execution daemon*). Ces démons d'exécution reçoivent une tâche à effectuer du démon qmaster et exécutent cette tâche localement. La capacité de la machine à effectuer une tâche est appelée un **slot**, la plupart du temps le nombre de *slot* d'une machine correspond au nombre de coeurs du CPU. Une fois la tâche effectuée, le démon d'exécution averti le démon qmaster qu'un *slot* est disponible. De plus, le démon d'exécution doit envoyer un rapport périodique au démon qmaster l'informant de l'avancement des tâches, sous peine de quoi le démon *qmaster* bannit la machine de sa liste de noeuds disponible.


## Soumettre des *jobs*

Pour soumettre un job au cluster, l'utilisateur peut utiliser une des commandes de soumission comme `qsub`,  une interface graphique (qmon), ou encore écrire un programme dans l'un des *binding* proposé (C ou Java).
Dans la commande de soumission de job, il faut inclure toute les infos importantes au sujet du job :
- les machines d'exécution nécessaires
- le programme à lancer
- la mémoire nécessaire
- le temps nécessaire

Toutes ces infos sont utilisées par le démon qmaster pour planifier et gérer les job.

Pour soumettre un *job* au SGE, on peut utiliser la commande `qsub` sur la machine maître. Par exemple on peut exécuter la commande `hostname` qui permet de connaître le nom de l'hôte :

    qsub -V -b y -cwd hostname
    Your job 1 ("hostname") has been submitted

`qsub` permet de spécifier de manière précise les informations importantes du *job*. Les options utilisées ici sont :

- **-V** pour que le job ait les même variables d'environnement que le shell exécutant `qsub` (recommandé)
- **-b** permet de spécifier si le 
- **-cwd** permet de lancer le *job* à partir du même dossier que celui du lancement de `qsub`

La commande renvoie le numéro du job (1 ici) ainsi que son nom. Le job est immédiatement ajouté à la file d'attente (ou *queueing list*) de qmaster.

> Il est primordial de lancer `qsub` à partir de la machine maître et non à partir d'autres machines du cluster

On peut alors vérifier l'état de la file d'attente à l'aide de `qstat` :

    sgeadmin@master:~$ qstat
    job-ID prior name user state submit/start at queue slots ja-task-ID
    -------------------------------------------------------------------
    1 0.00000 hostname sgeadmin qw 09/09/2009 14:58:00 1
    sgeadmin@master:~$
   
Notre job est initialement en attente (*qw*), si un *slot* ayant les spécifications demandées se libère, *qmaster* peut assigner au *slot* le *job*, l'état du *job* passe alors à en cours d'exécution (*running*, *r*) :

    sgeadmin@master:~$ qstat
    job-ID  prior   name       user         state submit/start at     queue  slots     ja-task-ID
    ---------------------------------------------------------------------------
    1 0.00000 hostname   sgeadmin     r     09/09/2009 14:58:14                1
    sgeadmin@master:~$

Une fois le *job* terminé, il disparaît de la file d'attente et n'est plus affiché par `qstat`.

## Gérer les *jobs*

Il peut arriver que l'on veuille annuler certains *jobs*, la commande `qdel` est faites pour ça.
On peut souhaiter annuler tous les jobs que l'on a lancé (en cours ou en attente), on lance alors :

    qdel -u arthur

si `arthur` est votre nom d'utilisateur.
Ou alors on peut annuler un *job* spécifique en précisant son numéro (celui visible via `qstat`). Ainsi :

    qdel 1

supprime le job numéro 1.





 
Sources :
[Sun Grid Engine (SGE) QuickStart - MIT](http://star.mit.edu/cluster/docs/0.93.3/guides/sge.html)
[Beginner's Guide to Oracle Grid Engine 6.2 (pdf)](http://www.oracle.com/technetwork/oem/host-server-mgmt/twp-gridengine-beginner-167116.pdf)