---
title: Continuations
layout: post
---

Long back i read about a blog post explaining a beautiful use-case of `call/cc`. It tried to solve on set of variables very elegantly, in a declarative style. I tried doing something similar in scala with `Cont` monad. The result looked like this


{% highlight scala linenos %}

for(
  a <- choose(1,2,3,4,5);
  b <- choose(1,2,3,4,5);
  c <- choose(1,2,3,4,5)
) yield {
  check((a*a + b*b) == c*c)
  (a,b,c)
}).get
 
{% endhighlight %}

The above code snippet tries to find pythogorean triples. In fact we can replace the condition and solve any constraint satisfaction problem which can be solved with back tracking. First of all lets see what continuations are.

Continuations
-------------

Imagine a programming language which doesnt have return statements. How would functions look like? 

Every function would take a function as a parameter to which it would pass the result to. The parameter function is called as Continuation. It can be thought of sort of like a callback. 

In static typed languages, a continuation of `A` is any function which takes `(A => R)` and gives back an `R` (Usually `R` is bottom type, like `call/cc`)

So lets see a simple example of sum of squares of 2 integers written with continuations.,

{% highlight scala linenos %}

def add[R](a: Int, b: Int)(cont: Int => R) = cont(a+b)
def square[R](a: Int)(cont: Int => R) = cont(a*a)
def sumOfSquares[R](a: Int, b: Int)(cont: Int => R) = {
  square(a)(asquare => 
    square(b)(bsquare => 
      add(asquare, bsquare)(cont))
}
 
{% endhighlight %}

The `add` and `square` functions looks clean. It expresses our intent clear, but the `sumOfSquares` looks bit clumsy. But the structure of it says there could be a `Monad` instance for the it. 

Cont Monad
----------

We would first abstract `Cont` as a class and implement `map` and `flatMap` on it.

{% highlight scala linenos %}
abstract class Cont[A] {
  def run[R](cont: A => R): R
}

def add(a:Int, b:Int): Cont[Int] = new Cont[Int] {
  def run[R](cont: Int => R) = cont(a+b)
}
{% endhighlight %}

This would make the code much clearer than our original reference of `(A => R) => R` since the type parameter `R` in case of `run` method is bound only when you call it. For instance following code snippet is valid in case the `add` returns `Cont[Int]` instead of `(Int => R) => R`

{% highlight scala linenos %}
val sumOf1And2 = add(1,2)
val asString = sumOf1And2.run(_.toString)
val asDouble = sumOf1And2.run(_.toDouble)
{% endhighlight %}

Now going back to our Monad instance, lets implement `map` on it, signature basically looks like this

{% highlight scala linenos %}
abstract class Cont[A] {
....
  def map[B](fn: A => B): Cont[B] 
}
{% endhighlight %}

What map does is, it transforms the result into something else, ie., we apply the transformation, before we call the next continuation. Its like calling a function on the return value of some function.

What about `flatMap`? whose signature is this

{% highlight scala linenos %}
abstract class Cont[A] {
....
  def flatMap[B](fn: A => Cont[B]): Cont[B] 
}
{% endhighlight %}

Its exactly similar to map is, except that even the transformation function follows the convention of not returning the result and instead takes a continuation. So we have our `flatMap` and `map` implementation like this,

{% highlight scala linenos %}
abstract class Cont[A] {
  def run[R](cont: A => R): R
  def map[B](fn: A => B): Cont[B] = {
    val self = this
    new Cont[B] {def run[R](cont: B => R) = self.run(cont compose fn)}
  }
  def flatMap[B](fn: A => Cont[B]): Cont[B] = {
    val self = this
    new Cont[B] {def run[R](cont: B => R) = self.run(a => fn(a).run(cont))}
  }
  def get = run(identity)
}
{% endhighlight %}

We also have a get function in `Cont` which basically turns a `Cont` back into a real return value.

Control structures
------------------

Why would anyone want to give back a `Cont` when its easier to return a value back? Because clearly `Cont` is way more powerful. You can choose to call the `cont` parameter in a run function once, or multiple times or never at all. Thats exactly what we are going to do with our original pythagorean triple example,

So the `choose` function instead of returning a chosen value, gives back a `Cont`. And the run function in that `Cont` keeps calling the `cont` parameter with each value until it finds the one which doesnt get an exception. And `check` function is even simpler, it throws an exception everytime it get a false value. 

Like this,
{% highlight scala linenos %}
def choose[A](as: A*):Cont[A] = new Cont[A] {
  override def run[R](cont: A => R): R = {
    doChoose(cont, as)
  }

  def doChoose[R](cont: A => R, values: Iterable[A]):R = {
    if(values.isEmpty) throw new RuntimeException("No choice is valid")
    Try(cont(values.head)).getOrElse(doChoose(cont, values.tail))
  }
}
def check(bool: => Boolean): Unit = {
  if(!bool) throw new RuntimeException("Constraint fails")
}
{% endhighlight %}

Thus we keep trying for each combination of values for `a`, `b` and `c` until we hit upon one which satisfies our check conditions.

The entire code is in [here](https://gist.github.com/yellowflash/361c6e31d5f2323f603b5fb5e02404df)