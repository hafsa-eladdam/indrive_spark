# InDrive Spark : 2 millions de courses analysées avec Spark

## Mon projet en une phrase
Comment faire analyser **2 millions de courses InDrive** par plusieurs "ordinateurs" qui travaillent en même temps, sans que tout ralentisse ?

## L'idée qui m'a aidée à comprendre
J'ai imaginé Spark comme une salle d'examen :

- **2 millions de copies à corriger** = mes 2 millions de courses.
- **Des piles de copies** = les partitions.
- **Des correcteurs** = les cœurs du processeur.
- **Un correcteur qui doit récupérer des copies chez les autres** = un shuffle, c'est lent.

Presque tout le projet revient à une question : comment répartir les piles pour que tous les correcteurs finissent en même temps ?

## Mon parcours, partie par partie

**A. Découper en piles.** J'ai réparti les courses en 8 partitions. Pourquoi 8 ? Parce que j'ai peu de cœurs, et que 2 à 4 partitions par cœur est une bonne règle.

**B. Spark est paresseux.** J'ai écrit toute une recette (filtrer, calculer le prix au km, faire la moyenne par quartier) et il ne s'est rien passé. Spark n'a travaillé qu'au moment du `count()`, en 0,48 s. Écrire les étapes et les exécuter sont deux moments différents.

**C. Opérations narrow ou wide.** Certaines opérations (`filter`) se font dans chaque pile sans échange. D'autres (`groupBy`) obligent à échanger des données entre piles. Je repère ces échanges grâce au mot `Exchange` dans le plan d'exécution.

**D. Deux façons de faire un join.** J'ai comparé un join avec shuffle (1,52 s) et un join avec broadcast (0,79 s). Copier une petite table partout est presque deux fois plus rapide que déplacer 2 millions de lignes.

**E. Le chauffeur qui fait la moitié des courses.** Dans mes données, le chauffeur 3 prend environ 50 % des courses. Résultat : une seule partition contenait **69 %** des lignes. J'ai corrigé ça avec le salting, une astuce qui coupe le travail du chauffeur en plusieurs morceaux, puis les additionne. Les totaux sont restés exactement les mêmes.

**F. Une table qui a de la mémoire.** Avec Delta Lake, j'ai changé la note du chauffeur 3 (4,9 puis 4,95), et j'ai pu relire l'ancienne version. J'ai aussi calculé, pour chaque chauffeur et chaque jour, ses gains, son total cumulé et son classement.

## Les chiffres à retenir

| Ce que j'ai mesuré | Résultat |
|---|---|
| Partitions | 8 |
| Temps du calcul lazy | 0,48 s |
| Join avec shuffle | 1,52 s |
| Join avec broadcast | 0,79 s |
| Part de la plus grosse partition (skew) | 69 % |
| Note du chauffeur 3 | 4,9 puis 4,95 (l'ancienne version est toujours lisible) |

## Ce qui m'a posé problème
- Une erreur `NameError: name 'rides' is not defined` : la cellule de départ doit tourner avant les autres, et la session peut se réinitialiser. Maintenant, je vérifie toujours que la première cellule affiche bien ses 2 millions de lignes.
- La différence entre transformation et action. Ce qui m'a aidée : "écrire la recette" n'est pas "cuisiner le plat".

## Petit lexique avec mes mots
- **Partition** : un morceau des données.
- **Lazy** : Spark note les étapes et ne calcule qu'à la fin.
- **Shuffle** : échanger des données entre partitions (coûteux).
- **Broadcast** : copier une petite table sur chaque cœur.
- **Skew** : des données mal réparties, un morceau est beaucoup plus gros que les autres.
- **Salting** : ajouter un numéro au hasard pour répartir un morceau trop gros.
- **Delta Lake** : une table qui garde l'historique de ses versions.

## Mes fichiers
- `exercise_indrive_spark.py` : le notebook complété.
- `exercise_indrive_spark.html` : le notebook avec tous les résultats (la preuve que tout fonctionne).
- `README.md` : cette page, avec mes réponses aux questions Q1 à Q6 en bas.

## Pour le refaire
1. Créer un compte Databricks Free Edition.
2. Importer le fichier `exercise_indrive_spark.py` dans le Workspace.
3. Se connecter au calcul Serverless.
4. Lancer les cellules dans l'ordre.

Rien à téléverser : le notebook fabrique les 2 millions de courses tout seul. Quand tout est juste, les vérifications affichent `8 partitions`, `C: OK`, `E: OK` et `F: OK`.

## Mes réponses aux questions (Q1 à Q6)

Q1: J'ai mis 8 partitions parce que j'ai peu de cœurs (environ 4), donc ça fait 2 partitions par cœur. 200 c'est trop pour 2 millions de lignes. Si on a trop peu de partitions, des cœurs ne font rien et les morceaux sont trop gros. Si on en a trop, Spark perd du temps à gérer plein de petites tâches.

Q2: C'est la ligne `plan.count()` qui a lancé le job, parce que `count()` est une action. Quand j'ai construit `plan`, Spark n'a rien calculé, il a juste noté les étapes (c'est la lazy evaluation). Il a calculé seulement quand j'ai demandé le résultat (0,48 s).

Q3: Avec le broadcast, la petite table `dim_driver` est copiée sur chaque cœur, donc les 2 millions de courses n'ont pas besoin de bouger. La taille de la table est limitée par la mémoire, parce qu'elle doit tenir en entier sur chaque cœur. Il a fallu un hint parce que Spark choisit le broadcast tout seul quand la table est petite. Mes temps : shuffle join 1,52 s, broadcast join 0,79 s.

Q4: Je regarderais dans l'onglet Stages du Spark UI. Je chercherais une tâche beaucoup plus longue et plus grosse que les autres. Dans mon cas, une partition contenait 69 % des lignes à cause du chauffeur 3.

Q5: Oui, `cache()` aiderait, parce que `running` est utilisé plusieurs fois et Spark le recalculerait à chaque fois. Mais sur Databricks Free Edition (serverless), `cache()` n'est peut-être pas disponible. Comme alternative, on peut enregistrer `running` dans une table Delta et la relire.

Q6: Je mettrais en Job l'écriture de la table Delta avec le MERGE et le calcul de `running`, parce que ce sont les résultats utiles qu'on veut refaire régulièrement. Le reste, c'est de l'exploration : compter les partitions, regarder les plans avec `explain()`, comparer les temps et mesurer le skew.
