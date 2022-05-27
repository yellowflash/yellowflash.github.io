---
title: Implementing FP language Part 3 - Execution with environments
layout: post
---

Substitution gave a simple and formal way to describe beta reduction. But the problem with substitution is the alpha renaming of variables to remove conflicts. Its very inefficient to do alpha renaming of variables everytime we execute a subexpression. That brings the question of order of evaluation.

Normal Order
------------

Given a lambda expression like $(\lambda x. x)((\lambda y.z)(\lambda z.z))$, there are 2 choices of reduction here, either we reduce it to $(\lambda y.z)(\lambda z.z)$ or $(\lambda x.x)z$. Though in this case both of them eventually terminates, but thats not the usually the case.

Its simpler to think of the reduction order as either to evaluate arguments first or the function applied to first, called as *applicative* and *normal* order of evaluation respectively. What we did with our substitution implementation is normal order.

Applicative order of evaluation is strict evaluation where every argument, even if its not being used would get evaluated. But normal order doesnt evaluate arguments in case there are no usages of arguments.

Consider the example $(\lambda x.y)((\lambda x.xx)(\lambda x.xx))$ would never terminate in applicative order of evaluation but when executed with normal order would give the result $y$.

Lazy evaluation
---------------

But the problem with normal order evaluation is there are times the argument might get executed multiple times, which the applicative order evaluation completely avoids. So in-order to get the best of both worlds we need to delay the evaluation until its being used and evaluate only once. This mode of evaluation is called as *lazy* evaluation.

Every expression that need to be executed in the lazy evaluation is represented as *thunk*. Thunk contains the lambda expression and the bindings for all the variables that are in scope. 

{% highlight scala linenos %}
case class Binding(vars:Map[TermVariable,Expression]=Map.empty) {
  def apply(variable: TermVariable)  = vars.getOrElse(variable, variable)
  def +(more:(TermVariable, Expression)) = Binding(vars + more)
}

...

class Thunk(expr:Expression, binding: Binding) extends Expression {
  lazy val evaluated = expr.reduce(binding)
  override def freeVariables = expr.freeVariables
  override def reduce(binding: Binding) = this
  override def toString = evaluated.toString
}
case class TermVariable(name:String) extends Expression {
  ...
  override def reduce(binding:Binding) = binding(this)
}

case class Abstraction(param: TermVariable, exp: Expression) extends Expression {
  ...
  override def reduce(binding: Binding) = Abstraction(param, exp.reduce(binding))
}

case class Application(fn: Expression, arg: Expression) extends Expression {
  ...
  override def reduce(binding: Binding) = fn match {
    case Abstraction(param, expr) => expr.reduce(binding + (param -> new Thunk(arg,binding)))
    case _ => Application(fn.reduce(binding), arg.reduce(binding))
  }
}

{% endhighlight %}

There are two problems here. First a minor one, consider the case of $(\lambda x. \lambda y. x)(\lambda x. y)$, gets evaluated to $\lambda y. (\lambda x. y)$. Thats not a problem, since internally it knows they are two different $y$s, once we report unbound variables we can fix that.

The next one is more around garbage collection of bindings. We keep accumulating bindings. And most of them aren't of any use inside the expression anyway. We should capture only those that are free inside the bound expression.


Other approaches
----------------

As we see the free variable conflicts are the ones that almost always make the execution tricky. There are alternate approaches to do the execution. One of them is to rewrite terms with [De-Bruijn Index](https://en.wikipedia.org/wiki/De_Bruijn_index), where instead of naming the variables we use the positions. Another approach is to use combinatory logic, like in [Miranda](). The combinators in combinatory logic gets rid of this whole notion of free variables and bound variables completely by using a fixed set of combinators. Both the approaches removes variable names, so the debuggability is lost in some sense. 
