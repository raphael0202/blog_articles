# Oracle Grid Engine
*Oracle Grid Engine* (anciennement *Sun Grid Engine*) est un système de gestion de grille informatique. Il permet de gérer le plus efficacement possibles les ressources d'un grand nombre de serveurs (jusqu'à plusieurs milliers).
Le système de file d'attente de *Oracle Grid Engine* peut s'avérer utile lorsque l'on a de nombreuses tâches à effectuer et que l'on souhaite distribuer ces tâches sur un cluster de machines. Tout l'intérêt d'un cluster est de pouvoir effectuer certaines tâches en quelques heures là où un seul serveur mettraient des jours voir des semaine, si la tâche est parallélisable. Ces tâches peuvent être très variées : il peut s'agir de la conversion de fichiers d'un format vers un autre, ou encore d'une simulation informatique pour laquelle on veut tester l'effet des valeurs des paramètres du modèle,...

Personnellement, j'ai l'occasion d'utiliser *Oracle Grid Engine* pour analyser le génome de patients atteint d'une maladie. Plusieurs étapes sont nécessaires, dont certaines qui nécessitent un temps de calcul de plusieurs heures/jours. Et comme je dois analyser le génome de 150 patients, le temps de calcul serait trop long sur un ordinateur classique, même puissant.
Mon laboratoire ayant accès à des infrastructures de calcul, je me suis donc tourné vers le cluster de calcul disponible tournant sous *OGE*. C'est dans ce genre de cas où un système de gestion de grille comme *OGE* est pertinent, car il permet de faire l'analyse des 150 génomes en parallèle.

## Et en pratique, comment ça marche ?

En pratique, *Oracle Grid Engine*, comme tout DRM (*distributed resource managers*), permet à l'utilisateur de planifier l'exécution d'un grand nombre de tâche (ou *job*), et de lui laisser le soin d'occuper au mieux le *cluster* en fonction des ressources disponibles (nombre de coeur des processeurs et RAM principalement), et des *jobs* des autres utilisateurs du *cluster*. Si il n'y a pas de ressources disponibles pour exécuter le *job*, celui-ci est mis en attente, le temps que les ressources soient libérées par d'autres *jobs*.

Le cluster est formé de plusieurs machines, aussi appelées **noeuds**. Ces machines peuvent être de plusieurs types.
C'est sur la **machine maître** (*master machine*) que tourne en arrière plan le démon *qmaster*, qui est central dans le fonctionnement du cluster. Il permet d'accepter les tâches soumises par les utilisateurs, d'assigner ces tâches à des noeuds du cluster, et de surveiller son utilisation. C'est via cette machine que l'utilisateur communique, l'utilisateur n'a pas d'interaction avec les autres machines du cluster.
On peut également prévoir une ou plusieurs *shadow master machine*, dont le rôle est de prendre le relai de la machine maître dans le cas d'une défaillance de celle-ci ou du démon *qmaster* qu'elle héberge.
Les autres machines du cluster sont des **machines d'exécution**, sur lesquelles tournent des démons d'exécution (*execution daemon*). Ces démons d'exécution reçoivent une tâche à effectuer du démon qmaster et exécutent cette tâche localement. La capacité de la machine à effectuer une tâche est appelée un **slot**, la plupart du temps le nombre de *slot* d'une machine correspond au nombre de coeurs du CPU de cette machine. Une fois la tâche effectuée, le démon d'exécution averti le démon qmaster qu'un *slot* est disponible. De plus, le démon d'exécution doit envoyer un rapport périodique au démon qmaster l'informant de l'avancement des tâches, sous peine de quoi le démon *qmaster* bannit la machine de sa liste de noeuds disponible.


## Soumettre des *jobs*

Pour soumettre un job au cluster, on peut utiliser une des commandes de soumission comme `qsub`,  une interface graphique (qmon), ou encore écrire un programme dans l'un des *binding* proposé (C ou Java).
Dans la commande de soumission de job, on peut inclure les informations relatives au *job*, dans le cas où on ait des exigences spécifiques :
- les machines d'exécution nécessaires
- le programme à lancer
- la mémoire minimale nécessaire
- le temps nécessaire

Toutes ces informations sont utilisées par le démon qmaster pour planifier et gérer les *jobs*.
Mais si vous n'avez pas besoin d'une architecture matérielle spécifique, il n'est pas obligatoire de fournir ces informations (sauf le programme à lancer).

Pour soumettre un *job* à *OGE*, on peut utiliser la commande `qsub` sur la machine maître. Par exemple on peut exécuter la commande `hostname` qui permet de connaître le nom de l'hôte :

    dl175comp.ens-lyon.fr% qsub -V -b y -cwd hostname
    Your job 7306157 ("hostname") has been submitted

Ici on utilise une commande (`hostname`), mais dans la plupart des cas, on indique le chemin du script à exécuter (script bash par exemple).

`qsub` permet de spécifier de manière précise les informations importantes du *job*. Les options utilisées ici sont :

- **-V** pour que le job ait les même variables d'environnement que le shell exécutant `qsub` (recommandé)
- **-b** permet de spécifier si la commande (ici `hostname`) doit être traitée comme un fichier binaire ou un script (**[y]es/[n]o**).
	- Si la valeur vaut **y**, seul le chemin du fichier binaire/script est envoyé à la machine d'exécution, le chemin n'existant pas forcément sur la machine d'exécution.
	- Si la valeur vaut **n**, alors la commande est forcément le chemin d'un script, et ce script est envoyé à la machine d'exécution
- **-cwd** permet de lancer le *job* à partir du même dossier que celui du lancement de `qsub`

La commande renvoie le numéro du job (7306157 ici) ainsi que son nom. Le job est immédiatement ajouté à la file d'attente (ou *queueing list*) de qmaster.

> Il est primordial de lancer `qsub` à partir de la machine maître et non à partir d'autres machines du cluster

On peut alors vérifier l'état de la file d'attente à l'aide de `qstat` :


    dl175comp.ens-lyon.fr% qstat
    job-ID prior name user state submit/start at queue slots ja-task-ID
    -------------------------------------------------------------------
	7306157 0.00000 hostname   rbournho     qw    11/13/2015 17:14:52                                    1 
   
Notre job est initialement en attente (*qw*), si un *slot* ayant les spécifications demandées se libère, *qmaster* peut assigner au *slot* le *job*, l'état du *job* passe alors à en cours d'exécution (*running*, *r*) :

    dl175comp.ens-lyon.fr% qstat
    job-ID prior name user state submit/start at queue slots ja-task-ID
    -------------------------------------------------------------------
	7306157 0.00000 hostname   rbournho     r    11/13/2015 17:14:52              

Une fois le *job* terminé, il disparaît de la file d'attente et n'est plus affiché par `qstat`.

En cas d'erreur, on peut obtenir des informations détaillées sur le *job* à l'aide de :
    
    qstat -j 7306157

où 7306157 est l'identifiant du *job*.

Par défaut, *stdout* et *stderr* sont redirigés vers des fichiers portant le nom de la commande (ou du script le cas échéant), suivi respectivement de o et e ainsi que l'identifiant du *job*.
Les options `-e` et `-o` permettent d'indiquer dans quel dossier ces fichiers logs seront stockés (`-e` pour *stderr* et `-o` pour *stdout*).


## Lancer un script bash

Si vous lancer des *job* relativement simples avec peu d'option ou d'arguments, vous pouvez vo
Souvent, il est préférable d'écrire toutes les options relatives au *job* dans un script bash, au lieu de le lancer directement en ligne de commande. Toutes les lignes du script commençant par `#$` sont interprétées par le démon d'exécution comme étant des paramètres de soumission de *job*. Par exemple on peut réécrire la commande de soumission de *job* ci-dessus comme :

    #!/bin/bash
    #$ -b y
    #$ -cwd
    #$ -V
    
    hostname


## Gérer les *jobs*

Il peut arriver que l'on veuille annuler certains *jobs*, la commande `qdel` est faites pour ça.
On peut souhaiter annuler tous les jobs que l'on a lancé (en cours ou en attente), on lance alors :

    qdel -u arthur

si `arthur` est votre nom d'utilisateur.
Ou alors on peut annuler un *job* spécifique en précisant son numéro (celui visible via `qstat`). Ainsi :

    qdel 1

supprime le job numéro 1.

## Conclusion
L'utilisation d'un logiciel de gestion de grille comme *OGE* peut faire gagner beaucoup de temps passé une fois que l'on sait le maîtriser correctement. La file d'attente est l'une fonctionnalités que je préfère, on peut ainsi planifier des centaines de *jobs*, qui seront exécutés une fois que les ressources seront disponibles.

Sources :
[Sun Grid Engine (SGE) QuickStart - MIT](http://star.mit.edu/cluster/docs/0.93.3/guides/sge.html)
[Beginner's Guide to Oracle Grid Engine 6.2 (pdf)](http://www.oracle.com/technetwork/oem/host-server-mgmt/twp-gridengine-beginner-167116.pdf)