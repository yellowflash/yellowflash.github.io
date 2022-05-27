---
title: Implementing FP language  Part 2 - Beta reduction with substitution
layout: post
---

Before we get on to the actual execution of our functional programs, ie., reductions, we would define some of the notions and properties of lambda calculus, as it would help us tackle reductions more easily.

Free variables
--------------

Free variables are those variables which are called unbound in almost all programming languages. To give an example., in `λx.xy`, `y` is a free variable while `x` is not. We call `x` just before `y` bound and the other `x` as binding.

In certain lambda expressions, a variable can both be bound and free. For instance., in `x(λx.xy)`, the first `x` is free, since its not in scope of binding occurence, while the last, bound. To be clear, a variable can be both bound and free but an occurence of the variable can either be bound or binding or free.

Thus we can define free variables in `M` inductively as follows. 

- If `M` is a term variable `x`, then free variables in `M` is itself ie., `{x}`
- If `M` is an abstraction of form `λx.P` then free variables in `M` is free variables in `P` with `x`removed, since it gets bound.
- If `M` is an application of form `PQ` then free variables in `M`, is free varibles in `P` union free variables in `Q`

{% highlight scala linenos %}
sealed trait Expression {
  def freeVariables:Set[TermVariable]
}

case class TermVariable(name:String) extends Expression {
  override def freeVariables = Set(this)
}
case class Abstraction(param: TermVariable, exp: Expression) extends Expression {
  override def freeVariables = exp.freeVariables - param
}
case class Application(fn: Expression, arg: Expression) extends Expression {
  override def freeVariables = fn.freeVariables ++ arg.freeVariables
}
{% endhighlight %}

Substitution
------------

When we talk of function applications like., `(λx.xy)z` we usually think of it as substituting `z` for `x` in `xy`. Generally we denote such substitutions like `[z/x]xy`. Substitutions can be thought of as replacement of certain term variables with other lambda expressions. But., seeing substitutions as replacements have a minor snag to it. For example., in `[ab/x]x(λx.xy)` seeing substitutions as replacements would give rise to `ab.(λab.aby)` which is both syntactically and semantically invalid. Even if we replace just the non binding occurence, the result `ab(λx.aby)` is not exactly we want. 

So ideally what we want is, to replace free occurences alone. Even then there is another edge case to handle, where the expression we substitute has some free variables which gets bound after substitution, like in `[x/y]y(λx.xy)` with our naive approach of replacing free occurences would give us `x(λx.xx)` but thats not what we want. 

In order to overcome the second problem we do renaming of bound variables such that it doesnt collide. Since renaming of bound variable and its binding form doesn't change its meaning, ie., both `λx.x` and `λz.z` mean the same thing. Interesting thing is we could even see this renaming as substitution, just that we choose to substitute a term variable with another carefully chosen term variable. So we can fix our problem by doing another substitution to get rid of conflicts like., `[x/y]y(λz.[z/x](xy))` which would give rise to the acceptable form of `x(λz.zx)`. This is called as *alpha renaming*

We can now define substitution `[N/x]M` inductively as follows.,

- If `M=x` then the result is `N`
- If `M=y` where `y` is some other term variable., then there is nothing to substitute. The result is `y`.
- If `M=λx.P` then the term variable ain't free anymore in `P` so the result is `λx.P`
- If `M=λy.P` where `y` is some other term variable and there are no free variable collisions, ie., `y` is not free in `N` then the result is `λy.[N/x]P`.
- If `M=λy.P` where `y` is free in `N` then we chose `z` not free in `P` or `N` and the result is `λz.([N/x]([z/y]P))`
- If `M=PQ` then the result is just doing substitutions on both ie., `([N/x]P)([N/x]Q)`

{% highlight scala linenos %}
sealed trait Expression {
  ...
  def substitute(variable: TermVariable, expr: Expression):Expression
}

case class TermVariable(name:String) extends Expression {
  ...
  override def substitute(variable: TermVariable, expr:Expression) = if(variable == this) expr else this
}

object TermVariable {
  var index = 0
  def newVar = {
    index = index + 1
    TermVariable("_" + index)
  }
}

case class Abstraction(param: TermVariable, exp: Expression) extends Expression {
  ...
  override def substitute(variable: TermVariable, expr: Expression) = {
    if(expr.freeVariables contains param) this.alphaRename.substitute(variable, expr)
    else Application(param, expr.substitute(variable, expr))
  }
  def alphaRename:Expression = {
    val newVariable = TermVariable.newVar
    Application(newVariable, this.substitute(param, newVariable))
  }
}
case class Application(fn: Expression, arg: Expression) extends Expression {
  ...
  override def substitute(variable: TermVariable, expr: Expression) = Application(fn.substitute(variable, expr), arg.substitute(variable,expr))
}

{% endhighlight %}

The `newVar` function in `TermVariable` is guaranteed to be non colliding since, we dont allow variables with starting with `_` in our language also we generate a new variable each time using the counter.

Reduction
---------

With our notion of substitution clear we can now define reduction or the execution of our lambda expressions. Reductions are much simpler in the sense wherever we find a sub term of the form `(λx.M)N` we replace that with `[N/x]M` and keep doing that until there is no subterm of the form `(λx.M)N`. 

There is only one minor detail to take care of, which is the order of application. For now we would follow what is called as *left most reduction* also called as *normal order evaluation*, in which we apply the function without reducing the argument first. Its defined like this,

{% highlight scala linenos %}
sealed trait Expression {
  ...
  def reduce:Expression
}

case class TermVariable(name:String) extends Expression {
  ...
  override def reduce = this
}

case class Abstraction(param: TermVariable, exp: Expression) extends Expression {
  ...
  override def reduce = Abstraction(param, exp.reduce)
}
case class Application(fn: Expression, arg: Expression) extends Expression {
  ...
  override def reduce = fn match {
    case Abstraction(variable, exp) => exp.substitute(variable, arg).reduce
    case _ => Application(fn.reduce, arg.reduce)
  }
}

{% endhighlight %}

Thus we have a working functional language. In the next post we would see other orders of evaluations and also talk about *normal forms* and termination *conditions*.

The complete source code is in another branch called 'substitution' [here](https://github.com/yellowflash/hindley/tree/substitution).