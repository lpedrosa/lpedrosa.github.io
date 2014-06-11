---
layout: post
title: "Keep trying! Try in Java 8"
date: 2014-06-10 16:33:04 +0100
comments: false
published: false
categories: [Java, Functional Programming]
---

!!!! I AM MISSING filter method in Try

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
