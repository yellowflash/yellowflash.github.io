---
title: Defining Functors in Scala
layout: post
---

Functors are like homo-morphisms between categories. It means, structure preserving maps between 2 structures, in our case categories. Intuitively, we can think of Functors as things that can be mapped. For example, `List`, `Option`, `Future` etc.. Yes, we are talking about the `map` function in Scala. Lets actually see what the signatures of each of the `map` function look like.,

{% highlight scala %}

class List[T] {
  ....
  def map[K](fn: T => K): List[K] = ???
  ....
}

class Option[T] {
  ....
  def map[K](fn: T => K): Option[K] = ???
  ....
}

class Future[T] {
  ....
  def map[K](fn: T => K): Future[K] = ???
  ....
}
 
{% endhighlight %}

Though the actual Scala signature would look a lot more bizarre, this simplistic view would be enough for our discussion. 

The thing i would like to do now is, to define a trait which could be overriden by all `Functor`s. Though it might look simple, it's not. Lets take a first stab at it.



{% highlight scala %}

trait Functor[T] {
  def map[K](fn: T => K): Functor[K]
}

{% endhighlight %}


Well, the above definition isn't really a `Functor`. Say for instance when i am implementing it in `Option`, the `map` function could as well return a `List` (which will also be a functor). The types should be stricter. Inorder to have a stricter definition, a notion of `Kinds` is needed.

Kinds
-----

Kinds can be thought of as type for types. Inductively, every concrete type has a kind (read it as type, for simplicity) `*`

Say for example, `Int`,`T` (type variable), `List[Int]`, `List[T]` all are of kind `*` . Every type we can infer or assign for a variable is of kind `*`.

We also have kinds for, if i can say partial types, like `List`. The kind of `List` will be `* -> *`. Why? Because it takes a concrete type `*` to give back a `*`, ie., `List[Int]`, `List[T]` .. 


So `Kinds` of some common classes.,

- `Option` is of kind `* -> *`
- `Either` is of kind `* -> * -> *` (Takes 2 type parameters to build a concrete type and its curried)

The distinction has to be made between `Option[T]` which has kind `*` and `Option` which has kind `* -> *`

Another important difference of types from kind is, kinds aren't polymorphic (well unifiable), For ex, when we have identity function with signature `T -> T` we can pass another function `K -> K` as an argument for that, which means `T` and `K -> K` can be unified. But with kinds, if someone asks for `*`, we cannot give it a `* -> *`. 


Functor
-------

With that defined, we are clear now what our `Functor` actually was lacking. We need a `Kind` which we can instantiate and give it as result type. Like this.,

{% highlight scala %}

trait Functor[T, F[_]] {
  def map[K](fn: T => K): F[K]
}

{% endhighlight %}


The `F[_]` is a kind of `* -> *` . That is how we define higher kinds saying i dont know what the argument is going to be but leave it blank for now.

One interesting thing to note is, what is the kind of `Functor` now? Its of kind, `* -> (* -> *) -> *`.

This is pretty good from where we started, but there is still another problem that we might want to address. The problem is, some one might implement the functor on `List` but pass `Option` as a kind parameter. The type signature doesnt restrict that. In which case the `map` function on `List` would return `Option`. 


Self type constraint
--------------------

In Scala traits, we can add constraints on what else a concrete class must implement for it to implement a given trait. We can use that to say that, anything that implement the `Functor` should implement `F[T]` too. Like this

{% highlight scala %}

trait Functor[T, F[_]] { self: F[T] =>
  def map[K](fn: T => K): F[K]
}

{% endhighlight %}

That now fixes our `Functor` for good. A `List` now cannot return an `Option` on `map`.


An interesting case is, `Tuple2` can have a `Functor` implementation on the second value. But how will it look like? Why is this interesting? Since the kind of `Tuple2` is `* -> * -> *`. [Check here](https://gist.github.com/yellowflash/bbdd8e68aca1e5cb1e7f)
