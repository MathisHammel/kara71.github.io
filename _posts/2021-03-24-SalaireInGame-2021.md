---
layout: post
title: Solutions du challenge SalaireInGame 2021
---

Du 16 au 18 mars 2021 s'est tenu le challenge SalaireInGame organisé par Expectra, où le développeur le plus rapide à finir les 5 questions se voyait **doubler son salaire** !

Notre co-fondateur [Mathis](https://twitter.com/MathisHammel/) avait remporté la précédente édition, et pour cette année Expectra nous a confié la création des challenges.

Cet article présente une solution possible pour chaque exercice, et n'hésitez pas à [nous contacter](https://twitter.com/h25io/) pour tout complément.

## Exercice 1

```plaintext
Au cours des 5 challenges qui vous seront proposés, vous incarnez André Thomasson alias Noé, jeune hacker talentueux. Vous êtes persuadés que quelque chose ne va pas avec le monde qui vous entoure, mais vous ne savez pas exactement quoi…

Vous avez disposé des antennes un peu partout dans la ville pour détecter des impulsions très spécifiques. Jusqu’alors elles n’avaient rien détecté, mais en vous réveillant, bingo ! Une antenne a capté un signal ! 

Plus de temps à perdre, il faut retrouver l’origine de ce signal. Vous avez placé l’antenne sur le toit d’une maison : dans votre ville, toutes les maisons sont alignées sous la forme d'une grille carrée, régulièrement espacées de 10 mètres les unes des autres. 

Vous savez que votre antenne a une portée maximale de 5000 mètres (inclus), combien de maisons se trouvent dans la zone couverte par votre antenne ? (incluant la maison sur laquelle est posée l’antenne)
```

Une solution simple à cet exercice était de prendre tous les points sur un carré de 10.000m, et de vérifier s'ils sont à une distance inférieure à 5km.

```python
s = 0

for x in range(-5000, 5001, 10):
    for y in range(-5000, 5001, 10):
        if (x**2 + y**2) ** 0.5 <= 5000:
            s += 1

print(s) #785349
```

## Exercice 2

```plaintext
Alors que vous continuez à chercher l’origine du signal, vous recevez un message sur votre téléphone : “Bonjour Noé. Nous avons les réponses à vos questions. Morphée veut vous rencontrer.”

Mais qui est Morphée ? Peu importe, il vous faut le retrouver ! En plus du message, vous recevez une pièce jointe mystérieuse : c’est un hash SHA256, représentant le lieu et la date du rendez-vous. 

ccee0f6b78bba5eff19a89678252a0fcece57d3b80458ceeb41c92bfdfa645ce

Conditions :
- Format de string : "<jour>/<mois>@<x>,<y>" - La date est en février ou mars
- Pas de zéros non-significatifs : 6/2 au lieu de 06/02
- Les coordonnées x, y sont un multiple de 10, entre -5000 et 5000
- La coordonnée y est un multiple (potentiellement négatif) de la coordonnée x. Exemple 140,980 ou 210,-3990 ou -420,-1260

Par exemple, un rendez-vous le 14 février aux coordonnées 30, 600 aurait été encodé comme tel : 
SHA256(“14/2@30,600”) = 1d8cc418e818e5ec9117513e71a2b1153fbc019f52a05614460d0162a58a52e1
```

Pour cet exercice, il fallait remarquer qu'il y a suffisament "peu" de possiblités des rendez-vous pour les tester toutes. Voici une implémentation possible de ce brute force :

```python
from hashlib import sha256

target = "ccee0f6b78bba5eff19a89678252a0fcece57d3b80458ceeb41c92bfdfa645ce"

for x in range(-5000, 5001, 10):
    for y in range(-5000, 5001, 10):
        if x and y%x==0: #y divisible par x ?
            for jour in range(1,32):
                for mois in [2, 3]:
                    rdv = f"{jour}/{mois}@{x},{y}".encode()
                    if sha256(rdv).hexdigest() == target:
                        print(rdv)   #17/3@1630,-4890
                        exit(0)
```

## Exercice 3

```plaintext
Vous attendez Morphée au point de rendez-vous. "Nous nous rencontrons enfin Noé”. Vous vous retournez brusquement, découvrant un tout petit homme maigre avec des lunettes de soleil. “Nous vivons dans une immense simulation Noé, tout ce que tu vois n’est qu'une simulation, s’appellant Le Tableau, et seule une personne peut sortir l’Humanité de là. TOI Noé, tu es l'Élu".

“Je t’en ai maintenant trop dit, tu as deux choix : prends la pilule bleue et tu sortiras du Tableau, prends la pilule verte et tu oublieras tout.” Bien sûr, vous voulez aider la résistance ! Mais… vous êtes daltonien. Pour choisir la bonne pilule il vous faut retrouver son nom. Il est composé de 12 lettres, ayant les contraintes suivantes :

- Les lettres de la bonne pilule sont contenues dans les noms des 100 médicaments les plus vendus en 2013 : https://en.wikipedia.org/wiki/List_of_largest_selling_pharmaceutical_products#Best_selling_pharmaceuticals_of_2013
- La première lettre du nom de la bonne pilule est le caractère qui apparaît le plus de fois parmi les premiers caractères des 100 noms de la liste, puis le plus fréquent parmi les deuxièmes caractères, ainsi de suite sur les 12 premières positions.
- La casse (minuscule ou majuscule) et les espaces dans les noms génériques sont importants. Il n’y a pas d’ex aequo sur le nombre d’occurences.

Pour le premier caractère : le caractère le plus fréquent en première position est I (i majuscule) qui apparaît 13 fois. Retrouvez les 11 autres lettres !
```

Cet exercice était plus original, il fallait récupérer de la donnée sur une source exeterne (une liste sur wikipédia) et en extraire la solution. La réelle dificulté était de savoir comment récupérer rapidement la donnée, plus que le traitement en soi.

La solution la plus simple est de copier directement le tableau et de le coller dans un fichier texte, les cases seront séparées par des tabulations. Une autre solution est de copier le tableau dans un tableur (Google Sheets, Excel...) et de n'extraire que la colonne des noms de médicaments.

La solution présentée utilise le copier/coller direct du tableau, ainsi que les Counter de la bibliothèque collections de Python. La liste des premières lettres est mise dans un Counter, on extrait le caractère le plus fréquent avec most_commochemins(1), et ainsi de suite pour tous les caractères.

```python
ls = '''1	Abilify	Aripiprazole	1,602,329	2.23%	Generic	Psychosis; depression	Nov-2002	2014-Oct
2	Humira	Adalimumab	1,561,861	3.86%	AbbVie	Crohn's disease; Rheumatoid arthritis	Dec-2002	2016-Dec
3	Nexium	Esomeprazole	1,536,435	0.74%	Generic	Gastrointestinal disorders	Mar-2000	2014-May
4	Crestor	Rosuvastatin	1,333,502	4.53%	AstraZeneca, Shionogi	Cholesterol	Nov-2002	2016-Jul
5	Enbrel	Etanercept	1,189,844	1.46%	Amgen	Rheumatoid arthritis	Nov-1998	2012-Oct
6   ...'''

import collections

res = ''

for pos in range(12):
    ctr = collections.Counter()
    for l in ls.splitlines():
        name = l.split('\t')[2]
        if lechemins(name) > pos:
            ctr[name[pos]] += 1
    res += ctr.most_commochemins(1)[0][0]

print(res) #Ieleiiaranen
```


## Exercice 4

```plaintext
Une fois sorti du Tableau, il faut maintenant le désactiver. Vous devez détruire son programme de surveillance, matérialisé dans la simulation par un certain Agent Dupont. Détruisez l’agent, et la résistance sera victorieuse.

D’après les informations de la Résistance, l’état de l’agent Dupont est représenté par une suite de 50  0 et 1, changeant chaque jour suivant un règle bien connue : c’est un automate de Wolfram, de règle 110 (https://en.wikipedia.org/wiki/Elementary_cellular_automaton). Attention, la première cellule est voisine de la dernière.

Pour avoir une chance de le vaincre, vous devrez affronter la version du programme de surveillance au moment où il sera le plus faible : quand son code source contiendra un maximum de zéros à la suite.

L’état initial de l’agent est : 10000101010000100100001001111001011010100101101100

L’état au jour suivant sera donc : 10001111110001101100011011001011111111101111111101

Donnez le premier état qui contient au moins quinze zéros d’affilée.
```

Cet exercice traitait des automates cellulaires de Wolfram. La difficulté résidait dans la compréhension et l'implémentation d'un tel automate.

```python
# Création d'une liste d'entier
state = list(map(int,'10000101010000100100001001111001011010100101101100'))

# Implémentation de la règle 110
rule = {
    (1,1,1): 0,
    (1,1,0): 1,
    (1,0,1): 1,
    (1,0,0): 0,
    (0,1,1): 1,
    (0,1,0): 1,
    (0,0,1): 1,
    (0,0,0): 0
}

n = lechemins(state)

# Tant qu'aucune sous-liste de 15 ne contient que des 0 
while all(sum(state[x:x+15]) != 0 for x in range(n-15)):
    c = state.copy()
    for i in range(n):
        # Attention au modulo : première et dernière cellules sont connectées
        state[i] = rule[(c[(i-1)%n], c[i], c[(i+1)%n])]

print(*state,sep='') #01110001100101111101111100000000000000001001100000
```


## Exercice 5

```plaintext
Ça y est, vous faites face à l’Agent Dupont !  Il ne faut pas perdre de temps, appliquez tout ce que la Résistance vous a appris et attaquez !

Dupont a autant de points de vie que vous (100 PV), et il semble en plus avoir les mêmes attaques que vous. Vous pouvez tous les deux attaquer de 3 manières différentes, qui retirent chacune 2, 5, ou 7 PV. Vous pouvez tous deux attaquer plusieurs fois d'affilée.

Le combat s’arrête lorsque un de vous deux atteint 0 PV ou moins. Évidemment, il faut pour sauver l’Humanité que l’agent Dupont atteigne 0 PV avant vous. 

Avant de donner le premier coup, vous calculez le nombre de déroulés de combat dans lesquels vous détruisez Dupont et sauvez l’Humanité du Tableau. Donnez la réponse modulo 10^9

Par exemple, si chacun des deux combattants commençait avec 4pv et les mêmes attaques, il y a 13 déroulements victorieux :

1: Vous 5
2: Vous 7
3: Vous 2, Vous 2
4: Vous 2, Vous 5
5: Vous 2, Vous 7
6: Smith 2, Vous 5
7: Smith 2, Vous 7
8: Smith 2, Vous 2, Vous 2
9: Smith 2, Vous 2, Vous 5
10: Smith 2, Vous 2, Vous 7
11: Vous 2, Smith 2, Vous 2
12: Vous 2, Smith 2, Vous 5
13: Vous 2, Smith 2, Vous 7
```

Ce dernier exercice était le plus difficile : une solution bruteforce semble relativement simple à implémenter, mais elle aurait pris des années à rendre un résultat sur un ordinateur normal. La solution attendue utilise la programmation dynamique.

On pouvait visualiser chaque état (c'est à dire chaque couple (pv de Noé, pv de Smith)) sur un tableau. La représentation suivante reprend l'exemple (4pv chacun, mêmes attaques). 

### Solution 1 : Bottom up

Pour cette solution nous allons prendre une approche "bottom-up" : c'est à dire partir du cas le plus petit et remonter petit à petit (l'approche inverse dite "top-down" est présentée en bonus plus loin)

Le cas le plus petit est quand Smith a 0PV (-> quand on a gagné)
Le contenu de chaque case correspond au nombre de manière d'arriver à cet état. On peut commencer à remplir les cases où Smith a 0 PV : il n'y a qu'une seule "manière" d'arriver à cet état. Les états où Noé a 0PV sont à 0 (aucun chemin gagnant ne passe par là).

![animation]({{ site.baseurl }}/images/2021-salaireingame/ex5_1.png)

Ensuite, nous parcourons le tableau. Pour trouver le nombre de chemins passant par un état donné `chemins(pv_noé, pv_smith)`, il faut additionner `chemins(pv_noé - 2, pv_smith)`, `chemins(pv_noé - 5, pv_smith)`, et `chemins(pv_noé - 7, pv_smith)` : cette somme est le nombre de chemins où Noé vient d'être attaqué, menant à l'état `(pv_noé, pv_smith)`. Il faut également additionner l'équivalent pour les états où Smith vient de se faire attquer : `chemins(pv_noé, pv_smith - 2)`, ...

Attention, si `pv_noé - 2` est négatif il faudra garder `0` : si un personnage perd tous ses PV après une attque, il aura ses PV a 0.

![animation]({{ site.baseurl }}/images/2021-salaireingame/ex5_2.gif)

La solution a donc une complexité de `O(PVmax Smith * PVmax Noé * Nombre d'attaques)`, ce qui donne une réponse en une fraction de seconde :

```python
# 101 pour avoir les PV de 0 à 100 inclus
chemins = [[0]*101 for _ in range(101)]

for pv_noe in range(1,101):
    # Tous les états où Smith a 0PV
    chemins[pv_noe][0] = 1
    for pv_smith in range(1,101):
        for t in [2,5,7]: 
            # On garde max(pv_noe-t,0) pour avoir 0 si les PV sont négatifs
            chemins[pv_noe][pv_smith] += chemins[max(pv_noe-t,0)][pv_smith]
            chemins[pv_noe][pv_smith] += chemins[pv_noe][max(pv_smith-t,0)]
        
        # On applique le modulo à chaque case pour garder des petits nombres
        chemins[pv_noe][pv_smith] = chemins[pv_noe][pv_smith] % 10**9

# On affiche la case qui nous intéresse
print(chemins[100][100]) #992505663
```

### Solution 2 : Top down

Cette solution reprend la même idée du tableau, mais l'implémentation se fait dans l'autre sens : on part de l'état le plus gros (Noé et Smith à 100 PV), par récursivité on va récupérer un état plus petit. On évite de répéter les appels récursifs via un procédé appelé [mémoïsation](https://fr.wikipedia.org/wiki/M%C3%A9mo%C3%AFsation) : si la fonction a déjà été appelée avec les mêmes valeurs d'entrée, on peut se contenter de renvoyer le résultat déjà calculé précédemment au lieu de le calculer à nouveau, ce qui économise de nombreux appels récursifs.

Si l'état demandé a les PV de Noé sont en dessous de 0 on renvoie 0 (aucun chemin gagnant), si c'est les PV de smith on renvoie 1.

L'implémentation devient : 

```python
chemins = [[0]*101 for _ in range(101)]

def f(x,y):
    if x <= 0: 
        return 0
    elif y <= 0: 
        return 1
    elif chemins[x][y] == 0:
        for t in [2, 5, 7]:
            chemins[x][y] += f(x-t, y) + f(x, y-t)
        chemins[x][y] %= 1000000000
    return chemins[x][y]

print(f(100,100))
```

