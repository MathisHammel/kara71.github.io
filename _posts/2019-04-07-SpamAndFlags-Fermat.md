---
layout: post
title: SpamAndFlags CTF Writeup - Fermat's Last Theorem
---

This weekend, I played SpamAndFlags CTF Teaser 2019, which was a very interesting CTF. I didn't play for too long, having another CTF and the Google Code Jam quals at the same time, but there was a particular task that I immediately fell in love with : the goal was to find a counter-example to Fermat's last theorem in order to get a flag.

Fermat's last theorem states that there are no x,y,z > 1 and n > 3 such that a^n + b^n = c^n. This famous conjecture made by Pierre de Fermat around year 1630 was proven to be true by Andrew Wiles in 1994.

Of course, we are not asked to find a counter-example in the general case, which would of course be impossible. The checker is flawed, and we have to find an example that the checker accepts.

```c
#include <stdio.h>
#include <stdlib.h>

void fail(void)
{
    puts("FAIL!");
    exit(0);
}

long long safe_add(long long x, long long y)
{
    long long r = x + y;
    if (r < x)
        fail();
    return r;
}

long long safe_mul(long long x, long long y)
{
    long long r = x * y;
    if (r < x)
        fail();
    return r;
}

long long safe_pow(long long x, long long n)
{
    long long r0 = 1, r1 = x;
    for (int i = 63; i >= 0; i--) {
        if (n & (1LL << i)) {
            r0 = safe_mul(r0, r1);
            r1 = safe_mul(r1, r1);
        } else {
            r1 = safe_mul(r0, r1);
            r0 = safe_mul(r0, r0);
        }
    }
    return r0;
}

int main(void)
{
    long long a, b, c, n;
    if (scanf("%lld%lld%lld%lld", &a, &b, &c, &n) != 4)
        fail();
    if (n < 3 || a <= 0 || b <= 0 || c <= 0)
        fail();
    if (safe_add(safe_pow(a, n), safe_pow(b, n)) != safe_pow(c, n))
        fail();

    FILE* f = fopen("flag.txt", "rt");
    char buf[1024];
    if (f == NULL || fgets(buf, 1024, f) == NULL) {
        puts("Cannot read flag, pls contact organizers!");
        return 0;
    }
    puts(buf);
    return 0;
}
```

## Integer overflows

The flaw lies in the fact that C uses fixed-size numbers. Here, all variables are of type `long long` which have to be *at least* 64 bits long according to the C specification, but it's safe to assume here that the compiler will make it exactly 64 bits, not more.

If you are not aware of the main issue with fixed-size integers, let's demonstrate with a small multiplication calculator :

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    long long a, b, product;
    while(1)
    {
      scanf("%lld %lld", &a, &b);
      product = a * b;
      printf("%lld * %lld = %lld\n\n", a, b, product);
    }
    return 0;
}
```

It reads two integers, multiplies them together and prints the result. Here is the calculator in action :

```
$ ./calc
4 7
4 * 7 = 28

3 1000
3 * 1000 = 3000

12345 6789
12345 * 6789 = 83810205

6195341542 2977518503
6195341542 * 2977518503 = 10
```

Wait, what just happened ???

In the last example, we witnessed an integer overflow. It happens when a number is too large for the size allocated to it, so the bits to the left are discarded. Let's have a detailed look at what happened.

Remember that our `long long` integers have 64 bits to work with ? That's usually plenty of memory to work with, but it's sometimes not enough. The result of 6195341542 * 2977518503 should be 18446744073709551626, which is just a little more than 64 bits can hold.

Here are the two numbers that are multiplied together, represented in binary :

![magicmult]({{ site.baseurl }}/images/2019-04-spamandflags/magicmult.png)

These numbers are 33 and 32 bits long, so the result has 65 bits :

![overflow0]({{ site.baseurl }}/images/2019-04-spamandflags/overflow0.png)

Do you see that 65th bit all the way to the left ? It doesn't fit in the 64 bits allocated to the result, so the CPU just gets rid of it. All that's left is 1010 in binary, which translates to 10 in decimal. So this explains why the result is only 10 whereas we multiplied two large numbers together.

This is different from a buffer overflow, where the computer writes a string of bytes in a memory section which is too small to contain it. In that case, the overflowing data is still written and can be exploited to overwrite another memory section. In the case of integer overflows, leftover bits are simply ignored.

Some languages such as Python make the developer's life almost too easy by using arbitrary length integers, where the number of bits allocated is dynamic and integer overflows can't happen.

## Protections against overflows

This challenge implements some protection against integer overflows : for multiplication and addition, it checks that the result is not less than the first operand. If we tried to use 6195341542 * 2977518503 in the function safe_mul, the program would detect the overflow since 10 < 6195341542.

However, this protection is not perfect. For instance, computing 4 * 4611686018427387914 with `long long` integers, the result is 40. Since the result is higher than the first operand, the execution continues normally and we have proven a flaw in the overflow detection.

The exponentiation is implemented as the square-and-multiply algorithm, which uses only log(N) multiplication operations to compute a number to the power N, instead of N multiplications with a naive algorithm. This is nice because it means less verifications against overflows.

## Solving the challenge

### Birthday attack

My first idea for the challenge was to pick a power N, and then compute random numbers to the power N until a collision is found. This is similar to a cryptographic birthday attack which is somehow feasible when working with 64 bits. However, the protection against overflows makes it harder to find valid powers, which I could probably ignore if I hadn't such a toaster PC to work with :)

We need to get smarter. Let's get into some math and binary operations.

### Working with binary product

The exponentiation algorithm is based on multiplication, and we have to get a correct understanding of what this operation does under the hood. Multiplying binary numbers might not sound easy, but we can use some properties of numbers to make it simple. Let's look at the product of 581 by 1686 :

![mult0]({{ site.baseurl }}/images/2019-04-spamandflags/mult0.png)

Now this result isn't easy to come by if you're not familiar with binary operations. Let's break it down into simpler operations. First, we can decompose the first number into simpler binary numbers, like we could decompose 1234 as 1000 + 200 + 30 + 4. This effectively decomposes our binary number into powers of two.

![decomp0]({{ site.baseurl }}/images/2019-04-spamandflags/decomp0.png)

Then, we use distributivity of multiplication over the addition. This is the complicated term which says (a + b + c + d) * x = a*x + b*x + c*x + d*x.

![distrib0]({{ site.baseurl }}/images/2019-04-spamandflags/distrib0.png)

We now have 4 terms to compute and add together. The fourth term is trivial to compute, and the others aren't too hard either : in the same way you can multiply 156246 * 1000 = 156246000, you can multiply any binary number by a power of two by adding the right amount of zeros to its right. In binary arithmetics, it is more common to see it as "shifting the number to the left" instead of "adding zeros to the right", but it's really the same thing.

When all four terms have been computed, we add them together through simple binary addition which I won't detail :

![mult1]({{ site.baseurl }}/images/2019-04-spamandflags/mult1.png)

You will notice that this multiplication algorithm is strangely similar to the method you learn in primary school to multiply decimal numbers. It turns out working with binary numbers is even easier since there are no multiplication tables to learn !

### Smart Fermat collisions

As we have seen in the previous example, the result of binary multiplications can quickly become "unpredictable", especially when we do several rounds of multiplication in a row, as is the case when raising to the Nth power.

However, in some carefully chosen cases, we can make sure that the output is clean and predictable, then exploit it with integer overflows. Let's take the product of 129 by itself. It's hard to do in my head, but gets much easier when I see the binary decomposition :

![sparse0]({{ site.baseurl }}/images/2019-04-spamandflags/sparse0.png)

We compute the two terms separately like last time, and add them together.

![sparse1]({{ site.baseurl }}/images/2019-04-spamandflags/sparse1.png)

This time, the addition was clean : we only had to add two terms with a two 1s in each. The "unpredictability" of the result directly depends on the number of 1s (also called the Hamming weight) of each operand.

By working with such numbers, we can craft special powers that will be predictable through exponentiation and overflows. Let's look at what happens when squaring 2^33 + 1 in 64-bit arithmetic:

![overflow1]({{ site.baseurl }}/images/2019-04-spamandflags/overflow1.png)

The leftmost bit does not fit in the 64 bits allocated, so it is discarded. This does not trigger the overflow check because the result is greater than the first operand.

We end up with an interesting result, which is that (2^33 + 1) ^ 2 = (2^34 + 1) when it overflows. This property is global : we have (2^N + 1) ^ 2 = (2^(N+1) + 1), given that N ≥ 32 (otherwise the result is too small and the overflow doesn't happen) and N < 63 (otherwise the middle bit also overflows).

Recursively, we also have

(2^N + 1) ^ 4 = (2^(N+2) + 1)

(2^N + 1) ^ 8 = (2^(N+3) + 1)

and so on.

### Closing the deal

Let's get back to the inital problem. In order to verify the equation A^N + B^N = C^N, we could pick C in the form (2^K + 1) with 32≤K≤62 and N as a power of 2, let's say N = 2^M. Then we can apply the previous property and have C^N be in the form of (2^(K+M) + 1).

We can then choose A such that A^N = 2^(K+M) and B = 1, and we will have a correct solution to the equation.

The hardest part is finding a good candidate for A. The value of M is constrained by 32≤K+M≤62, otherwise the above property would not be conserved. Having K strongly constrained as well between 32 and 62, we can search exhaustively for candidates of A in each possible value of (K,M).

It turns out there are only 3 candidates for this strategy. We pick any and submit it :

```
$ nc 35.195.26.244 1337

8 1 17592186044417 16
SaF{You_should_have_been_faster!_https://etherscan.io/tx/0x4249377e6f654d6c814c2ae1390a60e5133b68ce772c7e04ee7644972e56d9e5}
```

And there we get the flag ! It took me around 30 minutes to solve this challenge, which is a time I'm really proud of. My actual solution involved less mathematical analysis and more bruteforcing than this article shows, but the idea was the same :)

With only this flag after 10 hours of competition, my team took 6th place in the CTF, and slowly moved down to 16th place as other teams got more flags.