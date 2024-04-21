+++
title = "Resources Demystified"
date = "2024-04-20T20:22:13+01:00"
draft = false
description = "What is a java resource and how to work with resources"

tags = []
+++

I wanted to add a default configuration file to a Java project I was working on. This file would
work as a fallback, in case the user did not provide their own configuration file.

For the fallback file to be bundled into the final jar, I added the file to the `resources` folder
of the project. The project looked like this:

```
src/
  main/
    java/
      com/github/lpedrosa/myapp
        Application.java
    resources/
      application.properties
```

This setup allow us to access the file as a resource in Java code:

```java
InputStream resource = Application.class.getResourceAsStream("application.properties");

var properties = new Properties();
properties.load(resource);
```

Pretty cool huh? And when you run the application:

```
Exception in thread "main" java.lang.NullPointerException: inStream parameter is null
        at java.base/java.util.Objects.requireNonNull(Objects.java:259)
        at java.base/java.util.Properties.load(Properties.java:408)
        at com.github.lpedrosa.myapp.Application.main(Application.java:12)
```

Wait a second...

The file is in the `resources` folder, so I should be able to access it. I mean, isn't that what
the `getResourceAsStream` method is supposed to do?

Let's look at [what the javadocs say](<https://docs.oracle.com/en/java/javase/22/docs/api/java.base/java/lang/Class.html#getResourceAsStream(java.lang.String)>):

> **Returns:**
>
> A InputStream object; null if no resource with this name is found (...)

That should explain the `NullPointerException` on the `Properties#load(InputStream)` method. But
what else am I doing wrong?

## What is a resource?

According to Oracle's technotes[^1], a resource is _data (images, audio, text, and so on) that a
program needs to access in a way that is independent of the location of the program code_.

The `application.properties` file we tried to load is a text file, so it should be fine.

The previous definition also mentions something about the _location of a resource_ and the strategy
behind _accessing a resource_. Let's look at that folder structure again:

```
src/
  main/
    resources/
      application.properties
```

This folder structure respects Maven's [Standard Directory Layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html),
which the two biggest java build tools (Maven and Gradle) respect and use, in order to build java
projects.

If we use maven to build our example project, it creates a `target` folder that looks like this:

```
target/
  classes/
    com/github/lpedrosa/myapp
      Application.class
    application.properties
```

It looks like our `application.properties` file is at the _root_ of the `classes` folder. Let's look
at how we were retrieving the resource again:

```java
Application.class.getResourceAsStream("application.properties");
```

The way `Class#getResourceAsStream` tries to find the resource by name is a bit complex. In our
example, this method will try to find a resource in either the module path or the class path of
where the `Application` class is located:

- our `Application` class is in the `com.github.lpedrosa.myapp` package
- so the name given will resolve to `com/github/lpedrosa/myapp/application.properties`

So how do I access a file at the root of the `classes` folder?

```java
// note the slash (/) before the name
Application.class.getResourceAsStream("/application.properties");
```

If we run the application now, everything should work fine.

## So a resource is just like a file?

While it may look like accessing a resource with the `Class#getResourceAsStream` method obeys the
same rules as traditional filesystem access, there is a catch.

Once our project is packaged as a jar, **the resources contained inside become read-only**[^2].

If you want to change the resources inside a jar you can either:

- change the project files and re-package your project;
- use a tool like `unzip` or `jar` to extract the resources as files

## Can I use File I/O to read a resource?

No. Use the `getResourceAsStream` to obtain an `InputStream` and use the `Reader` family to read the
contents of the resource.

For example, if you wanted to print all the lines of the `application.properties` file:

```java
InputStream resource = Application.class.getResourceAsStream("application.properties");
try (var reader = new BufferedReader(new InputStreamReader(resource))) {
  // dump the file on stdout
  reader.lines().forEach(System.out::println);
} catch(IOException e) {
  // handle errors
};
```

## Conclusion

- A java resource is read-only
- The resource's location inside the JAR or classpath is important in order to access it
- You should not use File I/O to read a resource

[^1]: https://docs.oracle.com/javase/8/docs/technotes/guides/lang/resources.html
[^2]:
    [Check the posts by Tim Holloway](https://coderanch.com/t/734998/java/Resources-jar-file-practices),
    which clarify how a resource is different from a file
