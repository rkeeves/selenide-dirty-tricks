# Can I preattach listeners to drivers by factory?

Mutating an `ArrayList` with multiple threads is not a safe thing.

Can we somehow preattach listeners?

## Overview

Use custom driver factories.

This will be more safe, but add one more layer of indirection over the underlying `WebDriver` instance, slowing things down a bit.

Also, whatever negatives apply to listeners, you are still suffering from those negatives. Simply we attach them differently.

## Custom Driver Factory

As we discussed, Selenide enables you to use your own custom factories.

As a refresher, we create `HackedChromeDriverFactory`:

```java
package com.hack;

public static class HackedChromeDriverFactory extends ChromeDriverFactory
```

Then in selenide.properties we write:
```
selenide.browser=com.hack.HackedChromeDriverFactory
```

Or we mutate `Configuration.browser`:
```java
Configuration.browser = HackedChromeDriverFactory.class.getName();
```

The factory itself relies on the super for creation of the driver instance.

Then it wraps it inside the observable, and attaches the listener:

```java
public class HackedChromeDriverFactory extends ChromeDriverFactory {

    @Nonnull
    @Override
    public WebDriver create(Config config, Browser browser, @Nullable Proxy proxy, @Nullable File browserDownloadsFolder) {
        final WebDriver webDriver = super.create(config, browser, proxy, browserDownloadsFolder);
        WebDriverEventListener listener = ...;
        return new EventFiringWebDriver(webDriver).register(listener);
    }
}
```

For the outside world it seems like a normal `WebDriver`. But the real driver was wrapped within a `EventFiringWebDriver`.

## But then Selenide wraps it

When Selenide decides to create a driver, it looks up the factory.

Creates a new `WebDriver` and decorates it with both a `EventFiringWebDriver` and a `EventFiringDecorator`.

```java
class CreateDriverCommand {

  public Result createDriver(Config config,
                             WebDriverFactory factory,
                             Proxy userProvidedProxy,
                             List<WebDriverEventListener> eventListeners,
                             List<WebDriverListener> listeners) {
    
    WebDriver webdriver = ...

    // here
    @Nonnull
    private WebDriver addListeners(WebDriver webdriver,
                                 List<WebDriverEventListener> eventListeners,
                                 List<WebDriverListener> listeners) {
        return addWebDriverListeners(
                addEventListeners(webdriver, eventListeners), listeners);
    }

    @Nonnull
  private WebDriver addEventListeners(WebDriver webdriver, List<WebDriverEventListener> eventListeners) {
        if (eventListeners.isEmpty()) {
            return webdriver;
        }
        EventFiringWebDriver wrapper = new EventFiringWebDriver(webdriver);
        for (WebDriverEventListener listener : eventListeners) {
            ...
            wrapper.register(listener);
        }
        return wrapper;
  }

  @Nonnull
  private WebDriver addWebDriverListeners(WebDriver webdriver, List<WebDriverListener> listeners) {
        if (listeners.isEmpty()) {
            return webdriver;
        }

        log.info("Add listeners to webdriver: {}", listeners);
        EventFiringDecorator wrapper = new EventFiringDecorator(listeners.toArray(new WebDriverListener[]{}));
        return wrapper.decorate(webdriver);
  }
}
```

So, in vanilla Selenide a `WebDriver` is actually a:
- `EventFiringDecorator` wrapping
- an `EventFiringWebDriver` wrapping
- potentially a WebDriver.

In our case, a `WebDriver` is actually a:
- `EventFiringDecorator` wrapping
- an `EventFiringWebDriver` wrapping
- an `EventFiringWebDriver` wrapping
- potentially a WebDriver.

So we added one more layer of busyness to all calls to the actual `WebDriver`.

This might be acceptable to you, or not at all.
