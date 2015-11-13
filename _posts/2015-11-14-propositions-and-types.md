---
title: Propositions and Types
layout: post
math: true
---

Propositional logic in short is boolean logic as programmers would call it. It involves doing boolean operations on boolean variables and constants. In proposition logic we call boolean variables as *Atomic Propositions* and the boolean expressions as *Propositions*.

The usual boolean operations are `AND`, `OR` and `NOT` represented as $\wedge$, $\vee$ and $\neg$  respectively. Some example propositions are $A \wedge B$ ie., `A AND B`, $\neg A \vee B$ ie., `NOT(A) OR B`

Tautology and proofs
--------------------

What we are concerned with in a proposition is the notion of *satisfiability* ie., can the proposition ever be made `TRUE` by any assignment of boolean values to variables in it?

For example., $A \wedge B$ is satisfiable with assignment `A=TRUE` and `B=TRUE`, but $\neg A \wedge A$ is not satisfiable with any assignment.

Any proposition which isnt satisfiable is *inconsistent* and Any proposition which is satisfiable with every assignment is *tautology*. For example., $\neg A \vee A$ is a tautology. As we can see negation of any inconsistent proposition is a tautology and vice versa.

To verify something is satisfiable or tautology is simple if we deal with it semantically by applying every possible assignment of boolean values to variables and find whether its satisfiable or tautology  (but if you consider complexity its hard).

But if we consider *first order logic* we cannot verify a statement as `TRUE` or `FALSE` this way. So what we do is a method of derivation in those cases. By encoding the semantics of the system as set of axioms (or axiom schemas to be more accurate), and have an inference mechanism to derive all the truth statements. We call that a *proof*.

Hilbert System
--------------

Since propositional logic is simpler we would start doing the same here. [Hilbert](https://en.wikipedia.org/wiki/David_Hilbert) defined a set of axiom schemas for the propositional logic. But he didnt use `AND` and `OR` connectives. He used an operation called `IMPLICATION`, denoted with $\rightarrow$. For example., $X \rightarrow Y$ which informally means *If X is TRUE then Y is TRUE*.

Any proposition on $\wedge , \vee, \neg$ could be expressed with $\rightarrow , \neg$ (Its easier to prove that by simply expressing `AND`, `OR` with `IMPLICATION` and `NOT`) and vice versa., since $A \rightarrow B \equiv (\neg A \vee B)$


Now given $A$, $B$ and $C$ be any atomic propositions.

- $A \rightarrow (B \rightarrow A)$
- $(A \rightarrow (B \rightarrow C)) \rightarrow (A \rightarrow B) \rightarrow (A \rightarrow C)$
- $(A \rightarrow B) \rightarrow (A \rightarrow \neg B) \rightarrow \neg A$

We can verify all of them are tautology by writing down truth tables for it.

We also have an inference rule called **Modes Ponens**

- If $A \rightarrow B$ and $A$ is `TRUE` (or proved), then $B$ is `TRUE`

Example Proof
-------------

We know $P \rightarrow P$ is a tautology, we can give a proof for it as follows.

1: Axiom 2 - $(P \rightarrow ((P \rightarrow P) \rightarrow P)) \rightarrow (P \rightarrow (P \rightarrow P)) \rightarrow (P \rightarrow P)$ (Using $A=P$, $B= P \rightarrow P$ and $C=P$)

2: Axiom 1 - $(P \rightarrow ((P \rightarrow P) \rightarrow P))$ (using $A=P$ and $B=P \rightarrow P$)

3: Modes Ponens 1 & 2 - $(P \rightarrow (P \rightarrow P)) \rightarrow (P \rightarrow P)$

4: Axiom 1 - $P \rightarrow (P \rightarrow P)$ (Using $A=P$, $B=P$)

5: Modes Ponens 3 & 4 - $P \rightarrow P$

QED

Types 
-----

If we consider simplest typed programming language (or lambda calculus), the only types we need to consider are 

- Atomic type variables denoted by $A,B,C, ..$
- Functions from one type to another denoted by $P \rightarrow Q$ where $P$ and $Q$ can inturn be atomic type variables or functions.

One of the fascinating correlation of these types with propositional logic is, we can consider propositions as types and programs as proofs ([Curry Howard Isomorphism](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence)).

For instance the first two Hilbert's Axioms are nothing but types of `K` and `S` combinator respectively if we consider `IMPLICATION` as `FUNCTION` type.

- $K = \lambda x.\lambda y. x$
- $S = \lambda x.\lambda y.\lambda z. (xz)(yz)$

And *Modes Ponens* inference rule is nothing but function application.

The last one is tricky, its basically *Proof by contradiction* which can be used to prove *Law of excluded middle*. Constructive mathematics on which this isomorphism relies on doesnt allow it (Would talk more about it in seperate post). Without the negation and the last axiom, the reduced propositional logic is called *Positive implicational logic*.

Hence existence of a lambda function itself is a proof of the proposition, since lambda calculus is equivalent to `SKI` combinatory logic. Where

$I = \lambda x.x$

$I$ has type $A \rightarrow A$ and we proved that as tautology expressible by other axioms and our proof is nothing but $I \equiv SKK$ ie., application of *Modes Ponens* to the *Modes Ponens* of *Axiom 2* and *Axiom 1* with *Axiom 1* again.

Hence when we are programming most of the times we are actually proving something in Propositional Logic :) .