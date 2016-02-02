---
layout: post
title: "How many coin-flips till heads?"
date:   2015-12-03 17:30:00
---

Given a possibly-unfair coin, what's the expected number of coin-flips until you get heads?

The answer isn't surprising, but proving it to myself was harder than I expected.

Let's get rigorous with random variables.

$$
\begin{align}
& X : \text{number of coin flips} \\
& p : \text{probability of heads} \\
\end{align}
$$

So, choose some example probababilities, then write the general formula.

$$
\begin{align}
P(X=1) & = p \\
P(X=2) & = p(1-p) \\
P(X=3) & = p(1-p)^2 \\
\\
P(X=x) & = p(1-p)^{x-1} \\
\end{align}
$$

Now the expected value...

$$
\begin{align}
E[X] & = 1*P(X=1) + 2*P(X=2) + \ldots \\
& = p + 2*p(1-p) + 3*p(1-p)^2 + \dotsm \\
& = \sum_{i=1}^\infty{ip(1-p)^{i-1}} \\
& = p \sum_{i=1}^\infty{i(1-p)^{i-1}}
\end{align}
$$

Now let's take a break and fool around with the geometric series formula. Here it is:

$$
\frac{1}{1-k} = \sum_{n=0}^\infty k^n = 1 + k + k^2 + k^3 + \cdots
$$

Take the derivative.

$$
\begin{align}
\frac{d}{dk}\left(\frac{1}{1-k}\right) & = \frac{d}{dk}\left( 1 + k + k^2 + \dotsm \right) \\
\frac{1}{(1-k)^2} & = 1 + 2k + 3k^2 + \dotsm \\
& = \sum_{i=1}^\infty ik^{i-1} \\
\end{align}
$$

Cool, that formula just saved the day.

$$
\sum_{i=1}^\infty ik^{i-1} = \frac{1}{(1-k)^2}
$$

Substituting $$ k = 1 - p $$ we get:

$$
\begin{align}
E[X] & = p \left( \frac{1}{(1 - (1-p))^2} \right) \\
& = \frac{p}{p^2} \\
& = \frac{1}{p}
\end{align}
$$

So with a fair coin, the answer is 2. If the coin turns up with $$ p = \frac{3}{4} $$, the expected number of coin-flips is $$ \frac{4}{3} $$.

This is a special case of the [negative binomial distribution](https://en.wikipedia.org/wiki/Negative_binomial_distribution), by the way.

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
