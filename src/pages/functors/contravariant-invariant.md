## *Contravariant* and *Invariant* Functors {#contravariant-invariant}

We can think of `map` as
"appending" a transformation to a chain.
We start with an `F[A]`,
run it through a function `A => B`,
and end up with an `F[B]`.
We can extend the chain further by mapping again:
run the `F[B]` through a function `B => C`
and end up with an `F[C]`:

```tut:book
Option(1).map(_ + 2).map(_ * 3).map(_ + 100)
```

We're now going to look at two other type classes,
one that represents *prepending* operations to a chain,
and one that represents building a *bidirectional*
chain of operations.

One great use case for these new type classes is
building libraries that transform, read, and write values.
The content ties in tightly to the [JSON codec](#json-codec)
case study later in the book.

### Contravariant functors and the *contramap* method {#contravariant}

The first of our type classes, the *contravariant functor*,
provides an operation called `contramap`
that represents "prepending" a transformation to a chain.
This is illustrated in Figure [@fig:functors:contramap-type-chart].

![Type chart: the contramap method](src/pages/functors/generic-contramap.pdf+svg){#fig:functors:contramap-type-chart}

We'll talk about `contramap` itself directly for now,
bringing in the type class in a moment.

The `contramap` method only makes sense for certain data types.
For example, we can't define `contramap` for an `Option`
because there is no way of feeding a value in an
`Option[B]` backwards through a function `A => B`.

`contramap` starts to make sense when we have a data types
that represent tranformations.
For example, consider the `Printable` type class
we discussed in [Chapter 2](#type-classes):

```tut:book:silent
trait Printable[A] {
  def format(value: A): String
}
```

A `Printable[A]` represents a transformation from `A` to `String`.
We can define a `contramap` method that
"prepends" a transformation from another type `B`:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String

  def contramap[B](func: B => A): Printable[B] =
    ???
}

def format[A](value: A)(implicit p: Printable[A]): String =
  p.format(value)
```

This says that if `A` is `Printable`, and we can transform `B` into `A`, then `B` is also `Printable`.

#### Exercise: Showing off with Contramap

Implement the `contramap` method for `Printable` above.

<div class="solution">
Here's a working implementation:

```tut:book:silent
trait Printable[A] {
  def format(value: A): String

  def contramap[B](func: B => A): Printable[B] = {
    val self = this
    new Printable[B] {
      def format(value: B): String =
        self.format(func(value))
    }
  }
}

def format[A](value: A)(implicit p: Printable[A]): String =
  p.format(value)
```
</div>

Let's define some basic instances of `Printable`
for `String` and `Boolean`:

```tut:book:silent
implicit val stringPrintable =
  new Printable[String] {
    def format(value: String): String =
      "\"" + value + "\""
  }

implicit val booleanPrintable =
  new Printable[Boolean] {
    def format(value: Boolean): String =
      if(value) "yes" else "no"
  }
```

```tut:book
format("hello")
format(true)
```

Define an instance of `Printable` that prints
the value from this case class:

```tut:book:silent
final case class Box[A](value: A)
```

Rather than writing out
the complete definition from scratch
(`new Printable[Box]` etc...),
create your instance using
the `contramap` method of one of the instances above.

<div class="solution">
To make the instance generic across all types of `Box`,
we base it on the `Printable` for the type inside the `Box`:

```tut:book:silent
implicit def boxPrintable[A](implicit p: Printable[A]) =
  p.contramap[Box[A]](_.value)
```
</div>

Your instance should work as follows:

```tut:book
format(Box("hello world"))
format(Box(true))
```

If we don't have a `Printable` for the contents of the `Box`,
calls to `format` should fail to compile:

```tut:book:fail
format(Box(123))
```

### Invariant functors and the *imap* method {#invariant}

The second of our type classes, the *invariant functor*,
provides a method called `imap` that is informally equivalent to
a combination of `map` and `contramap`.
We can demonstrate this by extending `Printable`
to produce a typeclass for encoding and decoding to a `String`:

```tut:book:silent
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): Option[A]

  def imap[B](dec: A => B, enc: B => A): Codec[B] =
    ???
}

def encode[A](value: A)(implicit c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(implicit c: Codec[A]): Option[A] =
  c.decode(value)
```

The type chart for `imap` is showin in Figure [@fig:functors:imap-type-chart].

![Type chart: the imap method](src/pages/functors/generic-imap.pdf+svg){#fig:functors:imap-type-chart}

#### Transformative Thinking with Imap

Implement the `imap` method for `Codec` above.

<div class="solution">
Here's a working implementation:

```tut:book:silent
trait Codec[A] {
  def encode(value: A): String
  def decode(value: String): Option[A]

  def imap[B](dec: A => B, enc: B => A): Codec[B] = {
    val self = this
    new Codec[B] {
      def encode(value: B): String =
        self.encode(enc(value))

      def decode(value: String): Option[B] =
        self.decode(value).map(dec)
    }
  }
}

def encode[A](value: A)(implicit c: Codec[A]): String =
  c.encode(value)

def decode[A](value: String)(implicit c: Codec[A]): Option[A] =
  c.decode(value)
```
</div>

Here's an example `Codec` representing parsing and serializing `Ints`:

<div class="solution">
```tut:book:silent
implicit val intCodec =
  new Codec[Int] {
    def encode(value: Int): String =
      value.toString

    def decode(value: String): Option[Int] =
      scala.util.Try(value.toInt).toOption
  }
```
</div>

Demonstrate your `imap` method works by creating a
`Codec` for conversions between `Strings` and `Boxes`:

```tut:book:silent
case class Box[A](value: A)
```

<div class="solution">
```tut:book:silent
implicit def boxCodec[A](implicit c: Codec[A]): Codec[Box[A]] =
  c.imap[Box[A]](Box(_), _.value)
```
</div>

Your instance should work as follows:

```tut:book
encode(Box(123))
decode[Box[Int]]("123")
```

### What's With the Name?

What's the relationship between contravariance, invariance, and covariance as we usually understand them in Scala,
and the names for the functors above?

The usual meaning of these terms in Scala relates to subtypes.
We say that `B` is a subtype of `A` if we can use `B` anywhere we want an `A`.
Put another way, we can convert `A` into `B` and our program keeps on working.

Co- and contravariance usually arises in Scala when working with type constructors like `List` and `Option`.
If we declare a type constructor `F`,
and we want `F[B]` to be a subtype of `F[A]` when `B` is a subtype of `A`,
we declare the type parameter to be covariant.

```tut:silent
trait F[+A] // A is covariant
```

If `B` is a subtype of `A`,
and we want `F[A]` to be a subtype of `F[B]`,
then we declare `F` to have a contravariant type parameter.

```tut:silent
trait F[-A] // A is contravariant
```

Co- and contravariant functors capture the same principle without the limitations of subtyping.
As we said above subtyping can be viewed as a conversion.
`B` is a subtype of `A` if we can convert `A` to `B`.
In other words there exists a function `A => B`.
A covariant functor, which is what the standard `Functor` is,
captures exactly this.
If `F` is a (covariant) functor,
whenever we have a `F[A]` and a conversion `A => B`
we have a `F[B]`.
A contravariant functor captures the case in the opposite direction.
If `F` is a contravariant functor,
whenever we have a `F[A]` and a conversion `B => A`
we have a `F[B]`.
