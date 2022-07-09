# How does it find Y inside X?

When we do:

```java
Selenide.$("#foo").$(".bar");
```

we know that it is *lazy enough* for us.

But what happens when we do:

```java
Selenide.$("#foo").$(".bar").click();
```

How does it find '.bar' inside '#foo'?

## TLDR

If it has no parent: calls the driver with its field `By`.

If it has parent: it acquires `WebElement` from parent and calls `find` on that with its field `By`.

Although, things get complicated due to a lot of inheritance and helper functions.

## List

In CS-101 we learned that:

```java
(cons 1 (cons 2 (cons 3 nil)))
```

can be referred to as a list in some way.

In Java it looks like this.

```java
interface Node {

    String getName();

    static Node empty() {
        return () -> "";
    }

    static Node cons(String s, Node tail) {
        return () -> s + tail.getName();
    }
}

psvm {

    cons("A",
        cons("B",
            cons("C",
                empty())));
}
```

There's an other form which takes advantage of `null`. Most of us are more familiar with this:

```java
class Node {
    
    String name;

    Node aminull;

    static Node first(String s) {
        return new Node(s, null);
    }

    Node child(String s) {
        return new Node(name, this);
    }
}

psvm {
    first("h").child("i");
}
```

This can be referred to as a list too.

We can iterate over them, or recurse on them.

## Finding relatively

Selenide's `ElementFinder` finds 'relatively' by using a list.

It is made possible by keeping a reference to a 'parent'.

```java
public class ElementFinder extends WebElementSource {
    WebElementSource parent;
}
```

Unfortunately for us, inheritance gets involved.

To simplify things I rewrote it into a more simpler form:

```java
public class ElementFinder {

    ElementFinder parent;
}
```

Now imagine you do a `click`. We know that it will be turned into `Click` at the end. `Click` needs a `WebElement` so let's see how it acquires it.

Click is a bit complicated so I simplified it to fit our basic needs:

```java
public class Click implements Command<Void> {
  
    public Void execute(SelenideElement proxy, WebElementSource locator, Object[] args) {
   
    WebElement webElement = locator.getWebElement();

    ...

    return null;
  }
```

`ElementFinder` - as all subclasses of `WebElementSource` - can get actual `WebElement`s.

It does it by using a `WebElementSelector`. (We'll talk about that `inject` thing, but in short it is for SPI).

```java
class ElementFinder {

     WebElementSelector elementSelector = inject(WebElementSelector.class);
    
    ElementFinder parent;

    public WebElement getWebElement()
    throws NoSuchElementException,
            IndexOutOfBoundsException
    {
        return elementSelector.findElement(driver, parent, criteria);
    }
}
```

Once again, through a lot of simplification we get to this:

```java
class WebElementSelector {
    public WebElement findElement(Driver driver,
                                WebElementSource parent,
                                By selector) {
            SearchContext context = parent == null ? driver.getWebDriver() : parent.getWebElement();
            ...
        }
}
```

So as we can see the search context is the driver in the base case, otherwise the parent's `WebElement`.

Let's rewrite `ElementFinder` into an almost unrecognizeably simplified form called `LazyElem`.

```java
class LazyElem {

    Driver driver;

    By by;
    
    LazyElem parent;

    public WebElement getWebElement()
    throws NoSuchElementException,
            IndexOutOfBoundsException
    {
        if (parent == null) {
            return driver.find(by);
        } else {
            return parent.getWebElement().find(by);
        }
    }
}
```

This is an overly simplified form.

I just wanted to illustrate that this is indeed a list.

How is it guaranteed that parent has the same driver as child?

Somewhat like this (although the realiy is much more complex):

```java
class LazyElem {

    final Driver driver;

    final By by;
    
    final LazyElem parent;

    //Req Args Ctor
    private LazyElem(boilerplate) {
        boilerplate
    }

    static LazyElem first(Driver driver, By by) {
        return new LazyElem(driver, by, null);
    }

    LazyElem next(By by) {
        return new LazyElem(this.driver, by, this);
    }

    WebElement getWebElement() {
        if (parent == null) {
            return driver.find(by);
        } else {
            return parent.getWebElement().find(by);
        }
    }
}
```

Through construction rules a `LazyElem` is either:
- a root in which case find by the driver
- a child in which case find by the webelement (which is acquired from parent)
