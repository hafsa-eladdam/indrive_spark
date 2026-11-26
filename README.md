Q1: J'ai mis 8 partitions parce que j'ai peu de cœurs (environ 4), donc ça fait 2 partitions par cœur. 200 c'est trop pour 2 millions de lignes. Si on a trop peu de partitions, des cœurs ne font rien et les morceaux sont trop gros. Si on en a trop, Spark perd du temps à gérer plein de petites tâches.

Q2: C'est la ligne `plan.count()` qui a lancé le job, parce que `count()` est une action. Quand j'ai construit `plan`, Spark n'a rien calculé, il a juste noté les étapes (c'est la lazy evaluation). Il a calculé seulement quand j'ai demandé le résultat (0,48 s).

Q3: Avec le broadcast, la petite table `dim_driver` est copiée sur chaque cœur, donc les 2 millions de courses n'ont pas besoin de bouger. La taille de la table est limitée par la mémoire, parce qu'elle doit tenir en entier sur chaque cœur. Il a fallu un hint parce que Spark choisit le broadcast tout seul quand la table est petite. Mes temps : shuffle join 1,52 s, broadcast join 0,79 s.

Q4: Je regarderais dans l'onglet Stages du Spark UI. Je chercherais une tâche beaucoup plus longue et plus grosse que les autres. Dans mon cas, une partition contenait 69 % des lignes à cause du chauffeur 3.

Q5: Oui, `cache()` aiderait, parce que `running` est utilisé plusieurs fois et Spark le recalculerait à chaque fois. Mais sur Databricks Free Edition (serverless), `cache()` n'est peut-être pas disponible. Comme alternative, on peut enregistrer `running` dans une table Delta et la relire.

Q6: Je mettrais en Job l'écriture de la table Delta avec le MERGE et le calcul de `running`, parce que ce sont les résultats utiles qu'on veut refaire régulièrement. Le reste, c'est de l'exploration : compter les partitions, regarder les plans avec `explain()`, comparer les temps et mesurer le skew.