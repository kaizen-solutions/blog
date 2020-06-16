---
title:  "Understanding Covariance and Contravariance"
date:   2020-02-29 12:33:33 -0500
tags: ["scala", "variance", "development"]
categories: ["Variance"]
author: Calvin Lee Fernandes
---

## Introduction

There are at least 3 aspects to consider when dealing with contravariance and covariance in Scala when using container types.

```scala
def example(x: Container[T]): Container[T]
```
1. What can you feed in 
2. What can point to the result
3. What can the function return

It is important to remember:
* `-` means type itself and its subtypes
* `+` means type itself and supertypes

Given the following inheritance hierarchy:
```scala
sealed trait Animal
class Zebra() extends Animal
class Okapi() extends Zebra
```

Let's come up with some examples:
```scala
trait Foo[-T]
trait Foo2[+T]
```
Given that we have the above container types, here are some examples of using them:
```scala
def example(f: Foo[Zebra]): Foo[Zebra] = {
    new Foo[Animal] {}
}

val res: Foo[Okapi] = example(new Foo[Animal] {})
```
So here we are dealing with a contravariant type. 

1. Notice we have f in the method argument position. This means that the flip classification applies. We are dealing with a contravariant type parameter which means you can normally feed in subtypes and the type itself. However, the flip classification has affected this f meaning you can now only provide the type itself and supertypes. You can see that example expects a Zebra but we can actually provide its supertype Animal.

2. In terms of what can point to the function result, when Foo[Zebra] is in the result position, no flip classification occurs and T remains contravariant meaning only the type itself and more specific types can refer to this so a Foo[Okapi] can point to the result that is a Foo[Zebra]
	
3. Notice the function body returns a Foo[Animal] as it appears that the flip classification applies to this point as well. If you try to return a subtype of Foo[Zebra] it will fail (returning a Foo[Zebra] is fine however).

Let's look at the covariant case Foo2 now:
```scala
def example2(f: Foo2[Zebra]): Foo2[Zebra] = {
  println(f)
  new Foo2[Okapi] {}
}

val res2: Foo2[Animal] = example2(new Foo2[Okapi] {})
```

1. The flip classification applies to f. We are dealing with a covariant type parameter which means you can normally fill in the type itself and supertypes. However, the flip classification has affected this and as a result you can now only use the type itself and subtypes. So here we expect a Foo2[Zebra] but we can provide a subtype of Zebra so we can put in Foo2[Okapi].
	
2. In terms of what can point to the function result, when Foo2[Zebra] is in the result position, no flip classification occurs and T remains covariant meaning you can provide the type itself and supertypes. So an Animal which is a supertype of Zebra can point to this result so we can point a Foo2[Animal] to this result of Foo2[Zebra[
	
3. The function can return Zebra or subtypes of Zebra (Okapi) as it appears that the flip classification applies to this point as well (covariant means you can use the type itself and supertypes but the flip classification changes this to be the type itself and subtypes). If you try and return a supertype of Zebra, it will complain but returning a subtype is fine. 

## References

See [Covariance and contravariance in Scala](http://blog.kamkor.me/Covariance-And-Contravariance-In-Scala) for more information about the flip classification 