---
title: 'Determinant'
date: 2022-09-21
permalink: /posts/2022/09/determinant/
tags:
  - cool posts
  - category1
  - category2
---

This is a sample blog post. Lorem ipsum I can't remember the rest of lorem ipsum and don't have an internet connection right now. Testing testing testing this blog post. Blog posts are cool.

10 Key properties of Determinant
======
- **Rule 1: The determinant of the n by n identity matrix is 1.**
$$\left|\begin{array}{ll}
1 & 0 \\
0 & 1
\end{array}\right|=1 \quad \text { and } \quad\left|\begin{array}{lll}
1 & & \\
& \ddots & \\
& & 1
\end{array}\right|=1 \text {. }$$

- **Rule 2: The determinant changes sign when two rows are exchanged (sign reversal):**
$$
\text { Check: }\left|\begin{array}{ll}
c & d \\
a & b
\end{array}\right|=-\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right| \quad \text { (both sides equal } b c-a d \text { ). }
$$
Because of this rule, we can find det P for any permutation matrix. Just exchange rows of I until you reach P. Then det P = +1 for an even number of row exchanges and
det P = -l for an odd number.
The third rule has to make the big jump to the determinants of all matrices. 

- **Rule 3: The determinant is a linear function of each row separately (all other rows stay fixed).**
$$
\begin{aligned}
& \left|\begin{array}{cc}
t a & t b \\
c & d
\end{array}\right|=t\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right| \\
& \left|\begin{array}{cc}
a+a^{\prime} & b+b^{\prime} \\
c & d
\end{array}\right|=\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right|+\left|\begin{array}{cc}
a^{\prime} & b^{\prime} \\
c & d
\end{array}\right| .
\end{aligned}
$$
$$
\left|\begin{array}{ll}
2 & 0 \\
0 & 2
\end{array}\right|=2^2=4 \quad \text { and } \quad\left|\begin{array}{ll}
t & 0 \\
0 & t
\end{array}\right|=t^2 \text {. }
$$

- **Rule 4: If two rows of A are equal, then $let A = 0$.**

$$
\text { Equal rows } \quad \text { Check } 2 \text { by } 2:\left|\begin{array}{ll}
a & b \\
a & b
\end{array}\right|=0\text {. }
$$

Rule 4 follows from rule 2.

- **Rule 5: Subtracting a multiple of one row from another row leaves det A unchanged.**
$$\begin{aligned}
& \ell \text { times row } 1 \\
& \text { from row } 2
\end{aligned} \quad\left|\begin{array}{cc}
a & b \\
c-\ell a & d-\ell b
\end{array}\right|=\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right| .$$   
from Rule 3 (linearity).

**Conclusion**: The determinant is not changed by the usual elimination steps from A to U. 
Thus det A equals det U. If we can find determinants of triangular matrices U, we can 
find determinants of all matrices A. Every row exchange reverses the sign, so always 
det A= ± det U. Rule 5 has narrowed the problem to triangular matrices. 

- **Rule 6: A matrix with a row of zeros has det A = 0.**
$$
\text { Row of zeros } \quad\left|\begin{array}{ll}
0 & 0 \\
c & d
\end{array}\right|=0 \quad \text { and } \quad\left|\begin{array}{ll}
a & b \\
0 & 0
\end{array}\right|=0 \text {. }
$$

- **Rule 7: If A is triangular then $\operatorname{det} A=a_{11} a_{22} \cdots a_{n n}=$ product of diagonal entries.**
$$
\text { Triangular } \quad\left|\begin{array}{ll}
a & b \\
0 & d
\end{array}\right|=a d \quad \text { and also }\left|\begin{array}{ll}
a & 0 \\
c & d
\end{array}\right|=a d \text {. }
$$
further :
$$
\text { Diagonal matrix } \operatorname{det}\left[\begin{array}{cccc}
a_{11} & & & 0 \\
& a_{22} & & \\
& & \ddots & \\
0 & & & a_{n n}
\end{array}\right]=\left(a_{11}\right)\left(a_{22}\right) \cdots\left(a_{n n}\right) \text {. }
$$

- **Rule 8: If $A$ is singular then $\operatorname{det} A=0$. If $A$ is invertible then $\operatorname{det} A \neq 0$.**
$$
\text { Singular } \quad\left[\begin{array}{ll}
a & b \\
c & d
\end{array}\right] \text { is singular if and only if } a d-b c=0 \text {. }
$$
$$
\text { Multiply pivots } \quad \operatorname{det} A= \pm \operatorname{det} U= \pm \text { (product of the pivots). }
$$
$$
\text { The determinant is }\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right|=\left|\begin{array}{cc}
a & b \\
0 & d-(c / a) b
\end{array}\right|=a d-b c \text {. }
$$
$$
\text { If } P A=L U \text { then } \operatorname{det} P \operatorname{det} A=\operatorname{det} L \operatorname{det} U \text { and } \operatorname{det} A= \pm \operatorname{det} U \text {. }
$$

- **Rule 9: The determinant of $A B$ is $\operatorname{det} A$ times $\operatorname{det} B:|A B|=|A||B|$.**
$$
\text { Product rule } \quad\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right|\left|\begin{array}{ll}
p & q \\
r & s
\end{array}\right|=\left|\begin{array}{ll}
a p+b r & a q+b s \\
c p+d r & c q+d s
\end{array}\right| \text {. }
$$
$$
A \text { times } A^{-1} \quad A A^{-1}=I \text { so } \quad(\operatorname{det} A)\left(\operatorname{det} A^{-1}\right)=\operatorname{det} I=1 .
$$

- **Rule 10: The transpose $A^{\mathrm{T}}$ has the same determinant as $A$.**
$$
\text { Transpose }\left|\begin{array}{ll}
a & b \\
c & d
\end{array}\right|=\left|\begin{array}{ll}
a & c \\
b & d
\end{array}\right| \text { since both sides equal } a d-b c \text {. }
$$







You can have many headings
======

Important comment on columns
------
