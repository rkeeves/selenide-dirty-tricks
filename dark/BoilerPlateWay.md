# Can I create my own SelenideElement without Dynamic Proxy?

We saw that taking the Dynamic Proxy route with custom interfaces has its limitations, and it is quite clunky.

Can we create a new type without this Dynamic Proxy hassle?

## Overview

Yes, a simple wrapper.

If you create a simple wrapper, though, you'll write a ton of boilerplate.

## Throw more abstractions at the problem

Imagine we create a `CrazyElement` like so:

```java
@RequiredArgsConstructor(staticName = "crazy")
class CrazyElement {

    private final SelenideElement e;

    public void click() {
        arbitrary code
        e.click();
    }
}
```

In code we can use:

```java
CrazyElement e = crazy($("#foo"));
```

This seems reasonable.

## Many methods

If you want to mimic all methods from `SelenideElement`, you have to implement all of them:

```java
@RequiredArgsConstructor(staticName = "crazy")
class CrazyElement {

    private final SelenideElement e;

    public void click() {
        arbitrary code
        e.click();
    }

    public void clack() {
        arbitrary code
        e.clack();
    }

    public void cluck() {
        arbitrary code
        e.cluck();
    }
}
```

And also, you must manually construct new wrappers when the recursive data structure is grown:

```java
@RequiredArgsConstructor(staticName = "crazy")
class CrazyElement {

    private final SelenideElement e;

    public CrazyElement $(By by) {
        arbitrary code
        return new CrazyElement(e.$(by));
    }
}
```

And don't forget that methods have many overloads:

```java
@RequiredArgsConstructor(staticName = "crazy")
class CrazyElement {

    private final SelenideElement e;

    public CrazyElement $(By by) {
        arbitrary code
        return new CrazyElement(e.$(by));
    }

    public CrazyElement $(String css) {
        arbitrary code
        return new CrazyElement(e.$(css));
    }
}
```

It becomes a huge 500+ lines class fast.

Also, I personally don't recommend directly implementing interface `SelenideElement`.

Firstly, due to the recursive growth, this will force you in case of `$` to lose information by returning `SelenideElement` instead of `CrazyElement`.

Secondly, as we saw earlier, Selenide must do some crazy type casts, and compile time checks are off. So I cannot prove that if a wrapper takes up `SelenideElement` noone will do unappropriate things to it.

If you think this is acceptable it's fine, if you don't think it is acceptable to you it is fine too. What matters is that you know what it entails.
