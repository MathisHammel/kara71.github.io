---
layout: post
title: Facebook Hacker Cup 2019 - Qualification round
---

For the 4th year in a row, I've registered for the Facebook Hacker Cup, a great coding competition played in 4 rounds and a final, where my personal goal is to make it to the world's top 500 to earn a T-shirt. Last year I ended at the 282nd place out of 8200 which wasn't enough to advance to the last round, but I got the T-shirt ! If I win the same Hacker Cup T-shirt this time, I'll give it away to [one of my Twitter followers](https://twitter.com/MathisHammel) who retweets one of my Hacker Cup articles.

Let's get into the qualifier problems ! In order to advance to the next round, contestants needed to solve at least 1 problem correctly.

## Problem 1 : Leapfrog Ch. 1 (15 pts)

The problem is the following : We have a lake with N lilypads in a row. Each lilypad may be occupied by a frog. There are two types of frog :

- Alpha frog : there is only one Alpha frog, it starts on the first pad and wants to move to the last one. It can jump over a row of **one or more** Beta frogs **to its right**, and lands on the first free lilypad.
- Beta frog : it can move to a free lilypad to its immediate left or right.

Frogs can move in any order and can move more than once. The goal is to determine whether there is a way for the Alpha frog to reach its goal.

### Solving the problem

Let's solve a few examples by hand first.

The Alpha frog is represented as `A`, Beta frogs as `B` and empty lilypads as `.`

#### Example 1

`ABB.BBB.`

Without any Beta frogs moving, the Alpha frog can reach the right lilypad in two steps.

`.BBABBB.`

`.BB.BBBA`

So the answer for this testcase is yes. 

#### Example 2

`A....BBBBB`

We can move the beta frogs one by one (I'm not detailing the 14 steps) to reach the following state :

`ABB.B.B.B.`

Then, the Alpha frog can simply reach the exit by jumping over the strings of consecutive Beta frogs we have formed.

#### Example 3

In which case is it impossible to reach the end ? The trivial impossible case is when nobody can move :

`ABBBBB`

Since the Alpha frog can only reach an **empty** space and no Beta frogs can move to free a space, everyone is stuck and there is no solution.

#### Example 4

There is one other case where the Alpha frog can't reach its destination - when there aren't any Beta frogs to jump over. Look at this case :

`AB....`

Alpha can only reach the third space before getting stuck, and the Beta frog is stuck to its left and can't help.

#### Solution

We have identified two cases where the answer is no :

- Zero free lilypads
- Not enough Beta frogs

*How many Beta frogs is not enough ?*

We can start by seeing that when a Beta moves, it can be viewed as swapping positions with a free lilypad. It means we can shuffle any string of `.`s and `B`s to any other string with the same amount of `B` and `.`, and the only thing that matters in the testcase is the number of Beta frogs and lilypads.

There is also no need to move Beta frogs after we've finished placing them initially. It's a bit harder to explain, but I'll let you prove it by yourself if you want ;)

All in all, we need to find how to order Beta frogs to make a "bridge" for the Alpha to the rightmost free lilypad. The only way the bridge can't be crossed is if there are **two free lilypads in a row**.

With as many Beta frogs as free lilypads, we can make a string like `AB.B.B.B.B.`. Any less Beta frogs and we will have to make a hole in the bridge, and there is no solution. If we have more Beta frogs than lilypads, they can just fill an empty lilypad and the Alpha can still reach the end.

In the end, the solution is very simple :

```python
def solve(testcase):
    return '.' in testcase and testcase.count('B') >= testcase.count('.')
```

Just because the code is extremely simple doesn't mean the solution was easy to come up with !

## Problem 2 : Leapfrog Ch. 2 (15 pts)

This is almost the same problem as before, but now the **Alpha frog can also jump to the left**. Other rules are not changed.

### Solution

The only difference is that we can now use the following technique to cover more distance :

`AB.B..` (this would be a losing testcase in the first problem)

`.BAB..`

`.B.BA.`

`..BBA.`

`.ABB..`

`.AB.B.`

`..BAB.`

`..B.BA`

This time, we only need two or more Beta frogs to make the testcase solvable, no matter the number of empty lilypads.

```python
def solve(testcase):
    return '.' in testcase and testcase.count('B') >= 2
```

## Problem 2 : Mr. X (25 pts)

This problem was to me the most interesting of this round. My solution involves grammar parsing along with dynamic programming on a binary expression tree. I'll explain along the way.

In this challenge, we are given a boolean expression such as `(((X^1)&x)|(0&x))` (where ^ denotes the XOR operator). There is only one boolean variable in each expression : `X` is the negation of `x`.

The previous expression evaluates to 0 when x=0 and 1 when x=1. The goal of the problem is to edit the formula so that it evaluates to the same value regardless of the value of x. We need to return the smallest possible number of characters edited.

For example, changing `(((X^1)&x)|(0&x))` into `(((X^X)&x)|(0&x))` (1 character edited) makes the formula false every time.

We are only allowed to switch characters, not insert or remove anything.

### Grammar

The expressions use a very simple grammar which is explained in the problem statement. Having strict rules for valid expressions is very helpful, it helps with parsing the expressions and makes sure we do not miss any special case.

```
expression    := <atomicExpr> or <compositeExpr>

compositeExpr := (<expression><operator><expression>)
atomicExpr    := "0" or "1" or "x" or "X"

operator      := "&" or "|" or "^"
```

### Solution

Any expression and sub-expression has one input, x (used zero or more times in the expression). Thus, every formula has a simple truth table, such as :

![img truth1]({{ site.baseurl }}/images/2019-06-hackercup/truth1.png)

With one input, we only have four possible truth tables, which we can express as 0, 1, x and X. We can see that our formula has the same truth table as x :

![img truth2]({{ site.baseurl }}/images/2019-06-hackercup/truth2.png)

So, any expression can be reduced to one of the four atomic expressions.

Now, our goal is to edit the formula so that it reduces to either 0 or 1. If the expression is reduced to x or X, we have several ways to change it :

If it's an atomic expression, we can switch it to any other expression for an edition cost of 1.

If it's a composite expression, we can :

- Change the left sub-expression for a cost that depends on the left sub-expression
- Switch the operator for a cost of 1
- Change the right sub-expression for a cost that depends on the right sub-expression

We can write a recursive function that determines the edition cost of an expression to turn it into each of the 4 reduced forms. First, we need to manage atomic cases :

```python
if expr == '0':
    return [[0,1], [1,1]]
elif expr == '1':
    return [[1,1], [1,0]]
elif expr == 'x':
    return [[1,0], [1,1]]
elif expr == 'X':
    return [[1,1], [0,1]]
```

Here, we are representing the scores as a 2D array `[[cost_0, cost_x], [cost_X, cost_1]]`.

The index of a cost represents its truth table : for example, `cost[1][0]` represents the cost of X, which has truth table 1 0. We are using this representation to make it easier later, when we are dealing with the cost of switching the operator in a composite expression.

If the expression is composite, we will first have to parse it in order to find the operator and the two sub-expressions. For this, I'm using a simple linear pass on the expression which finds the operator. Then, the left and right children of the expression can be obtained by splitting on the operator.

```python
depth = 0
mid = 1
while True:
    if expr[mid] == '(':
        depth += 1
    elif expr[mid] == ')':
        depth -= 1
    elif expr[mid] in '&|^' and depth == 0:
        break
    mid += 1
```

`depth` counts how many brackets we are nested in. If an operator is found outside any brackets (= at depth 0), it means it's the operator of our current expression.

The last part is how we combine the scores of the two sub-expressions and a possible operator switch. It takes the cost of all combinations of reduced left and right sub-expressions and operator, and only keeps the lowest cost for each of the 4 reduced forms.

```python
lscore = scoreExpr(expr[1:mid])
rscore = scoreExpr(expr[mid+1:-1])
op = expr[mid]

score = [[500,500], [500,500]]
for lvl in range(2):
    for lvr in range(2):
        for rvl in range(2):
            for rvr in range(2):
                score[lvl & rvl][lvr & rvr] = min(score[lvl & rvl][lvr & rvr], lscore[lvl][lvr] + rscore[rvl][rvr] + (0 if op == '&' else 1))
                score[lvl | rvl][lvr | rvr] = min(score[lvl | rvl][lvr | rvr], lscore[lvl][lvr] + rscore[rvl][rvr] + (0 if op == '|' else 1))
                score[lvl ^ rvl][lvr ^ rvr] = min(score[lvl ^ rvl][lvr ^ rvr], lscore[lvl][lvr] + rscore[rvl][rvr] + (0 if op == '^' else 1))

return score
``` 

Here I'm using 500 for the base "infinity" cost. This is safe to use because the expression has a maximum length of 300, so there will always be a solution that costs 300 or less (for example, replace all x and X by constants).

In the end, we only have to submit `min(score[0][0], score[1][1])` since they represent the two acceptable reduced outcomes.

### Submitting the solution

The format of the Facebook Hacker Cup is quite special : the code runs exclusively on the contestant's machine. Facebook sends you a file containing several testcases and you have to upload the output of your program in the next few minutes.

This submission format used to be also used by Google Code Jam until 2017, and offers several advantages : 

- For the organizer, the competition can be run for tens of thousands of contestants at just the cost of a web server (no need for a judge server)
- For the contestant, you can use any language, and have full access to testcases

Everything ran smoothly and my program finished all testcases in a bit less than 2 seconds, which left me with about 5 minutes to check the inputs and outputs before locking in the submission.

There was a strange pattern in the outputs :

```
Case #1: 1
Case #2: 0
Case #3: 0
Case #4: 1
Case #5: 0
Case #6: 0
Case #7: 1
Case #8: 1
Case #9: 0
Case #10: 0
Case #11: 0
Case #12: 0
Case #13: 1
Case #14: 0
Case #15: 1
```

Every single one of my answers was a 0 or a 1 ! That is when you facepalm hard : either there is a bug and you have 5 minutes to fix it, or you just didn't take enough time to think.

I'm not sure which of these possibilities I would have preferred, but it turned out to be the second one.

### The return of Mr. X

This fact is hard to find if you haven't coded a working solution, but it's easy to prove that the cost of any expression is always 0 or 1 :

If the expression is atomic, make it a constant for a cost of 1 (or keep it if it's already a constant, for a cost of 0).

If the expression is composite, you can make its reduced form be a constant by only changing the operator for a cost of 1.

![img truth3]({{ site.baseurl }}/images/2019-06-hackercup/truth3.png)

Regardless of the reduced forms of the left and right sub-expressions, there is always one operator between them which makes the output constant. The cost is 0 if the output is already a constant, or 1 if we need to change the operator.

It's actually not even necessary to evaluate sub-expressions or determine which operator to use. The cost is 1 if the reduced form of the testcase is x or X, and 0 otherwise.

Now, how do we compute the reduced form ? It's very simple : we can evaluate the formula when x=0 and x=1, and it gives us its truth table. In python, we can even use the `eval` function, since the grammar yields valid Python expressions.

```python
def scoreExpr(expr):
    e0 = expr.replace('X','0').replace('x','1')
    e1 = expr.replace('X','1').replace('x','0')
    return int(eval(e0) != eval(e1))
```

This has to be the hardest one-liner solution I've ever come up with :)

## The end

Thankfully, the round went very well and I got all three submissions correct. I will be advancing to the next round, one step closer to the T-shirt !

![img score]({{ site.baseurl }}/images/2019-06-hackercup/score.png)
