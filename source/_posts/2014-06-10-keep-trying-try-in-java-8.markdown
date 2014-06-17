---
layout: post
title: "Keep trying! Try in Java 8"
date: 2014-06-10 16:33:04 +0100
comments: false
published: false
categories: [Java, Functional Programming]
---

Java 8 brought us some features that definitely improved the life of a Java developer. The inclusion of _lambdas_ (anonymous functions), as well as the _Stream_ API, allowed us to manage collection iteration in a much more functional way. 

Also, by adding `java.util.concurrent.CompletableFuture`, as well as some other concurrency API goodies, the developer is able to express concurrent tasks in a much more comprehensive way.

Overall, I think that Java 8 gives us the ability to express our intentions in a much more concise and readable way. This will also have a positive effect on code maintenance, which can translate in a lower barrier of entry to new codebases (e.g. when you start a new job, _dive_ into a open source project) and less boilerplate.

## Error handling and _Optional_

Java's way of error handling is through throwing and catching exceptions. These exceptions should only be thrown when an exceptional condition happens (e.g. illegal parameter passed into method, error parsing string, etc.).

At the time this was an huge improvement over _C_ code error handling, which usually consists in keeping track of a bunch of error constants. Unlike exceptions, which are simple Java objects, they are not able to hold any extra information regarding the error they represent.

You would normally handle exceptions by wrapping the code with a try-catch block such as:
```java
try {
    computationThatMightFail();
} catch (Exception e) {
    // do something the exception
}
```
All the error handling is deferred to the catch block. This all sounds pretty simple, until you start getting multiple try-catch blocks in a row:
```java
String configOption = null;
try {
    configOption = fetchFromConfig();
} catch (ConfigurationException e) {
    System.out.println("Failed to fetch config option from config", e);
    System.out.println("Using default value for config option");
    configOption = someDefaultValue;
}

Connection conn = null;
try {
    conn = openConnection(configOption);
} catch (IOException e) {
    System.out.println("Failed to open connection!", e);
}

String result = null;
try {
    conn.doSomething();
} catch(IOException e) {
    System.out.println("Something failed while doing something!", e);
}

// etc...
```
As you can see, your brain has to filter the try-catch blocks out in order to figure out what is really happening.

You might encounter a similar situation when handling computations that might return null, in order to avoid a `NullPointerException`:
```java
Person person = getPersonFromDB();
if (person != null) {
    Address address = p.getAddress();
    if (address != null) {
        String streetName = a.getStreetName();
        if (streetName != null) {
            System.out.println(streetName);
        } else {
            System.out.println("Street name unknown");
        }
    }
};
```
Again, your brain has to perform the mental exercise of navigating through the nested if chain, in order to understand what is really happening. This is all due these method calls returning null when a value is not present.

Fortunately, Java 8 introduces the `java.util.Optional` class. Instances of this class represent the presence or absence of a certain value. In pratice, you are basically "lifting" this value absence onto the type system. This enables your compiler to statically check if you are handling this absence properly or not.

An instance of the `Optional` class can be either empty or it can contain a reference to the value. For example:
```java
Optional<String> str = Optional.of("Some value"); // contains a value
Optional<String> empty = Optional.empty();        // does not contain a value
```
`Optional` also allows you to wrap a value that might be null through the static method `Optional.ofNullable()`. This allows, combined with `Optional`'s ability to support high-order functions (e.g. map, flatMap, filter), to express the above chain in the following manner:
```java
Optional<Person> person = Optional.ofNullable(getPersonFromDB());

String streetName = person.map(person -> person.getAddress())
                          .map(address -> address.getStreetName())
                          .orElse("Street name unknown");

System.out.println(streetName);
```
Optional's map implementation will short-circuit if the value you are mapping on is equal to `Optional.empty()`. This means the mapping function will not be applied to the value and the empty instance will propagate through the chain. Nevertheless, the code becomes a little bit more readable and puts less strain on your brain when trying to understand it.

Now, I am pretty sure this as crossed someone's mind at least once. What if we had a similar class to Optional, but instead wraps exceptions rather than null values. That could be insteresting...

## Try

Turns out the guys at Twitter had that exact same thought. They came up with this class called `Try` that holds the value of a computation or a failure (i.e. exception). It even made it into the [scala's API](http://www.scala-lang.org/api/current/#scala.util.Try "Scala's Try API docs"). One of the main reason's for the existence of this class was that this enabled you to pass computation results in between threads, even if they resulted in an error.
Like Optional it also "lifts" error handling onto to the type system. It warns the client of a specific method that it's result might wrap an error and it needs to handle this error accordingly.

But wait, isn't that what you do when you add `throws Exception` on a method's signature? Well yes, but this is an object rather then a language construct. Much like `Optional`, `Try` allows you to chain computations together, short-circuiting on the first error and propagating through the wrapped result.

However, Java 8 does not come with this `Try` class, so I decided to create one. Unlike scala's `Try`, this one does not use case subclassing internally (i.e. `Success`, `Failure`). I felt it wasn't needed since Java 8 does not support pattern matching (I believe the same reasoning was behind the implementation of `java.util.Optional`, whereas in `scala.Option` you have a `Some` and `None` case classes).

I tried to stay faithful to both the original scala API and the Java 8 Optional API. You guys can check out the code [here](https://github.com/lpedrosa/try "Java 8 Try implementation").

Much like `java.util.Optional`, this implementation provides the static method [Try.of()](https://github.com/lpedrosa/try/blob/master/src/main/java/com/lpedrosa/util/Try.java#L54 "Try.of method implementation") which allows you to wrap a value of a certain computation with a Try. For example:
```java
String supposedInt = "A";

Try<Integer> parsedInt = Try.of(() -> Integer.parseInt(supposedInt));
// parseInt now holds a NumberFormatException
```
Once you have the wrapped computation, you start chaining it with other computations (which may also fail of course!). `Try` allows you to perform high-order functions like `map`, `flatMap`, `filter` which will short-circuit either:

* when the `Try` value is already a failure or;
* when the computation you are trying to apply fails (e.g. due to an exception).

Once a failure occurs, it will be propagated through the chain, much like in an `Optional`. Here is an example of these features in action:
```java
// assume it fails with a PersonNotFoundException
Try<Person> person = Try.of(() -> PersonDB.getById(5)); 

Try<Address> address = person.map(person -> person.getAddress());
// address will still contain the PersonNotFoundException
```
### Error recovery
The `Try` class provides you with a couple of ways to recover from an error contained in it. Like `Optional`, you can use `orElse` to provide a default value. You can also use `orElseGet` when you want to call an alternative method instead (bearing in mind that this might also fail). For example:
```java
final long personID = 42;

// Tries to get the person from the cache, otherwise try to
// get it from the DB
Try<Person> person = Try.of(() -> PersonCache.get(personID))
                        .orElseGet(() -> PersonDB.get(personID));
```
Both methods, `orElse` and `orElseGet` disregard the type of the underlying exception. If you desire to act upon a specific exception then you can use `recover` or `recoverWith`.

These methods are similar to `map` and `flatMap` but they provide the underlying function with the current exception, so you can act upon a specific case. For example:
```java
Try<Integer> parsedInt = Try.of(() -> Integer.parseInt("A"))
                            .recover((throwable) -> {
                                if (NumberFormatException.class.isAssignableFrom(t.getClass()) {
                                    // do something
                                }
                            });
```
However this is not very readable (unfortunately Java does not have pattern matching, otherwise this would be a bit nicer). You can also re-wrap the exception using `recoverWith`:
```java
Try.of(() -> Integer.parseInt("A"))
   .recoverWith(t -> Try.failure(new MyException("wrapped another exception", t)));
```
---

In sum, `Try` allows you to wrap computations that might fail, so you can chain these with other computations, in a safe manner. It also forces you to deal with failure, since to get the underlying value you must unwrap the `Try`, which might result in a exception.

There are some downsides of using `Try`. Whenever you return a `Try` instance from a public method, you are forcing the method's client to deal with the error. This might not be what you want, especially if you were not a fan of checked exceptions.

Another downside is that dealing with `Try`'s might be a bit cumbersome for some, since Java does not provide a monadic comprehension (like scala's for comprehension and haskell's do).

It is not contained in the standard library, unlike `Optional`. This might not be a problem for some, but it is still a minor inconvenience.

Again, you can check the implementation of `Try` [here](https://github.com/lpedrosa/try "Java 8 Try implementation").

### Sources
- http://danielwestheide.com/blog/2012/12/26/the-neophytes-guide-to-scala-part-6-error-handling-with-try.html
- http://tersesystems.com/2012/12/27/error-handling-in-scala/
<!--
## Structure
- Intro to Try. Briefly mention Optional.
- Optional class review
- In comes Try (purpose, behaviour)
- Automatic error wrapping
- Function composition through high order functions
- Error recovery
- Contrast with CompletableFuture
- Contrast with Optional
- Conclusion

## Notes about Try in other blogs
> Optional<T> is a container for a value of type T that may be present or not
> Analogously, Try<T> is a container for a computation that may result in T or in a Throwable if something goes wrong
> We should return Try<T> from a method when we know something might fail and we want a client to handle that possibility in some way. (Pretty similar to throws IMO). Downside of this, the method will like a method with a checked exception
> Allows chaining operations (map and flatMap, so called high-order methods)
> Allows specifying defaults when you encounter an error case

### map, flatMap, filter
high order functions that allow chaining operations of a try

### recover, recoverWith
allow you to recover from a specific exception, similar to a catch block. With recoverWith you can even swap the thrown exception, e.g.

Try.of(() -> Integer.parseInt("A"))
   .recoverWith(t -> Try.failure(new MyException("wrapped another exception")));

### orElse, orElseGet
orElse allows you to provide a default value for the failed computation. orElseGet allows you to run another computation (which might fail) when the initial one failed.

**Reasons for using Try:**
> Exception handling is a bit ugly, and decreases readability of your code
> It can help in concurrency, if you wrap your exception in a value that you can pass around easly between threads.

## Cons 
- If you don't like checked exceptions, or don't agree with them you won't like this class. If you return it from a method, you are forcing the client to deal with it.
- Since java does not have a monadic comprehension (e.g. scala's for, haskell do) it might not look as clean as one would think.
- It is not in the standard library.

### Sources:
- http://danielwestheide.com/blog/2012/12/26/the-neophytes-guide-to-scala-part-6-error-handling-with-try.html
- http://tersesystems.com/2012/12/27/error-handling-in-scala/
-->
