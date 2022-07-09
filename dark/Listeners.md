# Can I use listeners for waiting?

Writing out manually the waiting becomes tiresome.

But we know that there are listeners.

Can we use listeners for waiting?

## Overview

Yes.

But there are many caveats.

## Choosing a listener

You have two kind of listeners:
- WebDriverEventListener
- WebDriverListener

We clarified earlier that `WebDriverEventListener` and `WebDriverListener` are fundamentally different due to how their respective Observables treat exceptions.

If you throw in a `WebDriverEventListener` method it will be rethrown by its Observable. And this type of listener is also deprecated.

On the other hand if you throw in a `WebDriverListener` method it will be consumed by its Observable.

This means that if you want to do any naughty things, like using FluentWait you must choose `WebDriverEventListener` to be able to throw.

## Fluently waiting in the listener

Both listeners are simple interfaces with methods corresponding to different events.

```java
public interface WebDriverEventListener {
    ...
    void beforeFindBy(By by, WebElement element, WebDriver driver);
    ...
    void beforeClickOn(WebElement element, WebDriver driver);
    ...
}
```

Although they have different 'callbacks'.

```java
public interface WebDriverListener {
    ...
    default void beforeAnyWebDriverCall(WebDriver driver, Method method, Object[] args) {
    }
    ...
    default void beforeFindElement(WebElement element, By locator) {
    }
    ...
    default void beforeClick(WebElement element) {
    }
    ...
}
```

If you want to wait in one a callback you'll need a reference to the driver.

An example would be:

```java
public interface WebDriverEventListener {
    ...
    @Override
    public void beforeClick(WebDriver driver) {
        new FluentWait<>(driver).withTimeout(Duration.ofSeconds(10L))
                .pollingEvery(Duration.ofMillis(500L))
                .until(webDriver -> {
                    if (webDriver instanceof JavascriptExecutor) {
                        Boolean result = ((JavascriptExecutor) webDriver)
                            .executeScript("return magicJs();");
                        return result;
                    }
                    return false;
                });
    }

}
```

Quite clunky, and inefficient, but you can get the basic idea from it.

## Adding listeners

If you want to add listeners to a thread local SelenideDriver, you can do it via `WebDriverRunner` in both cases:

```java
WebDriverRunner.addListener(new WebDriverEventListener() { ... });

WebDriverRunner.addListener(new WebEventListener() { ... });
```

But there's a problem though.

Selenide - as it was mentioned - simply keeps a non thread safe shared list for listeners.

```java
public class WebDriverThreadLocalContainer implements WebDriverContainer {
 
  private final List<WebDriverEventListener> eventListeners = new ArrayList<>();

  private final List<WebDriverListener> listeners = new ArrayList<>();

  public void addListener(WebDriverEventListener listener) {
    eventListeners.add(listener);
  }

  public void addListener(WebDriverListener listener) {
    listeners.add(listener);
  }
}
```

When it needs to create a new driver, it creates one then adds all listeners to it from those lists.

And when it creates a new driver, it creates one then adds all listeners to it from those lists.

So when you add a listener, you are basically mutating the factory which creates the drivers.

## One listener

In our test, before any `open` is called, we have to add listeners.

The problem is that there's no way for you to know whether you already added a listener or not.

This problem gets worse if you are using multiple threads.

They can now add too many listeners, or even mutate the `ArrayList` while something else is iterating over it.

You can't really fool proof the `ArrayList` from `ConcurrentModificationException`.

So you must create some obnoxious thing to ensure that:
- the listener is added only once
- multiple threads don't access the same resource at once

You can do this via a synchronized Singleton like the one below, but it is a disgusting hack:

```java
public class AddListener {

    private static volatile AddListener instance;

    private AddListener(WebDriverEventListener webDriverEventListener) {
        WebDriverRunner.addListener(webDriverEventListener);
    }
    
    public static AddListener orIgnoreIfAlready(WebDriverEventListener webDriverEventListener) {
        AddListener result = instance;
        if (result != null) {
            return result;
        }
        synchronized(AddListener.class) {
            if (instance == null) {
                instance = new AddListener(webDriverEventListener);
            }
            return instance;
        }

    }
}
```

You use it like this:

```java
WebDriverEventListener listener = ...;
AddListener.orIgnoreIfAlready(listener); // ctor runs
AddListener.orIgnoreIfAlready(listener); // ctor does NOT run
```

The ctor is guaranteed to run once (not really because I'm a bad programmer).

Whatever is in the ctor body will run only once.

## But what's the cost?

You hook up the listener ready to rock. But before we go and do things, let's think about how things will perform.

I won't go through all scenarios just one: `beforeFindBy`.

Imagine you do a `FluentWait` in `beforeFindBy`.

Then we call in the test `$("#foo").shouldBe(visible)`.

It will go to the Invocation Handler.

The Invocation Handler will do a do-while firing `Should` each time.

`Should` will call a method on `WebElementSource` passing the `Condition`.

So your code will do:

```
// the invocation handler's do-while
do {
    do { // the fluent wait's 'do-while'
        js test
    } while (whatever)
    find(By.id(""))
    check visibility
} while (whatever)
```

A lot of `FluentWait`s. You cannot control when it runs.

You decide if it is acceptable to you or not. I'm just saying that it can be done, but it has a pretty huge price.
