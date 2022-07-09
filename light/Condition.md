# How does a condition work?

In Selenide you can write `should` which accepts a `Condition` and retries it.

It's simply:

```java
$("#foo").should(visible);
```

So how does a `Condition` perform its task?

## Overview

Both `Condition` and `CollectionCondition` form two separate inheritance trees.

A subclass of `Condition` is almost like a function 'WebElement -> CheckResult'.

A subclass of `CollectionCondition` cannot be described as a simple function.

`CollectionCondition`s which have 'anyOrder' in their name simply do a nested for which can have some unexpected behavior coupled with `String::contains`.

## Should

A `should` is a mapping for a `Command` of type `Should`.

It gets an array of `Condition`s and does a for each iteration over them:

```java
class Should implements Command<SelenideElement> {

  public SelenideElement execute(SelenideElement proxy, WebElementSource locator, @Nullable Object[] args) {
    for (Condition condition : argsToConditions(args)) {
      locator.checkCondition(prefix, condition, false);
    }
    return proxy;
  }
}
```

So what really makes the check is the abstract class `WebElementSource`.

The corresponding code which actually does the task is:

```java
abstract class WebElementSource {
    
    private WebElement checkConditionAndReturnElement(String prefix, Condition condition, boolean invert) {
    Condition check = invert ? not(condition) : condition;

    Throwable lastError = null;
    WebElement element = null;
    CheckResult checkResult = null;
    try {
      element = getWebElement();
      checkResult = check.check(driver(), element);

      if (checkResult.verdict() == ACCEPT) {
        return element;
      }
    }
    catch (WebDriverException | IndexOutOfBoundsException | AssertionError e) {
      lastError = e;
    }

    if (lastError != null && Cleanup.of.isInvalidSelectorError(lastError)) {
      throw Cleanup.of.wrapInvalidSelectorException(lastError);
    }

    if (element == null) {
      if (check.missingElementSatisfiesCondition()) {
        return null;
      }
      throw createElementNotFoundError(check, lastError);
    }
    else if (invert) {
      throw new ElementShouldNot(driver(), description(), prefix, condition, checkResult, element, lastError);
    }
    else {
      throw new ElementShould(driver(), description(), prefix, condition, checkResult, element, lastError);
    }
  }
}
```

If we rewrite into a simpler form it becomes:

```java
abstract class WebElementSource {
    
    private WebElement check(Condition condition) {
        ErrorNoise lastError;
        try {
            element = getWebElement();
            checkResult = condition.check(driver(), element);

            if (checkResult.verdict() == ACCEPT) {
                return element;
            }
        } catch (ErrorNoise e) {
            lastError = e;
        }
        if (element == null) {
            if (condition.missingElementSatisfiesCondition()) {
                return null;
            }
            throw createElementNotFoundError(condition, lastError);
        }
        throw new ElementShould(driver(), description(), prefix, condition, checkResult, element, lastError);
  }
}
```

From this we can make some observations.

If `getWebElement` does not throw and `Condition` passes then it returns `WebElement` (which will be later swallowed by a helper function to not reach the outside world).

If `getWebElement` does not throw but the `Condition`s predicate fails then an error will be thrown and caught. Then at the bottom of the body a new `ElementShould` instance will be created. The `ElementShould` will create a String message for the super constructor within its ctor:

```java
public class ElementShould extends UIAssertionError {
  private static final ElementDescriber describe = inject(ElementDescriber.class);

  public ElementShould(Driver driver, String searchCriteria, String prefix,
                       Condition expectedCondition, CheckResult lastCheckResult,
                       WebElement element, Throwable lastError) {
    super(
      String.format("Element should %s%s {%s}%nElement: '%s'%s",
        prefix, expectedCondition, searchCriteria,
        describe.fully(driver, element),
        actualValue(expectedCondition, driver, element, lastCheckResult)
      ),
      expectedCondition,
      lastCheckResult == null ? null : lastCheckResult.actualValue(),
      lastError);
  }
}
```

If `getWebElement` does throw (and `element` local variable will remain null), then depending on whether the `Condition::missingElementSatisfiesCondition` is true two things can happen:
- returns without problem
- throws

Remember: the 'retry until timeout' is done by the invocation handler.

## Condition

A `Condition` is an abstract class. It represents basically a predicate about one `WebElement`.

Its internal state is the following:

```java
class Condition {
    String name;
    boolean missingElementSatisfiesCondition;

    public Condition(String name) {
        this(name, false);
    }
}
```

The `name` is used for logging purposes.

The `missingElementSatisfiesCondition` is a boolean flag, that we've discussed earlier.

This abstract class's compilation unit also stores static references and factory methods to all `Condition`s.

So when you write:

```java
$("#foo").shouldBe(visible);
```

You can statically import the following reference:

```java
public abstract class Condition {
  ...
  public static final Condition visible = new Visible();
}
```

A minor note: `Condition.visible` no longer has type `Visible` visible for you, but instead it is simply now a `Condition` hiding the original type `Visible`.

## String based Conditions

`Condition.text` is a static factory method for creating an instance of type `Text`:

```java
class Text extends TextCondition {

    Text(final String text) {
        super("text", text);
        if (isEmpty(text)) {
            throw new IllegalArgumentException("Argument must not be null or empty string. " +
            "Use $.shouldBe(empty) or $.shouldHave(exactText(\"\").");
        }
    }

    boolean match(String actualText, String expectedText) {
        // Focus here :)
        return Html.text.contains(actualText, expectedText.toLowerCase(Locale.ROOT));
    }
 
    String getText(Driver driver, WebElement element) {
        return "select".equalsIgnoreCase(element.getTagName()) ?
        getSelectedOptionsTexts(element) :
        element.getText();
    }

    String getSelectedOptionsTexts(WebElement element) {
        List<WebElement> selectedOptions = new Select(element).getAllSelectedOptions();
        return selectedOptions.stream().map(WebElement::getText).collect(joining());
    }
}
```

What we're interested is this line:

```java
return Html.text.contains(actualText, expectedText.toLowerCase(Locale.ROOT));
```

`Html.text` handles string transformations and comparisons.

Because the site's document is full of whitespaces and other irregularities, it has methods for handling these:

```java
public class Html {

  private static final Pattern REGEX_SPACES = Pattern.compile("[\\s\\n\\r\u00a0]+");
  public static Html text = new Html();

  public boolean contains(String text, String subtext) {
    return reduceSpaces(text.toLowerCase(Locale.ROOT))
        .contains(reduceSpaces(subtext.toLowerCase(Locale.ROOT)));
  }

  String reduceSpaces(String text) {
    return REGEX_SPACES.matcher(text).replaceAll(" ").trim();
  }
}
```

Let's rewrite it `Text::match` with two made up types `LeftRight` and `Transforms` to illustrate what happens:

```java
boolean match(String expected) {
    return LeftRight.of(e.getText(), expected)
        .mapBoth(Transforms.lowerCase())
        .mapBoth(Transforms.reduceSpaces())
        .mapBoth(Transforms.trim())
        .merge((actual, expected) -> actual.contains(expected));
}
```

My made up code was to illustrate that the concise name `text` is a shorthand for an otherwise lengthy function composition.

## CollectionCondition

`CollectionCondition` is an abstract class.

They are predicates over a list of WebElements.

We'll take a look at `TextsInAnyOrder` and what it does.

Let's say we have strings: "Themes", "Resources", "Templates",  "v8".

The below passes as expected:

```java
["Themes", "Resources", "Templates",  "v8"]

$$("li.topbar-submenu > a")
    .shouldHave(textsInAnyOrder("Themes", "Templates", "Resources", "v8"));
```

The below passes too, although it is not as straightforward why:

```java
["Themes", "Resources", "Templates",  "v8"]

$$("li.topbar-submenu > a")
    .shouldHave(textsInAnyOrder("He", "ate", "8", "PLATES"));
```

`TextsInAnyOrder` inherits from `ExactTexts` (which is a subclass of `CollectionCondition`).

```java
public class TextsInAnyOrder extends ExactTexts {
```

The `test` method is used to evaluate a List of Webelements:

```java
public boolean test(List<WebElement> elements) {
    if (elements.size() != expectedTexts.size()) {
      return false;
    }

    List<String> elementsTexts = elements.stream().map(WebElement::getText).collect(Collectors.toList());

    for (String expectedText : expectedTexts) {
      boolean found = false;
      for (String elementText : elementsTexts) {
        if (Html.text.contains(elementText, expectedText)) {
          found = true;
        }
      }
      if (!found) {
        return false;
      }
    }
    return true;
}
```

It goes through the expected strings in order, and checks whether the WebElement's text (lowercased, trimmed) contain the expected.

Aka the following things are exactly the same:
```java
$$().shouldHave(textsInAnyOrder("The", "The"));
$$().shouldHave(textsInAnyOrder("the", "the"));
$$().shouldHave(textsInAnyOrder("the", "t"));
```

Now let's revisit the case of "Themes", "Resources", "Templates",  "v8":

```java
$$("li.topbar-submenu > a")
    .shouldHave(textsInAnyOrder("He", "ate", "8", "PLATES"));
```

Step through each expected word and remember my made up function composition from earlier:

```java
boolean match(String expected) {
    return LeftRight.of(e.getText(), expected)
        .mapBoth(Transforms.lowerCase())
        .mapBoth(Transforms.reduceSpaces())
        .mapBoth(Transforms.trim())
        .merge((actual, expected) -> actual.contains(expected));
}
```

Below I listed which original expected substring matched with which actual string.

Remember: if an actual was matched once it can be 'reused' for further matches.

```
'He' matched 'Themes'
'ate' matched 'Templates'
'8' matched 'v8'
'PLATES' matched 'Templates'
```

So the result, despite looking strange, is actually right.
