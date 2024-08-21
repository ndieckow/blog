---
title: 'Some Thoughts on Programming Pearls: Column 12'
date: 2024-08-21
math: true
toc: true
author: Niklas
tags:
    - programming
    - probability
    - math
categories:
    - Math
---

The following question was inspired by Column 12 "A Sample Problem" from the book "Programming Pearls" by Jon Bentley. It's about randomly sampling $m$ integers from $\{1,\dots,n\}$, where $n$ is typically much larger than $m$. The first proposed solution runs in $\mathcal O(n)$ time, but the author also proposes a second solution. It works by uniformly sampling from $\{1,\dots,n\}$, adding the new integer to a set, and returning the set once it reaches a size of $m$.
Depending on how small $m$ is compared to $n$, the second solution might be faster: for small $m$ it works in $\mathcal O(m \log m)$ time. I was interested in the question of how the runtime changes, when $m$ becomes closer to $n$. Clearly, if we want to sample $90$ numbers from $\{1,\dots,100\}$, we will end up with quite a lot of repeating numbers toward the end.

To answer this, let's define a few random variables:
* $X$ is the number of iterations,
* $Y$ is the number of repeat draws; note that $Y = X - m$,
* Let $Y_i$ be $1$, if the $i$-th draw was a repeat draw and the current set size is at most $m$. Otherwise it is $0$. Note that $Y = \sum_{i=1}^\infty Y_i$,
* To add to the superscript confusion, let's also define partial sums $Y^{(i)} = \sum_{k=1}^{i} Y_k$.

Our goal is to compute $\mathbb E[X] = m + \mathbb E[\sum_{k=1}^\infty Y_i] = m + \sum_{k=1}^\infty \mathbb E[Y_i]$.

The first observation is that the probability of $Y_i$ being $1$ depends on the values of $Y_k$ for $k < i$: This is because the probability of a repeat draw is $C_i/n$, where $C_i$ is the size of our set at the start of iteration $i$, and, in a particular instance $\omega$ of our random experiment, $C_i(\omega) = (i-1)-Y^{(i-1)}(\omega)$. Because we cannot make the probability of a random variable depend on another random variable directly, we make use of the *law of total probability*.

$$\mathbb E[Y_i] = P(Y_i = 1) = \sum_{k=0}^{i-1} P(Y_i = 1 \mid Y^{(i-1)} = k) P(Y^{(i-1)} = k).$$

For the conditional probability, we have
$$P(Y_i = 1 \mid Y^{(i-1)} = k) = \frac{i-k}{n} P(Y^{(i-1)} \geq i-m),$$
where the second term is due to the fact that $Y_i$ should only be $1$, if the current size of the set is smaller than $m$.

For the probability of the partial sums, we get:
$$P(Y^{(i)} = k) = P(Y^{(i-1)} = k) P(Y_i = 0) + P(Y^{(i-1)} = k-1) P(Y_i = 1).$$

This formulation lends itself to a nice recursive algorithm that approximates $\mathbb E[X]$. I simply cap the computation at $4n$, which seems to work well, and further increasing it does not really change the result.

```python
import numpy as np

def EX_approx(m, n):
    if m == 1:
        return 1

    s = 4*n
    
    P = np.zeros((s,s))
    Y = np.zeros(s)

    # Initialization of values.
    Y[1] = 1 / n
    P[1][0] = (n-1) / n
    P[1][1] = 1 / n

    for i in range(2,s):
        for k in range(i):
            Y[i] += (i-k) / n * P[i-1][k]
        if i-m >= 0:
            Y[i] *= P[i-1][i-m:].sum()
        for k in range(i):
            P[i][k] = P[i-1][k] * (1 - Y[i]) + P[i-1][k-1] * Y[i]
    
    return m + Y.sum()
```

Note that I used 1-indexing in the equation but 0-indexing in the code. So, for example, `Y[1]` corresponds to $\mathbb E[Y_2]$ and `P[1][0]` is $P(Y^{(2)} = 0)$.

Below are the results for $n = 10$, rounded to two decimals.
$m$|Expected iterations
--|-----
1 | 1.0
2 | 2.36
3 | 3.77
4 | 5.37
5 | 7.20
6 | 9.33
7 | 11.84
8 | 14.87
9 | 18.62
10 | 23.48

I ran an experiment to see how well it holds up in real experiments. It looks okay, but slightly wrong:

![](plot.png)

So there is still an error in my calculations somewhere. Hopefully I'll find it!