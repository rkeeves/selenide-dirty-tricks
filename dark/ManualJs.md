# Can I wait with the driver?

Selenide itself is full of js, it does it all the time.

It is also easy for your to execute javascript in Selenide.

## Overview

You can execute javascript and wait if you have a reference to a driver.

## Executing javascript

To execute javascript, you need a webdriver.

If you are using the threadlocal managed driver, you're lucky!

You can use:

```java
boolean res = Selenide.executeJavaScript("return true;");
```

If you are using multiple drivers in one test (gosh the things management asks from poor testers sometimes makes me angry...):
```java
SelenideDriver ann = ...;
SelenideDriver bob = ...;
boolean a = ann.executeJavaScript("return true;");
boolean b = bob.executeJavaScript("return true;");
```

But...what are you running actually?

The script will be executed as the body of an anonymous function.

You return a value by using `return`.

The return value type must be one of these:
- Long
- String
- Boolean
- List
- WebElement

You can also supply arguments:

```java
Selenide.executeJavaScript("return arguments[0] + arguments[1];", "Hello", "World");
```

The above example is simple (and pointless), but imagine that you supply a `WebElement` and click on it from js(Selenide uses js this way sometimes too!), or supply a `String` to some jQuery function.

## Fluently

Executing javascript is crucial, but we want more.

We want to essentially retry it until a timeout is reached.

Suppose we have a driver and we want to run javascript predicate `magicJs` until it returns true, with custom timeout and polling.

In this case you can use `Selenide.Wait`:

```java
Selenide.Wait()
        .withTimeout(Duration.ofSeconds(6))
        .pollingEvery(Duration.ofMillis(500L))
        .until(webDriver -> {
            if (webDriver instanceof JavascriptExecutor) {
                return ((JavascriptExecutor) webDriver).executeScript("return magicJs();");
            }
            return false;
        });
```

If you want to use custom timeout and polling interval, then it simplifies into:

```java
Selenide.Wait()
        .until(webDriver -> {
            if (webDriver instanceof JavascriptExecutor) {
                return ((JavascriptExecutor) webDriver).executeScript("return magicJs();");
            }
            return false;
        });
```

Treat this as a simple `FluentWait`.
