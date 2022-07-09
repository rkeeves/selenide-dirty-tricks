# How does it know when to take screenshot?

Imagine you run:

```java
open("site")
$("#missing").shouldBe(visible);
```

After the timeout is reached, Selenide takes a screenshot.

But how does it know when to take a screenshot?

## Overview

The Invocation handler after timeout is reached uses the `ScreenshotLibrary` Singleton to take a screenshot.

## Should throws

So let's say we do this:

```java
$("#missing").should(visible);
```

Before `should` let's clarify what `visible` produces.

Condition `Visible` does the following:

```java
class Visible extends Condition {

    public CheckResult check(Driver driver, WebElement element) {
        boolean displayed = element.isDisplayed();
        return new CheckResult(displayed, displayed ? "visible" : "hidden");
    }
}
```

`CheckResult`'s ctor turns the `boolean` into an `enum`:

```java
class CheckResult {

    public CheckResult(boolean checkSucceeded, @Nullable Object actualValue) {
        this(checkSucceeded ? ACCEPT : REJECT, actualValue);
    }
}
```

**So remember that `visible` will return a `CheckResult` which has `REJECT` in our case.**


Back to the `should`.

We now know that the invocation handler will start repeatedly running a `Should` `Command` via the `Commands` singleton.

```java
class SelenideElementProxy implements InvocationHandler {
    protected Object dispatchAndRetry(...) throws Throwable {
        ...
        do {
        ...
                return Commands.getInstance().execute(proxy, webElementSource, method.getName(), args);
        ...
        while ...

        ...
  }
}
```

The `Should` simply calls upon the `WebElementSource::checkCondition`:

```java
public class Should implements Command<SelenideElement> {
  
  ...

  public SelenideElement execute(...) {
    ...
      locator.checkCondition(prefix, condition, false);
    ...
  }
}
```

**We are also 100% sure that an exception will be thrown by `locator.checkCondition`**. *Otherwise the invocation handler would return the control to you. This call MUST throw somewhere, somehow.*

But let's verify it!

`WebElementSource::checkCondition` is simple calling `WebElementSource::checkConditionAndReturnElement`:

```java
class WebElementSource {

    public void checkCondition(...) {
      checkConditionAndReturnElement(...);
    }
}
```

The called method's body is pretty big. Like this.

```java
class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(String prefix, Condition condition, boolean invert) {
        Condition check = invert ? not(condition) : condition;

        Throwable lastError = null;
        WebElement element = null;
        CheckResult checkResult = null;
        try {
            element = getWebElement();
            checkResult = check.check(driver(), element);

            if (checkResult.verdict() == ACCEPT) {
                return element;
            }
        }
        catch (WebDriverException | IndexOutOfBoundsException | AssertionError e) {
            lastError = e;
        }

        if (lastError != null && Cleanup.of.isInvalidSelectorError(lastError)) {
            throw Cleanup.of.wrapInvalidSelectorException(lastError);
        }

        if (element == null) {

            if (check.missingElementSatisfiesCondition()) {
                return null;
            }
            throw createElementNotFoundError(check, lastError);
        }
        else if (invert) {
            throw new ElementShouldNot(driver(), description(), prefix, condition, checkResult, element, lastError);
        }
        else {
            throw new ElementShould(driver(), description(), prefix, condition, checkResult, element, lastError);
        }
  }
}
```

So let's walk the execution all the way:

We know that the `Visible` will return `REJECT`, so we won't `return` from the `try`.

```java
class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(...) {
    ...

    CheckResult checkResult = null;
    try {
        element = getWebElement();
        checkResult = check.check(driver(), element);

        if (checkResult.verdict() == ACCEPT) {
            // We won't end up here
            return element;
        }
    ...
  }
}
```

We also did not throw, so we won't get into the `catch`.

```java
class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(String prefix, Condition condition, boolean invert) {
        ...
        catch (WebDriverException | IndexOutOfBoundsException | AssertionError e) {
            // We won't end up here
            lastError = e;
        }

        ...
  }
}
```

We did find the element previously, so we won't go into the `if` branch. We are also NOT inverted, so we won't go into the `else if` branch. We end up with going into the `else` branch.

```java
class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(String prefix, Condition condition, boolean invert) {
        WebElement element = null;
        try {
            element = getWebElement();
        ...

        if (element == null) {
            // We won't end up here
            ...
        } else if (invert) {
            // We won't end up here
            ...
        } else {
            ???
        }
  }
}
```

Which does instantiate an error `ElementShould` and throws it, according to our previous expectations:

```java
class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(String prefix, Condition condition, boolean invert) {
        ...
        } else {
            throw new ElementShould(driver(), description(), prefix, condition, checkResult, element, lastError);
        }
  }
}
```

`ElementShould` DOES NOT take a screenshot.

```java
public class ElementShould extends UIAssertionError {


    public ElementShould(Driver driver, String searchCriteria, String prefix,
                        Condition expectedCondition, @Nullable CheckResult lastCheckResult,
                        WebElement element, @Nullable Throwable lastError) {
        super(
        String.format(...),
        expectedCondition,
        lastCheckResult == null ? null : lastCheckResult.actualValue(),
        lastError);
    }
}
```

The `super` too DOES NOT take a screenshot. Although this one has a field for it.

```java
public  class UIAssertionError extends AssertionFailedError {

    private Screenshot screenshot = Screenshot.none();
    
    ...
```

In summary, `Should` just throws an error for the invocation handler.

## Invocation handler times out

Back at the invocation handler time is running out, and the `Should` keeps throwing.

Eventually the time runs out, and we fall out of the `do while`.

```java
class SelenideElementProxy implements InvocationHandler {

    protected Object dispatchAndRetry(...) throws Throwable {
    ...

    Throwable lastError;
        do {
            try {
                return unsafeWhatever();
            } (Throwable e) {
                lastError = e;
            } catch (InvocationTargetException e) {
                lastError = e.getTargetException();
            }
        } while (...);

        throw exceptionWrapper.wrap(lastError,
                                    webElementSource);
        }
    }
```

It is guaranteed that we have a `lastError`, because if it does not throw we immediately return.

The `exceptionWrapper` is just a stateless helper class. In our case it does simply return the passed in throwable.

```java
class ExceptionWrapper {
  
  Throwable wrap(Throwable lastError,
                WebElementSource webElementSource) {
    if (lastError instanceof UIAssertionError) {
      return lastError;
    }
    ...
```

So our UIAssertionError `ElementShould` still has no screenshot, but gets thrown from `SelenideElementProxy::dispatchAndRetry` up to its caller `SelenideElementProxy::invoke`.

## Taking screenshot

So we're in `invoke` without one single screenshot, but a throwable `ElementShould` gets thrown up.

```java

class SelenideElementProxy implements InvocationHandler {

    public Object invoke(...) throws Throwable {
    ...
    try {
     // we throw an ElementShould here
    }
    catch (Error error) {
        // And ElementShould gets caught here
        Throwable wrappedError = UIAssertionError.wrap(driver(), error, timeoutMs);
      ...

}
```

So `UIAssertionError.wrap` MUST take a screenshot.

And if we see the implementation it indeed takes a screenshot, although the actual screenshotting is done by calling within itself something else:

```java
public class UIAssertionError extends AssertionFailedError {
    public static Error wrap(...) {
        return ... ? ... : wrapThrowable(driver, error, timeoutMs);
    }

    private static UIAssertionError wrapThrowable(Driver driver, Throwable error, long timeoutMs) {
        ...
        Config config = driver.config();
        uiError.screenshot = ScreenShotLaboratory.getInstance()
        .takeScreenshot(driver, config.screenshots(), config.savePageSource());
    }
    ...
  }
}
```

`ScreenShotLaboratory` is a Singleton just like `Commands`. We'll later see how it works.


