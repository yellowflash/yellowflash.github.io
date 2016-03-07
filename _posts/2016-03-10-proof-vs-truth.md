---
title: Proof vs Truth
layout: post
---

Traditionally when we talk about logic we talk about whether a particular statement is `true` or `false`. Say for instance the statement, `2+3 = 5` is `true`. We can argue that `2+3` is *denotionally* equivalent to `5`. What we generally miss is that there is a underlying computation on the *sense* that makes `2+3` to equal to `5`. This notion of computation can be extended to the proofs too, where the entire question of whether a statement is `true` or `false` become irrelevant. Instead we would focus on whether a statement is provable.


BHK Interpretation
------------------

We will take a look at propositional logic again, with connectives $\wedge$ (AND), $\vee$ (OR), $\neg$ (NOT) and $\rightarrow$ (IMPLICATION). The traditional way to define them is by giving semantics through truth tables. But what we are going to define now instead is sort of an interpretation which relies more on the dynamics of the proofs.

Say for instance, consider the propositional statement $A \rightarrow B$. The usual semantics of propositional logic says when a statement is true, as in, this statement is always true except when $A$ is `true` and $B$ is `false`. The other way to look at it is, if some how we find a proof of $A$ then we can *construct* a proof of $B$. More systematically, $A \rightarrow B$ is a function which takes a proof of $A$ and builds a proof of $B$.

We can similarly define, 
$A \wedge B$ as a tuple of proofs of $A$ and $B$ as $(A,B)$. We will also define projection functions to take first and second components out. Why do we need it? We need it because, $A \wedge B \rightarrow A$ and $A \wedge B \rightarrow B$. As we saw earlier for the interpretation of implication, we need a function to take proof of one to give the proof of other, which is exactly what the projection functions does in this case.

$A \vee B$ is usually also represented as tuple, but not a tuple of proofs, instead we have it as tuple of an indicator variable $i \in \{1,2\}$, along with a proof of $A$ or $B$ depending on whether $i = 1$ or $i = 2$.

In-case of $\neg A$, we need a notion of absurdity, lets call that $\bot$, We can think of it as false, but this is bit more stronger. We can think of it as absurdity, as in once its `true` we can build proofs for everything in the world. So $\neg A$, is represented as $A \rightarrow \bot$, ie., as a function which takes a proof of $A$ and gives back a proof of absurdity.

Some tautology and Interpretations
----------------------------------

- $P \rightarrow (Q \rightarrow P)$ - We can easily build a function which takes a proof of $P$ and give back a function which take proof of something and gives back proof of $P$ (Since we already have that).

- $(P \rightarrow (Q \rightarrow R)) \rightarrow (P \rightarrow Q) \rightarrow P \rightarrow R$ - This is simple, we build a function, that takes the third argument ie., proof of $P$ and pass it to second argument which gives a proof of $Q$, We also pass the third argument followed by the proof of $Q$ we just found to the first argument and we get proof of $R$

We saw in the previous, **Proposition as types** post that above 2 are actually types of the $K$ and $S$ combinator respectively, if you see the interpretation of the proofs that we arrived at its exactly the same as the corresponding combinator functions.


Non constructive tautology
--------------------------

Not all tautology has a constructive interpretation. For ex, $((P \rightarrow Q) \rightarrow P) \rightarrow P$ (Its called [Pierce's law](https://en.wikipedia.org/wiki/Peirce%27s_law). Type theoretically, its the type of the continuation operator, call/cc in scheme) doesnt have an interpretation, how can we prove $P$ using the function which gives proof $P$ but asks for proof $P$ in first place? Intuitively its not possible. BTW, is such a function useful?, apparently it is, will cover it in the next post.

So not all truth statements are provable in this sort of system, since it blatantly ignores the principle of excluded middle $P \vee \neg P$ ie., either $P$ or its negation is `true` for any $P$. Why? since we could extend this system to first order logic, but the kind of truth system will allow us to claim, existence of elements with certain properties just because the non-existence of it causes absurdity. Such claims arent very useful, since we know nothing about those elements in general. Also, disregading it, doesnt let us claim about stuff we dont yet know about, reduces paradoxes we need to deal with. 

Inspired from: [Logic of buddhist philosophy](http://aeon.co/magazine/world-views/logic-of-buddhist-philosophy/). Which uses multi-valued logic. One alternative is to use constructive logic like this to deal with paradoxes he talks about.