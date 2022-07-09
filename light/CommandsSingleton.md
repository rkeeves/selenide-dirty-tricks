# Where is the behavior?

SelenideElementProxy itself doesn't know that a `click` or `should` is.

So who knows these things?

## Overview

If you call a method, the invocation handler gets invoked.

The invocation handler contacts a synchronized Singleton.

## Singleton for only one thread

First we have to familiarize ourselves with GoF pattern [Singleton](https://refactoring.guru/design-patterns/singleton).

Singleton has one main purpose: guarantee that outsiders can only interact with one instance of the class.

A simple Singleton could look like this:

```java
class Singleton {

    private static Singleton INSTANCE = new Singleton();

    private Singleton() {  }

    public static getInstance() {
      if (INSTANCE == null) {
        INSTANCE = new Singleton();
      }
      return INSTANCE;
    }
}
```

This `Singleton` is fine for basic **single thread** tasks.

## Singleton for many threads

The `Singleton` shown above has a problem.

Imagine 4 threads all going through `getInstance` at the same time. One thread sees that `INSTANCE` is null, so begins instantiation. Meanwhile an other still sees that `INSTANCE` is null, so it also begins instantiation.

In a multithreaded environment a more intricate access control is required.



```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {  }

    public static synchronized Singleton getInstance() {
        if (instance == null) {
        instance = new Singleton();
        }
        return instance;
    }
}
```

This way if one thread is in the process of acquiring the `Singleton` it basically blocks the other threads from doing so, until it finishes this block of code.

## Selenide and the Commands Singleton

Imagine we wrote something like this:

```java
$("#foo").setValue("a");
```

We already know that this will end up in an Invocation Handler's `invoke`. But what happens then?

```java
class SelenideElementProxy {

    public Object invoke(Object proxy,
                        Method method,
                        Object... args)
                        throws Throwable {
      return ???
    }
}
```

`SelenideElementProxy` does not know what to do, so it calls someone who knows.

In this case it calls a `getInstance` static method of a class called `Commands`.

```java
class SelenideElementProxy {

    public Object invoke(Object proxy,
                        Method method,
                        @Nullable Object... args)
                        throws Throwable {
      return Commands.getInstance()...
    }
}
```

From the looks of it we assume that this must be a `Singleton`.

To make sure, we check the `Commands` class:

```java
class Commands {

    private static Commands instance;

    public static synchronized Commands getInstance() {
        if (instance == null) {
            instance = ...instantiation;
        }
        return instance;
    }
}
```

We see that it is indeed a `Singleton`. Also, we can conclude that it was meant to be used in a multi threaded way.
