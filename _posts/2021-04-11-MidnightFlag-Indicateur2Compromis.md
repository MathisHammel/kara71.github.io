---
layout: post
title: Midnight Flag CTF - Indicateur 2 Compromis [Network 235]
---

Le dernier challenge de réseau du CTF Midnight Flag pouvait sembler très complexe à première vue, mais il était en réalité possible de trouver très rapidement le flag qui se cachait parmi les 35MB de capture réseau fournie.

## Capture réseau

Il se passe beaucoup de choses dans le pcap fourni. La très grande majorité des paquets semble être du trafic TLS, avec quelques échanges HTTP en clair et des paquets DNS, DHCP et DNS. Rien de très surprenant ici, nous avons affaire à une personne qui navigue sur internet.

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis1.png)

Cependant, on remarque aussi certains paquets UDP contenant des données qui semblent intéressantes. En écrivant `udp` dans le filtre Wireshark en haut de la fenêtre, les 206 paquets UDP de la capture se révèlent à nous :

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis2.png)

L'un des premiers paquets contient le texte `TUNURntuZXZlcl9nb25uYV9naXZlX3lvdV91cA==` qui semble être du base64. En le décodant, on obtient le texte `MCTF{never_gonna_give_you_up`. C'est évidemment un faux flag, mais la présence de plusieurs autres paquets de ce type laissent à penser que le flag correct se trouverait parmi eux.

On peut extraire le contenu des paquets UDP très simplement, en utilisant l'outil Tshark qui est l'équivalent en ligne de commandes de Wireshark.

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis3.png)

Sur chaque ligne apparaît un flag potentiel, on peut faire un petit script en Python ou en Bash pour les décoder une à une :

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis4.png)

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis5.png)

Sur l'une des lignes, on repère le flag `MCTF{IOC_R_fun}` qui semble être le moins fake de tous, et qui est effectivement le bon. C'est une technique assez maudite de créer des challenges qui ont des faux flags, surtout en aussi grand nombre et quasiment indiscernables du vrai, mais chaque créateur de CTF est libre de choisir son design et ce n'était ici pas trop gênant. On conclura donc ce writeup sur les sages paroles des amis Unactive et Mahal :

![]({{ site.baseurl }}/images/2021-04-midnightflag/compromis6.png)