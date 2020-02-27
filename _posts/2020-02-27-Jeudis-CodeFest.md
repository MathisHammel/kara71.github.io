---
layout: post
title: Solution au challenge de la CodeFest des Jeudis
---

Le challenge est disponible à l'adresse suivante : [challenge.h25.io](https://challenge.h25.io)

Il faut tout d'abord séparer les zones libres contiguës de la carte (celles composées de `.`), on peut éliminer les autres.

Pour remplir une zone vide de taille N, on doit effectuer N étapes successives de remplissage d'une case, avec à chaque fois l'option de remplir une case par la gauche ou par la droite sauf pour la dernière étape où on aura une seule option car la case de gauche est la même que celle de droite. Il existe donc 2^(N-1) manières de remplir un segment continu de la carte.

Il existe cependant un cas particulier pour les zones qui touchent un bord de la carte, il n'existe qu'une manière de remplir celles-ci.

Ensuite, il faut compter le nombre de manières d'entrelacer les actions gauche/droite de chaque groupe. Pour cela, on transforme la carte comme suit :

`..###....##...#####......##` devient `AA###BBBB##CCC#####DDDDDDD##` puis `AABBBBCCCDDDDDD`. Ensuite, on compte le nombre de permutations distinctes de cette chaîne transformée en utilisant la formule combinatoire correspondante.

On multiplie donc le nombre de manières de remplir chaque zone, et on multiplie par le nombre de possibilités d'entrelacer les actions sur les différentes zones.

La solution est potentiellement très grande avec la combinatoire, et deux options s'offrent à nous :

- Garder le modulo après chaque opération pour éviter un integer overflow
- Utiliser des entiers de type BigInteger (qui peuvent contenir des nombres gigantesques) pour ne pas se soucier des overflows

On choisit la deuxième option pour plus de simplicité, ces entiers sont utilisés par défaut en Python et sont disponibles dans des bibliothèques standard en Java et JS.

Nous espérons que vous avez apprécié la compétition, malgré les problèmes rencontrés par la plateforme qui n'était pas gérée par h25.

N'hésitez pas à nous suivre sur [Twitter](https://twitter.com/h25io) pour être tenus au courant de nos prochains challenges ! Nous sommes aussi en charge des exercices de la prochaine [Battle Dev](https://battledev.blogdumoderateur.com/) avec plus de 5000 développeurs, rendez-vous le 26 mars ;)

## Solution Python

```python
import itertools
import math

def countEndgames(grid):
    ls = []
    for elt, gp in itertools.groupby(grid):
        if elt == '.':
            ls.append(len(list(gp)))
    tot = 1
    for i,g in enumerate(ls):
        if i==0 and grid[0] == '.' or i==(len(ls)-1) and grid[-1] == '.':
            continue
        tot *= 2**(g-1)
    tot *= math.factorial(sum(ls))
    for g in ls:
        tot //= math.factorial(g)
    return tot % (10**9+7)
```

## Solution Java

```java
package myapp;

import java.math.BigInteger;
import java.util.ArrayList;

public class App
{
	private static BigInteger bigFact(int n) {
		BigInteger result = new BigInteger("1");
		for(int i=1; i<=n; i++) {
			result = result.multiply(new BigInteger(i+""));
		}
		return result;
	}
	
    public static long countEndgames(char[] map) {
		long MOD = 1000000007L;
		ArrayList<Integer> groups = new ArrayList<Integer>();
		char prevc = '#';
		int runlen = 0;
		int totrun = 0;
		for(int ptr=0; ptr<=map.length; ptr++) {
			if(ptr == map.length || (prevc != map[ptr])) {
				if(prevc == '.') {
					groups.add(runlen);
					totrun += runlen;
				}
				runlen = 0;
			}
			if(ptr != map.length)
			{
				prevc = map[ptr];
				runlen++;
			}
		}
		BigInteger result = bigFact(totrun);
		for(int i=0; i<groups.size(); i++) {
			if(!((i == 0 && map[0] == '.') || ((i == groups.size() - 1) && (map[map.length - 1] == '.')))) {
				result = result.multiply(new BigInteger("2").pow(groups.get(i)-1));
			}
			result = result.divide(bigFact(groups.get(i)));
		}
		return result.mod(new BigInteger(MOD+"")).longValue();
    }
}
```

## Solution Javascript

```javascript
module.exports = function countEndgames(grid){
    var groups = [];
    var prevc = '#';
    var runlen = 0;
    var totrun = 0;
    for(var ptr=0;ptr<=grid.length;ptr++){
        if(ptr===grid.length || prevc !== grid[ptr]){
            if(prevc==='.'){
                groups.push(runlen);
                totrun += runlen;
            }
            runlen = 0;
        }
        if(ptr !== grid.length){
            prevc = grid[ptr];
            runlen++;
        }
    }
    console.log(groups);
    var result = bigFact(totrun);
    for(var i=0;i<groups.length;i++){
        if(!(
            (i===0 && grid[0]==='.') || 
            (
                (i===groups.length-1) && 
                (grid[grid.length-1] ==='.')
            )
        )) {
            result *= 2n ** BigInt(groups[i]-1);
        }
        console.log(result);
        result /= bigFact(groups[i]);
    }
	return Number(result % 1000000007n);
};
```