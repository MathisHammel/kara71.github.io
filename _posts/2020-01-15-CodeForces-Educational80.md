---
layout: post
title: Solutions du round CodeForces Educational #80
---

On se retrouve pour le premier post de l'année, avec un peu d'algo pour changer de tous les writeups de CTF !

Parlons du round [Educational #80](https://codeforces.com/contest/1288) de la plateforme CodeForces, qui est à mon avis le meilleur site pour s'entraîner en algo avec des compétitions très régulières et des problèmes de qualité.

C'était un round typique de 2h avec 6 problèmes dont j'ai pu valider les 4 premiers (et presque un 5ème) sans trop de difficulté, ce qui m'a permis de gagner quelques points de classement :

![img main]({{ site.baseurl }}/images/2020-01-edu80/result.png)

Contrairement à mon habitude où je présente des solutions propres et travaillées aux problèmes les plus difficiles (cf. mes solutions du [problème 5](https://blog.h25.io/BattleDev-nov-2018-1/) et [problème 6](https://blog.h25.io/BattleDev-nov-2018-2/) du BattleDev saison 12) aujourd'hui je vous montre la solution que j'ai vraiment mis en oeuvre durant le round, en espérant que ça puisse vous éclairer sur la manière dont les compétitions se passent réellement. Les codes sources seront donnés sans aucune retouche, tels qu'ils ont été codés/soumis pendant le round.

Prenons les problèmes dans l'ordre.

# Problème A - Deadline

## Sujet

Adilbek doit exécuter un programme qui met **d** jours à renvoyer un résultat. Le souci est qu'il doit rendre son résultat obligatoirement dans les **n** prochains jours.

Heureusement, il a la possibilité de prendre **x** jours pour optimiser le programme avant de lancer l'exécution, et le programme s'exécutera en  `d/(x+1)` jours, arrondis à l'unité supérieure.

Il faut déterminer si Adilbek pourra rendre son résultat avant la deadline en choisissant bien le nombre de jours d'optimisation.

Les contraintes du problème sont `1 <= d, n <= 10^9`.

## Réflexion

PRemier feeling durant la lecture du sujet : c'est un exercice surprenant, normalement les problèmes A sur CodeForces sont vraiment très simples et je ne trouve pas de moyen immédiat pour trouver la valeur optimale de **x** qui minimisera `x + d/(x+1)`. J'imagine qu'il doit exister une formule qui doit sauter aux yeux de certains, mais c'est feuille blanche pour moi.

Comme souvent quand j'ai du mal à trouver une piste de solution, je code une solution naïve qui marche pour des petits testcases et j'essaie de trouver un pattern. Ici, on essaie toutes les valeurs possibles de **x** en essayant de voir laquelle minimise le nombre total de jours pour différentes valeurs de `d`.

```python
import math

d = 123

for x in range(0,1000):
    nd = x+d/(x+1)
    print(x,nd)

'''	
Output : 

0 123.0
1 62.5
2 43.0
3 33.75
4 28.6
5 25.5
6 23.571428571428573
7 22.375
8 21.666666666666664
9 21.3
10 21.18181818181818
11 21.25
12 21.46153846153846
13 21.785714285714285
14 22.2
15 22.6875
16 23.235294117647058
17 23.833333333333332
18 24.473684210526315
19 25.15
20 25.857142857142858
21 26.59090909090909
22 27.347826086956523
'''
```

L'output de ce programme simple est assez similaire pour les différentes valeurs de **d** : d'abord une descente rapide du nombre total de jours, puis une remontée progressive. (En même temps c'est ce qu'on attend d'une courbe en x+1/x, même si je ne me suis pas fait la remarque pendant le contest).

Essayons maintenant de voir où se trouve le minimum pour une valeur de **d** très grande, proche de la limite des 10^9. Même programme en itérant un peu différemment sur les x :

```python
import math

d = 10**9

for x in range(0,1000000,5000):
    nd = x+d/(x+1)
    print(x,nd)

'''	
Output : 

0 1000000000.0
5000 204960.0079984003
10000 109990.0009999
15000 81662.22251849876
20000 69997.50012499375
25000 64998.40006399744
30000 63332.22225925802
35000 63570.61226822091
40000 64999.37501562461
45000 67221.72840603543
50000 69999.60000799983
55000 73181.48760931619
60000 76666.38889351844
65000 80384.37870186612
'''
```

Le minimum est atteint aux alentours de 35000. Sans trop réfléchir, on peut donc se dire que le **x** optimal ne dépassera pas cette valeur dans les testcases, et la valeur est suffisamment basse pour se permettre de toutes les essayer.

## Solution

```python
import math

for i in range(int(input())):
    res = 'NO'
    n,d = map(int, input().split())

    for x in range(0,100000):
        nd = x+math.ceil(d/(x+1))
        if nd <= n:
            res='YES'
    print(res)
```

Aucune optimisation supplémentaire à faire, on va jusqu'à 100000 pour être large et on prend même pas la peine d'ajouter un break dans le if !

**Temps total de résolution : 5 minutes**

# Problème B - Yet Another Meme Problem

## Sujet

L'énoncé commence par une image, ce qui est très rare sur CodeForces :

![img main]({{ site.baseurl }}/images/2020-01-edu80/meme.png)

Nice. On nous explique ensuite que le but est de compter toutes les paires d'entiers positifs **a** et **b** tels que `a*b + a + b = concat(a,b)` comme montré dans l'image, et **a** et **b** ne doivent pas dépasser des seuils (respectivement **A** et **B**) donnés.

**A** et **B** sont compris entre 1 et 10^9.

## Solution

Pareil que pour le sujet précédent, après quelques reformulations de l'équation je ne vois aucune piste viable pour générer les solutions facilement. Comme précédemment, on va partir en bruteforce et espérer qu'un miracle mette un pattern évident sur notre chemin !

```python
for a in range(1,1000):
    for b in range(1,1000):
        if a*b+a+b == int(str(a)+str(b)):
            print(a,b)
```

Et comme espéré, le miracle se produit !

```
1 9
1 99
1 999
2 9
2 99
2 999
3 9
3 99
3 999
4 9
4 99
4 999
5 9
...
```

Il semblerait donc que les valeurs de **b** sont forcément composées du chiffre 9 uniquement et que toutes les valeurs de **a** fonctionnent.

J'ai fait une grosse erreur au moment de coder la solution, qui m'a coûté 20 minutes de pénalité et presque autant pour la débugger. Je m'étais convaincu que l'équation était symétrique (donc si elle est valide pour a,b elle sera valide pour b,a), ce qui est faux à cause de la fonction de concaténation des deux nombres. On passera ce moment sombre de mon existence, voici la solution que j'ai soumise.

## Solution

```python
for tc in range(int(input())):
    a,b = map(int, input().split())

    mxcnt = 0
    mxbase = 9
    while b >= mxbase:
        mxbase = 10*mxbase+9
        mxcnt += 1

    print(a*mxcnt)
```

Ici, on compte le nombre de **b** valides inférieurs à **B**, et on le multiplie tout simplement par **A** pour obtenir la réponse.

**Temps total de résolution : 24 minutes**

# Problème C - Two Arrays

## Sujet

On doit compter le nombre de manières de créer deux tableaux d'entiers **a** et **b** qui vérifient les conditions suivantes :

- les deux tableaux sont de longueur **m** donnée.
- les éléments de **a** et **b** sont tous compris entre 1 et **n** donné.
- **a** est trié en ordre croissant
- **b** est trié en ordre décroissant
- `a[i] <= b[i]` pour tout i.

Les contraintes sont `1 <= n <= 1000` et `1 <= m <= 10`.

Le nombre de solutions est potentiellement très grand, donc il faut le donner modulo 1000000007 (c'est très classique en compétition pour vérifier qu'un algo est bon sur des gros testcases sans avoir à lui faire gérer des valeurs gigantesques).

## Réflexion

Je commence enfin à reprendre mes marques, la solution me saute aux yeux assez rapidement et je commence directement l'implémentation sans passer par une nouvelle phase de bruteforce qui aurait certainement achevé ce cher Christophe Papazian :)

Il s'agira d'abord de trouver le nombre de tableaux triés en ordre croissant de longueur **m** et dont le dernier élément est **x** (pour tout **x** entre 1 et **n**). C'est un problème classique de programmation dynamique, qui va nous permettre de compter ces configurations sans les générer. J'expliquerai plus tard pourquoi on a besoin de calculer tout ça.

Un peu de notation d'abord : on notera **dp\[len\]\[elt\]** le nombre de tableaux croissants de longueur *len* et de dernier élément *elt*.

On va construire progressivement cette solution pour *len* de 1 à *l* en se basant à chaque fois sur les résultats de l'étape précédente. L'initialisation pour la longueur 1 est facile, il n'y a qu'un seul tableau valide de longueur 1 qui finit par **i**, c'est \[**i**\] donc **dp[1][\*] = 1**.

Ensuite, on va se créer deux propriétés qui sont nécessaires et suffisantes à ce qu'un tableau de longueur k soit trié :

- Les k-1 premiers éléments sont triés
- Le dernier élément est supérieur ou égal à l'avant dernier

Avec cette propriété, on peut construire **dp\[k\]** à partir de **dp\[k-1\]** en temps raisonnable, sans avoir à générer toutes les listes valides !

Maintenant qu'on a calculé **dp\[l\]**, on peut passer à la deuxième partie de la solution, et je vais enfin expliquer à quoi ça a servi de calculer **dp**.

Tout d'abord, on peut voir que la contrainte `a[i] <= b[i]` qui nous est imposée pour tout i peut être simplifiée. Comme le plus grand élément de **a** et le plus petit de **b** se trouveront forcément en dernière position, cette contrainte est équivalente à `a[-1] <= b[-1]`.

La solution de cet exercice est loin d'être triviale, mais accrochez vous on y est presque. **dp\[l\]** nous sert pour les listes triées en ordre croissant, il nous faut maintenant la même pour les listes en ordre décroissant. On pourrait s'amuser à calculer **dp2** qui nous donnerait la même chose pour les listes décroissantes, mais il possible de s'en passer complètement en voyant que **dp2\[len\]\[elt\]** = **dp\[len\]\[n - elt + 1\]**.

La réponse est égale à la somme des **dp\[len\]\[i\]** \* **dp2\[len\]\[j\]** pour toutes les paires i,j telles que **i** <= **j** (ici, **i** et **j** correspondent à **a\[-1\]** et **b\[-1\]**). Comme les contraintes de l'exercice sont assez basses, on peut ici se permettre d'énumérer toutes les paires i,j.

## Solution

```python
maxval, arrlen = map(int, input().split())

MOD = 10**9+7

dp = [[0 for i in range(maxval)] for j in range(arrlen)]

for i in range(maxval):
    dp[0][i] = 1

for i in range(maxval):
    for j in range(1,arrlen):
        for k in range(i+1):
            dp[j][i] += dp[j-1][k]
        dp[j][i] %= MOD

#print(dp)

res = 0
for i in range(maxval):
    for j in range(maxval-i):
        res += dp[-1][i]*dp[-1][j]
print(res%MOD)
```

Plus facile à coder qu'à expliquer :)

**Temps total de résolution : 12 minutes**

# Problème D - Minimax Problem

L'énoncé a été publié sur notre compte Twitter après le round, certains d'entre vous ont trouvé très rapidement !

## Sujet

On nous donne **n** listes de **m** entiers. Il faut donner deux listes **a** et **b** (pas forcément distinctes) parmi celles-ci afin de maximiser l'expression suivante (que l'on appellera **score**) : `min(max(a[i], b[i] for i in range(m)))`.

Prenons un exemple:

```
8 2 6 2 1 = a
1 6 2 1 3 = b

8 6 6 2 3 = max(a[i], b[i] for i in range(m))
      2   = score
```

Les limites sont `1 <= n <= 300000`, `1 <= m <= 8` et les éléments des tableaux ne dépassent pas 10^9.

## Réflexion

Là encore, la méthode de résolution m'apparaît assez clairement : on va à nouveau séparer la résolution en deux parties. Je pense que la technique a un vrai nom, mais le principe est de faire une recherche dichotomique qui utilise une technique gloutonne à chaque étape pour vérifier la validité du candidat considéré. On utilisera ici la recherche dichotomique pour trouver le score maximal que l'on puisse atteindre. Cette recherche se prête bien à la dichotome, car il y a un seuil en dessous duquel tous les scores sont réalisables et au dessus duquel tous les scores ne le sont pas (c'est une fonction monotone en quelque sorte).

Pour vérifier si un score est réalisable ou non, on va donc utiliser une technique gloutonne : pour chaque liste à notre disposition, on va créer un masque qui dira si chacun de ses éléments est plus petit ou plus grand que le score cible. On a 2^m masques possibles (donc 256 dans le pire cas), ce qui nous permet d'éliminer de la redondance parmi les **n** listes (deux listes de même masque sont équivalentes par rapport à un score cible donné). Ensuite, on cherchera si au moins deux des masques existants sont compatibles (= le OU logique des deux masques est vrai partout), ce qui peut se vérifier de manière exhaustive rapidement si l'on stocke les masques dans un set.

Il ne reste qu'à combiner la recherche dichotomique avec la vérification de faisabilité pour un score cible, et on peut désormais calculer le score maximal qu'il est possible d'obtenir.

Comme l'énoncé nous demande de donner **a** et **b**, il faudra faire une petite modification à notre fonction greedy mais le principe reste identique. Au moment du calcul des masques, on retiendra à chaque fois une liste qui correspond à chaque masque pour pouvoir la retrouver rapidement. Au lieu de renvoyer un booléen, la fonction retournera une solution ou une absence de solution.

## Solution

```python
def isPoss(n, arrs, nvals):
    masks = set()
    midx = {}
    for pos,arr in enumerate(arrs):
        mask = 0
        for i in range(nvals):
            if arr[i]>=n:
                mask += 1<<i
        midx[mask] = pos+1
        masks.add(mask)

    for m1 in masks:
        for m2 in masks:
            if m1|m2 == (1<<nvals)-1:
                return midx[m1], midx[m2]

    return -1, -1
                

narr, nvals = map(int, input().split())

arrs = []
for i in range(narr):
    arrs.append(list(map(int, input().split())))

mn = -1
mx = 10**9+1
while mn < mx-1:
    mid = (mn + mx) // 2
    a,b = isPoss(mid, arrs, nvals)
    if a != -1:
        mn = mid
    else:
        mx = mid - 1

for i in range(1,-1,-1):      
    a,b = isPoss(mn+i, arrs, nvals)
    if a != -1:
        print(a,b)
        break
```

La partie gloutonne se situe dans `isPoss`, et la recherche dichotomique est en deuxième partie.

**Temps total de résolution : 16 minutes**

# Conclusion

J'avais aussi une bonne idée de solution pour le problème E qui impliquait une structure de données complexe que je n'ai trouvé qu'en C++, mais ne maîtrisant plus que le python c'était assez compliqué de terminer dans l'heure qui me restait. On se contentera d'un +21 en rating, qui me ramène déjà au dessus de 2000 points :)

En espérant que cet article vous ait plu, n'hésitez pas à le partager !