# A listener listens to what exactly? - Mutable factories

Selenide offers a way to hookup listeners to the automanaged webdriver.

So let's explore how the listeners are hooked up.

## Overview

Selenide keeps a central non thread based list of listeners.

Upon driver creation all listeners from the list will be added to the decorator around the driver.

## Selenium Listeners

As already discussed, the `Driver` is - most of the time - managed by Selenide. It creates it, and delegates closing it to a shutdown hook.

Selenide can also add your custom listener to drivers at their creation time.

```java
public class WebDriverThreadLocalContainer implements WebDriverContainer {

  public void addListener(WebDriverEventListener listener) {
    eventListeners.add(listener);
  }

  public void addListener(WebDriverListener listener) {
    listeners.add(listener);
  }
}
```

Selenium has two types of listeners and their corresponding 'observables':
- (DEPRECATED) `WebDriverEventListener` and `WebDriverEventListener`
- (CURRENT) `WebDriverListener` and `EventFiringDecorator`

There's one crucial difference between the DEPRECATED and the CURRENT.

**DEPRECATED rethrows any exception which is thrown by your listener.**

**The CURRENT consumes all of your exceptions.**

## WebDriverRunner adding listeners

So we now understand the difference between the two observers.

Let's see how Selenide manages listeners:

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

We can see that `eventListeners` and `listeners` are kept in a list.

When Selenide creates a new driver, these listeners lists are passed to the `CreateDriverCommand`.

```java
class CreateDriverCommand

  public Result createDriver(Config config,
                             WebDriverFactory factory,
                             Proxy userProvidedProxy,
                             List<WebDriverEventListener> eventListeners,
                             List<WebDriverListener> listeners) {
    
    WebDriver webdriver = ...

    WebDriver webDriver = addListeners(webdriver, eventListeners, listeners);

     ... shutdown hook etc
    }
}
```

All listeners from these lists are added to the new driver.

In multithreaded tests, be cautios with `WebDriverRunner::addListener` because you are mutating a shared resource.

Also, be aware that if a driver is already created it won't get any new listeners. When you call `WebDriverRunner.addListener` you simply add the listener to a list which will be used for future driver creations.
