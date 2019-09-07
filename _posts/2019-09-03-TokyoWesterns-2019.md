---
layout: post
title: TokyoWesterns CTF 2019
---

In this article, I want to illustrate three levels of solving a crackme problem. The challenge is called Easy Crack Me, from TokyoWesterns CTF 2019. I really enjoyed playing this CTF which was very hard as expected (even the "easy" problems), but I managed to briefly make it to the top 100 even though I was playing alone.

We're going to solve the challenge in three different ways, starting with a very tedious way (which would usually be the beginner's approach) and ending with a fast method to solve many easy crackmes, used by all experts.

## The challenge

A single file is provided, which is an executable. Its behaviour is very typical of a crackme: it expects a single input which will be checked by the program. If the input is correct, the flag is displayed.

```
$ ./easy
./bin flag_is_here

$ ./easy 123456
incorrect

```

We need to understand what the executable does in order to provide a valid input and get the flag.

Here I'll be using the decompiled C code in IDA Pro to make the program logic clearer for non-experts, but there are a lot of free alternatives that are equally valid such as Radare2 or Binary Ninja.

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/graph.png)

Everything is contained in the `main` function, but the execution flowchart can look a bit overwhelming because of some nested loops and lack of code factorization. Let's break it down by looking at the code:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/main_signature.png)

The function's signature is very typical of any C program, but IDA doesn't guess symbol names so argc, argv and envp have ugly names... Let's do some quick renaming for our programmer friends:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/main_renamed.png)

## Flag checking

After quite a lot of variables allocations and other memory checks, we have the first useful lines of the program. Luckily the program isn't obfuscated, so it's quite easy to read and follow.

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/format_condition.png)

Here, we have the first two conditions that make the input (stored in `argv[1]`) valid:

- The input is 39 chars long
- It starts with `TWCTF{` and ends with `}`. This is the flag format for TokyoWesterns CTF, but we don't know what the 32 chars in the middle are going to be.

It turns out, the whole program is made of conditions like what we just saw. The structure is always the same:

```
if(!check) {
	puts("incorrect);
	exit(0);
}

```

If none of the checks have failed, one of the last lines of the function prints this message, along with the supplied input:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/correct.png)

There are 10 checks in total, including the 2 we saw above.

### Count check

The next verification counts how many of each hexadecimal character there is in the middle part of our input, and checks it against hard-coded values.

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/count_check.png)

The check can be split into three parts:

- Green: allocating variables. `s1` through `v45` are misinterpreted by the compiler, they should all be grouped into a `int[16]` array. Same for `v47` and `v48`, which is actually `0123456789abcdef` stored in `v47`.
- Blue: counting the hex chars in the input, and storing into `s1`. I've never seen such a way of counting occurrences with `strchr`, but it works.
- Red: comparing with expected values via `memcmp(&s1, &expected_counts)` and triggering a failure if they are different.

### Group checks

The next category regroups 4 very similar verifications on the input, which can be separated into two pairs of two checks. The middle part of the input (the 32 hex chars) are split into groups of 4 characters, in two different ways:

- Groups of 4 consecutive chars: `s[0], s[1], s[2], s[3]` then `s[4], s[5], s[6], s[7]`, etc.
- Groups of 4 chars with jumps of length 8: `s[0], s[8], s[16], s[24]` then `s[1], s[9], s[17], s[25]`, etc.

On each group of 4 chars obtained by these two methods, two simple sums are computed:

- The sum of the 4 ASCII values
- The bitwise xor of the 4 ASCII values

There are 32 checksums in total (2 ways of splitting x 8 groups x 2 checksums), and all of them must match the expected values. Here's what it looks like in C pseudocode (only the first splitting method is shown).

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/group_check.png)

Once again, we can split the verification in 3 parts:

- Green: allocating two arrays `int v22[8]` and `int v26[8]`. These two arrays will contain checksums for each group of 4 characters.
- Blue: computing and saving each checksum (addition and xor).
- Red: comparing both checksums with expected values.

### Type check

This one verifies the type of each character in the flag. There are three possible types: digit, letter between a and f, and other (which is unused). You know the drill, it's then compared with preset values and exits if the type of any char is different than expected.

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/type_check.png)

### Even checksum

This one computes the sum of the ASCII values of `s[0], s[2], s[4], ...`. The expected sum is 1160, and the program exits otherwise.

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/even_checksum.png)

### Fixed checks

Fixed checks, aka free characters in the flag:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/fixed_check.png)

Based on this, we already have many characters of the expected input: `TWCTF{_f___87__________2_______4_____5}`.

Now that we have all our constraints laid out, let's find a way to use all that information to make a flag!

## Crazy programmer method

The first solving method I want to show is a right mix between constraint resolution and exhaustive search (aka bruteforce). It's by far the most complex way to solve this challenge, but also the most educative because we do it all ourselves.

We are starting with a huge search space, which would take years to bruteforce. Little by little, we will use properties of the 10 available checks to prune the search space, ultimately ending with a single solution. There are many ways to do this reduction and the presented way might not be the most optimized, but trust me it works.

Our solution will mostly based on the group checks. Remember the hex string is split into groups using different methods: `s[0], s[1], s[2], s[3]` will be called method A, and `s[0], s[8], s[16], s[24]` is method B. We can visualize this:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/groups.png)

Groups of type A are surrounded by the same color, and groups of type B have the same background color. These constraints can be seen as some sort of complex sudoku, where the possibilities are huge and you have to be smart to solve it.

The first thing we will do, is extract all hard-coded values from the program checks. I'll spare you the details of the process, but it was mostly copy-pasting from IDA with some regex to extract the values.

```python
# Expected count of each char in the flag from 0 to f
CHARCNT = [3, 2, 2, 0, 3, 2, 1, 3, 3, 1, 1, 3, 1, 2, 2, 3]

# 0 = letter, 1 = digit
CATEGORY = [0, 0, 1, 0, 1, 1, 1, 1, 0, 1, 1, 0, 0, 1, 1, 0, 1, 1, 0, 1, 0, 0, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1]

# Checksums of groups A
A_ADDACCS = [350, 218, 303, 305, 256, 305, 251, 258]
A_XORACCS = [82, 12, 1, 15, 92, 5, 83, 88]

# Checksums of groups B
B_ADDACCS = [297, 259, 299, 305, 309, 267, 255, 255]
B_XORACCS = [1, 87, 7, 13, 13, 83, 81, 81]

EVEN_CHECKSUM = 1160

CONSTRAINTS = {31: '5',
               1: 'f',
               5: '8',
               6: '7',
               17: '2',
               25: '4'}
			   
```

A few checks can't hurt (and may save us hours of debugging):

```python
assert len(CHARCNT) == 16
assert sum(CHARCNT) == 32
assert len(CATEGORY) == 32
assert len(A_ADDACCS) == 8
assert len(A_XORACCS) == 8
assert len(B_ADDACCS) == 8
assert len(B_XORACCS) == 8

```
			   
A quick note on string indices: as we'll only be working on the middle part of the flag, we consider `s` to be of length 32, with the middle part starting at `s[0]` instead of `s[6]`.

### First batch of group constraints

Without using any other constraint, each group only has 4 characters with 16 possibilities each, which means they can only have 65536 different possibilities. We can see that the checksums on each group are computed with commutative operations: addition and xor. This means that a group `aa01` and `0a1a` would have the same checksums. Therefore, we can make the search a little bit faster by avoiding checking permutations multiple times.

Our first goal is to compute all possible values for each of the 8 groups of type A. Using the type constraint, we know how many letters and digits each group has, and we can reduce the search even more. For each possibility matching both checksums, we save it as a candidate for the group.

```python
import itertools

print('Starting group A smart enumeration')
all_group_a_solutions = []
for k in range(0,32,4):
    cats = CATEGORY[k:k+4]
    print('  Group A %d: %d digits, %d letters.' % (k, cats.count(0), cats.count(1)))
    
    letter_cs = itertools.combinations_with_replacement('abcdef', cats.count(0))
    digit_cs = itertools.combinations_with_replacement('012456789', cats.count(1))

    posscnt = 0
    solutions = []
    for letters, digits in itertools.product(letter_cs, digit_cs):
        posscnt += 1
        candidate = list(itertools.chain(letters,digits))
        xoracc = 0
        addacc = 0
        for c in map(ord, candidate):
            xoracc ^= c
            addacc += c
        if xoracc == A_XORACCS[k//4] and addacc == A_ADDACCS[k//4]:
            solutions.append(candidate)

    print('    [+] Found %d base solutions out of %d candidates' % (len(solutions), posscnt))

```

The second step is to take each potential solution for each group, and find all its correct permutations. To be correct, a permutation needs to:

- Have digits and letters in the correct positions.
- Not create any conflicts with the preset characters `TWCTF{_f___87__________2_______4_____5}`.

```python
	sol_permutations = []
    for solution in solutions:
        for perm in itertools.permutations(solution):
            for i in range(4):
                if ('0' <= perm[i] <= '9') ^  CATEGORY[k+i]:
                    break
                if CONSTRAINTS.get(k+i, perm[i]) != perm[i]:
                    break
            else:
                sol_permutations.append(perm)
    print('    [+] Found %d valid permutations.' % len(sol_permutations))
            
    all_group_a_solutions.append(sol_permutations)
	
```

If we only had a single base solution for each group, we could directly bruteforce all permutations of our 8 solutions and find the flag really quickly. However, reality is often disappointing:

```
Starting group A smart enumeration
  Group A 0: 3 digits, 1 letters.
    [+] Found 7 base solutions out of 504 candidates
    [+] Found 8 valid permutations.
  Group A 4: 0 digits, 4 letters.
    [+] Found 4 base solutions out of 495 candidates
    [+] Found 6 valid permutations.
  Group A 8: 2 digits, 2 letters.
    [+] Found 14 base solutions out of 945 candidates
    [+] Found 56 valid permutations.
  Group A 12: 2 digits, 2 letters.
    [+] Found 11 base solutions out of 945 candidates
    [+] Found 44 valid permutations.
  Group A 16: 1 digits, 3 letters.
    [+] Found 6 base solutions out of 990 candidates
    [+] Found 8 valid permutations.
  Group A 20: 2 digits, 2 letters.
    [+] Found 17 base solutions out of 945 candidates
    [+] Found 68 valid permutations.
  Group A 24: 1 digits, 3 letters.
    [+] Found 13 base solutions out of 990 candidates
    [+] Found 22 valid permutations.
  Group A 28: 1 digits, 3 letters.
    [+] Found 8 base solutions out of 990 candidates
    [+] Found 8 valid permutations.
```

We started from 65536 possibilities per group, and now we're down to 6-68. Pretty good, right ? But there's still a long way to go. Let's look at the search space now, by computing how many total combinations of the groups are possible:

```
from functools import reduce
import operator

space_a_size = reduce(operator.mul, map(len, all_group_a_solutions))
print('Initial group A search space has size %d' % space_a_size)
```

The result `Initial group A search space has size 11323834368` is pretty encouraging, compared to the size of 1208925819614629174706176 we started with.

We then perform the same operation to find all candidates on groups B.

### Individual cross-checking

We now have a (very) reduced list of candidates for each group A and B. What we can do now, similar to solving a sudoku, is eliminate the impossible values.

For example, all group A candidates say either `s[0] = 'e'` or `s[0] = '7'`. Now, all group B candidates say `s[0]` is either `'3'`, `'6'` or `'e'`. What we can deduce is that `s[0] = 'e'` and all candidates that say otherwise are out of the solutions pool.

By working iteratively, we can determine what are the possible and impossible values for each character, and eliminate candidates accordingly. Refining the candidates will allow us to update the possible values for the characters, which we can use to remove more candidates, and so on.

First, we determine what are the possible values of `s[i]` according to the group A candidates, for all values of `i`. This is stored in a `set` to avoid duplicates and make later operations easier:
```python
for i in range(10):
    avalues = [set() for _ in range(32)]
    for gp in range(8):
        for sol in all_group_a_solutions[gp]:
            for i in range(4):
                avalues[gp*4+i].add(sol[i])
				
```

Then, we do the same with B candidates:

```python
    bvalues = [set() for _ in range(32)]
    for gp in range(8):
        for sol in all_group_b_solutions[gp]:
            for i in range(4):
                bvalues[gp+8*i].add(sol[i])
				
```

Next, we look at all values that are both in `avalues` and `bvalues` for each character.

```python
    abvalues = [set() for _ in range(32)]
    for i in range(32):
        abvalues[i] = avalues[i].intersection(bvalues[i])
		
```

The last step is to remove all candidates that do not meet these new requirements:

```python
    new_group_a_solutions = []
    for gp in range(8):
        new_solutions = []
        for sol in set(all_group_a_solutions[gp]):
            for i in range(4):
                if sol[i] not in abvalues[gp*4+i]:
                    break
            else:
                new_solutions.append(sol)
        new_group_a_solutions.append(new_solutions)

    new_group_b_solutions = []
    for gp in range(8):
        new_solutions = []
        for sol in set(all_group_b_solutions[gp]):
            for i in range(4):
                if sol[i] not in abvalues[gp+8*i]:
                    break
            else:
                new_solutions.append(sol)
        new_group_b_solutions.append(new_solutions)
        
    all_group_a_solutions = new_group_a_solutions
    all_group_b_solutions = new_group_b_solutions


    space_a_size = reduce(operator.mul, map(len, all_group_a_solutions))
    print('New group A search space %s has size %d' % (list(map(len,all_group_a_solutions)), space_a_size))
    space_b_size = reduce(operator.mul, map(len, all_group_b_solutions))
    print('New group B search space %s has size %d' % (list(map(len,all_group_b_solutions)), space_b_size))

    space_ab_size = reduce(operator.mul, map(len, abvalues))
    print('Whole search space has size %d' % space_ab_size)
	
```

After 4 iterations, the algorithm has converged and greets us with a very pleasing search space size of 19440. The `Whole search space` indicates how many flag possibilities we have, if we take only possible values for each character. We can see it's much better to use the additional information of groups, which creates more constraints between individual characters.

```
Initial group A search space has size 11323834368
Initial group B search space has size 6440878080
Starting cross-checking step 1
  New group A search space [4, 2, 14, 5, 3, 16, 14, 2] has size 752640
  New group B search space [5, 1, 13, 9, 4, 6, 8, 5] has size 561600
  Whole search space has size 627174526648320
Starting cross-checking step 2
  New group A search space [4, 2, 11, 4, 3, 7, 12, 1] has size 88704
  New group B search space [5, 1, 12, 9, 3, 1, 3, 4] has size 19440
  Whole search space has size 36279705600
Starting cross-checking step 3
  New group A search space [4, 2, 11, 4, 3, 4, 12, 1] has size 50688
  New group B search space [5, 1, 12, 9, 3, 1, 3, 4] has size 19440
  Whole search space has size 3023308800
Starting cross-checking step 4
  New group A search space [4, 2, 11, 4, 3, 4, 12, 1] has size 50688
  New group B search space [5, 1, 12, 9, 3, 1, 3, 4] has size 19440
  Whole search space has size 3023308800
  
```

### Final search

The new search space of size 19440 now allows us to run an exhaustive search on all possible flags based on the cartesian product of group B candidates. First, we only keep solutions that verify the character count check.

```python
correct_b_cnt_candidates = []
for perm in itertools.product(*all_group_b_solutions):
    solution = ['' for i in range(32)]
    for gp in range(8):
        for i in range(4):
            solution[gp+8*i] = perm[gp][i]
    for i,c in enumerate('0123456789abcdef'):
        if solution.count(c) != CHARCNT[i]:
            break
    else:
        correct_b_cnt_candidates.append(solution)
print('Group B search found %d/%d results matching the count check' % (len(correct_b_cnt_candidates), space_b_size))

```

Then, we verify for each solution if all groups of type A created are in the candidate pool.

```python
correct_a_b_cnt_candidates = []
for solution in correct_b_cnt_candidates:
    if all(tuple(solution[4*gp:4*gp+4]) in all_group_a_solutions[gp] for gp in range(8)):
        correct_a_b_cnt_candidates.append(solution)
print('Solution space has size %d' % len(correct_a_b_cnt_candidates))

```

And the resulting search space is...

```
Group B search found 26/19440 results matching the count check
Final solution space has size 1
```

After about 3 seconds of processing time, we got it! The solution is unique, this must be the flag (and there is one check that we didn't even use). Print the flag and submit it, and bingo!

```
>>> 'TWCTF{' + ''.join(correct_a_b_cnt_candidates[0]) + '}'
'TWCTF{df2b4877e71bd91c02f8ef6004b584a5}'
```

While this was certainly the most brain-intensive solution, advanced CTF players are more likely to use tools to help them solve this kind of challenge. In my opinion it makes the whole thing way less fun, but at least you have time remaining to flag other challenges.

I decided to start with this method in order to avoid spoiling myself the challenge, and it took me a bit less than 3 hours between first download and flag submission. The other methods would easily have taken less thanhalf that time.

## Math nerd method

When it comes to clearly defined constraints, there is one tool that must come to your mind, the holy grail of constraint programming: Z3. It leverages many solvers in a single tool (SMT, SAT, Fixed point, ...) and can resolve huge sets of provided constraints. One of its main advantages is its Python interface, and the ease of installation of the solver `pip install z3-solver`.

What we did before by reducing our search space through various optimizations, z3 can do under the hood in a few milliseconds and find a result that satisfies the requirements.

Let's start by declaring our 32 variables, and creating a solver instance:

```python
from z3 import *
solver = Solver()
s = [BitVec('s%d'%i, 8) for i in range(32)]

```

Then, we can add our constraints one by one. First, the type (digit/letter) of each character.

```python
for i in range(32):
    lo, hi = (('a','f'),('0','9'))[CATEGORY[i]]
    solver.add(And(s[i]>=ord(lo), s[i]<=ord(hi)))

```

The single character constraints:

```python
for k in CONSTRAINTS:
    solver.add(s[k] == ord(CONSTRAINTS[k]))

```

Group checksums:

```python
for gp in range(8):
    solver.add((s[4*gp] + s[4*gp+1] + s[4*gp+2] + s[4*gp+3]) == A_ADDACCS[gp])
    solver.add((s[4*gp] ^ s[4*gp+1] ^ s[4*gp+2] ^ s[4*gp+3]) == A_XORACCS[gp])
    solver.add((s[gp] + s[gp+8] + s[gp+16] + s[gp+24]) == B_ADDACCS[gp])
    solver.add((s[gp] ^ s[gp+8] ^ s[gp+16] ^ s[gp+24]) == B_XORACCS[gp])
	
```

Even checksum:

```python
solver.add(sum(s[i] for i in range(0,32,2)) == 1160)

```

Count check:
```python
for i,c in enumerate('0123456789abcdef'):
    solver.add(PbEq([(s[j] == ord(c), 1) for j in range(32)], CHARCNT[i]))
	
```

Finally, we start the solver and export the flag.

```python
print(solver.check())
model = solver.model()

flag = 'TWCTF{'
for si in s:
    flag += chr(int(str(model[si])))
flag += '}'
print(flag)

```

The whole process takes 0.5 seconds.

Now that we've seen the power of z3, let's move to something even more powerful.

## Leet hacker method

For this one, we're going to use angr, a tool that puts a symbolic execution framework on top of z3. This is the equivalent of directly plugging a reverse engineer's brain into a program, using only a few lines of Python.

Let's load our executable and initialize our 39-byte input:
```python
import angr
import claripy

proj = angr.Project('./easy_crackme', load_options={"auto_load_libs": False})
argv1 = claripy.BVS("argv1", 39 * 8)
initial_state = proj.factory.entry_state(args=["./easy_crackme", argv1]) 

```

Now, we have to find the address we want the program to reach (aka the "goodboy"). In our case, it's going to be any of the following addresses:

![img]({{ site.baseurl }}/images/2019-09-TokyoWesterns/goodboy.png)

The only thing left is to start the exploration and export the flag:

```python
sm = proj.factory.simulation_manager(initial_state)
sm.explore(find=0x400e1a)
found = sm.found[0]
print(found.solver.eval(argv1, cast_to=bytes))

```


## Conclusion


I hope you liked the way of approaching a problem by three different angles. Make sure to share this post if you liked it!