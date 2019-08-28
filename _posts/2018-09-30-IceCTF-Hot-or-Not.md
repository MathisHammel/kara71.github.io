---
layout: post
title: IceCTF 2018 writeup - Hot or Not
---

I've been busy recently and couldn't play CTFs as much as I used to, but IceCTF was such a good memory in 2016 that I really wanted to play their second edition !

We are given a fat (70MB of jpeg !) image file. While downloading, I prepare my binwalk command because I am certain it's gonna be a big file "hidden" behind a normal image. It's only when I try to open the file that I realize the monstrosity that this challenge is : It's actually only an image file, but it's 5000x5000 pixels !

<blockquote class="twitter-tweet" data-lang="fr"><p lang="en" dir="ltr">You know it's real when paint.exe takes 7GB of RAM (playing <a href="https://twitter.com/icectf?ref_src=twsrc%5Etfw">@icectf</a>) <a href="https://t.co/3KM6VsP3A0">pic.twitter.com/3KM6VsP3A0</a></p>— Mathis Hammel (@MathisHammel) <a href="https://twitter.com/MathisHammel/status/1038770936026161152?ref_src=twsrc%5Etfw">9 septembre 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

After waiting 2 minutes for the image to load in the viewer, I can see a mosaic of small images with no obvious pattern.

![img mosaic1]({{ site.baseurl }}/images/2018-09-hotornot/mosaic1.jpg)

When I zoom in, pictures seem to be mostly fast food and dogs.

![img mosaic2]({{ site.baseurl }}/images/2018-09-hotornot/mosaic2.jpg)

It didn't take long to hit me : the pictures are either "dog" or "hot-dog", hence the name of the challenge "Hot or not" !
It might also be a reference to the Silicon Valley series where a character builds a "hot-dog"/"not hot-dog" machine learning app ([which was also released IRL](https://www.theverge.com/tldr/2017/5/14/15639784/hbo-silicon-valley-not-hotdog-app-download)).

I could also see that the whole mosaic has an orange-ish shade, except for 3 areas in the corner that are a bit more grey.

![img mosaic3]({{ site.baseurl }}/images/2018-09-hotornot/mosaic3.jpg)

This definitely looks like a QR code, with the three large markers in the grey areas. With all of that information, I have a strong intuition that I am supposed to paint dogs as one color and hot-dogs as another, and ultimately obtain a QR code. (This intuition turned out to be correct)

There are 7569 sub-images in the mosaic, so there is no way I will classify them by hand. Another solution would be to build a neural network for this classification task (or use a pre-made model), but I noticed something that I want to explore a little more to try for a more creative solution.

![img mosaic4]({{ site.baseurl }}/images/2018-09-hotornot/mosaic4.png)

Many of the sub-images appear more than once in the image. There are 7569 images on the mosaic, but only 1371 unique ones, which means that each image appears 5 or 6 times on average.

From there, I can either classify the 1371 images and be done in about an hour, or try to use another property : I noticed that hot-dog images tend to appear in groups, sane with dog pictures. I can try to build a graph where each node is an image and make the weight higher on the edges between images that appear together. In the end, I would try to extract two clusters from that graph, where each cluster represents a class (dog or hot-dog).

![img graph1]({{ site.baseurl }}/images/2018-09-hotornot/graph1.png)

Shortly after I started fiddling with Python to build the image relationship graph, I figured there was an extremely powerful property on the images. Not only do images of the same class appear more together, they also respect groups of 3x3 where all pieces of the mosaic have the same class !

![img mosaic5]({{ site.baseurl }}/images/2018-09-hotornot/mosaic5.jpg)

The principle of my relationship graph remains about the same, but I add the new property : There is an edge between two images if and only if they appear in the same 3x3 square. From there I will hopefully be able to extract two separate connected components in the graph, which will act as our previously defined clusters.

I will spare you the implementation details (feel free to DM me if you want the source code), but I managed my graph using a classic implementation of one of my favourite data structures, the [Merge-Find set](https://en.wikipedia.org/wiki/Disjoint-set_data_structure). (*Is there anything nerdier than having a list of favourite data structures ?*)

After the execution terminates, we are indeed left with 2 clusters ! Here is what the final graph would look like :

![img graph2]({{ site.baseurl }}/images/2018-09-hotornot/graph2.png)

In conclusion, we have successfully created a dog vs hot-dog image classifier, without even loading the contents of the images ! Of course, it only works in these conditions because it exploits some properties of the images that were probably left accidentally by the challenge designers. We only have to color back the mosaic in black and white :

![img qr1]({{ site.baseurl }}/images/2018-09-hotornot/qr1.png)

After adding the three QR markers that were removed by the authors, we can scan the QR code and get our flag !

![img qr2]({{ site.baseurl }}/images/2018-09-hotornot/qr2.png)

Flag : **IceCTF{h0td1gg1tyd0g}**
