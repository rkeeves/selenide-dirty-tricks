# How long will it take?

When we:

```java
$("#foo").click();
```

We expect it to run once.

On the other hand when we:

```java
$("#foo").shouldBe(visible);
```

We expect it to run until the condition is met or timeout is reached.

So, how does this work?

## Overview

The invocation handler calls the `Commands` singleton in a `do while`.

If the timeout is reached it throws.

If the `Commands::execute` did not throw it returns the result.

If the `Commands::execute` did throw it decides whether to continue based on the thrown exception type.

## Do While

The actual code is quite convoluted. I cut out a lot of things related to logging. The core of the algorithm is this:

```java
class SelenideElementProxy {

    public Object invoke(Object proxy, Method method, Object... args) throws Throwable {
   
    try {
      return dispatchAndRetry(noise);
    }
    catch (Error error) {
        return continueOrBreak(noise);
    }
    catch (WebDriverException error) {
        ...
        return continueOrBreak(noise);
    }
    catch (RuntimeException | IOException error) {
      throw error;
    }
  }
}
```

`continueOrBreak` is connected to `soft assertions`. `soft assertion` means that even if something fails, please *continue my test and don't stop*.

When you are NOT using `soft assertions`, then this `continueOrBreak` does simply throw.

So, after rewriting the code it becomes:

```java
class SelenideElementProxy {

    public Object invoke(Object proxy, Method method, Object... args) throws Throwable {
   
    try {
      return dispatchAndRetry(noise);
    }
    catch (Whatever w) {
        return throw w;
    }
  }
}
```

From this we can see that `dispatchAndRetry` is the actual `do while`.

The concrete code is quite long and handles both `WebElement`s and `SelenideElement`s:

```java
class SelenideElementProxy {

    protected Object dispatchAndRetry(long timeoutMs,
                                    long pollingIntervalMs,
                                    Object proxy,
                                    Method method,
                                    Object[] args)
    throws Throwable {
    Stopwatch stopwatch = new Stopwatch(timeoutMs);

    Throwable lastError;
    do {
      try {
        if (isSelenideElementMethod(method)) {
          return Commands.getInstance().execute(proxy, webElementSource, method.getName(), args);
        }

        return method.invoke(webElementSource.getWebElement(), args);
      }
      catch (InvocationTargetException e) {
        lastError = e.getTargetException();
      }
      catch (WebDriverException | IndexOutOfBoundsException | AssertionError e) {
        lastError = e;
      }

      if (Cleanup.of.isInvalidSelectorError(lastError)) {
        throw Cleanup.of.wrapInvalidSelectorException(lastError);
      }
      else if (!shouldRetryAfterError(lastError)) {
        throw lastError;
      }
      stopwatch.sleep(pollingIntervalMs);
    }
    while (!stopwatch.isTimeoutReached());

    throw exceptionWrapper.wrap(lastError, webElementSource);
  }
}
```

I'll edit it to only contain the parts necessary for us in case of `SelenideElement`:

```java
class SelenideElementProxy {

    protected Object dispatchAndRetry(long timeoutMs,
                                    long pollingIntervalMs,
                                    Object proxy,
                                    Method method,
                                    Object[] args)
    throws Throwable {
    Stopwatch stopwatch = new Stopwatch(timeoutMs);

    Throwable lastError;
    do {
      try {
          return Commands.getInstance().execute(noise);
      }
      catch (Whatever e) {
        lastError = e.getTargetException();
      }

      if (!shouldRetryAfterError(lastError)) {
        throw lastError;
      }
      stopwatch.sleep(pollingIntervalMs);
    }
    while (!stopwatch.isTimeoutReached());

    throw lastError;
  }
}
```

From this we can see that it talks to the synchronized Singleton in a do-while.

We can also already see that:
- if the `execute` does not throw it returns
- if timeout is reached it throws the last thrown problem

In all other scenarios `shouldRetryAfterError` decices whether to continue or not based on the thrown type:

```java
class SelenideElementProxy {

    static boolean shouldRetryAfterError(Throwable e) {
        if (e instanceof FileNotFoundException) return false;
        if (e instanceof IllegalArgumentException) return false;
        if (e instanceof ReflectiveOperationException) return false;
        if (e instanceof JavascriptException) return false;

        return e instanceof Exception || e instanceof AssertionError;
  }
}
```

From all of this we can conclude that when a `Commands::execute` does not throw, then the invocation handler returns to you the result.

On the other hand if `Commands::execute` throws then it has the possibility to be retried.

Actually finding out how a `Command`'s failure will be handled is quite complex, because it depends on external factors too as we've seen. For example: `soft assertions` is on or off.

As a last note, the invocation handler does not get an instance of the command it tries to execute.

It always reacquires the synchronized Singleton, and calls its `execute` method.
