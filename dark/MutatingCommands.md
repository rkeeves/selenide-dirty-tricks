# Can I override Click?

We know how we can click in Selenide:

```java
$("#foo").click();
```

But can we change the associated `Command`?

## Overview

Mutate the hashmap in `Commands`.

## Mapping a new value to a key in a hashmap

Imagine that for some odd reason we decide to create a new `Command` called `CrazyClick`.

`CrazyClick` will do what `Click` does, but will execute arbitrary code before, like this:

```java
class CrazyClick extends Click {

    public Void execute(SelenideElement proxy,
                        WebElementSource locator,
                        @Nullable Object[] args) {
        arbitrary_code
        return super.execute(proxy, locator, args);
    }
}
```

But how do we 'hook' it up?

By 'hook' I mean, how will `$("#foo").click()` now that it must run `CrazyClick`?

Let's think back and remember how the `Commands` singleton is responsible for resolving `Command`s.

If we mutate the hashmap, we can create a new mapping by:

```java
Commands.getInstance().add("click", new CrazyClick());
```

So now when we do:

```java
$("#foo").click();
```

The invocation handler will call `Commands.getInstance().execute(...)`.

And `Commands` will get by the string `click` an instance of `CrazyClick` instead of `Click`.

## Hook

Imagine we create a 'Hook' like so:

```java
@RequiredArgsConstructor(staticName = "hook")
class Hook<T> implements Command<T> {

    private final Command<T> hooked;

    public Void execute(SelenideElement proxy,
                        WebElementSource locator,
                        @Nullable Object[] args) {
        arbitrary_code
        return hooked.execute(proxy, locator, args);
    }
}
```

If you replace all commands in the hashmap:

```java
Commands.getInstance().add("click", hook(new Click()));
Commands.getInstance().add("cluck", hook(new Cluck()));
Commands.getInstance().add("clack", hook(new Clack()));
```

You can retroactively add basically a callback to all Selenide `Command`s.

The `Hook` instance itself is instantiated for each different command, so be careful about state, because multiple drivers from multiple parallel tests can and will access the `Command`s.
