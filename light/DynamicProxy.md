# How can I have a SelenideElement if nothing implements it?

All the things that you do in tests (assertions, effects) are done using `SelenideElement`.

```java
SelenideElement e = Selenide.$("#foo");
```

But noone seems to implement `SelenideElement` interface...

What's happening?

## Overview

SelenideElement is an interface.

Selenide creates dynamic proxies at runtime and adds an invocation handler.

## Static Proxy

First let's talk about what the [GoF Proxy](https://refactoring.guru/design-patterns/proxy) pattern is.

Proxy is a structural design pattern.

Imagine a use case where you want to read a file. IO is quite costly, and in some cases you know that the file won't change at all. So it is okay to read it only once, and return this result on subsequent calls.

Imagine you have an interface like:

```java
interface IOResource {
    String read();
}

class ActualIOResource implements {

    String read() {
        return // costly;
    }
}

class CachedIOResource implements IOResource {

    IOResource ioResource;

    String cached;

    String read() {
        if (cached == null) {
            cached = ioResource.read();
        }
        return cached;
    }
}

psvm {
    IOResource ioResource = new CachedIOResource(
        new ActualIOResource()
    );

    ioResource.read();
}
```

The proxy `CachedIOResource` tells the outside world that it is an `IOResource` but in reality it calls the `IOResource` inside it.

In more generic terms though `Proxy` stands between you and the resource you want to interact with. `Proxy` impersonates this resource so you think that you are actually interacting with the resource itself. But in reality `Proxy` decides what to do with your messages.

I call it a **static** proxy though, because everything is known at compile time to ensure type safety.

## Dynamic Proxy

In contrary to **static** proxy, there is a thing called [Dynamic Proxy](https://docs.oracle.com/javase/8/docs/technotes/guides/reflection/proxy.html).

`Dynamic Proxy` is created at runtime, so type safety and type information is almost lost. You can still use Reflection to cheat around this fact.

`Dynamic Proxy` is created like so:

```java
IOResource ioResource = (IOResource) Proxy.newProxyInstance(
  SomeClass.class.getClassLoader(),
  new Class[] { IOResource.class },
  new IOResourceHandler());
```

You define that you want a dynamic proxy acting as a `IOResource`.
You also define that you want dynamic proxy to 'relay all messages' to `IOResourceHandler`.

`IOResourceHandler` must implement an interface `InvocationHandler` with a single method `invoke`:

```java
public class IOResourceHandler implements InvocationHandler {


  public Object invoke(Object proxy,
                        Method method,
                        Object[] args) throws Throwable {
        return // behavior;
    }
}
```

Notice the `Object` types and `Method` type?

You lost all compile time safety, because your **Dynamic Proxy** does NOT exist at compile time.

## Selenide and Dynamic Proxy

SelenideElement is just an interface, and you'll find no implementing class.

And this is where **Dynamic Proxy** comes in.

Imagine you do this:

```java
Selenide.$("#foo")
```

The `Selenide::$` does call `SelenideDriver::find`:

```java
 public static SelenideElement $(String cssSelector) {
    return getSelenideDriver().find(cssSelector);
  }
```

Which through a lot of same method calling through different overloads of itself eventually ends up calling  `SelenideDriver::find`'s one of many overloads:

```java
public SelenideElement find(By criteria) {
    return ElementFinder.wrap(driver(), null, criteria, 0);
}
```

`ElementFinder`'s static method `wrap` does instantiate the Dynamic Proxy:

```java
public static <T extends SelenideElement> T wrap(Driver driver,
    Class<T> clazz,
    @Nullable WebElementSource parent,
    By criteria,
    int index) {
    return (T) Proxy.newProxyInstance(
        currentThread().getContextClassLoader(),
        new Class<?>[]{clazz},
        new SelenideElementProxy(new ElementFinder(driver, parent, criteria, index)));
}
```

`SelenideElementProxy` must implement `InvocationHandler`, and we're happy to see that our assumption holds:

```java
class SelenideElementProxy implements InvocationHandler {
    
    public Object invoke(Object proxy, Method method, @Nullable Object... args) throws Throwable {

    }
}
```

We'll later dive into how it does its job, and why it needs an `ElementFinder` in its ctor.

This concludes our journey into `SelenideElement` and `Dynamic Proxy`.
