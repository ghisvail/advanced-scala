## Example: *Eq*

We will finish off this chapter by looking at another useful type class:
[`cats.Eq`][cats.kernel.Eq].

### Equality, Liberty, and Fraternity

We can use `Eq` to define type-safe equality
between instances of any given type:

```scala
package cats

trait Eq[A] {
  def eqv(a: A, b: A): Boolean
  // other concrete methods based on eqv...
}
```

The interface syntax, defined in [`cats.syntax.eq`][cats.syntax.eq],
provides two methods for performing type-safe equality checks
provided there is an instance `Eq[A]` in scope:

 - `===` compares two objects for equality;
 - `=!=` compares two objects for inequality.

### Comparing *Ints*

Let's look at a few examples. First we import the type class:

```tut:book:silent
import cats.Eq
```

Now let's grab an instance for `Int`:

```tut:book:silent
import cats.instances.int._

val eqInt = Eq[Int]
```

We can use `eqInt` directly to test for equality:

```tut:book
eqInt.eqv(123, 123)
eqInt.eqv(123, 234)
```

Unlike Scala's `==` method,
if we try to compare objects of different types using `eqv`
we get a compile error:

```tut:book:fail
eqInt.eqv(123, "234")
```

We can also import the interface syntax in [`cats.syntax.eq`][cats.syntax.eq]
to use the `===` and `=!=` methods:

```tut:book:silent
import cats.syntax.eq._
```

```tut:book
123 === 123
123 =!= 234
```

### Comparing *Options*

Now for a more interesting example---`Option[Int]`.
To compare values of type `Option[Int]`
we need to import instances of `Eq` for `Option` as well as `Int`:

```tut:book:silent
import cats.instances.int._
import cats.instances.option._
```

Now we can try some comparisons:

```tut:fail:book
Some(1) === None
```

We have received a compile error here
because the `Eq` type class is invariant.
The instances we have in scope
are for `Int` and `Option[Int]`, not `Some[Int]`.
To fix the issue we have to re-type the arguments as `Option[Int]`:

```tut:book
(Some(1) : Option[Int]) === (None : Option[Int])
```

We can do this in a friendlier fashion using
the `Option.apply` and `Option.empty` methods from the standard library:

```tut:book
Option(1) === Option.empty[Int]
```

or using special syntax from [`cats.syntax.option`][cats.syntax.option]:

```tut:book:silent
import cats.syntax.option._
```

```tut:book
1.some === None
1.some =!= None
```

### Comparing Custom Types

We can define our own instances of `Eq` using the `Eq.instance` method,
which accepts a function of type `(A, A) => Boolean` and returns an `Eq[A]`:

```tut:book:silent
import java.util.Date
import cats.instances.long._
```

```tut:book:silent
implicit val dateEqual = Eq.instance[Date] { (date1, date2) =>
  date1.getTime === date2.getTime
}
```

```tut:book:silent
val x = new Date() // now
val y = new Date() // a bit later than now
```

```tut:book
x === x
x === y
```

### Exercise: Equality, Liberty, and Felinity

Implement an instance of `Eq` for our running `Cat` example:

```tut:book:silent
final case class Cat(name: String, age: Int, color: String)
```

Use this to compare the following pairs of objects for equality and inequality:

```tut:book:silent
val cat1 = Cat("Garfield",   35, "orange and black")
val cat2 = Cat("Heathcliff", 30, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]
```

<div class="solution">
First we need our Cats imports.
In this exercise we'll be using the `Eq` type class
and the `Eq` interface syntax.
We'll bring instances of `Eq` into scope as we need them below:

```tut:book:silent
import cats.Eq
import cats.syntax.eq._
```

Our `Cat` class is the same as ever:

```scala
final case class Cat(name: String, age: Int, color: String)
```

We bring the `Eq` instances for `Int` and `String`
into scope for the implementation of `Eq[Cat]`:

```tut:book:silent
implicit val catEqual = Eq.instance[Cat] { (cat1, cat2) =>
  import cats.instances.int._
  import cats.instances.string._

  (cat1.name  === cat2.name ) &&
  (cat1.age   === cat2.age  ) &&
  (cat1.color === cat2.color)
}
```

Finally, we test things out in a sample application:

```tut:book
val cat1 = Cat("Garfield",   35, "orange and black")
val cat2 = Cat("Heathcliff", 30, "orange and black")

cat1 === cat2
cat1 =!= cat2
```

```tut:book:silent
import cats.instances.option._
```

```tut:book
val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]

optionCat1 === optionCat2
optionCat1 =!= optionCat2
```
</div>

### Take Home Points

In this section we introduced
a new type class---[`cats.Eq`][cats.kernel.Eq]---that lets us
perform type-safe equality checks:

 - we create an instance `Eq[A]` to
   implement equality-testing functionality for `A`.

 - [`cats.syntax.eq`][cats.syntax.eq] provides two methods of interest:
   `===` for testing equality and `=!=` for testing inequality.

Because `Eq[A]` is invariant in `A`,
we have to be precise about the types of the values we use as arguments.
We sometimes need to manually type expressions in our code
to help the compiler locate the correct type class instances.
