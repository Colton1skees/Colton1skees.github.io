# Existence of binary polynomials with inverses

If you're familiar with [modular multiplicative inverses](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse), you may be interested to know that this concept extends to a certain class of binary polynomials.

Consider the 8-bit polynomial:
```
P(x) = 42*x**2 + 185*x + 132
``` 

There exists another polynomial, 

```
Q(x) = 24*x**4 + 40*x**3 + 102*x**2 + 153*x + 188
```
, that is an inverse of `P`. That is to say `Q(P(x)) == x` modulo 256. 

We can verify this by substituting`P` for all instances of `x` in `Q`

```
f(x) = 24*(42*x**2 + 185*x + 132)**4 + 40*(42*x**2 + 185*x + 132)**3 + 102*(42*x**2 + 185*x + 132)**2 + 153*(42*x**2 + 185*x + 132) + 188
```
, expanding

```
f(x) = 128*x**8 + 128*x**6 - 96*x**5 + 32*x**4 + 32*x**3 + 96*x**2 - 63*x
```

, and proving the equivalence using an SMT solver.
```
from z3 import *
x = BitVec('x', 8) 
op1 = x
op2 = 128*x*x*x*x*x*x*x*x + 128*x*x*x*x*x*x - 96*x*x*x*x*x + 32*x*x*x*x + 32*x*x*x + 96*x*x - 63*x
prove(op1 == op2)
```

# Binary permutation polynomials in the context of software obfuscation
In 2007, Zhou et al proposed using binary permutation polynomials as a means of software obfuscation[1]. The idea was that one can conceal an expression or constant by encoding it with an invertible polynomial, then recover the original expression at a later point using the polynomial inverse.

In general it is difficult for an attacker to undo this transformation. As we saw earlier, the polynomials can be obscured by partially (or fully) expanding them. Compounding on this effect, permutation polynomials are usually applied in combination with other transformations such as linear MBA[1]. As an example, consider the expression `x+y`. We can first apply the identity `x+y` == `2*(x&y) + (x^y)`, then apply the polynomial encoding from earlier:

```
128*(x&y)**2*(x^y) + 128*(x&y)**2 + 64*(x&y)*(x^y)**4 - 64*(x&y)*(x^y)**2 + 128*(x&y)*(x^y) - 126*(x&y) + 128*(x^y)**8 + 128*(x^y)**6 - 96*(x^y)**5 + 32*(x^y)**4 + 32*(x^y)**3 + 96*(x^y)**2 - 63*(x^y)
```

To break this technique, an attacker must implement [multivariate binary polynomial reduction](https://www.ndss-symposium.org/wp-content/uploads/bar2024-14-paper.pdf) *and* some algorithm for dealing with bitwise parts. Notably multivariate binary polynomial reduction was not (to the best of my knowledge) even well understood until very recently. Though the bitwise parts can be dealt with trivially. You can refer to [https://github.com/mazeworks-security/Simplifier](https://github.com/mazeworks-security/Simplifier) for implementations of both of these algorithms.

# Characterization of binary permutation polynomials
A polynomial modulo a power of two is a permutation polynomial if and only if:
1. The degree 1 monomial (i.e. `x`) has an odd coefficient
2. The sum of the coefficients of all even degree monomials *excluding* 0 (e.g. `x**2`, `x**4`, `x**6`, ... ) is even
3. The sum of the coefficients of all odd degree monomials *excluding* 1 (e.g. `x**3`, `x**5`, `x**7`, ... ) is even

All binary permutations permutations have inverses. Below are some 8-bit examples in the form of `forward, inverse`:
```
114*x*x + 213*x + 7, 24*x*x*x*x + 104*x*x*x + 14*x*x + 33*x + 251
30*x*x + 47*x + 157, 24*x*x*x*x + 56*x*x*x + 102*x*x + 235*x + 27
50*x*x + 155*x + 23, 8*x*x*x*x + 88*x*x*x + 50*x*x + 47*x + 5
42*x*x + 185*x + 132, 24*x*x*x*x + 40*x*x*x + 102*x*x + 153*x + 188
26*x*x + 97*x + 162, 24*x*x*x*x + 40*x*x*x + 86*x*x + 169*x + 246
```

# Formulating inversion as polynomial interpolation
There are several known methods for inverting binary permutation polynomials, but I'm only going to touch on what I think is the most intuitive approach (interpolation). Interested readers can refer to [1], [2], and [3] for other methods. 

Formulating inversion as interpolation should be straight forward. Consider our example from earlier `42*x**2 + 185*x + 132`. Recall that a polynomial of degree `d` is uniquely determined by `d+1` points, and the inverse of this polynomial has a degree less than or equal to 6, so we first evaluate the polynomial on 7 points (`0 ... 7`). 

```
f(0) = 132 + 185*0 + 42*0**2 = 132
f(1) = 132 + 185*1 + 42*1**2 = 103
f(2) = 132 + 185*2 + 42*2**2 = 158
f(3) = 132 + 185*3 + 42*3**2 = 41
f(4) = 132 + 185*4 + 42*4**2 = 8
f(5) = 132 + 185*5 + 42*5**2 = 59
f(6) = 132 + 185*6 + 42*6**2 = 194
```

We know that the inverse is of the form:
```
f(x) = c0 + c1*x + c2*x**2 + c3*x**3 + c4*x**4 + c5*x**5 + c6*x**6
```
, and we are looking for a polynomial that and returns 0 on `f(132)`, 1 on `f(103)`, 2 on `f(158)`, so on and so forth. Plugging this information in, we must solve for

```
f(132) = c0 + c1*132 + c2*132**2 + c3*132**3 + c4*132**4 + c5*132**5 + c6*132**6 = 0
f(103) = c0 + c1*103 + c2*103**2 + c3*103**3 + c4*103**4 + c5*103**5 + c6*103**6 = 1
f(158) = c0 + c1*158 + c2*158**2 + c3*158**3 + c4*158**4 + c5*158**5 + c6*158**6 = 2
f(41) = c0 + c1*41 + c2*41**2 + c3*41**3 + c4*41**4 + c5*41**5 + c6*41**6 = 3
f(8) = c0 + c1*8 + c2*8**2 + c3*8**3 + c4*8**4 + c5*8**5 + c6*8**6 = 4
f(59) = c0 + c1*59 + c2*59**2 + c3*59**3 + c4*59**4 + c5*59**5 + c6*59**6 = 5
f(194) = c0 + c1*194 + c2*194**2 + c3*194**3 + c4*194**4 + c5*194**5 + c6*194**6 = 6
```

then

```
f(132) = 1*c0 + 132*c1 + 16*c2 + 64*c3 + 0*c4 + 0*c5 + 0*c6 = 0
f(103) = 1*c0 + 103*c1 + 113*c2 + 119*c3 + 225*c4 + 135*c5 + 81*c6 = 1
f(158) = 1*c0 + 158*c1 + 132*c2 + 120*c3 + 16*c4 + 224*c5 + 64*c6 = 2
f(41) = 1*c0 + 41*c1 + 145*c2 + 57*c3 + 33*c4 + 73*c5 + 177*c6 = 3
f(8) = 1*c0 + 8*c1 + 64*c2 + 0*c3 + 0*c4 + 0*c5 + 0*c6 = 4
f(59) = 1*c0 + 59*c1 + 153*c2 + 67*c3 + 113*c4 + 11*c5 + 137*c6 = 5
f(194) = 1*c0 + 194*c1 + 4*c2 + 8*c3 + 16*c4 + 32*c5 + 64*c6 = 6
```

after constant folding and reducing the coefficients modulo 256. This leaves us with a system of linear equations to solve for.

# Solving interpolation
First note that polynomial interpolation over rings is weirder than fields. Lagrange interpolation will not work on all binary permutation polynomials because division is not well defined - but fortunately we just formulated interpolation as a system of linear equations using the steps above. Note that I am by no means the first person to touch on this [3][4].

I won't go into too much detail (enough has already been written about this elsewhere). To solve the system of equations we can apply gaussian elimination - but the rules are *slightly* different modulo m.
- Swapping two rows is still allowed
- Multiplying a row by a nonzero number is *not* allowed
- Adding a multiple of one row to another is still allowed
- Any odd leading coefficient can be used to eliminate all other coefficients in the same column, because the odd coefficient has a modular inverse. Similarly if only even leading coefficients remain, you should be able to eliminate all but one coefficient by solving for `ax = b` where b is a negation of the other even entry.


Applying gaussian elimination for this example yields:
```
81*c6 + 135*c5 + 225*c4 + 119*c3 + 113*c2 + 103*c1 + 1*c0 == 1
0*c6 + 34*c5 + 224*c4 + 34*c3 + 192*c2 + 34*c1 + 160*c0 == 162
0*c6 + 0*c5 + 152*c4 + 200*c3 + 112*c2 + 80*c1 + 136*c0 == 208
0*c6 + 0*c5 + 0*c4 + 40*c3 + 36*c2 + 94*c1 + 209*c0 == 66
0*c6 + 0*c5 + 0*c4 + 0*c3 + 184*c2 + 104*c1 + 62*c0 == 0
0*c6 + 0*c5 + 0*c4 + 0*c3 + 0*c2 + 196*c1 + 29*c0 == 112
0*c6 + 0*c5 + 0*c4 + 0*c3 + 0*c2 + 0*c1 + 71*c0 == 36
```

Note that the order of the operands was flipped, just because I prefer to solve for the constant offset first. Reading off a solution is a bit more involved, but not terrible. In short, you can't just pick any solution when solving the linear congruence - because there may be multiple solutions. Consider the made up example 
```
0*c6 + 0*c5 + 0*c4 + 0*c3 + 0*c2 + 0*c1 + 32*c0 == 64
```


, where there are multiple solutions(146, 66) for the bottom row. One could contrive a set of rows where this is no solution if you pick the wrong one.

Anyways, the solutions we read off for our example were
```
c0 = 188
c1 = 153
c2 = 70
c3 = 56
c4 = 136
c5 = 112
c6 = 48
```

, yielding `188 + 153*x0 + 70*x0**2 + 56*x0**3 + 136*x0**4 + 112*x0**5 + 48*x0**6`. This result can be verified with an SMT solver, or alternatively one could expand `Q(P(X))` and apply a multivariate polynomial reduction algorithm to verify its equivalence. Note that this solution is not unique, because there are arbitrarily many polynomials that evaluate to zero mod m. E.g., `128*x**2 - 128*x` is zero mod 256.

Source code for the implementation is available [here](https://github.com/mazeworks-security/Simplifier/blob/13e9de8bdc1f0e773c904c543f99f6da68ce0d6e/Mba.Simplifier/LinEq/PolyInverter.cs#L17).


# References
1. [Information Hiding in Software with Mixed Boolean-Arithmetic Transforms
](https://link.springer.com/chapter/10.1007/978-3-540-77535-5_5)
2. [https://www.ndss-symposium.org/wp-content/uploads/bar2024-14-paper.pdf](https://www.ndss-symposium.org/wp-content/uploads/bar2024-14-paper.pdf)
3. [Mixed Boolean-Arithmetic (Part 3): Binary Permutation Polynomials
](https://plzin.github.io/posts/perm-poly)
4. [Binary Permutation Polynomial Inversion and Application to Obfuscation Techniques
](https://dl.acm.org/doi/10.1145/2995306.2995310)