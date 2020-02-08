---
layout: post
title: How to get the perfect score on Google Hash Code - Part 1
---

Have you ever heard of Google's competition called Hash Code ? This article will discuss several techniques your team can use to get a top score on the next edition coming February 20th, 2020.

But first, let's go back to basics.

# 1. Goal of the contest

Hash Code is a very unique programming competition. Any team of 2-4 players can play it, and the goal is to find the best possible solution to a real-life problem using programming and optimization techniques. Teams can play from anywhere in the world, and can also register to play from an on-site hub (more on that later).

During the contest, we are given multiple datasets and the goal is to get the highest score you can on all datasets.

The challenge is always in the NP-complete class. I won't go into too much theory of computational complexity, but NP-complete basically means you can only solve it through brute-force, and it's impossible to find the optimal solution unless the dataset is very small. Let's review a few examples of such problems that we will use later to illustrate solving strategies.

### Traveling Salesman Problem

The most common example is the **Traveling Salesman Problem**, aka TSP. In instances of the TSP, we are given coordinates of several points, and the goal is to find the shortest path going through all the provided points. Here is an example with large cities in Europe, and a possible solution :

![img]({{ site.baseurl }}/images/2020-02-hashcode/tsp-dataset.png)
![img]({{ site.baseurl }}/images/2020-02-hashcode/tsp-solved.png)

The shortest path can be found through bruteforce or dynamic programming for instances that don't exceed 30-50 cities, but higher than that our search will only give an approximation of the shortest path.

### Number partitioning

We are given a list of numbers such as {2, 8, 4, 9, 6, -13}. The goal is to split this set into two groups such as the sums of the groups are equal (or as close as possible).

Here, we could take {2, 8, 9, -13} one one side with sum 6, and the other group is {4, 6} with sum 10. The difference of sums is 4 which is pretty good, but there are two solutions where the sums are equal. Can you find them ?

### Graph coloring

![img]({{ site.baseurl }}/images/2020-02-hashcode/graphcoloring.png)

Given a graph, we have to determine a way to assign colors to each node such that every node has a different color from all its neighbors. The goal here is to find an assignment that minimizes the number of different colors.

You may have heard of this problem for coloring maps like the one below, where it has been proven that the maximum amount of colors you'll ever need is 4. On more complex (non-planar) graphs, that number can be much higher.

![img]({{ site.baseurl }}/images/2020-02-hashcode/mapcoloring.jpg)

## Scoring

So, Google Hash Code is about NP-complete problems. But what's the point in trying to solve problems that would take billions of years to bruteforce ? 

The answer is optimization. Our goal isn't to reach the perfect solution, but to reach the best solution we can. For example, it's easy to say whether a Traveling Salesman path is good or bad by looking at its total distance, and we can give a higher score to a shorter path. The same is true for all Hash Code problems : we have an objective to maximize (or minimize), and teams are ranked according to how well their proposed solution fits the rating criteria.

## Examples of Hash Code problems

Problems given during the Google Hash Code are often famous NP problems such as the TSP with a real-world application, but they are always disguised and the actual NP problem isn't easy to identify. In the previous editions, there have been problems such as:

- [Organizing drone deliveries from warehouses to customers](https://storage.googleapis.com/coding-competitions.appspot.com/HC/2016/hashcode2016_qualification_task.pdf) (2016)
- [Compiling gigabytes of source code across multiple servers](https://storage.googleapis.com/coding-competitions.appspot.com/HC/2019/hashcode2019_final_task.pdf] (2019 finals)
- [Planning self-driving taxi courses](https://storage.googleapis.com/coding-competitions.appspot.com/HC/2018/hashcode2018_qualification_task.pdf) (2018)

---

# 2. Strategies

In this part, we'll first study several strategies you can use on an NP-complete problem to find a good solution. Then, we'll study how to organize your team and implement those strategies in the contest's very short time.

## Solving strategies

We will discuss 6 strategies of ascending complexity, each will be illustrated with examples that could work on TSP, number partitioning or graph coloring.

### Level 0 : Minimal

Our goal for this simple strategy is to find a solution. The score of the solution doesn't matter, it just has to be valid.

- For the TSP, we can visit the cities in the same order as the input
- For number partitioning, one group has all the numbers and the other has none
- For graph coloring, each node has a unique color

As you can see, the solutions are extremely bad and would rank us at the bottom of the scoreboard. But the minimal strategy is useful for one thing : we can make sure very quickly that every component of our program is working as intended without having to wait for a complex solving algorithm to be ready.

### Level 1 : Monte Carlo

The name of the Monte Carlo technique is a reference to a famous area in Monaco where a huge casino is located. Here, we will use a lot of randomness and hope that it brings us a good solution.

The goal is to generate as many random solutions as possible, and only keep the best one only. With a high enough computing power this can yield some interesting results, but the Monte Carlo strategy is easily beaten by more advanced solutions.

The bottleneck of execution time will often be the scoring algorithm, which will also need to be optimized for good Monte Carlo runs.

Examples of this strategy on typical NP-complete problems :

- For TSP, we can shuffle the order of the visited nodes
- For number partitioning, we put each element in a random group with probability 0.5 each time

### Level 2 : Greedy

As its name suggests, we will have a short-term vision to build an efficient solution. Starting from an empty solution, we evaluate all possible options and go towards the best one. We keep doing this until a valid solution is made, which should be somehow optimized.

- Starting in one of the cities, we can find a good solution to TSP by going to the nearest non-visited city each time
- For graph coloring, we start from a blank graph and iterate through all nodes sorted by decreasing number of neighbors, giving each the valid color with the lowest possible ID.

The performance of this strategy heavily depends on how well we can evaluate the pertinence of the available options. Of course we can't give an absolute and exact score to all options (otherwise our greedy algorithm would give the perfect solution), so we have to rely on heuristic methods to get a correct estimation.

The greedy option tends to start very good, but loses performance near the end when all the remaining options are bad.

### Level 3 : Randomized greedy

Greedy algorithms have a big problem : if we run the algorithm several times, it will always start at the same point and take the same steps to build its solution, and always produces the same answer.

We can introduce small perturbations in the scoring function to make the algorithm take a different path in building solutions each time. The short-term benefits will be a bit lower, but it will sometimes result in a better long-term situation that gives us a higher score. Randomized greedy is similar to Monte Carlo, we have to use a lot of computing power to maximize the efficiency of the strategy.

- In TSP, instead of always taking the closest node, we can take one of the 2 or 3 closest nodes picked randomly instead. We can also randomize the city we start from.
- In the graph coloring greedy heuristic, we sorted nodes by descending degree. To add some randomness, we could break sorting ties in random order, or also introduce some random swaps in the order we visit nodes.

### Level 4 : Customized

This is the most vague of all strategies, but also one that you should aim for. Get creative and find a smart way to solve the problem !

TSP can be optimally solved for small instances. What if you divided your large dataset of cities into smaller local groups that you can solve, and stitched all the parts back together ?

### Level 5 : Heaviy artillery

This level is made for the Hash Code experts (who have probably fallen asleep by now), and it's pretty safe to say that you will be in the top 100 teams if you manage to implement this solution.

To make it work, you have to identify the underlying NP-complete problem, and use a solver made for that problem (such as [OR-Tools](https://developers.google.com/optimization) made by Google).

These solvers are extremely optimized and have years of research behind them, so this strategy is way better than all previous propositions. However it's easier said than done, as identifying an NP-complete problem isn't trivial (try to identify it in [last year's problem](https://storage.googleapis.com/coding-competitions.appspot.com/HC/2019/hashcode2019_qualification_task.pdf) -- hint : it's one of our 3 examples), and also the Hash Code problems often introduce some variations that aren't managed by your solver.

### Bonus : meta-heuristics

Meta-heuristics are techniques that can be used in a large range of problems. Monte-Carlo and greedy can be considered meta-heuristics, but there are other ones that would need an entire blog post dedicated to them, you should check them out if you're interested in optimization techniques. Meta-heuristics can be classified into two groups :

- Global : They are the base of the solution, and can work on their own. The most common and interesting ones are simulated annealing, ant colonies, and genetic algorithms. As you can see, most of these algorithms are inspired by nature !
- Local : We use local meta-heuristics to optimize an existing solution. We can start from any solution, but they most often run faster when initialized from a good starting point. There are two main techniques here : hill climbing (where we progressively tweak the solution to increase its score), and partial reconstruction (where we disassemble a small part of the solution to rebuild it better).

---

# 3. Applied optimization contests

On paper, all strategies presented previously are very promising. But once you are with your team with only 4 hours to produce a good solution, everything becomes much harder than anticipated. We will study a practical methodology to get most of the short time you have.

While it would be tempting to try a super advanced and complex solution, it makes it very easy to lose track of progress and time, and can be frustrating when things don't work out as intended. That's why I recommend to work in cycles if you're not a seasoned expert. This year will be my 6th Hash Code and I still love this method, so you can probably trust me on that one ;)

First of all, let's build a small timeline of the contest. The first 20 to 30 minutes will be dedicated to reading and understanding the contest subject. After that, you need to work in cycles where your solution is gradually improved. This is very similar to agile project management, where you first build the MVP and work on incremental improvements. The goal here is to squeeze at least 3 or 4 development cycles into the 3.5 hours you will have left.

### First cycle : building the MVP

The entire algorithm will be divided in four software blocks that we'll detail in an instant, each person of your team will work on a single block (assuming there's 4 of you which is strongly recommended).

![img]({{ site.baseurl }}/images/2020-02-hashcode/blocks.png)

The first thing you have to do is to agree on a common basis for your data structures. All information is passed between blocks through variables so it's essential for teamwork that the format is 100% clear for everyone so the blocks can be assembled without bugs. You should specify the formats **before you start coding**.

![img]({{ site.baseurl }}/images/2020-02-hashcode/blocks2.png)

There are two formats to specify, namely **dataset** and **solution**. Your initial version of specification will look closely like the input and output formats in the problem statements, but you can also precompute informations that can be reused later in the chain. 

Now that the formats are clear for everyone in your team, here is what each block has to do :

- Parser : transforms an input file into the specified dataset variable
- Solver : it's the heart of the program's logic, the solver implements the strategy
- Scorer : computes the score according to the formula specified in the problem statement. It should output exactly the same score as the Hash Code server
- Writer : it turns a solution variable into the Hash Code text format for submissions

### Second cycle : incremental improvements

The remainder of the contest will be dedicated on implementing better strategies. Most of the work will be done on the solver block, sometimes with improvements of the parser and optimizations of the scorer.

This is definitely the more chaotic part of the contest where you will have to deal with all small deviations you made previously (such as using a library that only works on one of your teammates' computer, true story). The team tends to divide into three roles, but you can definitely adapt this :

- Run : a single person is in charge of running the code. The solution will often rely on some sort of Monte Carlo, so the runner will have to watch closely several processes and upload the new submission each time the best score is beaten.
- Strategy : two teammates to design and implement new solver evolutions
- Utility : one person to code useful functions that the rest of the team needs. For example, the two strategy players might ask for a Dijkstra implementation on a graph, which the utility player provides. If you have someone in your team with a speed-code background, it's definitely one of the fitting roles.

## Tools

As a team, you will have to work on the same codebase in a very limited time, so code collaboration tools will be absolutely necessary.

While git might be recommended in large-scale projects, the process of committing and potentially resolving merge conflicts will be burdensome on your tight time management.

Here are some other options you should explore :

- Google Docs : not the best, but you can always copy-paste your functions in there and have a single person run the program
- Visual Studio Live Share : you can edit the same file across multiple computers, and there are even shared terminals
- Google Colaboratory : a jupyter-style code collaboration tool with free online computing power including a GPU

# Conclusion

I hope you enjoyed this introduction to Google Hash Code. This was the first of a two-part article, the second half is coming shortly and will focus on the [2020 Hash Code practice round](https://hashcodejudge.withgoogle.com/#/rounds/4684107510448128/) that you can play if you're registered.

If you are interested in participating in this year's Google Hash Code on Feb. 20, Sogeti is hosting seven hubs in France where you can come with your team and play alongside local rivals ! There will be pizza and prizes for the best teams in each hub, and registration is completely free ! Hubs are in the following cities :

- Paris
- Lyon
- Toulouse
- Aix
- Nantes
- Strasbourg
- Rennes

Register for Google Hash Code on [this link](https://codingcompetitions.withgoogle.com/hashcode/register), then create your team (2 players minimum) on the [Judge system](https://hashcodejudge.withgoogle.com/), and don't forget to specify the Sogeti hub of your choice as spots are limited !