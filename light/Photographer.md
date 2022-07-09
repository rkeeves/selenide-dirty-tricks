# How does it take screenshots?

How does Selenide take a screenshot (in case of web pages)?

# Overview

Selenium's `TakesScreenshot`.

Because of support for mobile things are a bit more complicated.

## Screenshot laboratory

Remember how `ScreenshotLaboratory` was called?


```java
public class UIAssertionError extends AssertionFailedError {


    private static UIAssertionError wrapThrowable(Driver driver, Throwable error, long timeoutMs) {
        ...  = ScreenShotLaboratory.getInstance()
        .takeScreenshot(driver,
                config.screenshots(),
                config.savePageSource());
    }
    ...
  }
}
```

`ScreenShotLaboratory` is a Singleton without synchronization.

```java
public class ScreenShotLaboratory {

    private static final ScreenShotLaboratory instance = new ScreenShotLaboratory();

    public static ScreenShotLaboratory getInstance() {
        return instance;
    }
    ...
```

`takeScreenshot` is overloaded and travels through the overloads of itself multiple times, but eventually it calls:

```java
public class ScreenShotLaboratory {

    ...
        photographer.takeScreenshot(driver, outputType)
    ...
```

`Photographer` is an interface:

```java
public interface Photographer {
 
  <T> Optional<T> takeScreenshot(Driver driver, OutputType<T> outputType);
}
```

For webdrivers there exists a class `WebdriverPhotographer` which does the actual screenshot via Selenium's `TakesScreenshot`:

```java
public class WebdriverPhotographer implements Photographer {
 
  public <T> Optional<T> takeScreenshot(Driver driver, OutputType<T> outputType) {
    if (driver.getWebDriver() instanceof TakesScreenshot) {
      T screenshot = ((TakesScreenshot) driver.getWebDriver()).getScreenshotAs(outputType);
      return Optional.of(screenshot);
    }
    return Optional.empty();
  }
}
```
