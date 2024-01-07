---
layout: post
title:  "Calculating n! mod p in O(sqrt n log n) Time"
date:   2023-11-12 00:22:04 -0700
toc: true
---
<!-- date is set in the future to disable viewing -->

In this article we compute $n! \bmod p$ in
$O(\sqrt n \log n)$ time.

## Prerequisites

You should be familiar with
- Modulo and modular inverse
- Convolution and FFT

All formulas and calculations are done$\bmod p$, and $\dfrac ab$ should be interpreted as $ab^{-1}$,
where $b^{-1}$ the modular inverse of $b$,$\bmod p$.

## Formula
For simplicity, suppose that $n$ is a square number and $v^2 = n$. We let $g(x) = \displaystyle\prod_{i=1}^v (x + i)$.  Then,
- $g(0) = \displaystyle\prod_{i = 1}^v i$, the product of the first $v$ terms of $n!$
- $g(v) = \displaystyle\prod_{i = v+1}^{2v} i$, the product of the next $v$ terms of $n!$
- ...
- $g(v^2-v) = \displaystyle\prod_{i={v^2-v+1}}^{v^2} i$, the product of the last $v$ terms of $n!$

Therefore, $n! = g(0)g(v)g(2v) \cdots g(v^2-v)$.

If $n$ is not a perfect square, we can choose $v = \lfloor \sqrt n \rfloor$, and compute the rest of the terms the regular way, yielding the formula
$n! = g(0)g(v)g(2v) \cdots g(v^2-v) \times (v^2+1)(v^2+2)\cdots(n)$.

For the rest of the article, we focus on how to compute $[g(0), g(v), g(2v), \dots, g(v^2-v)]$.

## The Method

Our goal is to compute $g(0)g(v)g(2v) \cdots g(v^2-v)$. For simplicity, suppose $v$ is a power of 2.

Let $g_d(x)$ be the first $d$ terms of $g(x)$, that is, $\displaystyle\prod_{i=1}^d (x+i)$.

The main idea is to find an algorithm that takes in array $[g_d(0), g_d(v), g_d(2v), \dots, g_d(dv)]$
and outputs $[g_{2d}(0), g_{2d}(v), g_{2d}(2v), \dots, g_{2d}(2dv)]$ in $O(d \log d)$ time.
We call this algorithm the doubling algorithm.

Then, starting with $[g_1(0),g_1(v)]$, we can apply our algorithm $\log v$ times,
yielding $[g_2(0), g_2(v), \dots, g_2(2v)]$, then, $[g_4(0), g_4(v), \dots, g_4(4v)]$, and so on
until we have $[g_v(0), g_v(v), \dots, g_v(v^2)]$, and since $g_v(x) = g(x)$, this gives us
$[g_v(0), g_v(v), \dots, g_v(v^2-v)]$ by ignoring the last element of the array.

By doing this, our time complexity is $T(n) = T(n/2) + O(n \log n) = O(v \log v) = O(\sqrt p \log p)$.

In general, if $v$ is not a power of 2, we can round up to the nearest one.

## The Doubling Algorithm

Our goal is, given $g_d(0), g_d(v), \dots, g_d(dv)$, compute
$g_{2d}(0), g_{2d}(v), \dots, g_{2d}(2dv)$ in $O(d \log d)$ time.

The plan is to
1. Compute $[g_d(d), g_d(d + v), g_d(d + 2v), \dots, g_d(d + dv)]$
2. Compute $[g_d(dv), g_d(dv + v), g_d(dv + 2v), \dots, g_d(2dv)]$
3. Compute $[g_d(dv + d), g_d(dv + d + v), g_d(dv + d + 2v), \dots, g_d(2dv + d)]$
4. Using the arrays from Steps 1 through 3, for each $x$ in $[0, v, 2v, \dots, 2dv]$, compute $g_{2d}(x) = g_d(x) g_d(i + dv)$

Steps 1 through 3 all follow the same form, given $[g_d(0), g_d(v), \dots, g_d(dv)]$,
compute $[g_d(a), g_d(a + v), \dots, g_d(a + dv)]$. We discuss how to do this in the next section.

### Polynomial Sample Shift

First, we solve a simplified version of the problem.

Our goal is, if $h(x)$ is a degree-$d$ polynomial, given $h(0), h(1), \dots, h(d)$,
compute $h(m), h(m + 1), \dots, h(m + d)$.

To do this, we tak advantage of the Lagrange Interpolation formula on sampling points $[0, 1, \dots, d]$:

$$h(x) = \sum_{i=0}^d h(i) \prod_{j = 0, j \neq i}^d \frac{x-j}{i-j}$$

We aren't actually going to interpolate anything, we are just taking advantage of this formula.

Plugging in $m + k$ into $x$, we get

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d h(i) \prod_{j = 0, j \neq i}^d \frac{m+k-j}{i-j} \\
&= \sum_{i=0}^d \frac{h(i)}{\prod_{j=0,j\neq i}^d (i-j)} \prod_{j=0,j\neq i}^d (m + k - j) \\
\end{align*}
$$

For simplicity, let $\delta(i, d) = \prod_{j=0,j\neq i}^d (i-j)$.
<!--We can precompute values using the formula $\frac1{\delta(i,d)} = \frac{i-d-1}i \frac1{\delta(i-1,d)}$.-->

<!--
$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \prod_{j=0,j\neq i}^d (m + k - j) \\
\end{align*}
$$
-->

We note that $\displaystyle\prod_{j=0,j\neq i}^d (m + k - j)$
is the product of all terms $\displaystyle\prod_{j=0}^d (m + k - j)$ except for the i-th term, which we can write as $\dfrac1{m + k - i} \displaystyle\prod_{j=0}^d (m + k - j)$.
For simplicity, let $\Delta(m, k, d) = \displaystyle\prod_{j=0}^d (m + k - j)$.
We therefore have

$$
\begin{align*}
h(m + k) &= \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \prod_{j=0}^d (m + k - j) \\
&= \Delta(m, k, d) \left( \sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \right) \\
\end{align*}
$$


<!--
For $k > 0$,
$\Delta(m, k, d) = \frac{m+k}{m+k-d-1}\Delta(m, k - 1, d)$, giving us a $O(d)$ way
to precompute it.
-->

The sum in the formula is a sum of a product of $\dfrac{h(i)}{\delta(i, d)}$, a function of $i$, and
$\dfrac1{m + k - i}$, a function of $-i$. This lets us express it as a convolution and compute it in
$O(d \log d)$ time using FFT.

<!-- Use [p(x)]_i notation here too -->
In particular, we define the polynomials $p(x) = \displaystyle\sum_{i=0}^d \dfrac{h(i)}{\delta(i, d)} x^i$
and $q(x) = \displaystyle\sum_{i=0}^{2d} \dfrac{1}{a + i - d}x^i$

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

which is exactly the sum in the formula.

#### Summary of Polynomial Sample Shift

To summarize, we are given $h(0), h(1), \dots, h(d)$ and we want to
compute $h(m + k)$ for all $0 \leq k \leq d$, we

- Precompute $\delta(i, d)$ for all $0 \leq i \leq d$ by computing
	- $\delta(0, d) = \displaystyle\prod_{j=0,j\neq i}^d (0-j)$
	- For $0 < i \leq d$, $\delta(i,d) = \dfrac i{i-d-1}{\delta(i-1,d)}$
- Precompute $\Delta(m, k, d)$ for all $0 \leq k \leq d$
	- $\Delta(m, 0, d) = \displaystyle\prod_{j=0}^d (m + 0 - j)$
	- For $k > 0$, $\Delta(m, k, d) = \dfrac{m+k}{m+k-d-1}\Delta(m, k - 1, d)$
- Compute the coefficients for $p(x)$, where $[p(x)]_i = \dfrac{h(i)}{\delta(i, d)}$ for $0 \leq i \leq d$
- Compute the coefficients for $q(x)$, where $[q(x)]_i = \dfrac{1}{a + i - d}$ for
$0 \leq i \leq 2d$
- Compute $r(x) = p(x)q(x)$ using FFT
- The $(k + d)$-th coefficient of $r(x)$
is $\displaystyle\sum_{i = 0}^d \frac{h(i)}{\delta(i, d)} \frac{1}{m + k - i}$ for $0 \leq k \leq d$
- Compute $h(m + k) = 
\Delta(m, k, d) \left( \displaystyle\sum_{i=0}^d \frac{h(i)}{\delta(i, d)} \frac1{m + k - i} \right)$
for $0 \leq k \leq d$

To relate back to the original problem, where given $G_d(0)$, we want to compute $G_d(a)$ for $a = d, dv,$ and $dv + d$.

- We choose $h(x) = g_d(vx)$ and $m = \frac av$
- Using the above, we compute
$h(m), h(m + 1), \dots, h(m + k)$
which since $h(x) = g_d(vx)$, is $g_d(a), g_d(a + v), \dots, g_d(a + dv) = G_d(a)$


## The Overall Algorithm

Given $n$ and $p$, we

- Choose $v = \lfloor \sqrt n \rfloor$, rounding up to the nearest power of 2
- Calculate $g(0) = 1, g(v) = v + 1$, set $d = 1$
- Until $d = v$, using the doubling algorithm to compute $[g_{2d}(0), \dots, g_{2d}(2dv)]$
from $[g_d(0), \dots, g_d(dv)]$, doubling $d$ each time
	- Within the doubling algorithm, given $g_d(0), g_d(v), \dots, g_d(dv)$, compute
		1. $[g_d(d), g_d(d + v), g_d(d + 2v), \dots, g_d(d + dv)]$
		2. $[g_d(dv), g_d(dv + v), g_d(dv + 2v), \dots, g_d(2dv)]$
		3. $[g_d(dv + d), g_d(dv + d + v), g_d(dv + d + 2v), \dots, g_d(2dv + d)]$
		4. For each $x$ in $[0, v, 2v, \dots, 2dv]$, compute $g_{2d}(x) = g_d(x) g_d(i + dv)$

## Implementation

This is a C++ implementation that solves `Factorial` from Library Checker, linked below. It uses the modulo 998244353
version of FFT. There is no optimization in this code, it's mostly used to illustrate the concepts seen in this article.

```cpp
#include <stdio.h>
#include <algorithm>
#include <vector>
using namespace std;

typedef long long int lli;
typedef vector<lli> vint;
const int MAXN = 1<<23;
int revd[MAXN];
const lli MOD = 998244353;

// Modulo operations
lli pow(lli b, lli e){
	lli r = 1;
	while(e){
		if(e & 1) r = r * b % MOD;
		b = b * b % MOD;
		e /= 2;
	}
	return r;
}

lli inv(lli b){
	return pow(b, MOD - 2);
}

// Convolution/FFT Functions
unsigned rev(unsigned n){
	n = (n & 0x55555555) << 1 | (n >> 1 & 0x55555555);
	n = (n & 0x33333333) << 2 | (n >> 2 & 0x33333333);
	n = (n & 0x0f0f0f0f) << 4 | (n >> 4 & 0x0f0f0f0f);
	n = (n & 0x00ff00ff) << 8 | (n >> 8 & 0x00ff00ff);
	n = n << 16 | n >> 16;
	return n;
}

void init_fft(){
	for(int size = 2, power = 1; size <= MAXN/2; size *= 2, power++)
		for(int i = 0; i < size; i++)
			revd[i + size] = rev(i) >> (32 - power);
}

void fft(vint &arr){
	int N = arr.size();
	lli iroot = pow(3, (MOD - 1) / N);
	vint root(N/2);
	root[0] = 1;
	for(int i = 1; i < N/2; i++)
		root[i] = root[i - 1] * iroot % MOD;
	for(int i = 0; i < N; i++){
		int j = revd[i + N];
		if(i < j) swap(arr[i], arr[j]);
	}

	for(int size = 2; size <= N; size *= 2){
		int half = size / 2, step = N / size;
		for(int start = 0; start < N; start += size){
			for(int off = 0; off < half; off++){
				lli u = arr[start + off];
				lli v = arr[start + off + half] * root[off * step] % MOD;
				arr[start + off] = (u + v) % MOD;
				arr[start + off + half] = (u - v + MOD) % MOD;
			}
		}
	}
}

void ifft(vint &arr){
	reverse(arr.begin() + 1, arr.end());
	fft(arr);
	int N = arr.size(), divn = pow(N, MOD - 2);
	for(int i = 0; i < N; i++)
		arr[i] = arr[i] * divn % MOD;
}

vint mult(vint arr, vint brr){
	int N = 2, M = arr.size() + brr.size() - 1;
	while(N < M) N *= 2;
	arr.resize(N);
	brr.resize(N);
	fft(arr);
	fft(brr);
	for(int i = 0; i < N; i++)
		arr[i] = arr[i] * brr[i] % MOD;
	ifft(arr);
	return arr;
}

// Factorial functions
vint shift(vint h, int m){
	int d = h.size() - 1;
	vint delta(d + 1), Delta(d + 1);
	
	delta[0] = 1;
	for(int j = 1; j <= d; j++)
		delta[0] = delta[0] * (MOD + 0 - j) % MOD;
	delta[0] = inv(delta[0]);
	for(int i = 1; i <= d; i++)
		delta[i] = (MOD + i - d - 1) * inv(i) % MOD * delta[i - 1] % MOD;

	Delta[0] = 1;
	for(int j = 0; j <= d; j++)
		Delta[0] = Delta[0] * (MOD + m + 0 - j) % MOD;
	for(int k = 1; k <= d; k++)
		Delta[k] = Delta[k - 1] * (m + k) % MOD * inv(MOD + m + k - d - 1) % MOD;

	vint px(d + 1), qx(2*d + 1);
	for(int i = 0; i <= d; i++)
		px[i] = h[i] * delta[i] % MOD;
	for(int i = 0; i <= 2*d; i++)
		qx[i] = inv(MOD + m + i - d);
	
	vint rx = mult(px, qx);

	vint res(d + 1);
	for(int k = 0; k <= d; k++)
		res[k] = Delta[k] * rx[k + d] % MOD;
	return res;
}

vint grow(vint samples, lli v){
	lli d = samples.size() - 1;
	vint &G0 = samples;
	vint Gd = shift(samples, d * inv(v) % MOD);
	vint Gdv1 = shift(samples, (d*v + v) * inv(v) % MOD);
	vint Gdvd1 = shift(samples, (d*v + d + v) * inv(v) % MOD);

	vint res(2*d+1);
	for(int i = 0; i <= d; i++)
		res[i] = G0[i] * Gd[i] % MOD;
	for(int i = 1; i <= d; i++)
		res[i + d] = Gdv1[i - 1] * Gdvd1[i - 1] % MOD;
	
	return res;
}

lli factorial(lli n){
	lli v = 1;
	while(v * (v + 1) < n)
		v *= 2;

	int d = 1;
	vint Gd = {1, v + 1};
	while(d < v){
		d *= 2;
		Gd = grow(Gd, v);
	}
	
	lli total = 0, prod = 1;
	for(size_t i = 0; i < Gd.size(); i++){
		if(total + v > n) break;
		prod = prod * Gd[i] % MOD;
		total += v;
	}
	for(lli i = total + 1; i <= n; i++)
		prod = prod * i % MOD;

	return prod;
}

int main(){
	init_fft();
	int T;
	scanf("%d", &T);
	while(T--){
		lli N;
		scanf("%lld", &N);
		lli ans = factorial(N);
		printf("%lld\n", ans);
	}
}
```

## Practice Problems
- [[Library Checker] Factorial](https://judge.yosupo.jp/problem/factorial) can be solved directly using the knowledge
from this article
- [[DMOJ] Fast Factorial Calculator 3](https://dmoj.ca/problem/factorial3) requires some constant-optimizations
- [[SPOJ] Factorial Modulo Prime](https://www.spoj.com/problems/FACTMODP/) is similar

## References

- This article is based off of [this article](https://web.archive.org/web/20201026035551/https://min-25.hatenablog.com/entry/2017/04/10/215046) by Min_25
- Some of the math and derivations are from *Linear recurrences with polynomial coefficients and application to integer factorization and Cartier-Manin operator* by Alin Bostan, Pierrick Gaudry, and Eric Schost. [(Link)](https://mathexp.eu/bostan/publications/BoGaSc07.pdf). As far as I can tell, this is the first (English) source to describe this technique.