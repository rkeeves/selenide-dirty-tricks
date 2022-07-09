# Can I wait with a custom Command?

You can manually execute some js or wait with the driver.

But what if you don't have a direct reference to the driver?

## Overview

You can create custom `Command`s where you can interact with the driver.

If you return a value, polling will end.

If you throw something (which signals retry), then polling will continue.

## Custom Commands and interacting with the driver

Remember how `Command` works?

```java
interface Command {

     T execute(SelenideElement proxy,
                WebElementSource locator,
                @Nullable Object[] args) throws IOException;
}
```

Touching the `SelenideElement` is basically pointless because calling anything on it will - via Dynamic Proxy - run a `Command` (through `Commands` Singleton). But we are already in a `Command`. So all the things we want to play with are in `WebElementSource`. Forget about `proxy`.

We can acquire the driver from the `WebElementSource`. Let's write a new `Command` called `MyCommand` and see how the driver can be accessed:

```java
class MyCommand implements Command {

     T execute(SelenideElement proxy,
                WebElementSource locator,
                @Nullable Object[] args) throws IOException {
      
        return locator.driver()
            .executeJavaScript();
    }
}
```

Seems reasonable.

But let's back off a bit from using javascript, and let's think about what happens in general when we do anything in `Command::execute`.

Let's imagine we want to write a `Command` called `CountTo` which **does NOT interact with anything** simply counts how many times it was retried, and returns after an arbitrary number of retries were complete (remember it's a manual do-while in Invocation Handler, no magic).

How do we call a custom `Command` such as `CountTo`? `SelenideElement::execute` accepts any `Command<T>`.

```java
$("#foo").execute(new CountTo());
```

'execute' method name is mapped to `Execute`:

```java
public class Commands {
    ...
    add("execute", new Execute<>());
```

`Execute` has the following implementation:

```java
public class Execute<ReturnType> implements Command<ReturnType> {

  @Override
  public ReturnType execute(SelenideElement proxy, WebElementSource locator, @Nullable Object[] args) {
    Command<ReturnType> command = firstOf(args);
    try {
      return command.execute(proxy, locator, NO_ARGS);
    } catch (IOException e) {
      throw new RuntimeException("Unable to execute custom command", e);
    }
  }
}
```

You'd think at first sight that it is simply an adapter for accepting custom `Commmand`s.

You are right...almost. But there's a `catch` for `IOException`, so keep that in mind if you ever want to throw specifically an `IOException`.

So, what happens when you return something from `CountTo`?

The invocation handler will stop the do-while, and return control to you.

But what happens if an exception is thrown in `CountTo`?

If you throw something which is an instance of either an `Exception` or an `AssertionError` then it'll be retried.

So consider `CountTo` does this:

```java
@RequiredArgsConstructor
class CountTo implements Command<Integer> {

    private final int expected;

    // I'll increment this to expected to keep things simple
    int x = 0;

    @Nullable
    @Override
    public Integer execute(SelenideElement selenideElement, WebElementSource locator, @Nullable Object[] objects) throws IOException {
        if (x == expected) {
            return expected;
        } else {
            x++;
            throw new AssertionError("not yet " + expected);
        }
    }
}
```

Now we run `CountTo`, expecting that the third 'execution of it' will fail because it times out before it can return something (4000ms timeout, 200ms poll):

```java
open("dubioussite");
int res;
res = $(By.id("#foo")).execute(new CountTo(1));
System.out.println(res);
res = $(By.id("j_idt718:city")).execute(new CountTo(2));
System.out.println(res);
res = $(By.id("j_idt718:city")).execute(new CountTo(20));
System.out.println(res);
```

We get the following:
```
1
2

AssertionError: not yet 20
Screenshot: file:/bla/1657049479358.0.png
Page source: file:/bla/1657049479358.0.html
Timeout: 4 s.
Caused by: java.lang.AssertionError: not yet 20
```

So the first two passed, and the the third one failed as expected.

## Retrying javascript

Let's imagine we want to create something like an `Await`, and we execute some arbitrary js via it which returns a boolean telling us something that we want:

```java
@RequiredArgsConstructor
class Await implements Command<Integer> {


    @Nullable
    @Override
    public Integer execute(SelenideElement selenideElement, WebElementSource locator, @Nullable Object[] objects) throws IOException {
        boolean result = locator.driver()
            .executeJavaScript("return arbitraryJs();");
    }
}
```

Can we return the result?

If we want the polling to end then yes. Otherwise we have to throw.

So it becomes:

```java
@RequiredArgsConstructor
class Await implements Command<Integer> {


    @Nullable
    @Override
    public Integer execute(SelenideElement selenideElement, WebElementSource locator, @Nullable Object[] objects) throws IOException {
        boolean result = locator.driver()
            .executeJavaScript("return arbitraryJs();");
        if (result) {
            return true;
        } else {
            throw new AssertionError("not yet");
        }
    }
}
```

So we can now do this:

```java
open("thatsite");
$("#foo").execute(new Await());
```

Or even store a reference to an instance of `Await` as it is **stateless**:

```java
Await await = new Await();
open("thatsite");
$("#foo").execute(await);
$("#foo").execute(await);
```

And you can use it even in tests which use multiple drivers in one test:

```java
Await await = new Await();
SelenideDriver ann = ...;
SelenideDriver bob = ...;

ann.open("site-a");
ann.$("#foo").execute(await);

bob.open("site-b");
bob.$("#bar").execute(await);
```

The behavior itself is encapsulated in a `Command`, you don't need a reference to the driver, because you'll get one.

Also, the polling is done by Selenide, not you.

Make sure to understand that: in the example we assumed that `WebElementSource`'s driver is not null, we didn't make an explicit null check.

And also if js error arises from running something like this:

```javascript
returnnnnnn I 1 + 66as MEAN;;;. YOU...MISTYPE,kre SOMETHING
```

That can mean a `org.openqa.selenium.JavascriptException` will come from Selenium (assuming it does not crash the whole system).

Which has the following inheritance hierarchy:

1. java.lang.Object
2. java.lang.Throwable
3. java.lang.Exception
4. java.lang.RuntimeException
5. org.openqa.selenium.WebDriverException
6. org.openqa.selenium.JavascriptException

And as such, it'll be polled, but it will always throw, and it'll reach timeout and the Invocation Handler will throw and take a screenshot.

## Chaining

Let's revisit the `Await`:

```java
Await await = new Await();
open("thatsite");
Boolean x = $("#foo").execute(await);
```

It returns an Boolean so if you want to do something with `foo` again, you have to reacquire it or save it as a local variable:

```java
Await await = new Await();
open("thatsite");
Boolean x = $("#foo").execute(await);
$("#foo").click();
```

Can we create a chainable `Await`?

Of course. We have to return the dynamic proxy instance, that we get as an arg, and change our return type:


```java
@RequiredArgsConstructor
class Await implements Command<SelenideElement> {


    @Nullable
    @Override
    public SelenideElement execute(SelenideElement selenideElement, WebElementSource locator, @Nullable Object[] objects) throws IOException {
        boolean result = locator.driver()
            .executeJavaScript("return arbitraryJs();");
        if (result) {
            return selenideElement;
        } else {
            throw new AssertionError("not yet");
        }
    }
}
```

So now we can do:

```java
Await await = new Await();
open("thatsite");
$("#foo")
    .execute(await)
    .click();
```

Quite clumsy still, but if this is acceptable to you, it is fine. If it is not then it's fine too.
