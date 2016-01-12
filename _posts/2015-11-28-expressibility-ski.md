---
title: Expressibility of SKI combinator calculus
layout: post
---

Its really amazing that 3 basic functions could actually simulate all of the computing that we could possibly do with lambda calculus. But most of the times we dont understand how its possible. 

When we say equivalence of SKI combinator calculus and lambda calculus, we mean two things. First that any SKI expression can be turned into a lambda expression. Second that any lambda expression can be turned into a SKI expression.

SKI to Lambda
-------------

The first part of equivalence is easy to establish. We just replace $S$, $K$ and $I$ with appropriate lambda expressions

- $S=\lambda x. \lambda y. \lambda z. (xz)(yz)$
- $K=\lambda x. \lambda y. x$
- $I=\lambda x. x$

For example: The equivalent expression for $$(SKK)K$$ is simple $$((\lambda x. \lambda y. \lambda z. (xz)(yz))(\lambda x. \lambda y. x)(\lambda x. \lambda y. x))(\lambda x. \lambda y. x)$$


Lambda to SKI
-------------

This side of the equivalence is bit tricky. The primary problem with SKI is that it doesnt allow unbound occurences of variables, the expressions in SKI have to be expressions like in programming languages. Infact the final expression we build would have no variables at all. This restriction not necessarily reduce the expressibility in any way.

We will do a step by step procedure with induction on the structure of lambda expression. Lets call that procedure `convert`

- $M \equiv PQ$ where $P$ and $Q$ are lambda expression $convert(M) = convert(P) convert(Q)$
- $M \equiv \lambda x.x$ straightforward $convert(M) = I$
- $M \equiv \lambda x.P$ where $x$ is not free in $M$ this is also simple, we just need to drop the argument $x$. The combinator which drops an argument is $K$. Hence $convert(M) = K convert(P)$
- $M \equiv \lambda x.Ux$ where $x$ is not free in $U$. This is the case of $\eta$ reduction ie., the abstraction is fruitless. Thus $convert(M) = U$.
- $M \equiv \lambda x. UV$ where none of the above cases hold. Then $convert(M) = Sconvert(\lambda x. U) convert(\lambda x. V)$
- $M \equiv x$ this case need to be $convert(M) = x$ since partial conversion need to be supported for the next case.
- $M \equiv \lambda x.P$ where none of the above cases hold and $x$ is free in $P$ $convert(M) = convert(\lambda x. convert(P))$


To verify that such a conversion is correct, we just have to do the reverse of it ie., replace $S$, $K$ and $I$ with their lambda forms in each case and find that we get back the same expression or an expression with same normal form.

Examples
--------

- $\lambda x.\lambda y.\lambda z. x$ 
  $\equiv \lambda x. \lambda y. Kx$
  $\equiv \lambda x. K(Kx)$
  $\equiv S(\lambda x. K) (\lambda x. Kx)$
  $\equiv S(KK)K$

- $\lambda x.\lambda y.\lambda z. zx$
  $\equiv \lambda x. \lambda y. S(\lambda z. z)(\lambda z.x)$
  $\equiv \lambda x. \lambda y. SI(Kx)$
  $\equiv \lambda x. K(SI(Kx))$
  $\equiv S(\lambda x.K)(\lambda x. SI(Kx))$
  $\equiv S(KK)(S(\lambda x. SI)(\lambda x.Kx))$
  $\equiv S(KK)(S(K(SI))K)$

Implementation
--------------

To keep things simple, especially to handle the last case, we consider $S$, $K$ and $I$ to be part of lambda calculus itself. The complete code for such conversion is [here](https://gist.github.com/yellowflash/01d7b6866de2a0aa0713)