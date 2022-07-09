# Can I keep the old Click?

We can achieve that `click` executes arbitrary code:

```java
$("#foo").click();
```

But what if we want to keep the old `click` and have a method `crazyClick`.

## Overview

Create a new mapping in `Commands`.

Create a new interface which extends `SelenideElement`.

Create a static factory methid for acquiring an instance of the Dynamic Proxy.

You won't be able to chain properly without a lot of work.

## New key to the hashmap

Given you have a `Command` called `CrazyClick`:

```java
class CrazyClick extends Click {

    public Void execute(SelenideElement proxy,
                        WebElementSource locator,
                        @Nullable Object[] args) {
            arbitrary_code
            return super.execute(proxy, locator, args);
        }
    }
```

If you add a new mapping:

```java
Commands.getInstance().add("crazyClick", new CrazyClick());
```

When you call `crazyClick`, the invocation handler will call `Commands.getInstance().execute(...)`.

But... you can't call it...

```java
SelenideElement e = $("#foo");
```

`SelenideElement` interface has no method `crazyClick`.

## Dynamic Proxy with arbitrary interface

We need an instance of a type which has all the methods of `SelenideElement` and the method `crazyClick`.

Let's say it is interface:

```java
interface CrazyElement extends SelenideElement {

    void crazyClick();
}
```

Now, remember how the Dynamic Proxy was constructed?

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

So if we must create a static method which does almost this, but with our `CrazyElement` interface instead.

Let's pack it into a utility class called `Crazy`:

```java
class Crazy {

    public static CrazyElement $crazy(String selector) {
        return ElementFinder.wrap(driver(), CrazyElement.class, null, By.cssSelector(selector), 0);
  }
}
```

Now in your code you can interact with `CrazyElement`:

```java
CrazyElement e = $crazy("#foo");
e.crazyClick();
```

## Chaining issues

So what type will be returned when you do this:

```java
CrazyElement e = $crazy("#foo");
? f = e.$("#bar");
```

The "$" is coming from interface `SelenideElement` which will return - just by looking at the interface - `SelenideElement`.

So you can't chain your new type.

Mutating the hashmap won't help. It is the `SelenideElement` interface which defines `$`'s return type.

So, unless you are ready to do override the whole library, you lost chaining.

You can define of course your own `$crazyChild` method in interface `Crazy` but then you'll suddently find that building recursive data structures with DynamicProxies and inheritance is quite a tricky thing to do correctly.
