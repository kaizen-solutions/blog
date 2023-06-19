---
title:  "Typeclass derivation with Magnolia"
date:   2020-12-28 12:33:33 -0500
tags: ["scala2", "typeclass", "derivation", "magnolia", "development"]
categories: ["Typeclass"]
authors: ["Calvin Lee Fernandes"]
---

# Introduction

Typeclass derivation is a way to automatically generate typeclass instances for a typeclass provided some initial 
conditions are satisfied. It is a very useful mechanism which cuts down on a lot of boilerplate and makes it very 
enticing for users of your typeclass. We will be demonstrating how to do typeclass derivation with a library called
[Magnolia](https://propensive.com/opensource/magnolia) written by the amazing [Jon Pretty](https://propensive.com). Jon
has done an immense amount of good supporting the Scala community, so I highly suggest checking out his website, and the
other libraries he has written.

# What are typeclasses

Typeclasses are a construct used to implement ad-hoc polymorphism. This is achieved by adding a constraint/capability to
type variables (used in parametric polymorphism). Technically, typeclasses also come with a set of laws baked in but for
the purposes of this blog post, we will ignore this. If you want to learn more, I strongly suggest you to have a look at
[zio-prelude](https://github.com/zio/zio-prelude) and [discipline](https://typelevel.org/blog/2013/11/17/discipline.html). 
All of this sounds abstract, so let's have a look at an example:

```scala
trait Show[A] {
  def show(a: A): String
}
```

This is an example of a typeclass named `Show`. It is a typeclass for simple types (like `Int`, `String`, `List[A]`). 
There are also typeclasses for type-constructors (like `List`, `Option` - notice I said `List` and not `List[A]`) for 
example, `Monad` and `Functor` but that is a topic for another blog post. Back to `Show`, this typeclass adds a 
capability to render values of a type that implements this typeclass to be turned into `String`s. So how does one 
implement an instance for this typeclass? Let's have a look:

```scala
object Show {
  // typeclass instances
  implicit val showString: Show[String] = new Show[A] {
    def show(a: String): String = a
  }
  
  // single abstract method syntax to minimize boilerplate
  implicit val showInt: Show[Int] = (a: Int) => a.toString
  
  implicit val showLong: Show[Long] = (a: Long) => a.toString
  
  // ...
}
```

Here we have defined some typeclass instances for the types `String`, `Int` and `Long`. We have purposely placed these
typeclass instances in the companion object for `Show` so the implicit resolution mechanism that Scala 2 uses to emulate 
typeclasses will easily be able to find these instances without any additional imports. Here's how we can use this:

```scala
// context bound
def render[A: Show](a: A): String = implicitly[Show[A]].show(a)

// alternative
def render2[A](a: A)(implicit evidence: Show[A]): String = evidence.show(a)

// Usage
render(1L)
render2("Hello")
```

So we have an idea of the basics of what typeclasses are, how to define them, how to write instances for them and how to
use them. Now, let's look at using the typeclasses we have created from a user's perspective. This means that someone 
is using the typeclasses we have written and does not have the ability to modify the companion object of `Show` and 
wants to add the `show` ability to some of their own custom data types (for example, their own case classes (products) 
and sealed trait hierarchies (co-products)). Let's say the user wants to add the `Show` capability to the following 
data-types:

```scala
// co-product
sealed trait Country
object Country {
  case object Canada                   extends Country
  case object USA                      extends Country
  final case class Other(name: String) extends Country
}

// product
final case class User(
  firstName: String,
  lastName: String,
  age: Int
)
```

What they would go about doing is manually implementing `Show` instances for each of these data-types:

```scala
object Country {
  // ...
  
  implicit val showCountry: Show[Country] = {
    case Country.Canada      => s"Canada"
    case Country.USA         => s"USA"
    case Country.Other(name) => name
  }
}

object User {
  // ...
  implicit val showUser: Show[User] = { user => 
    s"User(firstName=${user.firstName}, lastName=${user.lastName}, age=${user.age}"
  }
}
```

This does not seem too bad, but it can get very repetitive. It also feels like a very mechanical process. You may think 
that this is not so bad, however, if you have multiple typeclasses that your data-types need to implement and multiple 
data-types then expect to be writing a lot of this repetitive (and error-prone) code. 

**What if you did not have to write any of this code by hand and instead let the compiler do it for you?** 
This is what typeclass derivation is: given a set of building blocks, the compiler (along with some help) can figure out
how to automatically derive/generate these typeclass instances for you. This is where [Magnolia](https://github.com/propensive/magnolia)
comes into play. Concretely, Magnolia is a macro that generates typeclass instances for datatypes composed from case 
classes (products) and sealed traits (coproducts). It supports recursively-defined datatypes out-of-the-box, and most 
importantly **incurs no significant time-penalty during compilation**. 

Note that there are other ways to do typeclass derivation in Scala (i.e. Shapeless) but alternative approaches require 
you to write a lot more low-level code or require a dramatic increase in compile-time. I personally feel that Magnolia 
is currently the best solution out there for Scala 2 when it comes to solving the typeclass derivation problem. 

So let's take a look at what is needed for typeclass derivation using Magnolia. Here is our typeclass from before with
some typeclass instances which will function as our building blocks:

```scala
trait Show[A] {
  def show(a: A): String
}

object Show {
  implicit val showString: Show[String] = identity[String]
  implicit val showInt: Show[Int] = (a: Int) => a.toString
  implicit val showLong: Show[Long] = (a: Long) => a.toString
}
```

Now let's talk about Magnolia. First off, if you are using `sbt`, you will need to add the following imports to your 
`build.sbt`:

```scala
libraryDependencies += "com.propensive" %% "magnolia" % "0.17.0" // at the time
libraryDependencies += "org.scala-lang" % "scala-reflect" % scalaVersion.value % Provided
```

To add support for typeclass derivation to your typeclass, Magnolia requires you to fill in the following pattern:
```scala
import magnolia._
import scala.language.experimental.macros

object YourTypeclassCompanionObject {
  type Typeclass[T] = ???

  // for products (case classes)
  def combine[T](caseClass: CaseClass[Typeclass, T]): Typeclass[T] = ???

  // for co-products (sealed trait hiearchies)
  def dispatch[T](sealedTrait: SealedTrait[Typeclass, T]): Typeclass[T] = ???

  implicit def gen[T]: Typeclass[T] = macro Magnolia.gen[T]
}
```

The Magnolia macro will use `combine` and `dispatch` to automatically derive typeclass instances for your case classes
and sealed traits so you don't have to write any typeclass instances for your data-types with a small caveat. Your case
classes and sealed traits must be comprised of the building blocks (i.e. the typeclass instances that you provided). In
the case of Show, we are allowed to build case classes consisting of `String`s, `Int`s and `Long`s but not `Boolean`s 
since we never bothered to implement a typeclass instance for those. In addition, you can have case classes or sealed trait
hierarchies consisting of other case classes and other sealed trait hierarchies but in the end the leaves must consist
of the types that you have provided typeclass instances for. As an example, Magnolia would happily derive the typeclass
instance for this sealed trait hierarchy:
```scala
sealed trait Country
object Country {
  final case object Canada             extends Country
  final case object USA                extends Country
  final case class Other(name: String) extends Country
}
```

Now, lets walk through the `CaseClass` and `SealedTrait` data-types and see how they will assist us in automatic 
derivation for `Show`:

## Automatically deriving case classes and case objects

```scala
// NOTE: The following are approximations taken from Magnolia sources to help assist in understanding the abstraction
trait Param[Typeclass[_], DataType] {
  /**
   *   For example, DataType = case class User(name: String, age: Int)
   *   ParameterType(s) would be name: String and age: Int
   */
  type ParameterType
  
  /** the name of the parameter */
  def label: String

  /** the position of the parameter (in the above example name would be index 0 and age would be index 1 */
  def index: Int

  /** tells you if the parameter is varargs */
  def repeated: Boolean

  /** typeclass associated with this particular parameter */
  def typeclass: Typeclass[ParameterType]

  /** 
   * Access the value of the parameter. For example:
   * given a User("Calvin", 29), if our ParameterType was name, then
   * deference(User("Calvin", 29)) would give us the value of name which is "Calvin"
   */
  def dereference(param: DataType): ParameterType

  // ...
}

// Tells you the name of the case class
final case class TypeName(owner: String, short: String /** ... */)

trait CaseClass[Typeclass[_], DataType] {
  def parameters: Seq[Param[Typeclass, DataType]]
  
  def typeName: TypeName
  
  def isObject: Boolean
  
  def isValueClass: Boolean

  // ...
}

```

As you can probably imagine, we will be iterating over each parameter in the case class and using a combination of 
`typeclass`, `dereference`, possibly using `label` and `typeName` along the way in order to help us achieve automatic 
derivation for any case class. Let's see an example of how to implement `combine` for `Show` which will provide us with 
automatic typeclass derivation for any case class.

```scala
import magnolia._
import scala.language.experimental.macros

object Show {
  type Typeclass[T] = Show[T]

  // for products (case classes)
  def combine[T](caseClass: CaseClass[Typeclass, T]): Typeclass[T] = { in: T =>
    if (caseClass.isObject) caseClass.typeName.short
    else caseClass.parameters.map { each =>
      val label        = each.label
      val showInstance = each.typeclass
      val paramValue   = each.dereference(in)
      val rendered     = showInstance.show(paramValue)
      s"$label=$rendered"
    }.mkString(start = s"${caseClass.typeName.short}(", sep = ", ", end = ")")
  }

  implicit def gen[T]: Typeclass[T] = macro Magnolia.gen[T]
  
  // ...
}
```

The code above, will be invoked if you ask for a `Show` instance for any case class. First we check if we have a case
class, or a case object and if we have a case object we just spit out the short name (`caseClass.typeName.short`) as the 
result. Otherwise, we are looking at a case class. In this case we iterate over every parameter of the case class and
extract the name of the parameter, the `Show` typeclass instance that is associated with each parameter and the value
of the parameter. We invoke the typeclass for each parameter on the parameter's value to get out a String. And we show
it as the parameter name = parameter rendered value. Finally we add the short name of the case class and wrap it all up.
So for example if you had a `case class User(name: String, age: Int)` and an instance like `User("bob", 30)` you can 
expect the output to be `"User(name=bob, age=30)`. Usage looks like this:

```scala
final case class User(
  firstName: String,
  lastName: String,
  age: Int
)

def render[A](a: A)(implicit evidence: Show[A]): String = evidence.show(a)

render(User("Bob", "Builder", 30))
// "User(firstName=Bob, lastName=Builder, age=30)"
```

## Automatically deriving sealed trait hierarchies

Now lets take a look at `SealedTrait` to see how it can help us do automatic typeclass derivation for sealed trait 
hierarchies:

```scala
// NOTE: The following are approximations taken from Magnolia sources to help assist in understanding the abstraction
trait SealedTrait[Typeclass[_], DataType] {
  def typeName: TypeName  // same as CaseClass
  
  /** A list of all the subtypes of the sealed trait hierarchy */
  def subTypes: Seq[Subtype[Typeclass, Type]]

  /** convenience method for delegating typeclass application to the typeclass corresponding to the
   *  subtype of the sealed trait which matches the type of the `value`
   *
   *  @tparam Return  the return type of the lambda, which should be inferred
   *  @param value   the instance of the generic type whose value should be used to match on a
   *                 particular subtype of the sealed trait
   *  @param handle  lambda for applying the value to the typeclass for the particular subtype which
   *                 matches
   *  @return  the result of applying the `handle` lambda to subtype of the sealed trait which
   *           matches the parameter `value` */
  def dispatch[Return](value: Type)(handle: Subtype[Typeclass, Type] => Return): Return

  // ...
}

trait Subtype[Typeclass[_], Type] {
  /** the type of subtype 
   * For example
   * sealed trait Country                            // this would be Type
   * object Country {
   *  case object Canada             extends Country // SType
   *  case object USA                extends Country // SType as well
   *  case class Other(name: String) extends Country // SType as well
   * }
   */
  type SType <: Type


  /** the [[TypeName]] of the subtype
   *
   *  This is the full name information for the type of subclass. */
  def typeName: TypeName

  def index: Int

  /** the typeclass instance associated with this subtype
   *
   *  This is the instance of the type `Typeclass[SType]` which will have been discovered by
   *  implicit search, or derived by Magnolia. */
  def typeclass: Typeclass[SType]

  /** partial function defined the subset of values of `Type` which have the type of this subtype */
  def cast: PartialFunction[Type, SType]
  
  // ...
}
```

The `SealedTrait` offers similar capabilities as `CaseClass` so you would use `typeclass` and `cast` to get 
the associated type class of the subtype, and the value of the subtype, and finally `dispatch` to bring it all together. 
Let's look at implementing `dispatch` for `Show`: 

```scala
object Show {
  // ...
  // derivation for sealed traits and their inhabitants (coproducts)
  def dispatch[A](sealedTrait: SealedTrait[Typeclass, A]): Typeclass[A] = { a: A =>
    sealedTrait.dispatch(a) { eachSubtype =>
      val showInstance = eachSubtype.typeclass
      val value = eachSubtype.cast(a)
      showInstance.show(value)
    }
  }
}
```

So we use `SealedTrait`s `dispatch` to iterate over all the subtypes of the sealed trait. For each subtype, we extract
the typeclass instance associated with the subtype, and the value associated with each subtype, and with this, we have 
all the pieces to render the subtype's value. This allows us to automatically derive `Show` typeclass instance for any 
sealed trait hierarchy:

```scala
sealed trait Country
object Country {
  case object Canada                   extends Country
  case object USA                      extends Country
  final case class Other(name: String) extends Country
}

def render[A](a: A)(implicit evidence: Show[A]): String = evidence.show(a)

render(Country.Canada)
// "Canada"

render(Country.Other("Germany"))
// "Other(name=Germany)"
```

# Conclusion

Putting everything together, we have the following typeclass definition along with the automatic derivation mechanism:

```scala
// Typeclass
trait Show[A] {
  def show(a: A): String
}

object Show {
  // typeclass instances
  implicit val showString: Show[String] = identity[String]
  implicit val showInt: Show[Int] = (a: Int) => a.toString
  implicit val showLong: Show[Long] = (a: Long) => a.toString
  
  // Magnolia
  type Typeclass[T] = Show[T]

  // for products (case classes)
  def combine[T](caseClass: CaseClass[Typeclass, T]): Typeclass[T] = { in: T =>
    if (caseClass.isObject) caseClass.typeName.short
    else
      caseClass.parameters.map { each =>
        val label        = each.label
        val showInstance = each.typeclass
        val paramValue   = each.dereference(in)
        val rendered     = showInstance.show(paramValue)
        s"$label=$rendered"
      }.mkString(start = s"${caseClass.typeName.short}(", sep = ", ", end = ")")
  }

  // for co-products (sealed trait hierarchy)
  def dispatch[T](sealedTrait: SealedTrait[Typeclass, T]): Typeclass[T] = { in: T =>
    sealedTrait.dispatch(in) { eachSubtype =>
      val showInstance = eachSubtype.typeclass
      val value        = eachSubtype.cast(in)
      showInstance.show(value)
    }
  }

  implicit def gen[T]: Typeclass[T] = macro Magnolia.gen[T]
}

object Usage {
  // Product
  final case class User(firstName: String, lastName: String, age: Int)

  // Coproduct
  sealed trait Country
  object Country {
    final case object Canada             extends Country
    final case object USA                extends Country
    final case class Other(name: String) extends Country
  }

  def render[A](a: A)(implicit evidence: Show[A]): String = evidence.show(a)

  val calvin: User = User("Calvin", "Fernandes", 29)
  val can: Country = Country.Canada

  println(render(calvin)) // User(firstName=Calvin, lastName=Fernandes, age=29)
  println(render(can))    // Canada
}
```

We have just shown the process of automatic derivation of a contravariant typeclass. Now, users of this library can
simply go ahead and start writing down their own custom types (provided they are backed by `String`, `Int` and `Long`)
and will now automatically be able to have Show instances.
