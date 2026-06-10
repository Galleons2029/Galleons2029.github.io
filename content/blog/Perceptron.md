---
title: An introduction of Perceptron
date: 2022-09-14
tags:
  - machine-learning
  - math
publish: true
---

The perceptron algorithm:

$$
\begin{aligned}
&\text{Perceptron}\left(\left\{\left(x^{(i)}, y^{(i)}\right), i=1, \ldots, n\right\}, T\right): \\
&\quad \text{initialize } \theta=0 \text{ (vector);} \\
&\quad \text{for } t=1, \ldots, T \text{ do} \\
&\quad \quad \text{for } i=1, \ldots, n \text{ do} \\
&\quad \quad \quad \text{if } y^{(i)}\left(\theta \cdot x^{(i)}\right) \leq 0 \text{ then} \\
&\quad \quad \quad \quad \text{update } \theta=\theta+y^{(i)} x^{(i)}
\end{aligned}
$$
