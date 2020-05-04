---
layout: post
title: FCSC 2020 - SSEcret [Reverse 500]
---

Comme tous les autres challenges de reverse du FCSC, SSEcret n'avait aucun mécanisme d'antidebug, ce qui est appréciable car le principe du challenge ne repose ainsi pas sur des contournements de sécurités mais réellement sur la compréhension de la logique du programme.

L'exécutable se présente sous la forme de 2 fonctions : la première assez courte semble transformer l'input (probablement un shuffle, xor, ou changement d'encodage), et la seconde plus longue sera l'objet de notre étude approfondie.

### Fonction de transformation

La transformation de l'input est une fonction assez complexe avec de nombreuses boucles imbriquées. Une technique pour en comprendre le fonctionnement serait de lancer gdb, et d'observer plusieurs valeurs d'entrée/sortie de cette fonction. Mais pour ceux qui me connaissent, vous savez que je ne crois que ce que je guess. On peut repérer un bout de code alléchant dans la transformation :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret1.png)

Les opérations de type `3*n>>2` sont un signe clair que nous avons affaire à de la base 64. On peut donc se dire que toute cette fonction est simplement un décodeur de base 64, et cette intuition se révèlera correcte.

### Fonction de vérification

La décompilation de cette fonction dans IDA Pro fonctionne mal (ça vaut bien le coup de payer 1500 balles par an pour que ça fonctionne moins bien que Ghidra :smirk:), pour deux raisons :

- La fonction utilise des instructions et registres [SSE](https://fr.wikipedia.org/wiki/Streaming_SIMD_Extensions) (les registres xmm0 à xmm7) qui sont mal gérés
- Le code se termine bizarrement, ce qui est souvent lié à une mauvaise analyse statique d'IDA, par exemple des instructions invalides qui sont inaccessibles en pratique (spoiler alert : c'est autre chose)

Ghidra gère mieux les instructions SSE, mais on va donc faire l'exercice pour une fois de travailler exclusivement en assembleur : si si vous verrez, c'est relativement simple et on ira doucement. La fonction est composée de 3 parties :

#### Initialisation

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret2.png)

Le buffer correspondant à notre input décodé en base64 est passé en paramètre dans le registre `rdi`, et sa longueur est dans `rsi`. Tout d'abord, on vérifie que le buffer fait bien au moins 16 caractères : `cmp rsi, 10h`. Si ce n'est pas le cas, on saute à l'adresse 6039e7 qui termine le programme immédiatement.

Ensuite, on charge les 16 premiers octets du de l'input dans le registre xmm0 : `movdqu  xmm0, xmmword ptr [rdi]`, et on supprime les 16 premiers caractères dans le buffer (ils sont conservés dans xmm0) : `add rdi, 10h` et `sub rsi, 10h`.

La dernière étape consiste à initialiser deux registres :

- xmm3 = 0 : `pxor xmm3, xmm3`
- xmm2 = 0x80000000000000000000000000000000 : `movq xmm2, rax` et `pinsrq xmm2, rbx, 1`

L'initialisation de xmm2 est intéressante : Le registre xmm2 a une capacité de 128 bits, mais l'assembleur x64 ne nous permet de gérer nativement que des nombres de 64 bits. On est ainsi obligé de charger les deux moitiés du registre séparément en insérant rax puis rbx.

#### Calcul de checksum

Une checksum est ensuite calculée sur le buffer d'entrée, en effectuant 128 opérations de calcul de parité sur les bits de l'entrée.

L'un de ces 128 blocs est présenté ci-dessous :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret3.png)

Décomposons le bloc comme nous l'avons fait pour l'initialisation :

- Tout d'abord, on décale xmm2 d'1 bit vers la droite : `pslrq xmm2, 1`
- Ensuite, on charge une constante de 128 bits dans xmm1 (cette constante est différente dans chaque bloc)
- On calcule le AND logique de xmm1 (la constante) et xmm0 (les 16 octets du buffer) : `pand xmm1, xmm0`
- On compte le nombre de bits à 1 dans le résultat du AND : `popcnt rax, rax` et `popcnt rbx, rbx` (opération effectuée sur deux moitiés de 64 bits, pour la même raison que précédemment)
- Si le nombre de bits à 1 est pair (`jz short_loc...`), on ajoute xmm2 à xmm3 (`pxor xmm3, xmm2`).

On peut analyser ces calculs selon un autre angle : les opérations de AND et de comptage de bits peuvent être considérées comme le calcul de parité d'un sous-ensemble choisi de bits de notre entrée. Prenons l'exemple suivant sur 8 bits :

```
input = 10001101
mask  = 10010011
AND   = 10000001  (parité 0 car le nombre de bits à 1 est pair)
```

Comme xmm2 est initialisé à une valeur binaire de 1 suivi de 127 zéros et qu'on le décale à chaque bloc d'un bit vers la droite, notre registre xmm3 aura des bits à 1 seulement aux emplacements qui correspondent à des parités impaires.

La checksum finale stockée dans xmm3 est finalement constituée de 128 bits correspondant aux parités de 128 sous-ensembles de bits de notre input.

#### Finalisation

La fin de notre fonction est composée de deux moitiés :

Tout d'abord, on vérifie que la checksum calculée est égale à 0x62E9EED78A6718200F72389798F7CA4F4 :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret4.png)

Si la checksum est correcte, l'algorithme se poursuit et va effectuer plusieurs opérations de chiffrement AES :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret5.png)

L'analyse ne sera pas détaillée en profondeur, mais cette partie va déchiffrer une section de mémoire en se basant sur notre buffer d'entrée, puis exécute le code ainsi déchiffré. Il s'agit sûrement de la fonction qui affichera le flag, nous sommes proches ! (ou pas)

### Analyse mathématique

Nous devons donc trouver un input de 128 bits qui vérifiera 128 conditions de parité sur des sous-ensembles de bits. Ces contraintes sont étonnamment très difficiles à résoudre pour des solveurs comme OR-Tools ou z3, mais on peut les modéliser comme un ensemble de 128 équations linéaires qui seront instantanées à résoudre avec le bon algorithme !

Les 128 variables de notre système d'équation sont les 128 bits de notre entrée. Pour chacun des 128 masques binaires, on peut aussi extraire la parité attendue (le bit correspondant dans la constante 0x62E9EED78A6718200F72389798F7CA4F4). Prenons un nouvel exemple avec 8 bits :

```
masque = 01010010
parité attendue = 1
```

Ce masque nous donnera donc l'équation `b1 xor b3 xor b6 = 1`

Si les 128 équations sont indépendantes linéairement entre elles, la solution de nos 128 bits sera unique. Mais comment trouver cette solution ?

#### Option 1 : backtracking

Dans notre équation précédente, on pourrait fonctionner itérativement sur quatre sous-solutions possibles :

```
b1  b2  b3
0   0   1
0   1   0
1   0   1
1   1   0
```

En essayant d'intégrer petit à petit de nouvelles équations, et en revenant d'un pas en arrière à chaque incohérence détectée. Cette solution fonctionne très bien pour des problèmes de type sudoku, mais on se heurte ici à un espace de recherche gigantesque avec des masques de plus de 20 bits. Les solveurs SMT comme z3 utilisent beaucoup ce genre de stratégie de recherche, et c'est ce qui explique qu'ils ne parviennent pas à trouver de solution à nos équations.

#### Option 2 : Calcul matriciel

Au lieu de considérer que nos équations sont basées sur des XOR, on peut les écrire d'une manière différente : `b1 + b3 + b6 = 1 (modulo 2)`. En effet, dans l'anneau des entier modulo 2 (appelé Z/2Z), l'addition est équivalente au XOR. Ainsi, nous récupérons un système d'équations linéaires qu'il sera très facile de résoudre avec la méthode classique.

Pour résoudre des équations linéaires, on place tous les coefficients des équations dans une matrice, puis on inverse cette matrice. Ainsi, notre équation d'exemple se traduira par une ligne `(0 1 0 1 0 0 1 0)` dans la matrice à inverser (vous remarquerez que la ligne est identique au masque.

De nombreuses bibliothèques Python permettent d'inverser des matrices, à commencer par `numpy`. Mais nous sommes ici dans un cas un peu spécial où toutes nos opérations sont modulo 2, et je n'ai pas trouvé de fonction toute prête permettant de faire cette inversion simplement. En revanche, l'algorithme d'élimination de Gauss-Jordan est très rapide à refaire soi-même, et c'est ce que j'ai choisi de faire.

### Validation du challenge (ou pas)

Il ne nous reste qu'à extraire les masques de l'exécutable et de les traduire en équations, et le tour est joué ! On encode les 128 bits ainsi trouvés en base 64, et on exécute le programme avec la bonne clé :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret6.png)

Il ne se passe absolument **rien**. Pas de flag, pas de message d'erreur ou d'encouragement... Allons inspecter dans gdb pour voir si notre input produit bien le checksum attendu :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret7.png)

En plaçant un breakpoint à l'adresse 0x6039d3, on voit bien que la valeur de xmm3 (notre checksum) est bien égale à xmm4 (checksum attendu). Le problème doit donc se situer dans le morceau de code qui est déchiffré puis exécuté, peut-être qu'il effectue une autre validation. On récupère le nouveau morceau de code déchiffré, que voici :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret8.png)

On voit que le nouveau code ressemble comme deux gouttes d'eau à celui que nous venons d'exécuter : calcul de checksum, vérification, puis déchiffrement de la section de code suivante. Et à en croire la taille de l'exécutable, cette opération se répète au moins une centaine de fois...

![cry](https://i.giphy.com/l3vQZ8ko4l0nvjm2Q.gif)

### Automatisation

Il va donc falloir être méthodique. Notre but va être de générer un nouvel exécutable où chaque section de code sera déjà déchiffrée, afin de pouvoir la charger dans IDA Pro et terminer le travail manuellement. Il faudra donc procéder itérativement, en extrayant les équations de chaque bout de code pour trouver la clé et déchiffrer le bloc suivant.

Tout d'abord, on va charger la première section de code, celle que nous avons analysée précédemment.

```python
with open('ssecret.bin','rb') as fi:
    with open('ssecret_decrypted.bin','wb') as fo:
        fo.write(fi.read(0x1050))
        for i in range(0x2c0):
            bufarr.append(fi.read(16))
        buf = b''.join(bufarr)
```

Ensuite, on extrait les valeurs des masques, et on les traduit sous forme matricielle. Comme vu dans le code assembleur, les masques sont chargés en deux moitiés donc haque masque est composé de deux valeurs `a` et `b` qu'il faudra extraire séparément dans l'assembleur.

```python
for blc in buf.split(b'\x48\xb8')[1:-1]:
    b = int.from_bytes(blc[:8], 'little')
    if b == 0x8000000000000000: continue
    bl.append(b)
for alc in buf.split(b'\x48\xbb')[2:-1]:
    a = int.from_bytes(alc[:8], 'little')
    al.append(a)
    
mat = [[None for i in range(128)] for j in range(128)]
for r in range(128):
    eq = (al[r]<<64)+bl[r]
    for bit in range(128):
        mat[r][bit] = (eq>>(127-bit)) & 1
```

Cette extraction est sale car elle récupère toutes les valeurs qui se situent derrière les opcodes `mov rax, ...` et `mov rbx, ...`. Rien n'empêche une valeur de masque de contenir les octets 48b8 ou 48bb (qui sont les opcodes détectés), ce qui faussera l'extraction. Ce problème ne se produit pas dans la première section de code (mais il survient assez rapidement, au bout de 13 déchiffrements). On peut donc se baser sur la première extraction pour créer automatiquement une table d'offsets `bidxl` et `aidxl`, qui nous permettront de récupérer les masques en se basant uniquement sur leur position dans la section de code.

```python
for blc in buf.split(b'\x48\xb8')[1:-1]:
    b = int.from_bytes(blc[:8], 'little')
    if b == 0x8000000000000000: continue
    bi = buf.index(blc[:8])
    bidxl.append(bi)
for alc in buf.split(b'\x48\xbb')[2:-1]:
    a = int.from_bytes(alc[:8], 'little')
    ai = buf.index(alc[:8])
    aidxl.append(ai)
```

Ensuite, on extrait la checksum attendue :

```python
expb = int.from_bytes(buf.split(b'\x48\xb8')[-1][:8], 'little')    
expa = int.from_bytes(buf.split(b'\x48\xbb')[-1][:8], 'little')    
expected = (expa<<64)+expb
```

Pour obtenir la clé de déchiffrement en résolvant les 128 équations, on inverse la matrice `mat` et la multiplie par le vecteur `expected`. Le code est lourd et peu intéressant, vous pourrez le retrouver sous ce writeup si cela vous intéresse.

Ensuite, il ne reste qu'à déchiffrer la prochaine section de code :

```python
nextbufarr = []
aes = AES.new(aesk, AES.MODE_ECB)
for i in range(0x2c0):
    bloc = fi.read(16)
    ctr = getctr(i)
    encd = aes.encrypt(ctr)
    encbloc = [a^b for a,b in zip(bloc, encd)]
    nextbufarr.append(bytes(encbloc))
bufarr = nextbufarr
```

Et on recommence cette opération de déchiffrement 128 fois avant que l'extraction se termine et nous laisse un bel exécutable prêt à être chargé dans IDA, et que le flag apparaisse sous nos yeux fatigués :

![img]({{ site.baseurl }}/images/2020-05-fcsc/ssecret9.png)

Merci à [\J](https://twitter.com/cryptanalyse) pour ce challenge qui était une belle promenade dans le jeu d'instructions XXE que je connaissais mal !

---

### Annexe - code source complet

```python
import binascii
import base64
from Crypto.Cipher import AES
import copy
import random
import numpy as np

def rowadd2(mat, rsrc, rdst):
    n = len(mat)
    for i in range(n):
        mat[rdst][i] = (mat[rdst][i] + mat[rsrc][i])%2
    return mat

def matinv2(matraw):
    mat = copy.deepcopy(matraw)
    n = len(mat)
    assert all(len(l)==n for l in mat)
    mati = [[int(i==j) for i in range(n)] for j in range(n)]
    for c in range(n):
        if not mat[c][c]:
            for r in range(c+1,n):
                if mat[r][c]:
                    mat = rowadd2(mat,r,c)
                    mati = rowadd2(mati,r,c)
                    break
            else:
                assert False, 'not invertible'
        for r in range(c+1,n):
            if mat[r][c]:
                mat = rowadd2(mat,c,r)
                mati = rowadd2(mati,c,r)

    for c in range(n):
        for r in range(c):
            if mat[r][c]:
                mat = rowadd2(mat,c,r)
                mati = rowadd2(mati,c,r)
    return mati

def getctr(n):
    ar = []
    for i in range(16):
        ar.append(n%256)
        n //= 256
    return bytes(ar)

k = base64.b64decode('eqFUxbL2zNoSFXuPo3P64A==')
aes = AES.new(k, AES.MODE_ECB)

bufarr = []

aidxl = []
bidxl = []
with open('ssecret.bin','rb') as fi:
    with open('ssecret_decrypted.bin','wb') as fo:
        fo.write(fi.read(0x1050))
        for i in range(0x2c0):
            bufarr.append(fi.read(16))

        for itr in range(250): # TODO 200
            print(itr)
            buf = b''.join(bufarr)
            fo.write(buf)
            bl = []
            al = []
            if itr == 0:
                for blc in buf.split(b'\x48\xb8')[1:-1]:
                    b = int.from_bytes(blc[:8], 'little')
                    if b == 0x8000000000000000: continue
                    bi = buf.index(blc[:8])
                    bidxl.append(bi)
                for alc in buf.split(b'\x48\xbb')[2:-1]:
                    a = int.from_bytes(alc[:8], 'little')
                    ai = buf.index(alc[:8])
                    aidxl.append(ai)
            for bi in bidxl:
                blc = buf[bi:bi+8]
                b = int.from_bytes(blc, 'little')
                bl.append(b)
            for ai in aidxl:
                alc = buf[ai:ai+8]
                a = int.from_bytes(alc, 'little')
                al.append(a)
                
            print(len(al))
            print(len(bl))
            mat = [[None for i in range(128)] for j in range(128)]
            for r in range(128):
                eq = (al[r]<<64)+bl[r]
                if r == 0 or r == 127:
                    print('eq first/last', hex(eq))
                for bit in range(128):
                    mat[r][bit] = (eq>>(127-bit)) & 1

            expb = int.from_bytes(buf.split(b'\x48\xb8')[-1][:8], 'little')    
            expa = int.from_bytes(buf.split(b'\x48\xbb')[-1][:8], 'little')    
            expected = (expa<<64)+expb
            print('expected', hex(expected))

            if itr == 128:
                break
            
            expvec = []
            for bit in range(128):
                expvec.append([(expected>>(127-bit)) & 1])
            expnp = np.matrix(expvec)
            mati = np.matrix(matinv2(mat))
            keyvec = (mati*expnp)%2
            key = 0
            for bit in keyvec:
                key <<= 1
                key += int(bit[0,0])
            aesk = key.to_bytes(16, byteorder='little')
            print(aesk)
            print(hex(key))

            nextbufarr = []
            aes = AES.new(aesk, AES.MODE_ECB)
            for i in range(0x2c0):
                bloc = fi.read(16)
                ctr = getctr(i)
                encd = aes.encrypt(ctr)
                encbloc = [a^b for a,b in zip(bloc, encd)]
                if i == 0:
                    print(encbloc)
                    print('enc(ctr)',encd.hex())
                nextbufarr.append(bytes(encbloc))
            bufarr = nextbufarr
        fo.write(fi.read())
```