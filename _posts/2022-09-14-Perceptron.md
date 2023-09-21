---
title: 'An introduction of Perceptron '
date: 2022-09-14
permalink: /posts/2012/08/2022-09-14-blog-post-5/
tags:
  - cool posts
  - category1
  - category2
---

This is a sample blog post. Lorem ipsum I can't remember the rest of lorem ipsum and don't have an internet connection right now. Testing testing testing this blog post. Blog posts are cool.

````latex
\begin{algorithm}
\caption{Perceptron Algorithm}
\begin{algorithmic}[1]
\Procedure{Perceptron}{$\left{\left(x^{(i)}, y^{(i)}\right), i=1, \ldots, n\right}, T$}
\State Initialize $\theta=0$ \Comment{Vector}
\For{$t = 1, \ldots, T$}
\For{$i = 1, \ldots, n$}
\If{$y^{(i)}\left(\theta \cdot x^{(i)}\right) \leq 0$}
\State Update $\theta=\theta+y^{(i)} x^{(i)}$
\EndIf
\EndFor
\EndFor
\EndProcedure
\end{algorithmic}
\end{algorithm}
````

Headings are cool
======

You can have many headings
======

Aren't headings cool?
------