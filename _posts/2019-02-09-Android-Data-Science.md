---
layout: post
title: Android Reverse Engineering for Data Science
---

Almost a year ago I started using an app called "Too Good To Go" (this article is not sponsored!). Its goal is to reduce food waste by allowing stores from all around Europe to sell their unsold products to the app users at a very small price. You usually pay 20 to 50% of the retail price, but the products have very short expiration dates and you don't know what you will get beforehand.

![img app1]({{ site.baseurl }}/images/2019-02-tgtg/app1.png) ![img app2]({{ site.baseurl }}/images/2019-02-tgtg/app2.png) ![img app3]({{ site.baseurl }}/images/2019-02-tgtg/app3.png)

I absolutely love the app (still not sponsored, but @toogoodtogo feel free to slide in those DMs) to get fancy stuff at a very low price, or to challenge my cooking creativity by getting random food from my local grocery store.

The only issue is that demand is always higher than supply on high-quality stores, so it's quite challenging to buy at the right time when a store just restocked and there is still stock available. On the most hyped stores, stock typically lasts less than 10 minutes after restock.

Until a few weeks ago, stores in France could only sell for the same day, which meant that next-day restocks were only taken into account at midnight. I only had to set an alarm and got my packages every time. However, a recent change made it possible for stores to restock for the next day, which means that restocks can now happen pretty much anytime and are much harder to predict.

Instead of being one of those normal people constantly watching the application for new offers on the best stores, I decided to put my degree to good use and automate it. Also, there are never enough open datasets in the world, so I decided to have one for myself and have some fun with it :)
## First step - Find the API

Apps usually communicate with servers through an API. By communicating with the backend through such an API, we can divert the stream of useful data for our own use. The first reflex is to search for a public API for the product, but there isn't one.

Having not found any hints about any endpoints online, we have to get dirty and find them ourselves. The app knows how to communicate with the API, and we can download a copy of the app to reverse engineer it.

After downloading the .apk (Android package) file corresponding to the app, let's load it into a visual decompiler for Android called JADX.

![img jadx]({{ site.baseurl }}/images/2019-02-tgtg/jadx.png)

We know that the app is inside package com.app.tgtg (it's also Google Play Store's internal identifier), so we can take a look at the class hierarchy and see if something API-related comes up.

![img jadx2]({{ site.baseurl }}/images/2019-02-tgtg/jadx2.png)

Sadly, all classes have names like e.d.b which makes it very hard to understand what's going on...

![img jadx3]({{ site.baseurl }}/images/2019-02-tgtg/jadx3.png)

Does it mean that the company obfuscated their source code to make it harder to reverse ? No. It's actually due to the fact that many informations are stripped from the original source files (including the filename) when compiling to an APK file, much like compiling a Java file to a .class file. JADX actually does a very good job converting Android bytecode back to source code, and it can also try to guess the original filename by enabling deobfuscation.

![img jadx4]({{ site.baseurl }}/images/2019-02-tgtg/jadx4.png)

Even with all the class hierarchy, the app is still quite a big piece and it's not easy to see how all the parts interact. But we don't really care, all that matters is how to get to the juicy API. For that, we can search the package for the string "https://" in hopes of finding an endpoint.

![img jadx5]({{ site.baseurl }}/images/2019-02-tgtg/jadx5.png)

Among the 142 results in the search, there are some that look very promising like https://apptoogoodtogo.com/index.php/api_tgtg/. There is no obvious endpoint to the API and hitting the URL returns an error so we have to dig deeper in the app (UPDATE: Swagger doc has since been added to this URL).

One way would be to find the string containing the URL, and finding all references to that string. JADX allows us to find references, so we quickly find a lot of pieces which use this string as a base to build URLs.

![img jadx6]({{ site.baseurl }}/images/2019-02-tgtg/jadx6.png)

![img jadx7]({{ site.baseurl }}/images/2019-02-tgtg/jadx7.png)

Owning this list of endpoints is cool but not very useful : the server often expects us to send parameters to the API (search criteria, user credentials, GPS position, etc.) but these query parameters are nowhere to be found in the source code related to these endpoints. This means the endpoints must be generated somewhere else. After a few more searches, we finally get it : a few lines of code where endpoints are actually called with all the query parameters needed !

![img jadx8]({{ site.baseurl }}/images/2019-02-tgtg/jadx8.png)

The reason this list didn't appear in the original search was because they actually use function annotations to make HTTP calls, and the URL is not explicitly specified.

We could also have used a technique to intercept live network communications made by an Android phone, but this proves to be rather challenging with HTTPS in recent OS versions, especially when certificate pinning is enabled (it wasn't, but this is quite a common security measure).

Now that we have a clear list of API endpoints, we can start retrieving it for exploitation.

## Second step - Extracting all the data

There are a few endpoints that we can use to retrieve all the data.

The first one is `list_all_business_map_v2_gz/`, which returns a list of elementary informations about all existing businesses. There are currently around 17000 registered stores, which means only a few fields are actually returned (business name and GPS coordinates) in order to minimize network traffic.

Another endpoint we can use is `list_business_guestv4/`, which also allows for text queries and searching stores within a certain radius. This one returns full business information, but is limited to at most 20 stores at a time (with pagination). Full content includes number of likes, business description, prices, opening hours, and much more.

Both endpoints aren't really satisfying for our use, they really feel like 'free' versions while we can unlock the premium one : by querying `list_business_guestv4/` and automatically querying each of the 800~ pages, we can get full business information on all stores in the world !

I won't post any source code for this - mostly to avoid having someone accidentally DDoS the API - but the scripts should be fairly easy to reproduce at your own risk.

## Third step - Have fun with it

Now that we have raw data over 17000 shops, we can extract and visualize some trends. This is mostly for fun, but this can also help identify the best stores and the best times to shop.

We have plenty of info on each business :

 - Latitude, longitude
 - Name
 - Picture, logo, website, email address
 - Original price
 - Selling price
 - Country
 - Address
 - Number of likes in the app
 - Description
 - Start and end of sales
 - Category (restaurant, bakery, ...)

First, some basic stats per country :

| Country    | Stores    | Stores per million inhabitants |
| ------- | --------- | ------------------ |
| Belgium | 1198 | 105.6 |
| Denmark | 1859 | 323.9 |
| Faroe Islands | 11 | 224.5 |
| France | 6226 | 92.8 |
| Germany | 2473 | 29.9 |
| Netherlands | 905 | 53.0 |
| Norway | 1260 | 239.6 |
| Spain | 242 | 5.19 |
| Switzerland | 523 | 62.11 |
| United Kingdom | 1359 | 20.58 |

Although France has almost as many stores as the rest of Europe combined, we can see that there is a much higher market penetration in Denmark and Norway which are smaller countries.

Many shops have pickup hours in the evening, because it's hard to predict which food items are going to waste until the last minute. Let's visualize how many stores are open along a day :

![img density_all]({{ site.baseurl }}/images/2019-02-tgtg/density_all.png)

No big surprise there, but it's also quite funny that the peak hour is around 18:40, not a very round figure.

Countries and cultures do not all operate on the same daily schedule, and it would be quite interesting to know if these differences are also reflected in Too Good To Go.

![img all_countries]({{ site.baseurl }}/images/2019-02-tgtg/all_countries.png)

Let's untangle this a bit :

![img all_countries_individual]({{ site.baseurl }}/images/2019-02-tgtg/all_countries_individual.png)

And so we can confirm, peak opening hour strongly correlates with [Bedtimes around the world](https://www.vox.com/2016/5/10/11639214/how-people-around-the-world-sleep) !

Each store also gives the original and selling price of packages, so we can get some stats on the price difference. Fun fact : The most expensive package costs 53 euros, and is originally worth 160 ! The package is 1.5kg of smoked salmon :)

![img old_new]({{ site.baseurl }}/images/2019-02-tgtg/old_new.png)

Finally, I was quite interested in visualizing where the stores are actually located around the world and in Paris where I live.

![img heat0]({{ site.baseurl }}/images/2019-02-tgtg/heat0.png)

![img heat1]({{ site.baseurl }}/images/2019-02-tgtg/heat1.png)

![img heat2]({{ site.baseurl }}/images/2019-02-tgtg/heat2.png)

Shops are mostly concentrated around large cities and the heatmap can be correlated with a population map. At a smaller scale, Paris is quite surprising : the highest concentration is on the northern side (right bank of la Seine), in the central arrondissements. I tried to find a correlation with population density or wealth, but didn't find one.

We're done playing with all these useless (but interesting) visualizations, congratulations for making it this far ! Here is the good stuff - stores with the most likes in Paris and in the world, so you can add them to your favourites :

| Rank    | Store    | Likes | Country |
| ------- | --------- | ------------------ | ------------------ |
| 1 | IKI sushi - Østerbro | 5339 | Denmark |
| 2 | Åpent Bakeri Produksjon | 4944 | Norway |
| 3 | Fuji Sushi - Frederiksberg | 4866 | Denmark |
| 4 | Restaurant Soya 2 - Aarhus | 4629 | Denmark |
| 5 | Restaurant Soya - Aarhus  | 4337 | Denmark |
| 6 | Baker Brun Bogstadveien  | 4244 | Norway |
| 7 | Slagter Friis - Frederiksberg | 4218 | Denmark |
| 8 | REINH. van HAUEN - Falkoner Allé | 4017 | Denmark |
| 9 | Kolonihagen Bakeri | 3994 | Norway |
| 10 | Yo! Sushi Østbanehallen | 3872 | Norway |
| 11 | Jaipur Indisk Restaurant - Middag | 3785 | Norway |
| 12 | Baker Hansen St.Hanshaugen | 3715 | Norway |
| 13 | Brødbakerne - Bislett | 3680 | Norway |
| 14 | Jaipur Indisk Restaurant - Lunsj | 3668 | Norway |
| 15 | Nibon Ya Sushi - Kbh V  | 3582 | Denmark |
| 16 | SMELT | 3427 | Norway |
| 17 | Åpent Bakeri Tranen | 3398 | Norway |
| 18 | Renaa Xpress Sølvberget | 3394 | Norway |
| 19 | MÆSK - Frederiksberg | 3365 | Denmark |
| 20 | Gutta på Haugen | 3340 | Norway |
| 21 | Ostehuset - Øst | 3302 | Norway |
| 22 | Det Grønne Køkken - KBH N | 3287 | Denmark |
| 23 | REINH. van HAUEN - Gammel Kongevej | 3275 | Denmark |
| 24 | Bio Nant - Racine | 3271 | France |
| 25 | Vincent Guerlais - Nantes | 3225 | France |
| 26 | Cafe Fika - Aarhus | 3222 | Denmark |
| 27 | Ostehuset Domkirkeplassen | 3192 | Norway |
| 28 | Comptoir Veggie | 3156 | France |
| 29 | Focacceria | 3128 | Norway |
| 30 | Lenôtre Bastille | 3115 | France |

---


| Rank    | Store    | Likes |
| ------- | --------- | ------------------ |
| 1 | Comptoir Veggie | 3156 |
| 2 | Lenôtre Bastille | 3115 |
| 3 | La Pâtisserie des Rêves - Bac | 3022 |
| 4 | Berko | 3007 |
| 5 | Lenôtre Vincennes | 2906 |
| 6 | Tandooright (soir) | 2879 |
| 7 | Les Poireaux de Marguerite - Paris 14ème | 2860 |
| 8 | Mandarin Oriental, Paris | 2853 |
| 9 | La Bossue | 2796 |
| 10 | Sushi Wasabi Saint Germain | 2786 |
| 11 | Les Poireaux de Marguerite - Saint Maur | 2730 |
| 12 | Big Fernand - Paris 13 | 2678 |
| 13 | VG Pâtisserie  | 2670 |
| 14 | Sol Semilla - Service 16h | 2605 |
| 15 | Lenôtre Lecourbe | 2604 |
| 16 | Opoa | 2602 |
| 17 | Le Garde Manger des Dames | 2487 |
| 18 | Eric Kayser - Bercy Village | 2485 |
| 19 | PAF le jus Pressé A Froid | 2456 |
| 20 | Mamy Thérèse la Madeleinerie | 2453 |
| 21 | L'Eclair de Génie - Pâte à choux | 2430 |
| 22 | Ten Belles Bread | 2408 |
| 23 | La Meringaie | 2404 |
| 24 | Wild & The Moon Charlot | 2370 |
| 25 | Les Belles Envies - Monge  | 2339 |
| 26 | Helmut Newcake  - Sans Gluten - Madeleine | 2326 |
| 27 | La Pâtisserie des Rêves - Bac - Entremets | 2303 |
| 28 | La Pâtisserie des Rêves - Poncelet | 2284 |
| 29 | Lenôtre Courcelles | 2234 |
| 30 | Top Primeur | 2227 |

## Bonus - How not to crypto

While reverse engineering the app, I stumbled upon strange API parameters like `subscription_id_encrypted` or `creditcard_encrypted`. Surely this means there is some cryptographic functions in the code, let's hope they did it right. (spoiler alert : no)

![img cryptation]({{ site.baseurl }}/images/2019-02-tgtg/cryptation.png)

First sign that the crypto's gonna be fucked is the name given to the class : `Cryptation`. Seriously, call the crypto police now.

Then, we dig into the actual code, and find that they use AES-CBC. Not the best block operating mode (CBC has several known attacks), but at least it's AES. Now, where are the keys and IVs located ? Some sort of secret token unique to each user ? No, it would be way too complex, let's use "12345678901234567890123456789012" and IV = 0 instead ! Yes, the function you see in the screenshot above is the one that contains the key...

Luckily, this example of terrible crypto is not used as the only security for sensitive information, because everything transits through HTTPS anyway. This is just an additional feature from the developers to avoid getting credit cards in plaintext everywhere in the server logs and in the phone's memory, but any attacker with a decompiler and half a brain can very easily render this layer useless.

This article is coming to an end, so here are some crypto wisdoms to remember kids :

 - Never roll your own crypto
 - Always hash and salt passwords in your database
 - Never use ECB
 - Avoid MD5
 - Avoid CBC
 - Never call anything `Cryptation` ;)
 
The visualizations have been made with matplotlib, seaborn, folium, plotly. If you have any special requests like publishing my viz source code, getting a custom data analysis, etc., please get in touch! My Twitter DMs are always open.