---
title: "Exploring the power of implicits in Scala!"
date: 2023-08-19
tags: ['scala', 'implicits']
draft: false
summary: >
    Unlock the full power of Scala with a deep dive into type classes and implicits. Learn how to write clean, modular, and extensible code by abstracting behavior without touching existing types. This guide walks through practical patterns and examples
---

## Intro


Have you ever wondered how something like this compiles in Scala?



```
val aPair = "Sugar" -> 25
// or maybe something even more absurd like
val check = 2.isEven

```

we already know that operators in Scala are *methods* in which the first operand is the instance whose method is called and the second operand is passed to it, let's look at an example



```
val sum = 5 + 6
// this is identical to writing it as using the + method inside the Int class
// with 6 as an argument
val sum2 = 5.+(6)

```

you might be wondering now how the first code block compiles even though neither the String class nor the Int class has a method `->` and `isEven` respectively, and that's where the power of implicits shines.


First, we will explain what exactly are implicits in Scala then we will explore the above code and some use cases.


## What exactly are implicits?


When we talk about implicits in Scala we are mainly dealing with two things


* Implicit parameters


* Implicit conversions




Both of these involve the compiler *implicitly* (invisibly) resolving some *type errors* using some extra information that we have provided, for *implicit parameters* that would be when we are calling a method without supplying all the parameters, and for *implicit conversions* when we are passing something of a different type than the method accepts.


### Implicit parameters


These are method parameters that we don't need to explicitly pass to the method when we are calling it, if they are missing the compiler will look in the surrounding scope for some info that can help fix this problem, let's look at an example



```
def add(x: Int)(implicit y: Int) = x + y
println(add(5)(6))    // prints 11 to the console
println(add(5))       // doesn't compile

implicit val z: Int = 6
println(add(5))       // prints 11 to the console

implicit val k: Int = 10
println(add(5))       // doesn't compile

```

let's break down what's happening here, we declared an add method which takes two parameters and returns their sum, but note that we are declaring the second parameter *implicitly* using the **implicit** keyword, which means that we can either explicitly supply it when calling the method, or just omit it, the compiler then would do a lookup in the current scope, searching for a value with the keyword "implicit" and an expected type of Int.


There is something to note here, the compiler matches the implicit parameters with an implicit value according to its ***type*** and will only supply if it finds exactly ***ONE*** value in scope that matches, that's why we get a compilation error when we declared another `implicit val k`, the compiler is now confused as to which to provide for the add method because both of them declared with *implicit* keyword and match the type of Int.


### Implicit conversions


Besides supplying implicit parameters, the compiler can implicitly also do some type conversions for us, whenever there is a type mismatch the compiler will try to resolve it using some extra information that we have provided, let's look at an example



```
class Person(val name: String) {
    def greet: String = s"Hello my name is $name!"
}

def StringToPerson(name: String): Person = new Person(name)

val person = new Person("Jack")
println(person.greet)    // prints Hello my name is Jack! to the console
println(StringToPerson("Jack").greet) // works fine

```

Let's break this out, we defined a class Person which has a name, a method called greet and a method that given a name returns a new Person instance.


Creating a new Person instance and then calling the greet method on it works fine, also calling StringToPerson(name).greet works as well since the method returns a Person instance which has a greet method, but wouldn't it be nice if we can just write something like `println("Jack".greet)`? It makes our code so much simpler and more expressive, so let's use the power of implicits!



```
// changing the declaration of StringToPerson
implicit def StringToPerson(name: String): Person = new Person(name)

println("Jack".greet)    // compiles!!!

```

So what happened here? We declared the StringToPerson function to be *implicit*, so when we are calling `"Jack".greet` the compiler doesn't give up, it looks for all implicit classes, objects, values and methods that can help turn this "Jack" string to *something* that has a `greet`method and it happens we have a class Person which has a greet method and a method that implicitly converts a String to a Person, so what happens behind the scenes is that the compiler rewrites our code to `println(StringToPerson("Jack").greet)`


Note that there has to be exactly ***ONE*** method that matches the conversion, or else the compiler will get confused and throw us an error!


Now we are ready to explore the very first code block we had at the beginning if we take a close look at the [***Predef object***](https://scala-lang.org/api/3.x/scala/Predef$.html) which provides definitions that are accessible in all Scala compilation units without explicit qualification, we will find this weird class



```
final implicit class ArrowAssoc[A](self: A) extends AnyVal {
    def ->[B](y: B): (A, B)
}

```

here we have a class that has a method `->` *which takes an argument of type B and returns a tuple of* `(A, B)`*.*


so writing something like `val aPair = "Sugar" -> 25` is valid, as the compiler looks up all implicit classes, objects and vals, and finds that there is an implicit class called `ArrowAssoc` which has a method `->` that takes two parameters and returns a tuple, so the compiler creates a new ArrowAssoc instance from "Sugar" and call the `->` method on it using the second operand "25" and returns a tuple of `("Sugar", 25)`.


I will leave the example `val check = 2.isEven` as an exercise for you, you can follow the same pattern we did above with `StringToPerson` method, this kind of behavior is called ***type enrichment*** when we want to add some new functionalities to existing classes that we have no access to, like adding isEven functionality to the Int class.


So far we have discussed the ability to either implicitly supply a method with some parameters or implicitly convert from types to some other types. The combination of these is called *type classes*.


## Type classes


A type class specifies a set of operations that can be applied to a given type, in Scala it means a `trait` that has at least one type variable, and a functionality that we want to add (ie. methods). Any class extending this trait should supply definitions for its methods.


### The need for type classes


Imagine we want to add new functionalities to the Person class we defined before along with a new Class called Cat, and let's assume that they are from a library whose source we don't have access to so we can't modify them. This would require us to pass a Person instance to some other method/class and add that functionality, does this ring a bell? Yes!, it's like the adapter pattern.



```
// we don't have access to this, thus we can't modify it
class Person(val name: String) {
    def greet: String = s"Hello my name is $name!"
}
class Cat(val name: String)
//

object PersonCanFly {
    def canFly(person: Person): String = s"Person ${person.name} can fly!"
}

object CatCanFly {
    def canFly(cat: Cat): String = s"Meow, ${cat.name} can fly!"    
}

val person: Person = new Person("Jack")
val cat: Cat = new Cat("fluffly")

println(PersonCanFlu.canFly(person))
println(CatCcanFly.canFly(cat))

```

This allows us to add Fly as new functionality to both Person and Cat this looks fine right? well, It isn't. It makes our Person and Cat fly, but nowhere is it captured that we are bestowing *common functionality* on different types in the way that this would be encoded in the **type system** if we make both Person and Cat extend some trait, we won't be able to define a method that can fly and those who can't.


### Solution using Type classes



```
// type class
trait CanFly[A] {
    def fly(x: A): String
}

```

This is called a **type class** and it defines a set of types instead of just one type (**A** could be anything whether its an Int, String or even a user-defined Class) we can extend this trait to anything that we want to add the fly functionality to by creating what we call **type class instances**, and these instances contain the actual implementation for our **different types**



```
// type class instances
object PersonCanFly extends CanFly[Person] {
    def fly(person: Person): String = s"Person ${person.name} can fly!!"
}

object CatCanFly extends CanFly[Cat] {
    def fly(cat: Cat): String = s"meow, cat ${cat.name} can fly!"
}

```

now that chat functionality is common to only a ***range of types (the ones we defined only)***, we can define a method that accepts *anything* that flies



```
object FlyUtility {
    def fly[A](x: A, canFly: CanFly[A]): String = canFly.fly(x)
}

val cat = new Cat("fluffy")
println(FlyUtility.fly(cat, CatCanFly))

```

We can even add more implementation to a **specific** type only, imagine some cats can fly higher than other cats, then we can do something like this and call it with the same FlyUtility object



```
object CatCanFlyHigher extends CanFly[Cat] = {
    def fly(x: Cat): String = s"meow, cat ${cat.name} can fly higher!"
}

val anotherCat = new Cat("garfield")
println(FlyUtility.fly(anotherCat, CatCanFlyHigher)

```

So far so good! but we can improve this a bit, if you notice in the above code we are always passing an object instance to the `FlyUtility` which contains the actual implementation. In some sense, this feels repetitious of the information we already have, we have a cat and we know we want it to fly why do we need to pass in the **type class instance** **every time**? on the other hand, we may need to specify if we want our cat to *just fly* or *fly higher.* Wouldn't it be nice if we can just say `println(FlyUtility.fly(cat))`?


Let's solve this by modifying our FlyUtility object and our **type class instances**



```
implicit object CatCanFly extends CanFly[Cat] {
    def fly(cat: Cat): String = s"meow, cat ${cat.name} can fly!"
}

object CatCanFlyHigher extends CanFly[Cat] {
    def fly(cat: Cat): String = s"meow, cat ${cat.name} can fly higher!"
}

implicit object PersonCanFly extends CanFly[Person] {
    def fly(person: Person): String = s"Person ${person.name} can fly!!"
}

object FlyUtility {
    def fly[A](x: A)(implicit canFly: CanFly[A]): String = canFly.fly(x)
}

println(FlyUtility.fly(cat))    // this works now!!

```

By making the CanFly[A] argument implicit, the compiler will look for something in our scope that is marked implicit and matches the type expected by the method, and because of the *type parameter* ***[A]***, *the matching type will be whatever the type of the first parameter* *is*, so when we call `println(FlyUtility.fly(cat))` the compiler looks for an implicit match with type Cat, calling `println(FlyUtility.fly(person))` matches with the PersonCanFly instead.  
Note that I left the CatCanFlyHigher as it is without using implicit, because then the compiler would throw us an error. Because both objects would be a suitable match and the compiler won't know which one to use.


This is looking great! but as always we can do better, let's make our FlyUtility object implicit and see what happens



```
implicit class FlyUtility[A](x:A) {
    def fly(implicit canFly: CanFly[A]) = canFly.fly(x)
}

// now we can do something like this
val cat = new Cat("Garfield")
val person = new Person("Jack")
println(cat.fly)
println(person.fly)

```

As you can see this is so much simpler and more expressive than what we had at the beginning, since `FlyUtility` defines a type that can fly, and our classes Person and Cat define a type that we would like to be able to fly, then by making our object implicit we can automatically convert our `cat` and `person` instances to a `FlyUtility` instance by calling the fly methods on `cat` and `person`.


## Pitfalls


The most common pitfall is when the type for matching an implicit is too general, let's look at a simple example



```
implicit convertToInt(string: String): Int = string.toInt

def incOne(x: Int): Int = x + 1

// this compiles, but obviously gives a runtime error
println(incOne("haha")

```

As we can see if we use an implicit parameter with a built-in type as we did in this example is not a good practice, because the odds of accidently importing an explicit value of type Int are high, our code compiles but has unexpected behaviour.


Mainly using implicits with type classes (***type enrichment***) exploits their usefulness while avoiding the pitfalls, as we saw before the implicit parameters used in the previous section (type classes example) were the parameters that we defined ourselves like `CatCanFly extends CanFly[Cat]` instead of any arbitrary existing types, it only works for Person and Cats that we defined, this is type-safe, if we try to use it with some other type like Int the compiler then would give us an error that it can't find a matching **type class instance** that works for Int (because simply we haven't defined one).


## Conclusion


Conceptually, implicits are not too hard to understand, but their interaction with Scala's sophisticated type system can be hard to grasp for a novice, especially with large or complex systems.


Although they are very powerful they can do as much harm as good, unconstrained use of implicits can make the code confusing to read and hard to debug.


