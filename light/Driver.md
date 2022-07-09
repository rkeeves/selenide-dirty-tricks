# Where's the driver?

Selenide has a static class `Selenide`.

You are probably familiar with it.

You interact with it always:

```java
Selenide.open("foo");
Selenide.$("bar")
```

But what happens when you open?

How does it magically pop up a driver?

How does the browser magically close after everything?

## Overview

There's a long indexed hashmap for keeping track of each `Thread`'s current driver.

Driver lifecycle management has nothing to do with this hashmap.

Driver lifecycle management is done by adding `shutdown hooks` to the `Runtime` for each new driver **created by Selenide**.

`Shutdown hooks` are added when Selenide **creates a driver**.

Therefore if you yourself create a driver, then Selenide does not have the ability to register shutdown hooks.

## Open creates driver on demand

The lifecycle management is quite complex, so I had to simplify things a lot.

But basically what happens when you open is that this static method:

```java
class Selenide {
    public static void open(URL absoluteUrl) {
        WebDriverRunner.getSelenideDriver().open(absoluteUrl);
    }
}
```

After a long-long travel it eventually executes `driver.getAndCheckWebDriver()`:

```java
class Navigator {

    private void navigateTo(SelenideDriver driver,
                            String relativeOrAbsoluteUrl,
                            AuthenticationType authenticationType,
                            Credentials credentials) {
    ...
        WebDriver webDriver = driver.getAndCheckWebDriver();
        webDriver.navigate().to(url);
    ...
    }
}
```

This is a pretty complicated thing and the management is done by `WebDriverThreadLocalContainer`.

## Current Drivers

Overly simplified `WebDriverThreadLocalContainer` keeps a hashmap which has mappings `Long -> WebDriver`. The `Long` is the `Thread`'s id. The map has to be concurrent, because multiple threads access the same resource:

```java
class WebDriverThreadLocalContainer {

    final Map<Long, WebDriver> threadWebDriver = new ConcurrentHashMap<>();
}
```

This map stores the **current** driver of a `Thread`.

When - for any reason - `Thread` creates a new `WebDriver`, the old mapping gets overridden by the new mapping.

## Shutting down

The `threadWebDriver` only keeps track of the current driver for each `Thread`.

```java
class WebDriverThreadLocalContainer {

    final Map<Long, WebDriver> threadWebDriver = new ConcurrentHashMap<>();
}
```

So how can it 'auto close' a `WebDriver` which was removed from the map?

The auto close has nothing to do with that map.

When Selenide creates a `WebDriver` intended to work as *thread local automanaged driver*, it adds a `Shutdown hook`. This is done by `CreateDriverCommand::createDriver`.

```java
class CreateDriverCommand {

    public Result createDriver(WebDriverFactory factory) {
       ...
            WebDriver webdriver = factory.createWebDriver(noise, noise, noise);

      
            Runtime.getRuntime().addShutdownHook(
                new Thread(new SelenideDriverFinalCleanupThread(webdriver))
            );

       ...
    };
}
```

But what is a shut down hook?

## Shutdown hook

[Shutdown hooks](https://docs.oracle.com/javase/7/docs/api/java/lang/Runtime.html#addShutdownHook(java.lang.Thread)) are simply a way for you to run something when the VM shuts down.

For example, take a look at the code below:

```java
public class Prog {

    public static void main(String[] args) {
        System.out.println("Hello");
        Runtime.getRuntime().addShutdownHook(new Thread(
                () -> System.out.println("World")
        ));
    }
}
```

Running this program will result in the following output upon execution:

```shell
Come over to the dark side
Cookies and free capes!
```

A shutdown hook runs when the VM normally terminates.

All shutdown hooks must be written with concurrency in mind. Especially keeping an exe for deadlocks.

## Abstract factory

Creating a driver automatically is hard. You have to download binaries, set up capabilities, handle edge cases.

Selenide uses [WDM](https://github.com/bonigarcia/webdrivermanager), but still has to do a lot of setup by itself.

`CreateDriverCommand` is actually not responsible for the actual creation of the driver itself.

Instead it relies on `WebDriverFactory` which it gets from its caller `WebDriverThreadLocalContainer`.

```java
class CreateDriverCommand {

    public Result createDriver(WebDriverFactory factory) {
        ...
            WebDriver webdriver = factory.createWebDriver(noise, noise, noise);
        ...
    });
  }
}
```

The `WebDriverFactory` is an [Abstract factory](https://refactoring.guru/design-patterns/abstract-factory), aka it is a factory of factories.

It has a hashmap of all factories (like chrome, or firefox)

```java
public class WebDriverFactory {
  
  private final Map<String, Class<? extends AbstractDriverFactory>> factories = factories();

  private Map<String, Class<? extends AbstractDriverFactory>> factories() {
    Map<String, Class<? extends AbstractDriverFactory>> result = new HashMap<>();
    result.put(CHROME, ChromeDriverFactory.class);
    result.put(FIREFOX, FirefoxDriverFactory.class);
    return result;
  }
}
```


```java
class WebDriverFactory {

  public WebDriver createWebDriver(Config config, noise) {

    ...

    Browser browser = new Browser(config.browser(), config.headless());
    
    WebDriver webdriver = createWebDriverInstance(config, browser, noise);
    
    ...
    return webdriver;
  }
}
```

What we're interested in is `createWebDriverInstance`.

It does looks up the actual factory from a hashmap and sets up the binary using `WDM`.

```java
class WebDriverFactory {
  
  final Map<String, Class<? extends AbstractDriverFactory>> factories;

  final RemoteDriverFactory remoteDriverFactory = new RemoteDriverFactory();

  private WebDriver createWebDriverInstance(Config config, Browser browser, noise) {
    // get by string a Class
    // and instantiate it via Reflection
    DriverFactory webdriverFactory = findFactory(browser);

    ...
      if (config.driverManagerEnabled()) {
        webdriverFactory.setupWebdriverBinary();
      }
      System.setProperty("webdriver.http.factory", "selenide-netty-client-factory");
      return webdriverFactory.create(config, browser, proxy, browserDownloadsFolder);
    ...
  }
}
```

So the abstract factory:
- finds the correct factory
- calls `webdriverFactory.setupWebdriverBinary()` (this is simply a call to `WDM`)
- sets a sys prop for http client
- and then finally calls `webdriverFactory.create(...)`

After this it returns the instance.

## When shutdown hook starts

Remember that `CreateDriverCommand` registered a `shutdown hook`. The `Runnable` was called `SelenideDriverFinalCleanupThread`.

```java
class CreateDriverCommand {

    public Result createDriver(WebDriverFactory factory) {
       ...
            WebDriver webdriver = factory.createWebDriver(noise, noise, noise);

      
            Runtime.getRuntime().addShutdownHook(
                new Thread(new SelenideDriverFinalCleanupThread(...))
            );

       ...
    };
}
```

`SelenideDriverFinalCleanupThread` is nothing more than a function and its arguments packaged together:

```java
public class SelenideDriverFinalCleanupThread implements Runnable {
  final Config config;
  final WebDriver driver;
  final SelenideProxyServer proxy;
  final CloseDriverCommand closeDriverCommand;

  SelenideDriverFinalCleanupThread(Config config, WebDriver driver, @Nullable SelenideProxyServer proxy,
                                   CloseDriverCommand closeDriverCommand) {
    this.config = config;
    this.driver = driver;
    this.proxy = proxy;
    this.closeDriverCommand = closeDriverCommand;
  }

  @Override
  public void run() {
    closeDriverCommand.close(config, driver, proxy);
  }
}
```

It is basically:

```java
public class Prog {

    public static void main(String[] args) {
        Consumer<String> close = (s) -> {
            System.out.println(s);
        };
        Runnable finalCleanup = () -> close.accept("World!");

        System.out.println("Hello");
        Runtime.getRuntime().addShutdownHook(new Thread(finalCleanup));
    }
}
```

We can clearly see that it does not interact with the hashmap that we talked about earlier (`Thread` ids mapped to `WebDriver`s). It has all the info it needs to fire the `CloseDriverCommand::close`.


```java
public class SelenideDriverFinalCleanupThread implements Runnable {
  
  ...
  final CloseDriverCommand closeDriverCommand;

  public void run() {
    closeDriverCommand.close(config, driver, proxy);
  }
}
```

If you look into `CloseDriverCommand` it uses the `Thread` id just for logging. It does not interact with the hashmap (or any cache).

```java
public class CloseDriverCommand {
 
  public void close(Config config, @Nullable WebDriver webDriver, ...) {
      if (webDriver != null) {
        ...
        close(webDriver);
        ...
  }

    private void close(WebDriver webdriver) {
        ...
            webdriver.quit();
...
```

## You are responsible for closing drivers created by you

`WebDriverRunner` has a method called `setWebDriver`.

In the javadoc it is highlighted by the authors that:

> When using your custom webdriver, you are responsible for closing it. Selenide will not take care of it.

But let's see why! First we jump to `WebDriverRunner::setWebDriver`:

```java
class WebDriverRunner {

  public static WebDriverContainer webdriverContainer = new WebDriverThreadLocalContainer();

  public static void setWebDriver(WebDriver webDriver) {
    webdriverContainer.setWebDriver(webDriver);
  }
}
```

The `WebDriverThreadLocalContainer::setWebDriver` does remove your thread's older driver from the hashmap and then puts your new custom driver into the map. We know that this map has nothing to do with lifecycle or cleanup. It is simply a cache of current webdrivers.

```java
class WebDriverThreadLocalContainer {

  public void setWebDriver(WebDriver webDriver, noise) {
    resetWebDriver();
    // put in the custom driver to cache
    long threadId = currentThread().getId();
    threadWebDriver.put(threadId, webDriver);
  }

  public void resetWebDriver() {
    // throw out the old automanaged from cache
    long threadId = currentThread().getId();
    threadProxyServer.remove(threadId);
    ...
  }
}
```

It does not remove the shutdown hook. The old driver - if it was managed by Selenide - is still 'scheduled to be closed'. The shutdown hook will call its `CloseDriverCommand` which has a reference to the old driver.

Your new driver on the other hand is automanaged by shutdown hook.

But why?

Remember where the shutdown hook was registered?

```java
class CreateDriverCommand {

    public Result createDriver(noise) {
          ...
            Runtime.getRuntime().addShutdownHook(...
           ...
    });
  }
}
```

Your manually added driver was not created by Selenide it won't automanage your driver.

## Custom driver, but automanaged

But what if you want to customize your driver and still be 'scheduled to quit via a shutdown hook'?

You can provide your own `DriverFactory`.

You must create a class which implements `DriverFactory`.

Or, if you just want to modify one of Selenide's own factories (Chrome, Firefox etc.) subclass them, and override what you need:

```java
public static class FooFactory extends ChromeDriverFactory
```

You must then set this factory via either:
- property file
- command line
- manually mutating Selenide's `Configuration`'s static browser field

```java
selenide.browser=com.acme.FooFactory
```

If you provide a factory this way, Selenide will add a shutdown hook just as we've discussed earlier.

## Listeners

Listeners are hooked on by `CreateDriverCommand` not any of the factories. (The listener list comes from WebDriverRunner, but those lists are not thread safe and are not thread specific)

```java
public class CreateDriverCommand {

    ....

    @Nonnull
    public Result createDriver(...
                                List<WebDriverEventListener> eventListeners,
                                List<WebDriverListener> listeners) {
        ...
        WebDriver webDriver = addListeners(webdriver, eventListeners, listeners);
        ...
  }
```

`CreateDriverCommand` creates the decorator and wraps the `factory-made driver` inside it (for both decorator types):

```java
public class CreateDriverCommand {

    private WebDriver addWebDriverListeners(WebDriver webdriver, List<WebDriverListener> listeners) {
        ...
        EventFiringDecorator wrapper = new EventFiringDecorator(listeners.toArray(new WebDriverListener[]{}));
        return wrapper.decorate(webdriver);
  }
}
```

In conclusion we've seen what an `Selenide.open("")` entails:
- concurrent hashmaps
- identifying your thread
- adding shutdown hooks
- abstract factories
- creating instances via Reflection
- and some Selenium

We now also know what Selenide really means when it says that **we're responsible for closing the driver**!
