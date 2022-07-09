# Can I use a custom Condition for waiting?

Remember the things we did with the custom `Command`?

```java
Await await = new Await();
$("#foo").execute(await);
```

It almost looks like a `Condition`.

Can we create a custom `Condition`? It'd look way cooler!

```java
Steady await = new Steady();
$("#foo").shouldBe(steady);
```

## Overview

The element is missing will mess up the truth table.

Aka you cannot use it in all scenarios reliably.

## Missing element satisfies

Due to how `Condition`'s work when the element is missing the `Condition`'s actual predicate does not run, but instead you'll get into a branch and your `missingElementSatisfiesCondition` method will be called.

```java
...
    if (element == null) {
      if (check.missingElementSatisfiesCondition()) {
        return null;
      }
      throw createElementNotFoundError(check, lastError);
    }
```

This method has no inputs, therefore you cannot get a reference to the driver. Therefore you cannot interact with Selenium.

Take a look at the truth table below (I wrote out all the rows, but the last two could be collapsed into one of course):

| ready | exists | result   |
| :--:  | :--:   | :--      |
| T     | T      | ACCEPT   |
| F     | T      | REJECT   |
| T     | F      | ?        |
| F     | F      | ?        |

You must decide a value for '?'. (aka `missingElementSatisfiesCondition`)


Let's say you return `true` from `missingElementSatisfiesCondition`:

| ready | exists | result   |
| :--:  | :--:   | :--      |
| T     | T      | ACCEPT   |
| F     | T      | REJECT   |
| T     | F      | ACCEPT   |
| F     | F      | ACCEPT   |

You'll accept blindly when the element does not exist.

Let's say you return `false` from `missingElementSatisfiesCondition`:

| ready | exists | result   |
| :--:  | :--:   | :--      |
| T     | T      | ACCEPT   |
| F     | T      | REJECT   |
| T     | F      | REJECT   |
| F     | F      | REJECT   |

You'll reject blindly when the element does not exist.

Yes. `$("body")` exists, but think about the bigger picture.

Suppose you start writing widget wrappers for reuse. Let's suppose it is `HamburgerOrWhateverMenu`.

```java
class HamburgerOrWhateverMenu {
    private final SelenideElement root;

    public HamburgerOrWhateverMenu someCoolChainedThing(String s) {
        root.click();
        root.shouldHave(text(s));
        return root;
    }
}
```

What do you `$().should` when the widget does not exist?

You cannot `Selenide.$("body")` because that'll couple your widget class to a thread local driver.


```java
class HamburgerOrWhateverMenu {
    private final SelenideElement root;

    public HamburgerOrWhateverMenu someCoolChainedThing(String s) {
        root.click();
        Selenide.$("body").whatever();
        root.shouldHave(text(s));
        return root;
    }
}
```

You cannot do `$().ancestor("whatever")` because root does not exist.


```java
class HamburgerOrWhateverMenu {
    private final SelenideElement root;

    public HamburgerOrWhateverMenu someCoolChainedThing(String s) {
        root.click();
        root.ancestor("whatever").whatever();
        root.shouldHave(text(s));
        return root;
    }
}
```

So, as far as I'm concerned, `Condition` and `Command` are different in the sense, that `Condition` cannot acquire a reference to a driver in case the element is missing / cannot be located.
