---
layout: post
title: FCSC 2020 - Slowppiness [Reverse 200]
---

Le challenge slowppiness est l'un de ceux qui m'a le plus donné de mal, malgré ses 200 points. Ce challenge est basé sur un algorithme vicieux qu'il faudra simplifier afin d'obtenir le flag.

Il n'y a aucun input à fournir, le programme calcule le flag tout seul et l'affiche. Le seul problème est que l'algorithme est extrêmement lent et qu'il lui faut plusieurs semaines pour afficher le sésame. Nous allons donc devoir lui donner un coup de pouce en l'optimisant beaucoup.

### Analyse initiale

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi1.png)

Le programme est livré avec 4 fichiers de différentes tailles contenant des données qui semblent aléatoires, chaque fichier est passé dans une fonction qui les transforme. Les 3 premiers fichiers servent de vérification car la sortie de la fonction de transformation est comparée à la valeur attendue.

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi2.png)

Le dernier fichier est traité puis affiché entre `FCSC{...}`, et c'est donc le résultat de la transformation de celui-ci que l'on cherche pour avoir le flag.

### Fonction de transformation

Avant de transformer le contenu du fichier, celui-ci est haché et comparé avec une constante pour vérifier que son contenu n'est pas corrompu. Le contenu est chargé sous forme d'un tableau d'entiers de 64 bits.

Ensuite, on entre dans une boucle qui va effectuer :

- Deux appels successifs vers une fonction `sub_C80`
- Fénération d'un entier pseudo-random `dword_202030` en utilisant un LCG, ou [Générateur congruentiel linéaire](https://fr.wikipedia.org/wiki/G%C3%A9n%C3%A9rateur_congruentiel_lin%C3%A9aire)
- Comparaison de deux valeurs du tableau. Si la première valeur est plus grande que la seconde, on effectue des opérations à base de xor sur les deux nombres.

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi3.png)

La boucle ne semble pas infinie à première vue, le souci de performance doit donc se situer dans les appels à `sub_C80`. Cette fonction est effectivement une horreur, il m'est impossible de montrer tout le code (autant pour des raisons d'affichage d'images que pour préserver votre santé mentale), mais en voici le flow chart pour vous donner une idée :

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi4.png)

![sad](https://www.reactiongifs.com/wp-content/uploads/2013/05/flame-war.gif)

Cette fonction effectue des appels à elle même dans tous les sens et comporte pas moins de 9 boucles imbriquées. Un début d'analyse statique me fait penser à un algorithme divide & conquer dont le principe est le suivant :

- On divise les données en deux moitiés
- On résout la moitié de gauche
- On résout la moitié de droite
- On combine les deux solutions partielles

Dans un algorithme divide & conquer, la fonction résout les deux moitiés des données en faisant des appels récursifs à elle-même, jusqu'à obtenir une donnée élémentaire qui est triviale à résoudre. Un cas typique d'un tel algorithme est le [quicksort](https://fr.wikipedia.org/wiki/Tri_rapide).

Ici, notre fonction semble faire un découpage en sous-parties et des appels récursifs sur celles-ci, mais il est très difficile de comprendre exactement son but final. On va donc devoir passer à une analyse dynamique avec gdb pour voir ce que fait cet algorithme mystère sur les petits jeux de données fournis.

Commençons par placer un breakpoint à l'entrée de `sub_C80` :

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi5.png)

Une fois le breakpoint atteint, on peut inspecter le contenu du tableau qui est passé en paramètre de la fonction, et constater qu'il s'agit bien du contenu du fichier `data16.u32` (ils sont retournés par blocs de 4 bits à cause de l'endianness).

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi6.png)

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi7.png)

On peut aussi voir que les registres rsi et rdx contiennent les arguments 0 et 7, ce qui confirme le fait que nous allons travailler sur la première moitié du tableau de 16 éléments. On continue pour atteindre le bloc suivant :

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi8.png)

Le tableau n'a pas changé, mais l'appel récursif concerne maintenant les indices 0 à 3. On peut continuer plusieurs fois pour se faire une idée plus précise des segments concernés par chaque appel récursif, mais cette belle symétrie de découpage en deux moitiés égales est rapidement cassée. Voici une visualisation des premiers appels de fonction sur le tableau `data16.u32`:

```
########.........
####.............
##...............
#................
..#..............
##...............
#................
#................
....##...........
....#............
......#..........
....##...........
....#............
....#............
####.............
##...............
#................
..#..............
##...............
#................
#................
....##...........
....#............
....#............
###..............
##...............
#................
#................
...##............
...#.............
...#.............
###..............
##...............
#................
#................
...#.............
##...............
#................
..#..............
##...............
#................
#................
(etc)
```

On peut constater que les appels de `sub_C80` repassent souvent sur les mêmes données, et c'est peut-être là que se trouve l'optimisation de l'algorithme. Regardons maintenant comment évolue le contenu du tableau au cours de l'exécution :

```
########......... [1710739839, 653111410, 712694519, 493492906, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
####............. [1710739839, 653111410, 712694519, 493492906, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [1710739839, 653111410, 712694519, 493492906, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [1710739839, 653111410, 712694519, 493492906, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
..#.............. [653111410, 1710739839, 712694519, 493492906, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [653111410, 712694519, 493492906, 1710739839, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [653111410, 712694519, 493492906, 1710739839, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [653111410, 493492906, 712694519, 1710739839, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....##........... [493492906, 653111410, 712694519, 1710739839, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....#............ [493492906, 653111410, 712694519, 1710739839, 3174068582, 484434820, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
......#.......... [493492906, 653111410, 712694519, 1710739839, 484434820, 3174068582, 290852104, 841426976, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....##........... [493492906, 653111410, 712694519, 1710739839, 484434820, 841426976, 290852104, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....#............ [493492906, 653111410, 712694519, 1710739839, 484434820, 841426976, 290852104, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....#............ [493492906, 653111410, 712694519, 1710739839, 484434820, 290852104, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
####............. [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
..#.............. [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....##........... [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....#............ [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
....#............ [493492906, 653111410, 712694519, 1710739839, 290852104, 484434820, 841426976, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
###.............. [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
...##............ [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
...#............. [493492906, 653111410, 712694519, 841426976, 290852104, 484434820, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
...#............. [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
###.............. [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
...#............. [493492906, 653111410, 712694519, 290852104, 484434820, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 653111410, 484434820, 290852104, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 653111410, 484434820, 290852104, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
..#.............. [493492906, 653111410, 484434820, 290852104, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
##............... [493492906, 484434820, 290852104, 653111410, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [493492906, 484434820, 290852104, 653111410, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
#................ [484434820, 290852104, 493492906, 653111410, 712694519, 841426976, 1710739839, 3174068582, 4042181011, 2636282546, 797548889, 3154741035, 3359734628, 555203089, 4191037080, 1954157709]
(etc)
```

Les valeurs du tableau sont conservées tout au long de l'exécution, et sont simplement réordonnées. Les opérations qui modifient les valeurs du tableau prennent toutes la même forme :

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi9.png)

On peut donc comprendre que c'est une opération qui échange deux valeurs, mais seulement si la première est plus grande que la seconde. Il s'agit donc d'un algorithme de tri, ce que l'on pourra confirmer en affichant la valeur du tableau après l'exécution de l'algorithme :

```
[ 290852104,  484434820,  493492906,  555203089,
  653111410,  712694519,  797548889,  841426976,
 1710739839, 1954157709, 2636282546, 3154741035,
 3174068582, 3359734628, 4042181011, 4191037080]
 ```
 
Il est donc désormais très simple de se passer de cet algorithme totalement inefficace, mais une dernière étape subsiste :

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi10.png)

Après le tri, le tableau est XORé avec des valeurs aléatoires provenant de notre LCG puis la fonction renvoie son SHA256. Cependant, le LCG est initialisé à la valeur `v23`, qui correspond au nombre d'itérations effectuées par le précédent LCG. Il nous suffit de connaître ce nombre d'itérations pour calculer le tableau et le flag.

Comme le LCG était itéré plusieurs fois dans chaque appel, le nombre d'itérations est très grand et nous ne pouvons pas simplement exécuter l'algorithme pour connaître le nombre d'itérations.

### Analyse de la complexité

D'après le concepteur du challenge, la solution attendue était de faire une analyse de toutes les opérations de la fonction de tri afin de déterminer précisément combien de nombres ont été générés par le LCG.

Malheureusement, je me suis juré de ne plus jamais regarder `sub_C80` (vous aurez remarqué que je me suis bien gardé de renommer cette fonction car je n'ai aucune affection envers elle). On va donc devoir trouver une technique magique comme je les aime tant.

Une chose que l'on peut remarquer est que le nombre d'appels au générateur congruentiel linéaire est toujours le même, quel que soit le sous-tableau passé à `sub_C80` et qu'il soit trié ou non. En effet, le nombre est généré même lorsque l'on n'en a pas besoin.

Pour confirmer cette théorie, on peut regarder l'écart de notre compteur d'itérations entre le moment où `sub_C80` est appelée et le moment où elle se termine. Le nombre d'itérations effectuées ne dépend de rien d'autre que la taille du sous-tableau à trier. J'ai automatisé des commandes gdb afin d'afficher le nombre d'itérations en fonction de la taille.

```
8 41
8 41
8 41
7 28
7 28
7 28
7 28
6 18
6 18
6 18
6 18
5 11
5 11
5 11
5 11
4 6
4 6
4 6
4 6
3 3
3 3
3 3
3 3
2 1
2 1
2 1
2 1
1 0
1 0
1 0
```

Le nombre d'appels respecte une sorte de suite logique 0, 1, 3, 6, 11, 18, 28, 41, ... Rien d'évident ne saute aux yeux, mais il existe une super astuce pour trouver des motifs logiques dans des suites de nombres : il existe une encyclopédie des suites de nombres entiers nommée [OEIS](https://oeis.org/), qui offre une fonction de recherche de suites très pratique.

![img]({{ site.baseurl }}/images/2020-05-fcsc/slowppi11.png)

La suite est effectivement présente dans la base, et on a même le droit à une formule nous permettant de calculer n'importe quel terme !

#### Calcul de la suite

Notre suite [A178855](https://oeis.org/A178855) est calculée à partir des termes d'une autre suite [A033485](https://oeis.org/A033485) dont la formule récurrente est `a(n) = a(n-1) + a(floor(n/2))`.

On peut donc coder une fonction Python pour calculer le 4096e terme de la suite et ainsi initialiser notre LCG à la bonne valeur :

```python
a = [1]
for i in range(1,10000):
    a.append(a[-1] + a[i//2])

b = [0, 0]
for i in range(1,4500):
    b.append((a[2*i+1]-1)//4)
    
print(b[4096])
```

Il ne nous reste qu'à initialiser un nouveau LCG avec la bonne seed (6964875559206489366, ça aurait effectivement pris un bon siècle), puis de XORer le tableau `daya4096.u32` trié. Le hash du tableau nous donnera le flag : 

`FCSC{a9b6e18e0ccd35f23401a32873b1f9ecc38e378df0517afb58d0e210d71c72e1}`

Merci à [\J](https://twitter.com/cryptanalyse) pour la création de ce super challenge qui m'aura bien fait fondre le cerveau (et fait perdre 4 heures dans une mauvaise piste !)
