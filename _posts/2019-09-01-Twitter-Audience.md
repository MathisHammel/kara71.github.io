---
layout: post
title: Twitter Audience Diagrams
---

Here's a simple dataviz I made while trying to understand the community behind our Twitter account `@h25io`. I especially wanted to see where the audience comes from, and how strongly it overlaps with the company's founders `@MathisHammel` and `@clement_hammel`.

![img]({{ site.baseurl }}/images/2019-09-audience-viz/setsize.png)

Here's how to make your own. First, install the following pip dependencies:

```bash
pip install python-twitter matplotlib
```

Now we can import the libraries in Python. Like all data visualization applications, I recommend using a Jupyter notebook.

```python
from matplotlib_venn_wordcloud import venn3_wordcloud
from matplotlib_venn import venn3, venn3_circles
import matplotlib.pyplot as plt
import twitter
```

The first step will be setting up the Twitter API to fetch follower lists. For this, you will need to [generate an access token](https://developer.twitter.com/en/docs/basics/authentication/guides/access-tokens.html).

```
API_KEY = 'TmljZSB0cnkgYnV0IG5v'
API_SECRET_KEY = 'QnJvdGhlciB3aHkgZG8geW91IGtlZXAgdHJ5aW5n'
ACCESS_TOKEN = '832653971248471283754-Tm90aGluZyBoZXJlIGVpdGhlcg'
ACCESS_TOKEN_SECRET = 'T2sgb2sgaGVyZSBpcyB0aGUgZmxhZzogaDI1aW97cmVkYWN0ZWR9'

twapi = twitter.Api(consumer_key=API_KEY,
            consumer_secret=API_SECRET_KEY,
            access_token_key=ACCESS_TOKEN,
            access_token_secret=ACCESS_TOKEN_SECRET)
```

Now we're querying the follower lists of our three targets. It's also possible to use only two targets, but you'll need to import `venn2` instead of `venn3`. Be careful, the Twitter API won't like it too much if you query big accounts, basic API accounts are rate limited to 3000 total followers fetched per 15 minute window.

```python
TARGETS = ['MathisHammel', 'h25io', 'clement_hammel']

followers_lists = []
for target_acc in TARGETS:
    followers_lists.append(twapi.GetFollowers(screen_name=target_acc))
```

You can play a bit with the user objects contained in `followers_lists`, they gather a lot of informations on each of the users fetched. Now, let's generate the viz:

```python
out = venn3([set(user.screen_name for user in followers_list) for followers_list in followers_lists],
     set_labels = ['@'+username for username in TARGETS])

for text in out.set_labels:
    text.set_fontsize(34)
for text in out.subset_labels:
    text.set_fontsize(34)
    
plt.title('Follower set size', fontsize=24)
```

Matplotlib offers a lot of options to customize the diagrams, feel free to play with the settings to get a cool render!

## Bonus

Here's a similar diagram you can make, with account names instead of set sizes.

![img]({{ site.baseurl }}/images/2019-09-audience-viz/names.png)

You will simply need another dependency built on top of Matplotlib, which displays a word cloud inside of each area.

```bash
pip install matplotlib_venn_wordcloud
```

The code is very similar to previously:

```python
from matplotlib_venn_wordcloud import venn3_wordcloud

out = venn3_wordcloud([set(user.screen_name for user in followers_list) for followers_list in followers_lists],
                    set_labels=['@'+username for username in TARGETS])

for text in out.set_labels:
    text.set_fontsize(34)
plt.title('Follower set size', fontsize=24)
```

Now, it's time for you to get creative, and build something cool with this. Send us your creations on Twitter!