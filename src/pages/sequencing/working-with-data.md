---
layout: page
title: Generic Folds for Generic Data
---

We've seen that when we define a class with generic data, we cannot implement very many methods on that class. The user supplies the generic type, and thus we must ask the user to supply functions that work with that type. Nonetheless, there are some common patterns for using generic data, which is what we explore in this section. We have already seen **fold** in the context of our `IntList`. Here we will explore fold in more detail, and learn the pattern for implementing fold for any algebraic data type.


## Fold

Last time we saw fold we were working with a list of integers. Let's generalise to a list of a generic type. We're already see all the tools we need. First our data definition, in this instance slightly modified to use the covariant sum type pattern.

~~~ scala
sealed trait LinkedList[A]
final case class Pair[A](head: A, tail: LinkedList[A]) extends LinkedList[A]
final case class End[A]() extends LinkedList[A]
~~~

The last version of `fold` that we saw on `IntList` was

~~~ scala
def fold[A](end: A, f: (Int, A) => A): A =
  this match {
    case End => end
    case Pair(hd, tl) => f(hd, tl.fold(end, f))
  }
~~~

It's reasonably straightforward to extend this to `LinkedList[A]`. We merely have to account for the head element of a `Pair` being of type `A` not `Int`.

~~~ scala
sealed trait LinkedList[A] {
  def fold[B](end: B, f: (A, B) => B): B =
    this match {
      case End() => end
      case Pair(hd, tl) => f(hd, tl.fold(end, f))
    }
}
final case class Pair[A](head: A, tail: LinkedList[A]) extends LinkedList[A]
final case class End[A]() extends LinkedList[A]
~~~

Fold is just an adaptation of structural recursion where we allow the user to pass in the functions we apply at each case. As structural recursion is the generic pattern for writing any function that transforms an algebraic datatype, fold is the concrete realisation of this generic pattern. That is, fold is the generic transformation or iteration method. *Any function* you care to write on an algebraic datatype can be written in terms of fold.

<div class="callout callout-info">
#### Fold Pattern

For an algebraic datatype `A`, fold converts it to a generic type `B`. Fold is a structural recursion with:

- one function parameter for each case in `A`;
- each function takes as parameters the fields for its associated class;
- if `A` is recursive, any function parameters that refer to a recursive field take a parameter of type `B`.

The right-hand side of pattern matching cases, or the polymorphic methods as appropriate, consists of calls to the appropriate function.
</div>

Let's apply the pattern to derive the `fold` method above. We start with our basic template:

~~~ scala
sealed trait LinkedList[A] {
  def fold[B](???): B =
    this match {
      case End() => ???
      case Pair(hd, tl) => ???
    }
}
final case class Pair[A](head: A, tail: LinkedList[A]) extends LinkedList[A]
final case class End[A]() extends LinkedList[A]
~~~

This is just the structural recursion template with the addition of a generic type parameter for the return type.

Now we add one function for each of the two classes in `LinkedList`.

~~~ scala
def fold[B](end: ???, pair: ???): B =
  this match {
    case End() => ???
    case Pair(hd, tl) => ???
  }
~~~

From the rules for the function types:

- `end` has no parameters (as `End` stores no values) and returns `B`. Thus its type is `() => B`, which we can optimise to just a value of type `B`; and
- `pair` has two parameters, one for the list head and one for the tail. The argument for the head has type `A`, and the tail is recursive call and thus has type `B`. The final type is therefore `(A, B) => B`.

Substituting in we get

~~~ scala
def fold[B](end: B, pair: (A, B) => B): B =
  this match {
    case End() => end
    case Pair(hd, tl) => pair(hd, tl.fold(end, pair))
  }
~~~

## Working With Functions

There are a few tricks in Scala for working with functions and methods that accept functions (known as higher-order methods). Here we are going to look at:

1. a compact syntax for writing functions;
2. converting methods to functions; and
3. a way to write higher-order methods that assists type inference.

### Placeholder syntax

In very simple situations we can write inline functions using an extreme shorthand called **placeholder syntax**. It looks like this:

~~~ scala
scala> ((_: Int) * 2)
res23: Int => Int = <function1>
~~~

`(_: Int) * 2` is expanded by the compiler to `(a: Int) => a * 2`. It is more idiomatic to use the placeholder syntax only in the cases where the compiler can infer the types. Here are a few more examples:

~~~ scala
_ + _     // expands to `(a, b) => a + b`
foo(_)    // expands to `(a) => foo(a)`
foo(_, b) // expands to `(a) => foo(a, b)`
_(foo)    // expands to `(a) => a(foo)`
// and so on...
~~~

Placeholder syntax, while wonderfully terse, can be confusing for large expressions and should only be used for very small functions.

### Converting methods to functions

Scala contains one final feature that is directly relevant to this section -- the ability to convert method calls to functions. This is closely related to placeholder syntax -- simply follow a method with an underscore:

~~~ scala
scala> object Sum {
         def sum(x: Int, y: Int) = x + y
       }
defined module Sum

scala> Sum.sum
<console>:9: error: missing arguments for method sum in object Sum;
follow this method with `_' if you want to treat it as a partially applied function
              Sum.sum
                  ^

scala> (Sum.sum _)
res11: (Int, Int) => Int = <function2>
~~~

In situations where Scala can infer that we need a function, we can even drop the underscore and simply write the method name -- the compiler will promote the method to a function automatically:

~~~ scala
scala> object MathStuff {
         def add1(num: Int) = num + 1
       }
defined module MathStuff

scala> Counter(2).adjust(MathStuff.add1)
res12: Counter = Counter(3)
~~~

### Multiple Parameter Lists

Methods in Scala can actually have multiple parameter lists. Such methods work just like normal methods, except we must bracket each parameter list separately.

~~~ scala
scala> def example(x: Int)(y: Int) = x + y
example: (x: Int)(y: Int)Int

scala> example(1)(2)
res2: Int = 3
~~~

Multiple parameter lists have two relevant uses: they look nicer when defining functions inline and they assist with type inference.

## Exercises

#### Folding Maybe

In the last section we implemented a sum type for modelling optional data:

~~~ scala
sealed trait Maybe[+A]
final case class Full[A](value: A) extends Maybe[A]
final case object Empty extends Maybe[Nothing]
~~~

Implement fold for this type.

<div class="solution">
The code is very similar to the implementation for `LinkedList`. I choose polymorphism for my solution. I belive it's more idiomatic and it provides an example to contrast with the pattern matching solution for `LinkedList`.

~~~ scala
sealed trait Maybe[+A] {
  def fold[B](full: A => B, empty: B): B
}
final case class Full[A](value: A) extends Maybe[A] {
  def fold[B](full: A => B, empty: B): B =
    full(value)
}
final case object Empty extends Maybe[Nothing] {
  def fold[B](full: A => B, empty: B): B =
    empty
}
~~~
</div>

#### Folding Sum

In the previous section we implemented a generic sum type:

~~~ scala
sealed trait Sum[A, B]
final case class Left[A, B](value: A) extends Sum[A, B]
final case class Right[A, B](value: B) extends Sum[A, B]
~~~

Implement `fold` for `Sum`.

<div class="solution">
~~~ scala
sealed trait Sum[A, B] {
  def fold[C](left: A => C, right: B => C): C
}
final case class Left[A, B](value: A) extends Sum[A, B] {
  def fold[C](left: A => C, right: B => C): C =
    left(value)
}
final case class Right[A, B](value: B) extends Sum[A, B] {
  def fold[C](left: A => C, right: B => C): C =
    right(value)
}
~~~
</div>


#### Tree

A binary tree can be defined as follows:

A `Tree` of type `A` is a `Node` with a left and right `Tree` or a `Leaf` with an element of type `A`.

Implement this algebraic data type along with a fold method.

<div class="solution">
This is a tricker example than the previous two exercises as we have a recursive data type. Follow the patterns and you should be ok.

~~~ scala
sealed trait Tree[A] {
  def fold[B](node: (B, B) => B, leaf: A => B): B
}
final case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A] {
  def fold[B](node: (B, B) => B, leaf: A => B): B =
    node(left.fold(node, leaf), right.fold(node, leaf))
}
final case class Leaf[A](value: A) extends Tree[A] {
  def fold[B](node: (B, B) => B, leaf: A => B): B =
    leaf(value)
}
~~~
</div>

Using `fold` convert the following `Tree` to a `String`

~~~ scala
val tree: Tree[String] =
  Node(Node(Leaf("To"), Leaf("iterate")),
       Node(Node(Leaf("is"), Leaf("human,")),
            Node(Leaf("to"), Node(Leaf("recurse"), Leaf("divine")))))
~~~

Remeber you can append `String`s using the `+` method.

<div class="solution">
Note it is necessary to instantiate the generic type variable for `fold`. Type inference fails in this case.

~~~ scala
tree.fold[String]((a, b) => a + " " + b, str => str)
~~~
</div>