# Can I communicate to fellow testers my intent via the type system?

Executing custom commands can become quite a spaghetti (just like all Selenide/Selenium/Cypress etc. code I've seen, including mines too).

```java
Await await = new Await();

var foo = $("#foo");
var bar = $("#bar");
var doodad = $("#doodad");

open("thatsite");

foo.click();
bar
    .execute(await)
    .shouldBe(visible);

foo.click();
doodad
    .execute(await)
    .shouldHave(text("qwerty"));
```

Can we communicate intent better somehow?

## Overview

I don't know. It depends on taste, (and performance requirements, because more abstractions means more instantiations and more indirections).

You are probably more capable than me, and can come up with an actual solution.

## Updated

One way to communicate the intent is to tell others, that something is a potential target of an update (the target of an event), so before interacting with it, some precaution is advised.

Let's imagine we write a functional interface like this:

```java
interface Updated {
        
    static Updated updated(SelenideElement updated) {
        return () -> updated.execute(new Await());
    }
    
    SelenideElement awaitUpdate();
}
```

Our code is still a mess, but it now has a bit of information for others about what is actually happening.

```java
var foo = $("#foo");
var bar = updated($("#bar"));
var doodad = updated($("#doodad"));

open("thatsite");

foo.click();
bar
    .awaitUpdate()
    .shouldBe(visible);

foo.click();
doodad
    .awaitUpdate()
    .shouldHave(text("qwerty"));
```

## Updater

Can we create something like this for the event's source?

It's much more trickier, because we have no control over Selenide. You can fire `click` and finish your code. We cannot force you to do anything after a click using pure `SelenideElement`.

You are also a bit out of luck due to type system awkwardness.

```java
interface Updater {

    static Updater updater(SelenideElement updater) {
        // ERRROR, target method is generic!!!!!
        return f -> f.apply(updater);
    }

    <T> T thenAwait(Function<SelenideElement, T> f);
}
```

Also, you cannot differentiate `Function<String, Integer>` from `Function<String, String>` due to type erasure.

If we are fine enough with effects, and don't really want to over do it, we can get away with:

```java
interface Updater {
        
    static Updater updater(SelenideElement updater) {
        return f -> {
            f.accept(updater);
            updater.execute(new Await());
            return null;
        };
    }

    Updater thenAwait(Consumer<SelenideElement> f);
}
```

So we end up with the clunky code below:

```java
var foo = updater($("#foo"));
var bar = $("#bar"));
var doodad = $("#doodad");

open("thatsite");

foo.thenAwait(SelenideElement::click);
bar
    .shouldBe(visible);

// or instead of method reference
foo.thenAwait(e -> e.click());
doodad
    .shouldHave(text("qwerty"));
```

Which does not stop you from doing things like this:

```java
var foo = Updater.of($("#foo"));
var bar = $("#bar"));
var doodad = $("#doodad");

open("thatsite");

foo.thenAwait(e -> {
    e.click();
    e.click();
    e.click();
    e.click();
    e.click();
});
bar
    .shouldBe(visible);

foo.thenAwait(e -> e.click());
doodad
    .shouldHave(text("qwerty"));
```
