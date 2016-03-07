---
title: Implementing FP language  Part 4  - Recursion and Fixed Points
layout: post
---

One of the missing pieces of lamba calculus as we have seen so far is the ability to express recursive functions, since the ability to define and use a function is absent in lambda calculus. But any reasonable functional programming relies on recursion to do anything non-trivial. Here we would develop some theory around how we implement recursion as a feature of the language.

Definitions
-----------

So far in our programming language we never named any function, all functions that we defined and used were anonymous. We would now add the syntax for naming/defining a function, since any recursive function needs a way to refer to itself somehow. The syntax we introduce will be like 

`let  x = P in Q` 

where $$x$$ is a term variable $$P$$ and $$Q$$ are lambda expressions. Though this form is equivalent to defining something like this 

`(\x. Q) P`

But we will still introduce this `let` form, as it is much clearer, and also it allows us to define polymorphic functions. We would talk more about it when we get to type systems.


Self recursion
--------------

Consider the program 

`let fact = \n . if n == 1 then 1 else (n * (fact (n - 1))) in (fact 5)`

Though we dont yet have a notion of numbers and `if ... then ... else ...`, consider they are available for the moment (We would make them available soon). How do we transform that into a lambda expression?

Any self-recursive function will be of the form

`let P = \x ... P ...`

The primary problem is we need a way to use `P` inside the definition of `P` itself. We can rewrite this expression into

`let P = (\q. (\x. .... q ...))P`

And also if we were some how able to build a lambda expression, lets call it `Y`, which could do infinite self application of a function applied to itself like,

`Yf = f(Yf)` ie., `Yf = f(f(f(f(f .....))))`

Lets call `E = \q.(\x. .... q ...)`, then the definition of `P` will be like

`P = EP` expanded to `P = E(EP)` to `P = E(E(EP))` ....

Now we can use `Y` as

`let P = Y(\q. (\x. .... q ...))`

that removes the recursive usage of `P` in the definition of P 

Our factorial example will be 


`let fact = Y(\fn. \n. if n == 1 then 1 else (n * (fn (n - 1)))) in (fact 5)`

The execution of the same would be like this

- `fact 5`
- `(\n. (if n ==1 then 1 else n * ((Y fn) (n - 1)))) 5`
- `5 * ((\n. (if n ==1 then 1 else n * ((Y fn) (n - 1)))) 4)`
- `5 * 4 * ((\n. (if n ==1 then 1 else n * ((Y fn) (n - 1)))) 3)`
- `5 * 4 * 3 * ((\n. (if n ==1 then 1 else n * ((Y fn) (n - 1)))) 2)`
- `5 * 4 * 3 * 2 * ((\n. (if n ==1 then 1 else n * ((Y fn) (n - 1)))) 1)`
- `5 * 4 * 3 * 2 * 1`


Fixed point combinator
----------------------

Now the question is more about is it possible to define such a `Y` in lambda calculus. We know of a lambda expression $$(\lambda x. xxx)(\lambda x. xxx)$$. which expands to $$(\lambda x. xxx) (\lambda x.xxx)(\lambda x.xxx)$$ and never terminates

Lets call $$Z = \lambda x.xxx$$ and $$E = ZZ$$. Its clear that $$E \rightarrow ZE \rightarrow Z(ZE) ...$$ which looks similar to our $$Yf$$ reduction, except if we could somehow keep generating $$f$$ instead of $$Z$$ so we can use that principle to build a lambda expression

$$ Y = \lambda f. (\lambda x. fxx)(\lambda x. fxx) $$

It is called a fixed point combinator, since it can be used to find fixed point of lambda expression. Fixed point of a function $$f$$ is any $$x$$ such that $$fx = x$$. More on fixed points in some other post. 

Thus we can get rid of recursion and compile all of `let x = Q in P` to `(\x. P)(Y(\x. Q))`. But we wont do that, since it gets really complicated when we have mutual recursion, though this is a nicer way to do execution incase we follow substitution. Execution with environments have better and easier ways to do recursion.

