---
layout: post
title: How to get the perfect score on Google Hash Code - Part 2
---

In the [first part]({{ site.baseurl }}/HashCode-Part1/) of this article, we studied several techniques and a competitive strategy for the Google Hash Code.

Today, we will have a look at this year's practice contest released by Google, to see the strategies proposed in the previous post applied in a practical example. The source code will be in Python, which we strongly recommend for the Hash Code due to its simplicity.

# 1. More Pizza

The problem statement is the following : we are an organizer of a Hash Code hub with M contestants, and have to order pizza for everyone. There are N different pizzas we can buy, each has a different number of slices and we can only buy each pizza at most once.

We have to order as many slices as possible without exceeding the number of players. Let's take this example with 17 contestants in the room and the following pizzas :

![img]({{ site.baseurl }}/images/2020-02-hashcode/pizzas.png)

If we take the first, third and fourth pizza, we'll end up with 16 slices, which is a pretty good approach since the best possible score on this set can be no greater than 17. It turns out that the best score for this dataset is actually 16 (and it's pretty easy to demonstrate through bruteforce), we can never reach 17.

You may already recognize a pretty well known problem if you have some experience in programming contests, and we'll get to that in a second.

## Datasets

Five testcases are provided, and a quick summary of each is provided below.

| Dataset | Contestants   | Pizzas | Avg. slices | Min. slices | Max. slices |
|---------|---------------|--------|-------------|-------------|-------------|
| A       | 17            | 4      | ~ 5         | 2           | 8           |
| B       | 100           | 10     | 42          | 4           | 95          |
| C       | 4 500         | 50     | ~ 94        | 7           | 195         |
| D       | 1 000 000 000 | 2 000  | ~ 504 916   | 476         | 999 867     |
| E       | 505 000 000   | 10 000 | ~ 75 396    | 223         | 150 000     |

The first three datasets are quite small (and thus will only account for a small fraction of the total score), with the interesting testcases being D and E. There is also no obvious pattern in the number of slices in each pizza, they appear to be randomly generated in a uniform distribution.

Now that we are done with the problem and the inputs, let's start cracking the problem and attempt a high score ! We'll try several strategies and see what happens for each one. But first, let's have a look at the parts of the code we'll need to support our solving algorithms.

# 2. Base of the project

Remember in the [last article]({{ site.baseurl }}/HashCode-Part1/), we saw a methodology to implement the full solution in 4 parts where each team member takes care of one part. Here's a refresher in case you forgot :

![img]({{ site.baseurl }}/images/2020-02-hashcode/blocks.png)

## Internal formats

You may remember that the first thing you should do **before you even start coding** in this methodology is to specify the internal format of datasets and solutions.

The proposed attributes of our internal variables are the following, stored in simple python data structures (dict and list) :

```python
dataset_format = {
            'pizzas': [2, 5, 6, 8],  # Number of slices in each pizza
            'knapsize': 17           # Number of players
}

solution_format = [0, 2, 3]          # Index of each selected pizza
```

These internal formats are very simple and contain all the basic information we need. We can now start implementing the solution.

## Main program

The main program is pretty much the same regardless of the Hash Code problem and can be coded in advance, the only customization is the line where the score is printed.

```python
import glob

import parsr, solver, scorer, writer

for idx, filename in enumerate(glob.glob('datasets/*')):
    dataset = parsr.parse(filename)
    solution = solver.solve(dataset)
	score = scorer.score(solution, dataset)
	print('Score for %s: %s/%s (%s to perfect score)' % (filename, score, dataset['knapsize'], dataset['knapsize'] - score))
	writer.write(solution, filename.replace('datasets','solutions'))
```

We now need to implement the four functions `parse`, `solve`, `score` and `write` which are the four elementary bricks of a good Hash Code solution. Each of the functions will be in a separate file so each team member can work in their own space.

## Parser

The specified dataset format is quite simple, we parse it directly to the python dictionary.

```python
def parse(filename):
    with open(filename, 'r') as fi:
        knapsize, ntypes = map(int,fi.readline().split())
        pizzas = list(map(int, fi.readline().split()))
        assert pizzas == sorted(pizzas)
        dataset = {
            'pizzas': pizzas,
            'knapsize': knapsize}
        return dataset
```

## Writer

Same as the parser, the conversion is very straightforward.

```python
def write(solution, filename):
    with open(filename, 'w') as fo:
        fo.write(str(len(solution))+'\n')
        for elt in solution:
            fo.write(str(elt)+' ')
```

## Scoring

The formula to compute the score is simply the sum of all pizzas. If the total amount of slices exceeds the capacity we could raise an error or a warning, but in this case we decided to return a score of 0.

```python
def score(solution, dataset):
    res = 0
    for idx in solution:
        res += dataset['pizzas'][idx]
    return 0 if res > dataset['knapsize'] else res
```

Remember, the `solution` variable is a list of indices, so we need to lookup the number of slices in the dataset (here with `dataset['pizzas'][idx]`).

# Solver

The last block we need to talk about is the solver. There are several strategies we discussed in the first part of this article, let's start with the minimal strategy and work upwards.

## Minimal strategy

Remember, the goal of this strategy is to produce any valid solution so we can test the entire program very quickly to get rid of any early bugs, and verify our scoring function matches the score given by the Hash Code judge server.

Our proposed minimal strategy is to only return the first pizza slice.

```python
def solve(dataset):
	return [0]
```

The score of this submission is obviously very low.

## Greedy strategy

Now that we made sure our basic program is free of bugs, we can work on a solution that will give a higher score.

Here, we use a greedy algorithm that will iterate over each pizza and order it as long at we don't exceed the slice limit.

```python
def solve(dataset):
	slices = 0
	solution = []
	for i in range(len(dataset['pizzas'])):
		if slices + dataset['pizzas'][i] <= dataset['knapsize']:
			solution.append(i)
	return solution
```

The total score of this solution is already very high : 1 504 818 713 / 1 505 004 617, which is more than 99.98% of the upper bound ! But there are ways to go even higher, we are still missing 185904 slices in total to reach the full capacity on all datasets. Note that we don't know if the capacity can be reached on each dataset, so the score of 1 505 004 617 is an absolute upper bound.

## Greedy II : the return

If you had to fill a jar with golf balls and sand, you wouldn't start pour the sand first, right ?

The same applies here, we can optimize by going through our pizzas in reverse order. The code is basically the same, but we add a reverse iterator.

```
Score for A.in: 16/17 (1 to perfect score)
Score for B.in: 99/100 (1 to perfect score)
Score for C.in: 4495/4500 (5 to perfect score)
Score for D.in: 999999725/1000000000 (275 to perfect score)
Score for E.in: 504999983/505000000 (17 to perfect score)
```

With only this improvement, we have drastically cut the gap remaining to our goal of reaching the perfect score, with only 299 slices to go !

## Optimal solution

### The knapsack problem

It turns out the NP-complete problem hidden behind the practice round isn't hidden very well. It is an instance of the famous [Knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem), which usually is presented in the following form :

You have a backpack which can contain at most K kilograms of items. For your next trip, you would like to make your luggage but don't have enough capacity in your bag to take all your items. Each item has a weight and a value, and you have to pick the set of items with maximum total value without exceeding the knapsack's weight capacity.

The variant in this practice problem can be modelized as a knapsack problem where the capacity of your bag is equal to the amount of participants, and each pizza with K slices is represented as an item worth K dollars and of weight K kg.

This also explains the name `knapsize` used in our example source code.

### Solving the knapsack

There are algorithms for the knapsack problem that have much better complexity than bruteforce over all pizza subsets which is in **O(N \* 2^N)** time complexity. The algorithm in itself isn't trivial to understand and would need an entire blog article to itself, but here it is :

```python
def knapsolve(dataset):
    maxscore = dataset['knapsize']
    dp = [None for _ in range(maxscore+1)]
    dp[0] = []
    for idx, item in progressbar.progressbar(list(enumerate(dataset['pizzas']))):
        for i in range(maxscore - item, -1, -1):
            if dp[i] is None: continue            
            dp[i+item] = dp[i] + [idx]
    i = -1
    while dp[i] is None: i -= 1
    return dp[i]
```

The complexity of this algorithm based on dynamic programming depends on the size of the knapsack M and the number of available pizzas N, for a complexity of **O(N \* M)**. While this might look like a quadratic time complexity to solve an NP-complete problem, it is actually considered an exponential-time algorithm by some computational theory magic that doesn't really need to be explained here (if it interests you, complexity is a function of the **size** of the input and not the input itself).

Using this program, we can solve the first three testcases perfectly, but the last two would take way too long due to a very high value of M.

```
Score for A.in: 16/17 (1 to perfect score)
Score for B.in: 100/100 (0 to perfect score)
Score for C.in: 4500/4500 (0 to perfect score)
Score for D.in: 999999725/1000000000 (275 to perfect score)
Score for E.in: 504999983/505000000 (17 to perfect score)
```

We only gained 6 points in total compared to the greedy method, which isn't much because most of the optimization left to do is on the 4th testcase. Here again we see that the best solution for the first dataset isn't equal to the capacity, and it might be the case for the remaining two datasets as well.

### Optimizing the knapsack : partial reconstruction

With a pseudo-quadratic complexity, the dynamic programming solution isn't fast enough to solve the large datasets. However, we can reduce our case to a simpler instance of the knapsack problem and solve that one instead. Here's how :

If we take an already good solution such as the one obtained with the greedy solution, we can destroy part of it (remove a few pizzas) and then rebuild the solution more optimally. To do this, we remove a random pizza from the solution which increases our remaining capacity by the value of the pizza.

For example, our solution for dataset D was 999999725, which means we had a remaining capacity of 275. If we remove the pizzas of size 24955, 68049 and 263303, we are freeing a capacity of 356307 in our knapsack and have a remaining capacity of 356582. If we can find a subset of unused pizzas that sums to this capacity, then our knapsack is solved optimally.

Since the capacity of the knapsack is now lower than 1 million instead of 1 billion, the computing power needed to run our dynamic programming algorithm is realistic.

```python
def rebuild(solution, dataset):
    newsol = list(solution)
    for i in range(min(len(newsol),3)):
        idx = random.randrange(len(newsol))
        print(idx, dataset['pizzas'][newsol[idx]])
        del newsol[idx]
    newscore = scorer.score(newsol, dataset)
    remaining = dataset['knapsize'] - newscore
    st = set(newsol)
    newpizz = []
    for i in range(len(dataset['pizzas'])):
        if i in st:
			# Pizzas already used in the solution are replaced with infinite slices
			# so we avoid selecting them twice while keeping the list indices correct
            newpizz.append(99**99)
        else:
            newpizz.append(dataset['pizzas'][i])
    print('rem', remaining)
    reco = knapsolve({'pizzas':newpizz, 'knapsize':remaining})
    return sorted(newsol + reco)
```

In practice, we only needed to run this algorithm once on each solution to reach the following result :

```
Score for A.in: 16/17 (1 to perfect score)
Score for B.in: 100/100 (0 to perfect score)
Score for C.in: 4500/4500 (0 to perfect score)
Score for D.in: 1000000000/1000000000 (0 to perfect score)
Score for E.in: 505000000/505000000 (0 to perfect score)
```

We did it ! But nobody is perfect, so we are 1 point away from full capacity on testcase A that can never be fulfilled. Sorry to that 1 contestant out of 1.5+ billion that won't have a slice !

This code runs in about 3 minutes on a single core of a modern CPU. Can we do it faster ?

## Upping the ante : perfect score in less than 1 second

There's another technique that we can use to break perfect score very fast, which is based on our previous greedy solution. Instead of taking the item when it fits the capacity, we can first decide randomly (with an arbitrary probability of 25%) to reject it.

This can create a sub-optimal solution especially near the end as we might reject the very last item, so we iterate a second time in the normal greedy way without rejection.

This strategy effectively randomizes a bit the greedy algorithm, and running it many times will eventually lead to the perfect score. In practice, we only need to run around 1000 iterations for the biggest testcases which took no more than 400ms in the best case.

```python
def solvemc(dataset):
    capa = dataset['knapsize']
    sol = []
    for i in range(len(dataset['pizzas'])-1, -1, -1):
        if random.getrandbits(2) and capa >= dataset['pizzas'][i]:
            sol.append(i)
            capa -= dataset['pizzas'][i]
    st = set(sol)
    for i in range(len(dataset['pizzas'])-1, -1, -1):
        if i not in st and capa >= dataset['pizzas'][i]:
            sol.append(i)
            capa -= dataset['pizzas'][i]
    return sol
```

As you can see, the code is very similar to the greedy code duplicated.

# Conclusion

We could also have used a heavy artillery approach to this problem, with a constraint solver such as [OR-Tools](https://developers.google.com/optimization). You can try it if you want to practice using a solver before the competition !

Remember : you only have until Monday Feb. 17th to register, so don't forget to do so on the [competition website](https://hashcode.withgoogle.com/) if you want to participate !

Feel free to register in one of many Sogeti hubs in France, where you will get free pizza (we ran our knapsack algorithms all night, there will be enough for everyone) and the opportunity to win prizes !