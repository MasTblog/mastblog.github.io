---
layout: post
title:  "BGS Factorial (Concise Version)"
toc: true
date:   2023-09-03 01:23:04 -0700
---

In this article we compute $n! \bmod p$ in
$O(\sqrt p \log p)$ time.
The time complexity is in terms of $p$ since if $n > p$, then $n! \equiv 0 \pmod p$.

## Credit

- This article is mainly based off of [this article](https://web.archive.org/web/20201026035551/https://min-25.hatenablog.com/entry/2017/04/10/215046) by Min_25
- The math and derivations are from *Linear recurrences with polynomial coefficients and application to integer factorization and Cartier-Manin operator* by Alin Bostan, Pierrick Gaudry, and Eric Schost. [(Link)](https://mathexp.eu/bostan/publications/BoGaSc07.pdf)
	- To my knowledge, this is the first written record of this method

## Prerequisites

You should be familiar with
- Modulo and modular inverse
- FFT and its applications

## Formula

We define a polynomial $f(x) = \prod_{i=1}^n (x + i)$. We note that $f(0) = \prod_{i=1}^n i = n!$.

Computing $f(x)$ this way is slow and doesn't yield any benefit. Instead, we use a square root decomposition strategy.

Let $v$ be $\lfloor \sqrt p \rfloor$ and $g(x) = \prod_{i=1}^v (x+i)$. Then

$$n! = \left(\prod_{i=0}^{v-1} g(vi) \right)\cdot \left(\prod_{i=v^2+1}^n i \right) $$

The rest of the article deals with computing the first product, evaluating $g(0), g(v), g(2v), \dots, g((v - 1)v)$.

## Main Idea

Let $g_d(x) = \prod_{i=1}^d (x+i)$. Our main task is, given
$g_d(0), g_d(v), \dots, g_d(dv)$, we compute
$g_{2d}(0), g_{2d}(v), \dots, g_{2d}(2dv)$ in $O(d \log d)$ time. We can then compute $g(x)$ by repeatedly performing the task.

Let $G_d(i)$ be the sequence
$[g_d(i), g_d(i + v), g_d(i + 2v), \dots, g_d(i + dv)]$. Given $[g_d(0), g_d(v), g_d(2v), \dots, g_d(dv)] = G_d(0)$, the plan is to

1. Compute $G_d(v)$, $G_d(dv)$, and $G_d(dv + d)$
2. Using the computed values, for each $i$ from $0$ to $2dv$, compute $g_{2d}(i) = g_d(x) g_d(i + dv)$

## Polynomial Sample Shift

We solve a more general version of Step 1 from above.

For some degree-$d$ polynomial $h(x)$, we are given $h(0), h(1), \dots, h(d)$ and
our goal is to compute $h(m), h(m + 1), \dots, h(m + d)$.

The Lagrange Interpolation formula on sampling points $(0, 1, \dots, d)$ and $x = m + k$ gives us

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d h(i) \prod_{j = 0, j \neq i}^d \frac{m+k-j}{i-j} \\
&= \Delta(m, k, d) \left( \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \right) \\
\end{align*}
$$

where

$$\delta(i, d) = \prod_{j=0,j\neq i}^d (i-j),\ \ \Delta(m, k, d) = \prod_{j=0}^d (m + k - j)$$


We can precompute the $\dfrac{1}{\delta(i, d)}$ part in $O(d)$ using the
formula

$$\frac1{\delta(i,d)} = \frac{i-d-1}i \frac1{\delta(i-1,d)}$$

and precompute $\Delta(m, k, d)$ in $O(d)$ using the formula

$$\Delta(m, k, d) = \frac{m+k}{m+k-d-1}\Delta(m, k - 1, d)$$

The second term is a sum of a product of $\dfrac{h(i)}{\delta(i, d)}$, a function of $i$, and
$\dfrac1{m + k - i}$, a function of $-i$, letting us express it as a convolution. In particular, we define the polynomials

$$p(x) = \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} x^i,\ \ q(x) = \sum_{i=0}^{2d} \frac{1}{a + i - d}x^i$$

Then for $0 \leq k \leq d$, the
$(k + d)$-th coefficient is

$$
\begin{align*}
\sum_{i = 0}^d
\frac{h(i)}{\delta(i, d)} \frac{1}{m + k - i}
\end{align*}
$$

which is exactly the second term.

## Constant-Factor Optimizations

- The bottleneck of this algorithm is computing a $d$ by $2d$ size convolution. This can supposedly
be reduced to just a $d$ by $d$ convolution using the "[middle product](https://inria.hal.science/inria-00071921/document)" idea
- As usual, if $p$ is not known at compile time, [Montgomery Multiplication](https://cp-algorithms.com/algebra/montgomery_multiplication.html) can be used