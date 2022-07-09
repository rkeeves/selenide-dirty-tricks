# How do you invoke the right command if it is not statically typed?

Calling `$().setValue("a")` ends up callings the `Commands` singleton.

But how does the `Commands` know how to set a value?

## Overview

The invocation handler passes the method name, all arguments and the 'locator' and many other info to the `Commands`.

`Commands` does a hashmap lookup based on a string key (aka method name like 'click').

Then `Commands` applies the function-like `Command` on the arguments which are not statically type checked due to Dynamic Proxy.

## Command lookup by Commands singleton

When a method is called on a `SelenideElement` the proxy calls an `execute` method of the singleton.

```java
class SelenideElementProxy {

    public Object invoke(Object proxy,
                        Method method,
                        @Nullable Object... args)
                        throws Throwable {
      return Commands.getInstance()
        .execute(proxy, webElementSource, method.getName(), args);
    }
}
```

`Commands` has all available `Command`s in a string keyed hashmap. It simply looks up by method name something. Unchecked casts it, then calls the `Command::execute` method with the args (whose types are not deducible at compile time, so they are `Object`s).

```java
class Commands {

    Map<String, Command<?>> commands = new ConcurrentHashMap<>();

    public <T> T execute(Object proxy,
                         WebElementSource webElementSource,
                         String methodName,
                         Object[] args) throws IOException {
        Command<T> command = (Command<T>) commands.get(methodName);
        return command.execute((SelenideElement) proxy, webElementSource, args);
    }
}
```

`Command` itself is simply an interface:

```java
interface Command<T> {
    
    T execute(SelenideElement proxy,
                WebElementSource locator,
                Object[] args) throws IOException;
}
```

How does the string keyed hashmap of `Command`s get populated? In the constructor:

```java
class Commands {

    Map<String, Command<?>> commands = new ConcurrentHashMap<>();
    
    protected Commands() {
        addFindCommands();
        ...
    }

    private void addFindCommands() {
        add("find", new Find());
        add("$", new Find());
        ...
    }

    public final void add(String method, Command<?> command) {
        commands.put(method, command);
    }
}
```

So, the `Commands` is basically a singleton hashmap responsible for instantiating, looking up and executing `Command`s.

The `Commands` singleton looks up and executes a `Command` on behalf of the invocation handler.

When the `Command` returns a value, the singleton returns back this value to the original caller, the invocation handler.
