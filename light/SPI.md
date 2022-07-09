# What is inject? - SPI

Remember `Plugins.inject`?

```java
public class ElementFinder extends WebElementSource {

    // Don't worry. We'll see later what inject is all about.
    WebElementSelector elementSelector = inject(WebElementSelector.class);
}
```

Now we'll talk about it.

## Overview

It is simply SPI. `Plugins` is a utility class.

If you need a service somewhere you use `Plugins.inject`.

`Plugins.inject` handles the instantiation and caching of the service (so it gets loaded only once).

This is needed, because some users might want to customize some services.

## Service locator

Plugins is basically a Singleton which stores a hashmap. Inject is a synchronized get from a hashmap.

```java
public class Plugins {

    static final Map<Class<?>, Object> cache = new ConcurrentHashMap<>();

    public static synchronized <T> T inject(Class<T> klass) {
        T plugin = (T) cache.get(klass);
        if (plugin == null) {
            plugin = loadPlugin(klass);
            cache.put(klass, plugin);
        }
        return plugin;
    }
}
```

It is like a `computeIfAbsent` or a memoization of `loadPlugin` results.

The `loadPlugin` is simply an abstraction over [Java Service Provider Interface](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html). Plugins is also able to load the default implementations to handle the edge case where the user does not supply her/his own implementation:

```java
public class Plugins {
    private static <T> T loadPlugin(Class<T> klass) {
        Iterator<T> loader = ServiceLoader.load(klass)
                                            .iterator();
        if (!loader.hasNext()) {
            T defaultPlugin = getDefaultPlugin(klass);
            return defaultPlugin;
        }
        T implementation = loader.next();
        return implementation;
    }
}
```

The `getDefaultPlugin` simply instantiates an object which has interface T. But how does it figure out which actual class to load? It simply reads it from src/main/resources/META-INF/defaultservices.
It uses basic Reflection for instantiation.

```java
private static <T> T getDefaultPlugin(Class<T> klass) {
    String resource = "/META-INF/defaultservices/" + klass.getName();
    URL file = Plugins.class.getResource(resource);
    if (file == null) {
      throw ...
    }

    String className = readFile(file).trim();
    try {
      // that method simply creates a new instance via Reflection
      return instantiate(className, klass);
    }
    catch (Exception e) {
      throw ...
    }
  }
```

So we can conclude that `Plugins` is a hashmap cached SPI + Reflection powered object instantiation mechanism.

