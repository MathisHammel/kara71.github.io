---
layout: post
title: Facebook Hacker Cup 2019 - Round 2
---

This is my solution to a cool problem I found while playing the Facebook Hacker Cup recently.

I reached my objective to make it to the top 500 of the competition, and won the official T-shirt! As I already have one from last year, I've decided to give it away randomly to one of my Twitter followers who retweets this post.

## Problem statement

We have to produce a bit string **S** of length **N** with several constraints :

- We have a list of requirements (**A**, **B**) where the substring between indices A and B must be a palindrome.
- The string must be balanced : The amount of 0s and 1s should be the closest to each other as possible.

## Solution

The solution consists in two separate parts : in the first half of the algorithm, we will solve the strong constraints posed by the palindrome requirements, and the second half will be dedicated to balancing the 0s and 1s.

### Bit groups

Given the complexity of dealing with palindrome substrings, it might seem very hard to find a non-trivial solution to the requirements,, even without the balancing part. However, it turns out to be super simple if we take the problem from another angle.

A palindrome is nothing more than a string where the first character is equal to the last, the second one is equal to the second last, etc. The same rule applies for our requirements : saying the substring of S between A and B is a palindrome can be written as `S[A + i] == S[B - i]` for any i in [0; B-A].

All the constraints can then be translated into pairs of equalities `S[x] == S[y]`. Since the equality is transitive (`a == b and b == c` implies `a == c`), we will have "communities" of equivalent bits which will all be equal.

Let's take an example and visualize what the communities of equal bits will look like : N = 6, and two constraints (0,2) and (1,5). The two constraints can be translated into unitary bit equalities :

From the first requirement:
- `S[0] == S[2]`
- `S[1] == S[1]` (trivial)
- `S[2] == S[0]` (redundant)

From the second requirement:
- `S[1] == S[5]`
- `S[2] == S[4]`
- `S[3] == S[3]` (trivial)
- `S[4] == S[2]` (redundant)
- `S[5] == S[1]` (redundant)

The bits can be made into nodes of a graph, where an edge is drawn between two bits that are equal :

![img equivgraph]({{ site.baseurl }}/images/2019-07-hackercup/equivgraph.png)

This graph can then be divided into connected components, where all bits in the same connected component will have the same value.

For this problem, I didn't exactly use a graph as the data structure to represent equal bits, but a Merge-Find Set. The functionality is the same, but it was simpler to implement and has slightly better time and memory performance ([read more about Union-Find data structures on Wikipedia](https://en.wikipedia.org/wiki/Disjoint-set_data_structure)). Here's my implementation of the Merge-Find Set on `n` integers representing the bits :

```python
class MergeFind(object):
    def __init__(self, n):
        self.parent = range(n)

    def merge(self, a, b):
        apar = self.find(a)
        bpar = self.find(b)
        self.parent[apar] = bpar

    def find(self, a):
        node = a
        path = []
        while self.parent[node] != node:
            node = self.parent[node]
            path.append(node)
        # Path splitting
        for pathnode in path:
            self.parent[pathnode] = node
        return node
```

Now that we have groups of equal bits, it's easy to generate a string matching the requirements. But how to generate a balanced solution ?

### String balancing

For our previous example, we have three groups of sizes 1, 2 and 3. If bits of the first two groups are 1 and bits of the other group are 0, we will have 3 bits of each type and the string is perfectly balanced.

This example is simple to solve by hand, but we need an algorithm to process cases with more groups of larger size. For this, we can reuse a famous dynamic programming algorithm : the knapsack solver.

You don't need to understand in detail how the KP solver works, the important thing is to know what problems it can solve.

#### The knapsack problem

We have a backpack which can hold a total weight of at most C kilograms. We also have N items we want to put in the bag. Each item has a weight and a value, and the bag can't hold all the items because their total weight exceeds C.

The goal is to pick a set of items of maximal total value, such that the weight of selected items does not exceed C.

In the following example (courtesy of Wikipedia), the best solution is 15$ by taking all items except the green one.

A knapsack solver can find a solution to any problem of this kind in O(N*C) time, using Dynamic Programming.

#### String balancing as a knapsack

We have to build a string of N bits, where bits are split into several groups of different sizes s1, s2, s3, etc.

We can represent the problem of balancing our bit string as a knapsack problem, and solve this using the dynamic programming algorithm : The backpack is of size N/2, and each item represents a group with both weight and value equal to the group's size.

The knapsack solver will give us a set of selected bit groups, where the total sum of the group sizes is as close as possible to N/2. If all the bits in the selected groups are 1 and the others are 0, our solution will be as balanced as possible, which is guaranteed by the knapsack solver.

## Conclusion

I really liked this problem, it combined two interesting algorithmic notions without making the problem statement too complicated.

I'm glad I could (barely) make it into the top 500 and win the competition's T-shirt, please make sure you follow me and retweet my article so you can enter the raffle :)

## Bonus : my code

If you're interested in seeing what my solution looks like, here it is. I haven't touched it since the round so it's dirty and uncommented, but I guess you can maybe learn something from it.

It took me 41 minutes in total to solve this problem which includes reading, thinking, implementation and testing.

As opposed to many competitive programmers, I use Python instead of C++, and almost always code my utilities from scratch instead of using my templates which requires too much maintenance and planning for me.

```python
import collections

class MergeFind(object):
    def __init__(self, n):
        self.parent = range(n)

    def merge(self, a, b):
        apar = self.find(a)
        bpar = self.find(b)
        if apar >= bpar:
            self.parent[apar] = bpar
        else:
            self.parent[bpar] = apar

    def find(self, a):
        node = a
        path = []
        while self.parent[node] != node:
            node = self.parent[node]
            path.append(node)
        for pathnode in path:
            self.parent[pathnode] = node
        return node
        

fi=open('inputB.txt','rb')
fo=open('outputB.txt','wb')
ncases=int(fi.readline().strip())
for case in range(1,ncases+1):
    n, m = map(int, fi.readline().strip().split())
    uf = MergeFind(n)
    for constri in range(m):
        xi, yi = map(lambda tok:int(tok)-1, fi.readline().strip().split())
        if xi == yi:
            continue
        assert xi < yi
        delta = yi - xi
        #print 'xy', xi, yi
        for i in range(delta/2 + 1):
            #print xi+i, yi-i
            uf.merge(xi+i, yi-i)
    bitequiv = map(uf.find, range(n))

    eqsize = collections.Counter()
    for cls in bitequiv:
        eqsize[cls] += 1

    eqclass = collections.defaultdict(list)
    for cls in eqsize:
        eqclass[eqsize[cls]].append(cls)

    dp = [None for _ in range(n/2+1)]
    dp[0] = []
    for clsize in eqclass:
        for clnum in eqclass[clsize]:
            for i in range(len(dp)-1-clsize,-1,-1):
                if dp[i] is not None and dp[i+clsize] is None:
                    dp[i+clsize] = dp[i] + [clnum]
            if dp[-1] is not None:
                break
        if dp[-1] is not None:
            break

    for elt in dp[::-1]:
        if elt is not None:
            selclass = set(elt)
            break

    res = ''
    for cls in bitequiv:
        res += '1' if cls in selclass else '0'

    fo.write('Case #%d: %s\n' % (case, res))
fi.close()
fo.close()
```