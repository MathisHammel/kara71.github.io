---
layout: post
title: Norzh CTF 2020 - La crypto quantique, c'est fantastique
---

Cette semaine à l'occasion du FIC, j'ai participé à plusieurs CTFs avec l'équipe des Sogeti Aces of Spades.

Pour sa deuxième édition, le NorzhCTF était organisé par 4 étudiants de l'ENSIBS sur un environnement réaliste : le réseau d'une centrale nucléaire était simulé sur un Cyber Range, et le but était de pénétrer plusieurs couches de sécurité sans déclencher les sondes de sécurité du SI.

Il y avait aussi quelques challenges dans un format plus standard sur lesquels j'ai passé une bonne partie de la soirée (étant une bille en pentest), en particulier le challenge "Quantum encryption" dont nous allons parler dans ce post. J'ai trouvé une solution originale que l'on pourra qualifier d'arnaque totale, absolument pas prévue par le créateur :)

## Sujet du challenge

Vous l'aurez compris au titre, ce challenge de cryptographie portera sur de la technologie quantique. On nous donne un fichier chiffré, ainsi qu'un algorithme de chiffrement implémenté en langage OpenQASM, qui est le langage bas niveau utilisé pour les machines quantiques chez IBM.

Le but est bien évidemment d'inverser le processus de chiffrement pour retrouver le flag. Le code source est le suivant :

```
// quantum encryption
OPENQASM 2.0;
include "qelib1.inc";

qreg q[4];
creg c[4];

barrier q[0],q[1],q[2],q[3];
cx q[0],q[1];
x q[1];
barrier q[0],q[1],q[2],q[3];
cx q[1],q[2];
barrier q[0],q[1],q[2],q[3];
cx q[2],q[3];
barrier q[0],q[1],q[2],q[3];
ccx q[0],q[2],q[1];
barrier q[0],q[1],q[2],q[3];
swap q[0],q[3];
swap q[1],q[3];
swap q[2],q[3];
barrier q[0],q[1],q[2],q[3];
cx q[1],q[3];
x q[3];
barrier q[0],q[1],q[2],q[3];
cx q[1],q[0];
ccx q[2],q[0],q[1];
cx q[1],q[0];
barrier q[0],q[1],q[2],q[3];
measure q[0] -> c[0];
measure q[1] -> c[1];
measure q[2] -> c[2];
measure q[3] -> c[3];
```

Une visualisation de l'algorithme est fournie avec les sources du programme :

![img]({{ site.baseurl }}/images/2020-01-norzh/circuit.png)

Ce programme prend 4 bits quantiques en entrée et produit 4 bits en sortie. Les seules opérations présentes sont des portes CNOT et Toffoli, ce qui signifie qu'il n'y a aucune superposition : les qubits se retrouvent donc réduits à des bits de valeur 0 ou 1 mais jamais les deux.

Pour chiffrer des messages, l'entrée est découpée en sous-parties de 4 bits qui seront chiffrées individuellement.

![img]({{ site.baseurl }}/images/2020-01-norzh/blocks.png)

## Solution attendue

La solution attendue était de créer le circuit logique inverse, comme l'absence d'intrication quantique rend le programme réversible.

Cette solution est présentée par [Baptiste "Creased" Moine](https://twitter.com/creased_), le concepteur du challenge lui-même, dans un article très complet que vous pourrez retrouver [ici](https://www.aperikube.fr/docs/norzhctf_2020/quantum/). Le circuit inverse donné par Baptiste est le suivant, il suffit de l'exécuter sur chaque bloc de 4 bits pour récupérer le flag :

![img]({{ site.baseurl }}/images/2020-01-norzh/decryption_circuit.png)

## Solution codebook

Au lieu de créer un circuit inverse, on peut utiliser une faiblesse du cryptosystème : la taille des blocs n'est que de 4 bits donc on peut créer un mapping exhaustif de toutes les entrées possibles vers leurs sorties correspondantes. Comme les blocs sont traités séparément (à la manière d'un [ECB](https://fr.wikipedia.org/wiki/Mode_d%27op%C3%A9ration_(cryptographie)#Dictionnaire_de_codes_:_%C2%AB_Electronic_codebook_%C2%BB_(ECB))) et qu'il n'y a pas de clé, on peut reconstituer le codebook de chiffrement et l'inverser opur obtenir le codebook de déchiffrement.

On a 16 entrées possibles, et il suffit d'exécuter le programme sur chacune de ces entrées pour recréer le codebook. Dans l'exemple suivant, on sait qu'un chiffré 0011 proviendra forcément du bloc 1011 en clair (en supposant que la transformation est biijective).

![img]({{ site.baseurl }}/images/2020-01-norzh/codebook.png)

Pour créer le mapping, on peut générer les 16 programmes et les exécuter sur une machine quantique virtuelle du toolkit Qiskit. Ensuite, il nous suffira de simuler la boîte noire de déchiffrement pour retrouver le flag.

Malheureusement, cette technique n'a pas fonctionné pour une raison que j'ignore (probablement un souci d'implémentation), mais en théorie c'est parfaitement viable. Je n'ai pas cherché trop longtemps à corriger mon programme car j'ai trouvé entre temps une nouvelle technique qui manque encore plus de respect au créateur du chall et qui m'a beaucoup fait rire.

## L'arnaque

En notation hexadécimale, 4 bits correspondent à 1 caractère. Par conséquent, ce cryptosystème correspond en réalité à une substitution sur du texte hexadécimal.

Par exemple, le message hexadécimal `6666` pourra donner `0000` ou `AAAA`, mais jamais `B46C`. En utilisant cette propriété ainsi qu'une partie du flag connue (`ENSIBS{   }`), on peut réduire la génération du codebook à un bruteforce sur un petit espace de recherche. Regardons le message chiffré et la partie connue du clair :

![img]({{ site.baseurl }}/images/2020-01-norzh/ciphertext.png)

![img]({{ site.baseurl }}/images/2020-01-norzh/plaintext.png)

Ici, le début du flag `454E` est traduite en `191A`. On peut donc déduire que le chiffrement transforme `4` -> `1`, `5` -> `9` et `E` -> `A`.

En utilisant tout le format de flag `ENSIBS{...}`, il est possible de déduire 9 substitutions sur les 16. Il nous manque 7 substitutions, ce qui correspond à 7! = 5040 déchiffrements possibles. C'est là qu'intervient l'arnaque : on va générer les 5040 flags possibles et les regarder un par un !

```python
import itertools
import binascii

missingvals = list('0168ACF')
missingkeys = list('2468DEF')

# On génère toutes les 7! combinaisons de déchiffrement
for perm in itertools.permutations(missingvals):

	# Le dictionnaire de substitution inverse, les 7 valeurs inconnues ont un X
    dct = {'0': '7',
        '1': '4',
        '2': 'X',
        '3': '2',
        '4': 'X',
        '5': 'D',
        '6': 'X',
        '7': '3',
        '8': 'X',
        '9': '5',
        'A': 'E',
        'B': 'B',
        'C': '9',
        'D': 'X',
        'E': 'X',
        'F': 'X'}
	
	# On remplit les 7 valeurs manquantes
    for i in range(7):
        dct[missingkeys[i]] = perm[i]
		
	# On déchiffre le message avec le nouveau codebook, et on l'affiche
    msg = '191A971C13970B0209718A0199859E10710119079E007291841E99709E97099619030676078C70728E1A05'
    flag = ''
    for c in msg:
        flag += dct[c]
    flag = binascii.unhexlify(flag)
    print(flag)

```

La plupart des flags affichés ont des caractères non-imprimables, mais on peut facilement deviner son contenu :

```
...
b'ENSIBS{pu4\xfetU\xfd\\G4tEs\\w0T\xf1LU7\\SuVErv6s\xf970\xfcN}'
b'ENSIBS{pu4\xfetU\xfdXG4tEsXw0T\xf1HU7XSuVErv6s\xf970\xf8N}'
b'ENSIBS{pu4\xfetU\xfd\\G4tEs\\w0T\xf1LU7\\SuVErv6s\xf970\xfcN}'
b'ENSIBS{pu4\xfetU\xfdXG4tEsXw0T\xf1HU7XSuVErv6s\xf970\xf8N}'
b'ENSIBS{pu4\xfetU\xfdZG4tEsZw0T\xf1JU7ZSuVErv6s\xf970\xfaN}'
b'ENSIBS{pu4ntUm\\G4tEs\\w0TaLU7\\SuXErx8si70lN}'
b'ENSIBS{pu4ntUm_G4tEs_w0TaOU7_SuXErx8si70oN}'
b'ENSIBS{pu4ntUmZG4tEsZw0TaJU7ZSuXErx8si70jN}'
b'ENSIBS{pu4ntUm_G4tEs_w0TaOU7_SuXErx8si70oN}'
b'ENSIBS{pu4ntUmZG4tEsZw0TaJU7ZSuXErx8si70jN}'
b'ENSIBS{pu4ntUm\\G4tEs\\w0TaLU7\\SuXErx8si70lN}'
b'ENSIBS{pu4\xaetU\xad\\G4tEs\\w0T\xa1LU7\\SuXErx8s\xa970\xacN}'
...
```

En faisant un grep sur `qu4ntUm_`, on se réduit à un petit ensemble de 12 flags, parmi lesquels le bon apparaît clairement :

```
b'ENSIBS{qu4ntUm_G4tEs_w1T`OU7_SuXErx8si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1T`OU7_SuZErz:si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1T`OU7_Su\\Er|<si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1ThOU7_SuPErp0si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1ThOU7_SuZErz:si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1ThOU7_Su\\Er|<si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TjOU7_SuPErp0si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TjOU7_SuXErx8si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TjOU7_Su\\Er|<si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TlOU7_SuPErp0si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TlOU7_SuXErx8si71oN}'
b'ENSIBS{qu4ntUm_G4tEs_w1TlOU7_SuZErz:si71oN}'
```

Flag : `ENSIBS{qu4ntUm_G4tEs_w1ThOU7_SuPErp0si71oN}`

## Conclusion

C'est toujours très amusant de trouver un flag en contournant le design du challenge, et la dernière attaque présentée n'a même pas besoin d'avoir le code Qasm de chiffrement pour casser le flag !

Merci aux organisateurs du NorzhCTF pour tous les challenges de qualité, et en particulier Creased <a href="https://twitter.com/Creased_?ref_src=twsrc%5Etfw" class="twitter-follow-button" data-show-count="false">Follow @Creased_</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Et bravo à tous mes coéquipiers des Sogeti Aces of Spades qui ont décroché la 1re et 3e place sur le podium !

![img]({{ site.baseurl }}/images/2020-01-norzh/scoreboard.png)