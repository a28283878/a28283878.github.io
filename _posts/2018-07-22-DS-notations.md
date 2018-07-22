---
layout: post
title: "Big O, Omega"
author: "Richard Lin"
categories: data-structure
tags: data-structure
image:
  feature: data-structure.jpg
---

## Big O O(n)
* * *
Big O means the upper bound for f(n).<br>
f of n is Big-O of g of n, if and only if positive c and n exit, such that
`
f(n) \leq cg(n)
`
<br><br>
**Example.1 Show that
`
400n^{3} + 20n^{2} = O(n^{3}).
`**
<br>
* * *
**Solution** By definition, we have<br>
0 <= h(n) <= cg(n)<br>
Substituing 400n^3 + 20n^2 as h(n) and n^3 as g(n) we have, <br>
`
0 \leq 400n^{3} + 20n^{2} \leq cn^{3}
`
<br>
all divided by n^3,<br>
`
0 \leq 400 + \frac{20}{n} \leq c
`<br>

Note that 20/n is 0 as n is , and 20/n is maximum when n = 1. Therefore,<br>
`
0 \leq 400 + 20/n \leq c
`
<br>
This means, c = 420<br>
<br>
To determine n0,<br>
`
0 \leq 400+20/n_{0} \leq 420 \n
-400 \leq 20/n_{0} \leq 20 \n
-400n_{0} \leq 20 \leq 20n_{0} \n
-20n_{0} \leq 1 \leq n_{0}
`<br>
that implies n0 = 1.

<br><br>
**Example.2 Show that  
`
  10n^{3} + 20n \ne O(n^{2}).
`**
<br>
* * *
**Solution** by definition, we have<br>
`
0 \leq h(n) \leq cg(n)
`
<br>Substituting
`
10n^{3} + 20n
`
as h(n),
`
n^{2}
`
as g(n), we get<br>
`
0 \leq 10n^{3} + 20n \leq cn^{2}
`
<br>Dividing by n^2<br>
`
0 \leq 10n + 20/n \leq c
`
<br>hence ,
`
10n^{3} + 20n \ne O(n^{2})
`


## OMEGA notation
* * *
Omega notation provide the lower bound of f(n), it means that the functions cannot perform better than certain value but can be worse.<br>
`
\Omega(g(n)) = {h(n):\exists \text{ positive constants } c>0,n_{0}\text{ such that } 0 \leq cg(n) \leq h(n), \forall n \geq n_{0}}
`
<br>
Examples of functions in
`
\Omega(n^{2}) \text{ include : } n^{2}, n^{2.9}, n^{3} + n^{2}, n^{3}
`
<br>
Examples of functions not in
`
\Omega(n^{3}) \text{ include : } n^{2}, n^{2.9}, n^{2}
`
<br><br>
**Example.1 Show that  
`
  5n^{2} + 10n = \Omega(n^{2}).
`**
<br>
* * *
**Solution** By definition, we get<br>
\begin{gather}
0 \leq cg(n) \leq h(n)\newline
\text{Substituting } 5n^{2} + 10n \text{ as } h(n)\ and\ n^{2}\ as\ g(n)\newline
0 \leq cn^{2} \leq 5n^{2} + 10n
\end{gather}
<br>Dividing by n^2
`
0 \leq c \leq 5 + 10/n
`
<br>Now,
`
\lim_{n\rightarrow\infty} 5+10/n = 5
`
<br>so 0 <= c <= 5
<br>Hence, c = 5
<br>now determine the n0<br>
`
0 \leq 5 \leq 5 + 10/n_{0},
-5 \leq 0 \leq 10/n_{0}
`
<br>So n0 = 1, as lim(n -> infinity)1/n = 0<br>
Hence,
`
5n^{2} + 10n = \Omega(n^{2}) \text{ for } c = 5 \text{  and  } \forall n \geq n_{0} = 1 
`


