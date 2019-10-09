---
layout: post
title: SIGSEGv2
---

J'ai récemment participé aux qualifications pour le CTF [SIGSEGv2](https://rtfm.re/), un événement qui aura lieu le 30 novembre à Paris. J'aurai l'honneur d'y présenter une conférence autour de l'informatique quantique et de ses impacts sur le monde de la cryptographie.

Ce writeup porte sur *Efficiency*, le challenge de Reverse-Engineering proposé par mon éternel rival et coéquipier [SIben](https://twitter.com/_SIben_). Le challenge est intéressant et agréable à résoudre, sans pièges ni techniques anti-reverse (MOVfuscator, antidebug, ...).

## Premier contact

On se retrouve face à un challenge de reverse ultra classique, avec un binaire Linux x64 fourni. Il semble que c'est un challenge compilé avec gcc sans rien qui sort de l'ordinaire, la principale difficulté sera donc de comprendre la logique du programme.

```
$ file efficiency_fixed
efficiency_fixed: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=67589639b035156448396d9b94d9f5f0c6096729, for GNU/Linux 3.2.0, stripped
```

Il y a deux versions du challenge, car la première version du programme acceptait des faux positifs. Lorsque ce genre de chose se produit en CTF, c'est généralement bénéfique de faire un diff entre les versions : sur un fichier contenant beaucoup d'infos inutiles, on peut économiser des heures de recherche en voyant quelle partie a été retravaillée. Ici le diff n'est pas vraiment utile, il apporte juste une correction à un dépassement d'entier dans un modulo.

On charge le fichier dans IDA Pro (ou tout autre outil gratuit, merci à mon ami D.L. pour la licence IDA) :

![img functions]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/functions.png)

Le challenge est composé d'une vingtaine de fonctions courtes, commençons par le main :

![img main]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/main.png)

### Fonction principale

Le main se contente de lire 20 caractères dans l'entrée standard, et envoie cette string à une fonction checker **sub_138B**. Cette dernière peut sembler très complexe à première vue avec ses *if* imbriqués, mais la logique est en réalité simple :

![img checkerflow]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/checkerflow.png)

![img checkerflow2]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/checkerflow2.png)

On peut séparer cette fonction en sous-parties.

### Initialisation

![img fstpart]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/fstpart.png)

Tout d'abord, une étape de pre-processing qui va mettre en place les variables pour l'exécution de la suite du programme. On va se concentrer sur la fonction **sub_1355** qui transforme des blocs de 4 caractères de l'entrée utilisateur sous forme d'entiers de 32 bits, en utilisant des opérations logiques bizarres.

![img endian_convert]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/endian_convert.png)

Pour comprendre le comportement de cette fonction, on peut copier-coller le code dans Python (les opérations utilisent ici les mêmes symboles entre C et Python), et jouer un peu avec.

![img endian_python]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/endian_python.png)

On constate que ces opérations servent à inverser la position des octets dans le nombre fourni. Mais à quoi ça sert ? Il s'agit en fait d'une correction liée à la manière dont le processeur représente la donnée :

![img char_dword]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/char_dword.png)

Dans les architectures x64, qui utilisent le little endian, l'ordre des octets en mémoire n'est pas le même si l'on considère des int ou des char. Il faut donc inverser la position des octets pour retomber sur nos pattes lors d'une conversion de l'un à l'autre.

La dernière ligne de l'initialisation va copier une longue section de mémoire dans une variable locale **v2**.

### Boucle d'exécution

Ensuite, le programme entre dans une boucle infinie d'exécution. Il récupère d'abord les 3 valeurs successives v2\[ptr\], v2\[ptr + 1\] et v2\[ptr + 2\].

En fonction de la valeur de v2\[ptr\], le programme va choisir (avec tous les *if* imbriqués) une fonction parmi 14 qui sera appelée avec les arguments v2\[ptr + 1\] et v2\[ptr + 2\]. Les experts en reverse auront reconnu un challenge typique de VM, où le programme du challenge a un format custom et s'exécute dans un interpréteur fait maison. C'est en quelque sorte du méta-assembleur !

Regardons de plus près quelques instructions à notre disposition :

![img sub_1256]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/sub_1256.png)

Cette fonction compare les deux valeurs pointées par ses arguments, et met à jour un bit en mémoire qui note si ces valeurs sont égales ou non. Ce type d'instruction est généralement suivi d'une instruction conditionnelle pour reproduire un if, for ou while. Voici l'un de ces sauts conditionnels implémentés dans cette VM :

![img sub_1207]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/sub_1207.png)

Si les deux valeurs sont égales, la variable ptr sera modifiée. Cela correspond à un jump conditionnel vers une autre instruction.

Plusieurs autres instructions classiques sont implémentées, par exemple xor, add, exit. L'une des fonctions est cependant plus complexe que les autres :

![img sub_1207]({{ site.baseurl }}/images/2019-10-SIGSEGv2-Efficiency/sub_12c6.png)
 
Un oeil avisé saura reconnaître ici l'algorithme d'[exponentiation modulaire rapide](https://fr.wikipedia.org/wiki/Exponentiation_rapide#Algorithme), équivalent à la fonction `pow(x, power, mod)` de Python. Le nombre et le modulo sont donnés en argument, mais la puissance est fixe (dword_4074) et vaut 65537. C'est au tour des amateurs de crypto de se réveiller, puisque 65537 (aussi écrit 0x10001) est l'exposant le plus répandu dans le cryptosystème RSA.

### Reconstruction du programme

Une fois que toutes les instructions ont été identifiées, on peut passer au désassemblage du programme de la VM. On commence par récupérer le contenu de la section mémoire qui nous intéresse, c'est très simple avec IDAPython :

```python
prog = ' '.join(map(lambda b:b.encode('hex'), idc.GetManyBytes(0x2020, 0x180)))
```

Ensuite, on reconstitue les instructions et les arguments dans un format lisible. Ici on se base uniquement sur le premier caractère de l'opcode car celui-ci est unique à chaque instruction :

```python
inst_map = {'789ABCDE': 'compare(mem[{arg1}], mem[{arg2}])',
            '6789ABCD': 'jmp {arg1}',
            '56789ABC': 'jmp_if_eq {arg1}',
            '456789AB': 'mem[{arg1}] += mem[{arg2}]',
            '3456789A': 'mem[{arg1}] ^= mem[{arg2}]',
            '23456789': 'mem[{arg1}] = {arg2}',
            '12345678': 'mem[{arg1}] = mem[{arg2}]',
            'EF012345': 'mem[{arg1}] -= mem[{arg2}]',
            'DEF01234': 'jmp_if_diff {arg1}',
            'CDEF0123': 'mem[{arg1}] = pow(mem[{arg1}], 65537, mem[{arg2}])',
            'BCDEF012': 'set_status({arg1})',
            'ABCDEF01': 'exit',
            '89ABCDEF': 'mem[{arg1}] <<= mem[{arg2}]',
            '9ABCDEF0': 'mem[{arg1}] >>= mem[{arg2}]'}

for i in range(0, len(prog), 12):
    instr = prog[i+3] + prog[i+2] + prog[i+1] + prog[i]
    
    arg1 = int(prog[i+7] + prog[i+6] + prog[i+5] + prog[i+4], 16)
    arg2 = int(prog[i+11] + prog[i+10] + prog[i+9] + prog[i+8], 16)

    print(str(i//12).zfill(2), inst_map[instr].format(arg1=arg1, arg2=arg2))
```

Le programme implémenté dans la VM est le suivant :

```
00 mem[4] = 3
01 jmp 4
02 set_status(1023)
03 exit
04 mem[5] = 65537
05 mem[256] = 19190605
06 mem[512] = 348201767
07 mem[257] = 801200469
08 mem[513] = 867023011
09 mem[258] = 369248485
10 mem[514] = 533432993
11 mem[259] = 93788620
12 mem[515] = 354559883
13 mem[260] = 195189490
14 mem[516] = 428166631
15 mem[16] = pow(mem[16], 65537, mem[512])
16 compare(mem[16], mem[256])
17 jmp_if_diff 2
18 mem[17] = pow(mem[17], 65537, mem[513])
19 compare(mem[17], mem[257])
20 jmp_if_diff 2
21 mem[18] = pow(mem[18], 65537, mem[514])
22 compare(mem[18], mem[258])
23 jmp_if_diff 2
24 mem[19] = pow(mem[19], 65537, mem[515])
25 compare(mem[19], mem[259])
26 jmp_if_diff 2
27 mem[20] = pow(mem[20], 65537, mem[516])
28 compare(mem[20], mem[260])
29 jmp_if_diff 2
30 mem[1023] = 1
31 jmp 2
```

C'est un script très simple que l'on peut résumer en 5 conditions que les 5 blocs d'input doivent vérifier :

```
pow(input_block[0], 65537, 348201767) == 19190605
pow(input_block[1], 65537, 867023011) == 801200469
pow(input_block[2], 65537, 533432993) == 369248485
pow(input_block[3], 65537, 354559883) == 93788620
pow(input_block[4], 65537, 428166631) == 195189490
```

### Résolution des conditions

Afin de vérifier ces conditions, deux options s'offrent à nous. On pourrait tester les 2^32 valeurs possibles pour chaque bloc, ce qui prendrait environ 1h de calcul sur un bon CPU, avec un bruteforce intelligent qui élimine les caractère non-imprimables.

```python
while pow(i, 65537, 348201767) != 19190605:
	i += 1
print(i)
```

L'autre option est d'utiliser les propriétés de RSA pour retrouver instantanément les valeurs attendues. On factorise chaque module (la factorisation naïve en testant tous les facteurs de 1 à sqrt(n) est très rapide), ce qui nous sert à retrouver l'exposant de déchiffrement. Ensuite, on applique la fonction de déchiffrement RSA qui nous donne directement le morceau du flag attendu :

```python
cs = [19190605, 801200469, 369248485, 93788620, 195189490]
ns = [348201767, 867023011, 533432993, 354559883, 428166631]

for i in range(5):
    n = ns[i]
    p = 2
    while n%p:
        p+=1
    q = n // p
    c = cs[i]
    e = 0x10001

    d = modinv(e, (p-1)*(q-1))
    
    plain = pow(c,d,n)
    print('Block',i,':', long_to_bytes(plain))
```

Et le résultat s'affiche instantanément :

```
Block 0 : sigs
Block 1 : egv{
Block 2 : VM3d
Block 3 : \_stu
Block 4 : ff!}
```

On peut maintenant valider avec le flag **sigsegv{VM3d_stuff!}** pour gagner 1337 points !