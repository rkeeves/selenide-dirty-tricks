# How lazy is lazy?

When we do:

```java
Selenide.$("#foo");
```

It is said that this *imperative act* of us is lazy in a sense.

When we do:

```java
Selenide.$("#foo").$("li", 2);
```

It is said that this *imperative act* of us is lazy in a sense too.

So what's this manufactured lazyness?

## TLDR

It simply doesn't call the driver on `find`.

Instead it constructs a new Dynamic Proxy and gives it back to you.

## What even is lazy in imperative terms?

When you do something it gets executed immediately.

```java
int x = 1 + 1;
```

In this example you add 1 to 1 immediately and store it into `x`. *(Not exactly true...)*

```java
int x = 0 / 0;
```

In this example you divide 0 by 0 immediately and store it into `x`. *(Not exactly true...)*

## Lazy enough...

On the other hand,  the below means that you create a Supplier immediately and store its result into x *(Not exactly true...)*:

```java
Supplier<Integer> x = () -> 0 / 0;
```

You are still running something and store back its resilt into something for later use.

`x` is still not an alias for `() -> 0 / 0` (whatever alias means), nor is it lazy (whatever lazy means).

But this thing won't throw, just when you `Supplier::get` so it kind of gets the job done without going full Scala/Haskell.

## Selenide Dynamic Proxy

Selenide says `find`, `$` and some other commands are lazy.

What Selenide tries to say is that `find` consructs an object which stores your selector and much more instrumentations.

It doesn't call the webdriver.

The below code we know creates a Dynamic Proxy.

```java
Selenide.$("#foo");
```

The below code we know 'runs' through the Dynamic Proxy, the invocation handler, then through the Singleton, getting a functor like object from the hashmap, executes that thing, then it comes back to you with the result.

```java
Selenide.$("#foo").$(".bar");
```

Technically speaking the second `$` is still a command. It will immediately run.

We also know that if we look into `Commands` Singleton hashmap, we must be able to find a mapping for key `$`.

```java
class Commands {

    private void addFindCommands() {
        add("find", new Find());
        add("$", new Find());
        ...
    }
}
```

The mapping corresponds to an instance of type `Find` which is calling `WebElementSource::find` after a bit of unsafe casts (due to the dynamic proxying):

```java
public class Find implements Command<SelenideElement> {

    public SelenideElement execute(SelenideElement proxy,
                                WebElementSource locator,
                                Object... args) {
    assert args != null;

    return args.length == 1 ?
        locator.find(proxy, args[0], 0) :
        locator.find(proxy, args[0], (Integer) args[1]);
  }
}
```

`WebElementSource` is an abstract class:

```java
public abstract class WebElementSource {
```

Remember `$("#foo")` meant the following:

```java
class ElementFinder {
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
}
```

Notice that `SelenideElementProxy`'s ctor received `ElementFinder`.

`ElementFinder` is a subclass of `WebElementSource`:

```java
public class ElementFinder extends WebElementSource
```

So, this `Find` in our case calls the find method on `ElementFinder`:

```java
public class Find implements Command<SelenideElement> {

    ...
    return ...
        locator.find(proxy, args[0], 0) :
        ...
  }
}
```

`ElementFinder::find` simply does more unchecked casts, but it eventually calls the static `wrap` method which creates the Dynamic Proxy:

```java
class ElementFinder {

    public SelenideElement find(SelenideElement proxy, Object arg, int index) {
    return arg instanceof By ?
      wrap(driver, this, (By) arg, index) :
      wrap(driver, this, By.cssSelector((String) arg), index);
  }
}
```

In summary: `find` begins a chain of complicated actions through dynamic proxy, singleton, hashmap, but eventually calls a static method which creates a Dynamic Proxy. No webdriver calls.
