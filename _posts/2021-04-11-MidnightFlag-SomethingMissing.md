---
layout: post
title: Midnight Flag CTF - Indicateur 2 Compromis [Crypto 335]
---

Au lancement du Midnight Flag CTF, ce challenge était le dernier de la catégorie crypto (avant que le chall sur qiskit soit ajouté). On nous donnait un flag chiffré avec la fonction de chiffrement associée.

![]({{ site.baseurl }}/images/2021-04-midnightflag/missing1.png)

On reconnaît ici la structure de base d'une fonction de chiffrement AES.

Les fonctions subBytes, mixColumns et shiftRow sont également fournies, mais elles ne semblent pas présenter de souci particulier. J'étais initialement parti sur cette piste comme le challenge valait un nombre conséquent de points, en pensant que les S-boxes avaient par exemple été modifiées pour affaiblir le cryptosystème.

En réalité, la solution est bien plus simple : en regardant de plus près le code de la fonction AES_Encryption, on peut constater que toutes les sous-clés du key schedule sont calculées, mais elles ne sont jamais utilisées ! Il manque en effet la fonction AddRoundKey, ce qui signifie que le chiffrement ne prend pas du tout en compte la clé, nous facilitant ainsi grandement la tâche de décryptage.

Pour récupérer le flag, on peut le passer dans une fonction de déchiffrement AES sur laquelle on aura retiré toutes les occurrences de AddRoundKey. J'ai choisi d'utiliser [cette implémentation](https://github.com/boppreh/aes) dont je m'étais déjà servi lors du FCSC 2020 pour le challenge Keykoolol.

![]({{ site.baseurl }}/images/2021-04-midnightflag/missing2.png)

Il ne reste qu'à appeler la fonction decrypt_block en lui passant le bloc chiffré qui nous est fourni, avec n'importe quelle clé de 16 bytes (comme le déchiffrement est le même quelle que soit la clé). On récupère ainsi le flag sans avoir besoin de bruteforce ni d'utiliser son cerveau :)