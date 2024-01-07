---
layout: post
title:  "BGS Factorial (Detailed Version)"
date:   2099-09-03 00:22:04 -0700
toc: true
---
<!-- date is set in the future to disable viewing -->

In this article we compute $n! \bmod p$ in
$O(\sqrt p \log p)$ time.
The time complexity is in terms of $p$ since if $n > p$, then $n!$ contains $p$ as a factor and
$n! \bmod p$ is just 0.

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

Let $v$ be $\lfloor \sqrt p \rfloor$ and
$g(x) = \prod_{i=1}^v (x+i)$. Then $g(0) = v!$,
$g(v) = (v + 1)(v + 2)\cdots(2v)$, and so on
giving us the formula

$$ n! = \left(\prod_{i=0}^{v-1} g(vi) \right)\cdot \left(\prod_{i=v^2+1}^n i \right) $$

The first product computes $(v^2)!$ and the
second product is just the product of the rest
of the terms since $v^2$ can be a little less than $n$. There are at most $O(\sqrt p)$ such terms.

The rest of the article deals with computing the first product, evaluating $g(0), g(v), g(2v), \dots, g((v - 1)v)$.

### $O(\sqrt p \log^2 p)$ Method

Directly applying the above formula, $g(x)$
is a polynomial with $O(\sqrt p)$ coefficients
and we want to evaluate it at $O(\sqrt p)$
points, which can be done in $O(\sqrt p \log^2 p)$ time
using [multipoint evaluation](https://cp-algorithms.com/algebra/polynomial.html#multi-point-evaluation).


### $O(\sqrt p \log p)$ Method

To save a log factor, we can take advantage of certain facts.

The roots of $g(x)$ are $-1, -2, \dots, -v$,
which form an arithmetic progression.
In addition,
the points at which we want to evaluate $g(x)$
are $0, v, 2v, \dots, (v - 1)v$, also an
arithmetic progression.

Let $g_d(x) = \prod_{i=1}^d (x+i)$ be a degree-$d$ polynomial. The main idea is, given
$g_d(0), g_d(v), \dots, g_d(dv)$, we compute
$g_{2d}(0), g_{2d}(v), \dots, g_{2d}(2dv)$ in $O(d \log d)$ time.

We start with $d = 1$, then keep doubling until we have
$d \geq v$, which gives us $g(x)$. Our overall time complexity is given by
$O(v \log v) + O\left(\frac v2 \log \frac v2\right) + O\left(\frac v4 \log \frac v4\right) + \dots = O(v \log v)$
which is $O(\sqrt p \log p)$.

Let $G_d(i)$ be the sequence
$[g_d(i), g_d(i + v), g_d(i + 2v), \dots, g_d(i + dv)]$. Given $[g_d(0), g_d(v), g_d(2v), \dots, g_d(dv)] = G_d(0)$, the plan is to

1. Compute $G_d(v)$, $G_d(dv)$, and $G_d(dv + d)$
2. Using the computed values, for each $i$ from $0$ to $2dv$, compute $g_{2d}(i) = g_d(x) g_d(i + dv)$

### Polynomial Sample Shift

We solve a more general version of Step 1 from above, then discuss how to do that step using this more general version.

For some degree-$d$ polynomial $h(x)$, we are given $h(0), h(1), \dots, h(d)$ and
our goal is to compute $h(m), h(m + 1), \dots, h(m + d)$.

The Lagrange Interpolation formula on sampling points $(0, 1, \dots, d)$ gives us

$$h(x) = \sum_{i=0}^d h(i) \prod_{j = 0, j \neq i}^d \frac{x-j}{i-j}$$

We aren't actually going to interpolate anything, we are just taking advantage of this formula. Plugging in $m + k$ into $x$, we get

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d h(i) \prod_{j = 0, j \neq i}^d \frac{m+k-j}{i-j} \\
&= \sum_{i=0}^d \frac{h(i)}{\prod_{j=0,j\neq i}^d (i-j)} \prod_{j=0,j\neq i}^d (m + k - j) \\
\end{align*}
$$

Let $\delta(i, d) = \prod_{j=0,j\neq i}^d (i-j)$.
We can precompute values using the
formula $\frac1{\delta(i,d)} = \frac{i-d-1}i
\frac1{\delta(i-1,d)}$.

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \prod_{j=0,j\neq i}^d (m + k - j) \\
\end{align*}
$$

We note that $\prod_{j=0,j\neq i}^d (m + k - j)$
is the product of all terms $\prod_{j=0}^d (m + k - j)$ except for the i-th term, which we can write as $\frac1{m + k - i} \prod_{j=0}^d (m + k - j)$. We therefore have

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \prod_{j=0}^d (m + k - j) \\
&= \left(\prod_{j=0}^d (m + k - j) \right) \left( \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \right) \\
\end{align*}
$$

We can precompute the first term for all $k = 0, 1, \dots, d$. Let $\Delta(m, k, d) = \prod_{j=0}^d (m + k - j)$. For $k > 0$,
$\Delta(m, k, d) = \frac{m+k}{m+k-d-1}\Delta(m, k - 1, d)$, giving us a $O(d)$ way
to precompute it.

The right-hand side is a sum of a product of $\frac{h(i)}{\delta(i, d)}$, a function of $i$, and
$\frac1{m + k - i}$, a function of $-i$. This lets us express it as a convolution.

<!-- Use [p(x)]_i notation here too -->
In particular, we define the polynomials $p(x) = \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} x^i$ and $q(x) = \sum_{i=0}^{2d} \frac{1}{a + i - d}x^i$

Then their product $r(x) = p(x)q(x)$ is
$$
\sum_{i = 0}^d \sum_{j=0}^{2d} \frac{h(i)}{\delta(i, d)}\frac{1}{a + i - d} x^{i + j}
$$

Let $[f(x)]_i$ denote the $i$-th coefficent of $f(x)$.
Then the $c$-th coefficient of $r(x)$, or $[r(x)]_c$ is

$$
\begin{align*}
&\phantom{=}\sum_{i = 0}^{\min(c, d)} [p(x)]_i [q(x)]_{c-i} \\
&= \sum_{i = 0}^{\min(c, d)} \frac{h(i)}{\delta(i, d)} \frac{1}{m + (c - i) - d}
\end{align*}
$$

In particular, for $0 \leq k \leq d$, the
$(k + d)$-th coefficient or
$[r(x)]_{k + d}$ is

$$
\begin{align*}
&\phantom{=}\sum_{i = 0}^{\min(k+d, d)}
\frac{h(i)}{\delta(i, d)} \frac{1}{m + (k + d - i) - d} \\
&=\sum_{i = 0}^d
\frac{h(i)}{\delta(i, d)} \frac{1}{m + k - i}
\end{align*}
$$

which is what we wanted. To summarize,

- We are given $h(0), h(1), \dots, h(d)$
- Our goal is to compute $h(m), h(m + 1), \dots, h(m + d)$
- We precompute $\delta(i, d)$ for all $0 \leq i \leq d$ and $\Delta(m, k, d)$ for all $0 \leq k \leq d$
- Compute the coefficients for $p(x)$, where $[p(x)]_i = \frac{h(i)}{\delta(i, d)}$ for $0 \leq i \leq d$
- Compute the coefficients for $q(x)$, where $[q(x)]_i = \frac{1}{a + i - d}$ for
$0 \leq i \leq 2d$
- Compute $r(x) = p(x)q(x)$ using FFT
- The $(k + d)$-th coefficients of $r(x)$
give $\sum_{i = 0}^d \frac{h(i)}{\delta(i, d)} \frac{1}{m + k - i}$ for $0 \leq k \leq d$
- Compute $h(m + k) = 
\Delta(m, k, d) \left( \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \right)$
for each $0 \leq k \leq d$

To relate back to the original problem, where given $G_d(0)$, we want to compute $G_d(a)$ for $a = d, dv,$ and $dv + d$.

- We choose $h(x) = g_d(vx)$ and $m = \frac av$
- Using the above, we compute
$h(m), h(m + 1), \dots, h(m + k)$
which since $h(x) = g_d(vx)$, is $g_d(a), g_d(a + v), \dots, g_d(a + dv) = G_d(a)$


## Constant-Factor Optimizations

- The bottleneck of this algorithm is computing a $d$ by $2d$ size convolution. This can supposedly
be reduced to just a $d$ by $d$ convolution using the "[middle product](https://inria.hal.science/inria-00071921/document)" idea
- As usual, if $p$ is not known at compile time, [Montgomery Multiplication](https://cp-algorithms.com/algebra/montgomery_multiplication.html) can be used