---
layout: post
title: Midnight Flag CTF - Le Chemin Tordu [Misc 200]
---

Pour continuer la série de writeups du CTF Midnight Flag, voici le premier challenge sur lequel je me suis penché dans la soirée.

## Le challenge

![]({{ site.baseurl }}/images/2021-04-midnightflag/chemin1.png)

Il s'agit de résoudre plusieurs équations générées aléatoirement en un temps très court, en communiquant avec le serveur via des sockets. L'énoncé du challenge nous indique que les équations se compliquent au fur et à mesure, mais cela ne devrait pas nous poser de problème comme python dispose d'une fonction `eval` qui permet d'évaluer des expressions mathématiques aussi compliquées qu'il le faut.

Ce genre de challenge se scripte assez simplement en utilisant `socket`, mais on peut en réalité s'en sortir beaucoup plus facilement en détournant l'utilisation du package `pwntools` qui propose une gestion améliorée des erreurs et des fonctions supplémentaires d'envoi/réception de données.

![]({{ site.baseurl }}/images/2021-04-midnightflag/chemin2.png)

Après avoir correctement répondu à la première question, le serveur nous en renvoie immédiatement une autre au même format :

![]({{ site.baseurl }}/images/2021-04-midnightflag/chemin3.png)

On peut donc mettre la réception, évaluation et envoi du résultat dans une boucle infinie jusqu'à recevoir le flag. J'ai décidé de mettre en place une simple précaution pour éviter de se faire piéger par les concepteurs du challenge : ici on passe des données arbitraires dans `eval`, et rien n'empêche le serveur de nous envoyer un bout de code malicieux au lieu de l'équation, qui sera directement exécuté sur notre machine. Pour se prémunir contre ce genre de blague, on fait en sorte que les équations de plus de 30 caractères passent par une revue manuelle avant d'être exécutées. En lisant dans l'énoncé que les équations devenaient de plus en plus complexes je pensais que ce serait un piège de ce type, mais finalement les équations étaient safe et le souci est différent.

![]({{ site.baseurl }}/images/2021-04-midnightflag/chemin4.png)

La voilà, la difficulté croissante des calculs ! Dans l'équation qui fait planter le script, `13 - forty-six + ninety-three`, certains nombres sont écrits en toutes lettres...

Il semblerait que les nombres restent inférieurs à 100 donc on pourrait coder une fonction toute simple, mais comme on ne sait pas si le format continue encore à se complexifier avec des nombres plus longs, c'est plus safe d'utiliser un package tel que `word2number`.

Il ne reste qu'à exécuter le script ci-dessous, qui nous donnera le flag après avoir renvoyé une cinquantaine de calculs corrects au serveur :

```python
from pwn import *
from word2number import w2n

r = remote('ctf.midnightflag.fr', 9000)

r.sendline('yes')

while True:
    try:    
        r.recvuntil('What is the result of ')

        eqraw = r.recvline().decode().split(' ?')[0]

        # On évite d'avoir un espace, pour ne pas planter sur le eqraw.split()
        eqraw = eqraw.replace('one hundred', '100')
        
        print(eqraw)
        eq = ''
        for w in eqraw.split():
            if w in '+-*/%':
                eq += w
            else:
                eq += str(w2n.word_to_num(w))
        print(eq)
        if len(eq) > 30:
            input('WARN')

        res = eval(eq)

        r.sendline(str(res))

    except:
        print(r.recv())
```

C'était un format de challenge assez connu, mais je me suis mis au défi de le résoudre en speedrun pour un temps de total en dessous de 11 minutes dont je suis très satisfait :

![]({{ site.baseurl }}/images/2021-04-midnightflag/chemin5.png)