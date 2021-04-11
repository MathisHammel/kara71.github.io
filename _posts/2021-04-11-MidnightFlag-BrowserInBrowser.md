---
layout: post
title: Midnight Flag CTF - Browser In Browser [Web 270]
---

Ce weekend a eu lieu le Midnight Flag CTF, organisé par une école française à destination d'étudiants. Les challenges étaient de bonne qualité et sans trop de guessing, je n'ai pas particulièrement été mis en difficulté sur mes disciplines favorites, mais j'en ai du coup profité pour explorer les catégories que je maîtrise moins comme le web qui sera mis à l'honneur dans ce writeup du challenge Browser In Browser.

## Le challenge

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser1.png)

On arrive sur un site permettant de faire une sorte de proxy vers un autre site web.

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser2.png)

#### Tentative d'accès privilégié

On peut se douter que la requête est effectuée directement depuis le serveur du chall, et qu'il faudra donc exploiter cette position pour accéder à une ressource visible seulement depuis ce serveur. La première chose qui vient en tête serait de faire une requête vers http://127.0.0.1 pour essayer d'accéder à une console d'administration :

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser3.png)

Malheureusement, 127.0.0.1 et localhost sont interdits d'accès. C'est cependant un très bon signe, car le message semble provenir d'un filtre en amont de la requête qui va vérifier que l'hôte n'est pas dans une blacklist donnée. Le fait que l'on nous interdise l'accès au localhost fait penser que les admins y ont caché quelque chose...

#### Contournement du filtre

Ma première intuition est de passer par des contournements custom de parsing d'URL, par exemple avec l'utilisation du @ ou de l'adressage IPv6, mais une idée beaucoup plus simple me vient immédiatement : le filtre semble se faire uniquement sur l'URL soumise, mais le serveur suit sûrement les redirections !

Il suffit donc de chercher un service permettant de créer des liens qui redirigent vers une autre URL de notre choix. Ce genre de service est très répandu sous la forme de raccourcisseur d'URL, j'ai choisi d'utiliser bit.ly pour ce challenge.

On commence par lui passer http://127.0.0.1 sous forme raccourcie, et bingo !

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser4.png)

Le filtre est contourné, et on accède à une interface qui semble très prometteuse avec des infos réservées à l'admin du site.

En essayant d'accéder à la page admin_panel.php avec la même technique, un message d'erreur nous apparaît : **Missing action. Available action are: viewUsers, viewFile, ...**

On peut donc essayer de lire le fichier /etc/passwd via le lien raccourci correspondant à l'URL http://127.0.0.1/admin_panel.php?action=viewFile&file=/etc/passwd mais une erreur se produit :

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser5.png)

#### Leak du code source

Il semblerait que le fichier à afficher soit importé dans la page via un include PHP. Pour mieux comprendre comment fonctionne le serveur, on peut leaker le code source en lisant admin_panel.php dont le chemin nous est donné dans l'erreur. Il faudra bien faire attention à utiliser les PHP filters (technique courante dans l'exploitation de LFI) pour éviter que le code importé ne soit interprété et s'importe lui-même dans une boucle infinie. En utilisant le filtre base64, on peut récupérer sans problème le source encodé en base64, que l'on pourra ensuite décoder localement.

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser6.png)

La payload correspondant à la capture d'écran ci-dessus correspond à la version raccourcie de http://127.0.0.1/admin_panel.php?action=viewFile&file=php://filter/read=convert.base64-encode/resource=/var/www/html/admin_panel

En décodant le base64 qui nous est renvoyé par le serveur, on retrouve donc le code source complet du panneau d'administration, et le flag apparaît parmi les commentaires :

![]({{ site.baseurl }}/images/2021-04-midnightflag/browser7.png)
