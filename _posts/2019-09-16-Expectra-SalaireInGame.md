---
layout: post
title: Concours SalaireInGame by Expectra
---

Vous l'avez peut-être vu passer, cette semaine j'ai doublé mon salaire du mois de septembre grâce à un concours d'algo.

<blockquote class="twitter-tweet"><p lang="fr" dir="ltr">Ils ont décodé les énigmes pour gagner et prouver leur valeur.<br>Bravo à eux et à tous les participants ! <a href="https://t.co/PISvhUw7WG">https://t.co/PISvhUw7WG</a> <a href="https://t.co/KwIUmUbz8Q">pic.twitter.com/KwIUmUbz8Q</a></p>&mdash; Expectra (@expectra_emploi) <a href="https://twitter.com/expectra_emploi/status/1173641306310139907?ref_src=twsrc%5Etfw">September 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

C'était organisé par Expectra, et le concours était globalement de bonne qualité malgré quelques bugs pas hyper gênants qui auraient pu être éliminés facilement en testant un peu mieux le contest.

Le format d'exercice était un équivalent [Project Euler](https://projecteuler.net), à savoir un énoncé simple et direct qui n'attend qu'une seule réponse, et il faut soumettre cette réponse pour valider l'exercice. J'aime bien ce format, il permet de ne pas se prendre la tête avec des limitations de langage et d'environnement comme tout le code est exécuté en local.

Le concours se divise en 5 problèmes. Je n'ai plus le contenu exact des énoncés, mais je tâcherai de les refaire en corrigeant les quelques erreurs d'origine.

J'ai décidé de partir en Python, qui est depuis quelques années mon unique choix de langage. Sa polyvalence me permet en général de m'en sortir dans beaucoup de compétitions de code et CTFs.

## Problème 1

Il s'agit de trouver le plus petit entier dont le SHA512 commence par `aaaaaa`. On part donc sur un script hashlib simple, pour se mettre en jambes:

```python
import hashlib
i=0
while True:
    if hashlib.sha512(str(i).encode()).hexdigest().startswith('aaaaaa'):
        0/0
    i += 1
```

Pour une fois, je vais mettre dans cet article le code que j'ai écrit pendant le concours, sans le retoucher. Par exemple le trick de `0/0` est horrible à voir mais permet de gagner quelques frappes au clavier sans se prendre la tête à choisir entre un return, break, exit(0), ...

L'algo s'exécute dans son coin pendant une petite minute avant de planter sur la division par zéro, l'interpréteur me rend alors la main et je peux récupérer la valeur de `i` qui est de `35318008`.

## Problème 2

Le second exercice traite encore de hashing. On doit retrouver un code secret généré entre deux instants donnés séparés d'environ 24h. On sait que ce code correspond au md5 de l'instant où il a été généré au format `YYMMDDhhmmss`. On sait aussi que les 4 premiers caractères du code sont égaux au 4 derniers.

Je me lance encore sur un petit script, en rétrospective il aurait été plus élégant de partir sur des objets `datetime` avec `strftime`, mais je n'avais plus la doc par coeur et j'avais peur d'y perdre trop de temps.

```python
import hashlib
YY=19
MM=7
DD=25
HH=16
mm=55
ss=0
while True:
    date='%02d%02d%02d%02d%02d%02d'%(YY,MM,DD,HH,mm,ss)
    hsh = hashlib.md5(date.encode()).hexdigest()
    if hsh[:4] == hsh[-4:]:
        print(hsh, date)
    ss += 1
    if ss == 60:
        ss = 0
        mm += 1
    if mm == 60:
        mm = 0
        HH += 1
    if HH == 24:
        HH = 0
        DD += 1
```

En une fraction de seconde, la première ligne sort. C'est la seule à matcher l'intervalle temporel donné, on peut donc soumettre le bon code secret.

## Problème 3

Celui-ci avait l'énoncé le plus succint des 5. Il faut retrouver la 100e valeur de la 200e ligne du triangle de Pascal.

Au lieu de me risquer à partir sur le calcul du coefficient binomial correspondant (99 parmi 199) qui est certainement plus rapide à faire mais moins simple à debug, je décide de générer les 200 premières lignes du triangle.

```python
prevline = [0,1]
for i in range(1,201):
    line = [0]
    for j in range(i):
        line.append(prevline[j+1]+prevline[j])
    line.append(0)
    #print(i,line)
    prevline=line

print(prevline[100])
```

Heureusement que j'utilisais python et ses entiers de taille infinie, car la réponse `45274257328051640582702088538742081937252294837706668420660` aurait fait pâlir les amateurs de C++ et de Java.

L'autre solution était d'évaluer [ncr(199,99) sur WolframAlpha](https://www.wolframalpha.com/input/?i=ncr%28199%2C+99%29), mais encore fallait-il comprendre que c'était 199,99 et non 200,100...

## Problème 4

On nous donne une longue chaîne de lettres minuscules, et la règle est simple : il faut trier la chaîne puis compter le nombre d'occurrences de chaque lettre. Ensuite, on fait la correspondance : 100 occurrences = A, 101 = B, ..., 126 = espace

On peut rapidement se rendre compte que cela ne sert à rien de trier la chaîne d'abord, le nombre d'occurrences ne changera pas. Je décide donc de compter les lettres avec un objet `collections.Counter`, puis de recomposer la chaîne avec des manipulations de codes ASCII.

```python
s = 'cnhhnhtanfreoeqitmptkaqcrjqhipabqhehanthitdposdgekjpqibceckmcmckfkcacdhggsdrdpinoedddofemdtrhjroepscdaskslgcclngaofmfprdnrodkftmgetiqrmlqimqreealabdibrmhfdqlidfltofhgrlospeemtdiholtmepcikatgprnhjlmpoglpcrktrbgdgtlkhiqssjdhtafsjffemikbjchkefgkeiarhoadgnteeeaomamnitlpfqalikbolbtrjitornkecitqsrblahlldgogceoeoeaeskmlqeshobkhckllnebsbetafesgcepsbdfkiahtjingpcjsoekkgncleesriqbcroidqorrtkhsnlikajdptptcinjpcglhsrlpnajpoctkhbikkcraaccsfrsjaeggmsbdorogdosmeofjamiqnsckgsaipjsfrasmlshcdgljcjqtpoelabkfkpjhjatigrfptbggknlpnnaojqrjodagaprahknentjarphosflnfklpeppfmtmooneracdjrjmpjcpjmknaootgscninqgoqedjffootcphihadosdefbkqhhoaotgkljknbflbjassaerfgnkaffjqgprbgntretnobsdrsbqmkseeqfbtadgopcqhpslrfoesimrhjpnhnbtidcrsqancnojmeoqphqqtbqhcregtsrlkedbqkjsgjbbpkhtjqsscfpkhtmkdaciirfgamabketrcsjdthackqfbijmgkpqnjanbesddtlfbmidbonnpsbiqbkmfhijptsphnshmlntgnskbqodngpiggqbcfcrcmkjincfdkotgcqgifialhhcjpqoocrnlebroiskplerloblhfrfajfnedfrcspcrpncchiqfqdfjsikmnhjffjlrnahdidllbnroqakpebeissgcbloqgdstnotlhknjscssqnheclkikehfbomioqfjtambhoadebcpsbsmbhhsjbkdtabbnsarghnbjgbfrrnlrmmnhitlojsondjinljfaforgdqhqpbhaongsismsdttbktlqkkfsefloooohalkmhpfdidjlssdhgeafkolncgknsmkkqgcmkdsmicmpjhmhfancllghbsmncdqpmhqdcenhtmtlafahhmdhnalgbbchlbigbnsomtqehteeodqhcngnrdfmjdlkbsinrarnciemhbljllqmnmnhrennfncljojfcrehbjgkesehghrdciebttbasdhqcsrlhplibscpatsjkjqagqreolechrfhsirjihsnchfdnaqmkpdplcnptjlflqmqtfcdcegfaeipdjqnorjtgdpjcfnlehmnqrpdlhobtchigkcdkgdmlkrbcdnirogdgmghpsihcftajoqroccacrremphrmmqgtrhfncntfjkdiitpmehgtqgehokoicqmjajfkfppklkkfdbrbimcqgkilsjjbfbnpkqglqkhllkcdobaacgjkmspldmlrgfkejribbesmmdljlbbaoftoenarodmaiasbolktbffidhfiecamdjehkqgoojanrsjohegpcnntebkbpjlorqglaqngrsqjmpgrpikgcrflkohhckqjctmhpapntmhrkipbnmrmoorglqraemdqkdhhbilpbdnjhgnnjblhctplsgtlqcenfrelmhgqdmrjjcsfljdkarnorppamnsichpeqshqjkroclegoameknrfbirodqstabbalirtnrlldopmmqdldoksrepsbpmkmtreqfpiffdekgqlkkgcjkappooolrldkmbbmnfqtrdsotjdsdqfhahdotlajgnsrmbkdsadoetpqiclqaaoqitrlijdasbbgimtqrhsgtneehgrcfbdmlofjpcqkaijtlqojstofpeltfsdantqngtajlsmebcokoeoiclfsqnkferebijamomccgklbhlbotksmlebdilliiqlnhdhkcnpmnmdsdshgaeckcpjitdsqqdktkfrsanorjnglqpgbhsospllegnkrsaimkhnccptfhbjlhpheqgcdbkgigltfanpmplaimrbqejbdlhlgeadsrbbmapccnjleknmjprktcdseipfifnddsnigdpkbobchrbgrbpfdrbqfnrlfcichsiieknhkjrhdmbqneicai'
import collections
ctr = collections.Counter(s)
flag=''
for c in 'abcdefghijklmnopqrst':
    flag += chr(ctr[c] - 101 + ord('A') + 1)
print(flag)
```

La chaîne de sortie est `JOURNEE[DEV[EXPECTRA`, les espaces sont remplacés par des crochets car c'est le caractère qui vient après Z dans la table ASCII.

## Problème 5

Le dernier problème est similaire au précédent, mais il faut cette fois-ci faire la transformation inverse, et ne pas faire le mélange. Pour valider le dernier exercice, il faudra donc transformer la chaîne `expectra`.

J'aurais pu utiliser le fait que la chaîne `expectra` apparaissait dans le challenge précédent, mais on peut faire beaucoup plus simple from scratch :

```python
res=''
for e,c in zip('expectra','abcdefgh'):
    res += (100+ord(e)-ord('a'))*c
print(res)
```

## Résultats

Après avoir soumis la dernière question, on m'annonce mon score final de 25 minutes pour les 5 problèmes. Même si j'aurais pu optimiser quelques minutes, je suis finalement satisfait de ma performance. Je n'avais pas l'impression que ce concours était très suivi par les gros joueurs en France, mais tout peut arriver donc je me fixe pour objectif d'atteindre au moins le podium.

![img scoreboard]({{ site.baseurl }}/images/2019-09-expectra/scoreboard.png)

Mon intuition à propos des meilleurs joueurs s'est donc confirmée, j'ai joué un peu seul en tête en finissant le concours deux fois plus rapidement que le 2ème du classement. Mais restons modeste sur ce classement, je peux trouver une petite dizaine de français qui auraient fait la moitié de mon temps à moi ! (bonjour à Jérémy [@jebouin](https://twitter.com/jebouin) et Guillaume [@gaubian](https://twitter.com/gaubian) s'ils me lisent)

Merci Expectra pour l'orga et les 16000 euros (si seulement) et féliciter mes concurrents pour leur belle performance, c'était serré dans le reste du top 5 !