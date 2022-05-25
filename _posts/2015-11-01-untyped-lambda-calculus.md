---
title: Implementing FP language  Part 1 - Untyped lambda calculus
layout: post
---

In this series we would implement a Typed ML style functional programming language. Also would talk about some of the design decisions and alternatives for the same. Would try to keep the formalism as minimal as possible, but introduce them as and when needed. Let's get started. 

Almost all FP languages have their basis on lambda calculus, which is rather simple.  The computation is modelled with just one operation - function application (or substitution). Hence basic constructs are the ones which are just necessary to define and apply functions nothing more. And nothing less. 

In lambda calculus everything is a lambda term. And it can be any of the following

- **Term variables** (just like variables in programming languages). We usually use small letters `(x,y,z...)` to represent term variables
- **Abstraction** - This is basically function definition. Defined as, `λx.M` where `x` is any term variable and `M` is any lambda term (which can be a term variable, application or abstraction)
- **Application** - Which is function application. If `M` and `N` are lambda terms `MN` is application.

Some examples
-------------

The identity function is defined as `λx.x` which takes an argument and gives back the same. We never name any functions, all functions are anonymous, which is why most programming languages call anonymous functions as lambda expressions. 

Another example is `(λx.x)y` which is an application of `y` as an argument to the identity function. The result of the application is `y`. But what is `y`? Unlike programming languages lambda calculus allows unbound variables since it allows some formalizations to be much simpler. Whenever there is  `MN` where `M` and `N` are lambda expressions, we can visualize it as `M(N)` in other programming languages.

So how do we define functions with multiple arguments? We use a technique called *Currying* (named after [Haskell Curry](https://en.wikipedia.org/wiki/Haskell_Curry)). For instance if we are to define a function which ignores the second argument and gives back the first. We define a function on one argument which gives a function which takes another argument but ignores it, like., `λx.λy.x`

Non trivial example
-------------------

With our informal notion of function application, I would make the claim that almost anything thats computable can be expressed this way aka., this system is *Turing complete*. In order to grasp the sense of such a small and beautiful system, we would write a function for finding whether a number is even or odd.

But where are numbers? booleans? Like in any computational model we encode them into the system. We encode numbers as binary in a system which understands binary. Similarly we encode numbers and booleans as functions here.

Boolean functions
-----------------

While choosing an encoding it would be nice if we could make some of the operations in it easier. For instance with booleans we usually make conditionals ie., choose one over other. Thus we define,


`TRUE: λx.λy.x`

`FALSE: λx.λy.y`

See `TRUE` chooses the first argument and `FALSE` chooses the second argument, which is more like how if-else statement behaves in a programming language. 

We would now define some useful functions on booleans now.

`NOT: λb.b(λx.λy.y)(λx.λy.x)`

Choose `FALSE` incase `b` is `TRUE` and choose `TRUE` incase `b` is `FALSE`.

`AND: λa.λb.ab(λx.λy.y)`

Choose `b` incase `a` is `TRUE` or choose `FALSE`

Similarly we can define `OR`, `XOR` and all the computable boolean functions.

Numerical functions
-------------------

Now we define numbers similarly. If `n` is a number the most useful operation is to repeat something `n` times. Hence we define numbers as

`0: λf.λx.x`

`1: λf.λx.fx`

`2: λf.λx.f(fx)`

`.`

`.`

<code>n: λf.λx.f<sup>n</sup>x</code>

*Note : Applications are left associative by default so `ffx` means `(ff)x` and not `f(fx)`.*

This method of encoding is called as *Church numeral* (Named after [Alonzo Church](https://en.wikipedia.org/wiki/Alonzo_Church)). So numbers are nothing but functions which take a function and an argument and applies the function `n` times on that argument.

We can define some of the useful functions on numbers too. Like.,

`SUCCESSOR: λn.λf.λx.f(nfx)`

Apply `f` one more time after applying `n` times.

`PREDECESSOR` is bit difficult. Church almost gave up on this, but finally figured out, after all its turing complete. I am not going to define it here though, would leave it as a fun challenge. I would jump to `SUM`

`SUM: λa.λb.λf.λx.af(bfx)`

Apply `a` times `f` on the result of applying `b` times `f` on `x` or using `SUCCESSOR` like,

`SUM: λa.λb. a SUCCESSOR b`

Though the right way is to replace `SUCCESSOR` with the expression, i used this form to explain what i mean.

Similarly we can use `SUM` to define `PRODUCT`. 

`PRODUCT: λa.λb. a (SUM b) 0`

Now we get to our original problem of defining a function which finds whether a given number is even or odd. The approach we will use is - given any number `n` if we alternate between `TRUE` and `FALSE` `n` times starting with `TRUE`, we can find whether the number is even or odd. Hence 

`EVEN: λn. n NOT TRUE`

Apply `NOT` `n` times on the value `TRUE`. It works like this 

`0: TRUE`

`1: NOT(TRUE)`

`2:  NOT(NOT(TRUE)`

`.`

`.`

To be precise 

`EVEN:  λn.n(λb.b(λx.λy.y)(λx.λy.x))(λx.λy.x)`

That would have given an idea of the power of lambda calculus. Note that there are multiple occurences of `x` and `y` each with its own *scope*. We would formalize this in next part along with substitution, hich would allow us to implement a small interpreter for the same. 

Implementation
--------------

The implementation of the interpreter for our programming language would be done with [Scala](http://www.scala-lang.org/). (Which is a cool functional programming language on JVM).

We need to define the Abstract syntax representation for our language, which would directly correspond to the inductive definition of lambda terms. 

{% highlight scala %}
sealed trait Expression

case class TermVariable(name:String) extends Expression
case class Abstraction(param: TermVariable, exp: Expression) extends Expression
case class Application(fn: Expression, arg: Expression) extends Expression
{% endhighlight %}

*Note: We have defined the first argument of application as `Expression`, rather than as `Abstraction`. This is intentional since `xy` is also a valid lambda expression, and x could be bound to an `Abstraction`.*

Coming to parser, *Scala* has a monadic parser library called *RegexParsers* in its standard library. Its very simple to use. (Monadic parsers are a topic on its own. Would cover it in another post.)

{% highlight scala %}
object Parser {
  def expression:Parser[Expression] =  (factor+)^^{case applications => applications.reduceLeft(Application.apply)}
  def factor = paranthesised | termVariable | abstraction
  def paranthesised = ("("~expression~")") ^^{case _~expr~_ => expr}
  def termVariable =  literal ^^ {case variable => TermVariable(variable)}
  def abstraction = ("\\"~literal~"."~expression)^^{case "\\"~argument~"."~expr => Abstraction(TermVariable(argument), expr)}
...
}
{% endhighlight %}

Few things to note here, We use spaces as delimiters for application, in-order to support multiple character variables. Also we use `\` to denote `λ`, which is easier to type.

Also note the way `Application` is defined, it makes sure that it is left-factored and left associative.

Thus we have a parser for our programming language. In the next part we would formally define substitution and implement the same.

Complete source code is available [here](https://github.com/yellowflash/hindley).