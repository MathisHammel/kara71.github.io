---
layout: post
title: FCSC 2020 - Mazopotamia [Misc 200]
---

Le challenge Mazopotamia fait appel à plusieurs compétences, notamment en automatisation et en algorithmique. J'ai obtenu le first blood sur ce challenge, qui m'a pris environ 2 heures sans difficulté majeure.

L'énoncé du problème est très simple, il faut résoudre des labyrinthes fournis sous forme d'image, en parcourant des portes de couleur dans un ordre donné. L'énoncé donne l'image suivante, dans laquelle nous devons trouver un chemin de l'entrée à la sortie en alternant obligatoirement les portes rouges et bleues :

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze1.png)

Le service nous accueille avec un bel exemple assorti de sa solution :

```
Hello stranger, welcome in Mazopotamia.
To check whether you are worthy to be here, please try
to get out of our mazes.

I will send you a PNG maze as a base64-encoded string.
Send me how you would like to move within the maze:
  N to go North
  S to go South
  E to go East
  W to go West
Along your journey, you will encounter coloured doors: you can
only cross them, but cannot stay on a cell with a door.

Important: you can only cross doors in a predetermined order,
which is listed on top of each problem.

Try to escape all the mazes in less than 30 seconds!
I will give you a beautiful flag if you succeed ;-)

Here is an example:
------------------------ BEGIN MAZE ------------------------
iVBORw0KGgoAAAANSUhEUgAAAsAAAAOACAIAAAC7YlJgAAA44ElEQVR4nO3d
d2BVhd34/3tJQGQJCgqiFkStKAXFhVpnaSviQhwoarVa96q1oOVxAYK7tmot
7j1R6vapxUHdT3GCoiguBFkKhCUQ7veP/H73uc+9ScgnCTlJeL3+Sk7OPfkk
RPPOmelMJpMCAIhokvQAAEDDIyAAgDABAQCECQgAIExAAABhAgIACBMQAECY
...
AABhAgIACBMQAECYgAAAwgQEABAmIACAMAEBAIQJCAAgTEAAAGECAgAIExAA
QJiAAADCBAQAECYgAICw/wfUufMIea1K7gAAAABJRU5ErkJggg==
------------------------- END MAZE -------------------------
One solution to get ouf of this maze is: NENNWWNWWSSSES
```

L'image de l'exemple ressemble bien à celle qui était fournie dans l'énoncé (j'ai ajouté en vert le trajet valide proposé), ce qui est rassurant vis-à-vis du processus d'extraction des données depuis l'image que nous allons devoir implémenter :

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze2.png)

On peut aussi remarquer une autre propriété intéressante : l'intégralité du contenu de l'image (y compris l'indicateur de l'ordre des couleurs en haut à gauche) est positionnée selon une grille de carrés de 64x64 pixels. Cela aidera grandement le parsing.

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze3.png)

### Modélisation algorithmique

Nous reprendrons le parsing plus tard, intéressons-nous tout d'abord à la manière dont nous allons calculer un chemin valide. Dans ce genre de challenge impliquant de l'automatisation, il est primordial de déterminer d'abord la manière dont nous allons résoudre chaque tâche **avant** de coder le parsing des données : autant ne pas avoir à faire le travail en double après s'être rendu compte que telle ou telle structure de données était plus adaptée à notre méthode de résolution.

Tout comme la plupart des exercices d'algorithmique impliquant des labyrinthes, nous allons modéliser notre problème en utilisant des graphes. La contrainte supplémentaire des portes de couleur s'intègre assez facilement en intégrant un paramètre supplémentaire : la couleur de la dernière porte empruntée.

On peut ainsi voir notre labyrinthe comme un labyrinthe à plusieurs étages, où chaque étage représente une couleur et les portes de couleur sont des escaliers permettent de passer d'un étage à l'autre.

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze4.png)

Les escaliers ne fonctionnent que dans une seule direction, car une porte rouge nous permet de passer de bleu à rouge mais pas de rouge à bleu.

Le fonctionnement de notre graphe est illustré ici avec uniquement deux couleurs, mais il est extensible à autant de couleurs que désirable : une porte de couleur K permet de passer de l'étage de couleur K-1 à l'étage K.

Une fois notre graphe créé, il suffit d'exécuter un algorithme de recherche de chemins (par exemple un [Parcours en largeur](https://fr.wikipedia.org/wiki/Algorithme_de_parcours_en_largeur) aka BFS) entre le noeud d'entrée et le noeud de sortie, qui nous trouvera un chemin valide dans le labyrinthe.

### Parsing et automatisation

Maintenant que nous avons une idée claire des algorithmes et structures de données à mettre en oeuvre, nous pouvons passer à la phase d'automatisation.

Pour communiquer avec des sockets de manière automatisée, je recommande chaleureusement l'utilisation de la bibliothèque `pwntools` en Python, qui est bien plus stable que son homologue `socket`. Pwntools n'est pas disponible sur Windows, mais elle est 100% compatible avec le WSL donc cela ne pose pas vraiment de problème. Les images seront quant à elles traitées à l'aide de `Pillow`, un équivalent de la Python Image Library (PIL).

On peut déjà commencer par s'entraîner sur l'image d'exemple fournie :

```python
from pwn import *
import base64
from PIL import Image
from io import BytesIO

r = remote('challenges2.france-cybersecurity-challenge.fr',
           6002,
           level='info')

rec = r.recvuntil('END MAZE ---')
maze = rec.decode().split('BEGIN MAZE ---')[1].split('\n')

# Ici, on concatène toutes les lignes qui se trouvent entre les lignes BEGIN MAZE et END MAZE
b64data = ''.join(maze[1:-1])
im = Image.open(BytesIO(base64.b64decode(b64data)))

# Enregistrer l'image, toujours utile pour debug
im.save('maze.png')

print(solve(im))
```

Nous devons maintenant écrire une fonction `solve` qui prendra en entrée une image et qui renverra une string contenant un chemin valide dans le labyrinthe correspondant. 

#### Ordre des portes

Commençons par récupérer l'ordre des couleurs donné : on peut remarquer que le premier carré de couleur se situe toujours à la même position dans l'image, et chaque carré qui suit se retrouve à 128 pixels du précédent. On peut itérer en augmentant x de 128 à chaque itération, et s'arrêter dès que l'on rencontre un carré blanc.

```python
def solve(im):
    order = []
    x,y = 160, 96
    while im.getpixel((x,y)) != (255,255,255):
        order.append(im.getpixel((x,y)))
        x += 128
    print(order)
```

#### Création du graphe

Pour créer le graphe, nous allons utiliser `networkx`, qui offre des structures de données efficaces pour traiter des graphes. Cette bibliothèque nous aidera grandement plus tard dans ce challenge, et vous allez sûrement regretter de ne pas l'avoir utilisé si vous avez participé au FCSC.

Il nous reste 3 données à récupérer de l'image et à traduire en graphe :

- contenu du labyrinthe
- position et couleur de l'entrée
- position et couleur de la sortie

Comme constaté précédemment, le labyrinthe est aligné sur la grille de 64x64 pixels, ce qui nous permet de travailler dans un système de coordonnées simplifiées : chaque coordonnée est divisée par 64 pour pouvoir itérer 0, 1, 2, 3, ... au lieu de 0, 64, 128, 192, ...

On parcourt tout l'intérieur du labyrinthe, et on récupère le pixel qui se situe au centre de chaque case :

```python
nx = im.size[0]//64
ny = im.size[1]//64
graph = networkx.DiGraph()
for y in range(4, ny-3):
    for x in range(2, nx-2):
        mypx = im.getpixel((64*x+32, 64*y+32))
```

Ensuite, 3 possibilités s'offrent à nous :

#### Cas 1 : pixel noir

Cela signifie que la case est un mur. On ne crée aucune arête, il n'y a aucun chemin valide passant par cette case.

#### Cas 2 : pixel blanc

Dans ce cas, nous allons chercher parmi les 4 cases voisines si certaines sont blanches. Pour chaque voisine qui est blanche aussi, on crée une arête bidirectionnelle à chaque étage de notre labyrinthe entre la case voisine et la case actuelle.

```python
elif mypx == (255,255,255):
    for dx, dy in ((-1,0),(1,0),(0,-1),(0,1)):
        ox = x+dx
        oy = y+dy
        opx = im.getpixel((64*ox+32, 64*oy+32))
        if opx == (255,255,255):
            for colori in range(len(order)):
                graph.add_edge((x,y,colori), (ox,oy,colori))
                graph.add_edge((ox,oy,colori), (x,y,colori))
```

Le format des noeuds est `(x,y,couleur)`. On peut voir que networkx est très pratique et nous permet de créer des noeuds et arêtes simplement, sans se soucier des structures de données utilisées sous le capot.

#### Cas 3 : pixel de couleur

Lorsque l'on rencontre un pixel de couleur, cela signifie qu'une porte est présente dans la case. On peut détecter si la porte est verticale ou horizontale en observant le pixel se situant directement au dessus de celle-ci.

```python
if mypx in order:
    colorid = order.index(mypx)
    colorprevid = (colorid-1)%len(order)
    pxabove = im.getpixel((64*x+32, 64*y-32))
    if pxabove != (0,0,0): # Vertical
        graph.add_edge((x,y-1,colorprevid), (x,y+1,colorid))
        graph.add_edge((x,y+1,colorprevid), (x,y-1,colorid))
    else:
        graph.add_edge((x-1,y,colorprevid), (x+1,y,colorid))
        graph.add_edge((x+1,y,colorprevid), (x-1,y,colorid))
```

On retrouve ici le fait qu'une porte de couleur K nous permet de passer de l'étage K-1 (`colorprevid`) à l'étage K (`colorid`).

#### Parsing du départ et de l'arrivée

Afin de détecter la position des flèches, on peut regarder dans chaque case sous le labyrinthe les pixels de coordonnées (22, 5) et (22, 32) illustrées ci-dessous :

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze5.png)

```python
startnode = None
endnode = None
for x in range(2, nx-2):
    y = ny-2
    pxout = im.getpixel((64*x+22, 64*y+5))
    pxin = im.getpixel((64*x+22, 64*y+32))
    if pxin != (255,255,255):
        pxabove = im.getpixel((64*x+32, 64*y-32))
        startnode = (x, y-2, order.index(pxabove))
    if pxout != (255,255,255):
        pxabove = im.getpixel((64*x+32, 64*y-32))
        endnode = (x, y-2, (order.index(pxabove)-1)%len(order))
```

#### Résolution du labyrinthe

Il ne nous reste plus qu'à trouver un chemin dans notre graphe entre `startnode` et `endnode`, et nous aurons terminé la fonction de résolution. Pour cela, nous allons exploiter encore toute la puissance de NetworkX (elle est ici l'astuce qui vous fera regretter de ne pas avoir connu cette bibliothèque plus tôt) :

```python
path = networkx.shortest_path(graph,startnode,endnode)
```

On peut maintenant transformer ce chemin en une série de caractères N/S/E/W :

```python
result = 'N'
for i in range(len(path)-1):
    myx, myy, _ = path[i]
    nxx, nxy, _ = path[i+1]
    if myx < nxx:
        result += 'E'
    if myx > nxx:
        result += 'W'
    if myy < nxy:
        result += 'S'
    if myy > nxy:
        result += 'N'
return result + 'S'
```

Il ne reste plus qu'à intégrer la fonction dans une boucle, et nous obtenons le flag au bout du 30e labyrinthe résolu :

`Congratulations! Here is your flag: FCSC{a630a31265cdc8516f5926d636f62d4c38ed4bf743bc84db095f1cc78cd3fb7f}`

La fonction `solve` s'exécute en moins de 3 millisecondes, ce qui n'est pas étonnant au vu de la "petite" taille des labyrinthes donnés :

![img]({{ site.baseurl }}/images/2020-05-fcsc/maze6.png)

Merci à [\J](https://twitter.com/cryptanalyse) pour ce challenge qui était un mix sympa entre de l'algo et du scripting, deux disciplines que je porte haut dans mon coeur <3

---

#### Annexe - Code source complet

```python
import base64
from PIL import Image
from io import BytesIO
import networkx
from pwn import *

def solve(im):
    order = []
    x,y = 160, 96
    while im.getpixel((x,y)) != (255,255,255):
        order.append(im.getpixel((x,y)))
        x += 128
    print(order)
    nx = im.size[0]//64
    ny = im.size[1]//64
    graph = networkx.DiGraph()
    for y in range(4, ny-3):
        for x in range(2, nx-2):
            mypx = im.getpixel((64*x+32, 64*y+32))
            if mypx in order:
                colorid = order.index(mypx)
                colorprevid = (colorid-1)%len(order)
                pxabove = im.getpixel((64*x+32, 64*y-32))
                if pxabove != (0,0,0):
                    graph.add_edge((x,y-1,colorprevid), (x,y+1,colorid))
                    graph.add_edge((x,y+1,colorprevid), (x,y-1,colorid))
                else:
                    graph.add_edge((x-1,y,colorprevid), (x+1,y,colorid))
                    graph.add_edge((x+1,y,colorprevid), (x-1,y,colorid))
            elif mypx == (255,255,255):
                for dx, dy in ((-1,0),(1,0),(0,-1),(0,1)):
                    ox = x+dx
                    oy = y+dy
                    opx = im.getpixel((64*ox+32, 64*oy+32))
                    if opx == (255,255,255):
                        for colori in range(len(order)):
                            graph.add_edge((x,y,colori), (ox,oy,colori))
                            graph.add_edge((ox,oy,colori), (x,y,colori))
    startnode = None
    endnode = None
    for x in range(2, nx-2):
        y = ny-2
        pxout = im.getpixel((64*x+22, 64*y+5))
        pxin = im.getpixel((64*x+22, 64*y+32))
        if pxin != (255,255,255):
            pxabove = im.getpixel((64*x+32, 64*y-32))
            startnode = (x, y-2, order.index(pxabove))
        if pxout != (255,255,255):
            pxabove = im.getpixel((64*x+32, 64*y-32))
            endnode = (x, y-2, (order.index(pxabove)-1)%len(order))
    
    path = networkx.shortest_path(graph,startnode,endnode)
    result = 'N'
    for i in range(len(path)-1):
        myx, myy, _ = path[i]
        nxx, nxy, _ = path[i+1]
        if myx < nxx:
            result += 'E'
        if myx > nxx:
            result += 'W'
        if myy < nxy:
            result += 'S'
        if myy > nxy:
            result += 'N'
    return result + 'S'

r = remote('challenges2.france-cybersecurity-challenge.fr', 6002, level='info')

rec = r.recvuntil('END MAZE ---')
print(rec[:200])
maze = rec.decode().split('BEGIN MAZE ---')[1].split('\n')
b64data = ''.join(maze[1:-1])
im = Image.open(BytesIO(base64.b64decode(b64data)))
im.save('maze.png')
print(solve(im))

r.sendline('')
mazeid = 0
while True:
    try:
        rec = ''
        rec = r.recvuntil('END MAZE ---')
        print(rec[:200])
        maze = rec.decode().split('BEGIN MAZE ---')[1].split('\n')
        b64data = ''.join(maze[1:-1])
        im = Image.open(BytesIO(base64.b64decode(b64data)))
        r.sendline(solve(im))
    except:
        print(r.recvrepeat(1.0))
        break
```