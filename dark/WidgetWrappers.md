# Can I represent widgets as wrappers?

We have all these cool widgets and gadgets.

Instead of fiddling around with `SelenideElement`, can we create abstractions for these widgets?

## Overview

Yes. But it is not that straightforward as one would hope.

## Pages, fragments, and widgets

Modeling the application's UI from the test's point of view is hard.

The `Page Object Model` comes up a lot in testing circles. I'll not discuss it at all though, as I don't understand them: for me `Page Object Model` is too vague.

Web applications are 'modeled' by the developers too.

Taking a look at their approach might interest us, because they way developers organize their code and how the testers organize their code has a limited, but still existing, correspondence.

I'll talk about bulky jsf applications. I will not deal with SPAs and other approaches.

Developers create pages. A page consists of UI elements. A page, most of the time, has no clear goal, it is just a container for UI elements. A page is never considered complete. A page can and will be modified according to user needs. It is a constantly changing rag-tag collection of UI elements.

Developers want to manage this ever changing page, so they use fragments and widgets.

`Fragment`s are like header, footer, navbar. They are almost like a page, in the sense that there's no clear goal for them, they keep changing. They are simply a recurring pattern in pages. The developers recognize these recurring patterns, and to evade code reuse, they 'cut out' the recurring pattern of the pages, create a self-contained fragment which contains the recurring pattern, and substitute this newly created fragment with all occurrences of the pattern. This helps maintainability in many ways. One really easy example is additive operations. Imagine all pages' menu bar needs a new menu item. If the developers had 20 seperate pages each with its own menu, then the 'addition of a new menu item' must be done in 20 places. On the other hand, with fragments, they need to modify only one fragment, and all pages that incorporate that fragment will be changed.

Pages tend to act as sets of fragments. By this I mean, that I personally never saw a page with more than one footer, or more than one instance of a topbar (it can have multiple menus in it, but there's only one fragment). A Page either has a fragment or not. The need to think about equality between two instances of the same fragment on one page simply never arises. The fragments are also maintained by the developers in-house. They have full control over them.

`Widget`s are like special input fields the user can type into, or menubar, or complicated tables. These most of the time come from a library outside of the control of developers. The widgets's main purpose - from the developer's point of view - is NOT reusing his/her own code. The widget is written by a third party (or in-house by a different team). Therefore the developer's goal is to use others pre-written code. The difference is huge between managing my team's code vs. relying on other parties' code which I have no control over. The widgets are pre written units, which can be customized and used in a page or fragment. A page can contain more than one instance of one type of widget (like two different instances of the same `Dropdown` widget). Due to this widgets are more tricky to locate for the testers.

A good `Widget` has to achieve a lot of goals.
- it 'must look nice' for the end user
- it 'must be configurable' for the developer

If a `Widget` can achieve these goals, it is considered to be a good widget.

As you can see the testers' needs are not on the list.

Some `Widget`s require pretty intricate interactions via Selenium. Instances of the same `Widget` can behave quite differently with minimal effort from the developer. For example: by changing essentially a boolean flag, the developer can delay the loading of options in a select until the user opens the options list (via ajax for example).

Tests which must frequently interact with complex widgets naturally want to reuse their code. The main driving force behind this - in my opinion - is NOT code reuse. The test aims to verify the developer's work. Interacting with a third party (the developers of the UI library) makes tests lengthy, and blurs together the verification of the developer's work with essentially non relevant boilerplate.

Testers want to abstract away interactions with a third party `Widget`, which is outside of the scope of the test.

## SelectDoodad experiment #0

Imagine we have a reoccuring select like doodad.

Its handling is quite complex so we want to encapsulate it.
*(I'll keep it simple though as the problem I want to highlight has nothing to do with a concrete scenario. So if visibility is too simple for you, imagine that the options are stored in an animated div, where the ul is lazily loaded by ajax, and the div is at the tail of the body, despite it being semantically the select's child.)*

So, let's create a reusable `SelectDoodad` with visibility.

```java
class SelectDoodad {

    SelenideElement root;

    SelectDoodad shouldBeVisible() {
        root.shouldBe(visible);
        return this;
    }

     SelectDoodad shouldNotBeVisible() {
        root.shouldNotBe(visible);
        return this;
    }
}
```

Ok. Now let's add enabled, which can be determined by a child's enabled.

```java
class SelectDoodad {

    SelenideElement root;

    SelectDoodad shouldBeVisible() {
        root.shouldBe(visible);
        return this;
    }

     SelectDoodad shouldNotBeVisible() {
        root.shouldNotBe(visible);
        return this;
    }

    SelectDoodad shouldBeEnabled() {
        root.$(".some").shouldBe(visible);
        return this;
    }

    SelectDoodad shouldNotBeEnabled() {
        root.$(".some").shouldNotBe(enabled);
        return this;
    }
}
```

It has a selected value too, which - due to how the UI library handles it - is a text of a label.

But we want to differentiate between text and exact text.

```java
class SelectDoodad {

    SelenideElement root;

    SelectDoodad shouldBeVisible() {
        root.shouldBe(visible);
        return this;
    }

     SelectDoodad shouldNotBeVisible() {
        root.shouldNotBe(visible);
        return this;
    }

    SelectDoodad shouldBeEnabled() {
        root.$(".some").shouldBe(visible);
        return this;
    }

    SelectDoodad shouldNotBeEnabled() {
        root.$(".some").shouldNotBe(enabled);
        return this;
    }

    SelectDoodad shouldHaveSelected(String value) {
        root.$(".text").shouldHave(text(value));
        return this;
    }

    SelectDoodad shouldHaveSelectedExact(String value) {
        root.$(".text").shouldHave(exactText(value));
        return this;
    }
}
```

This will quickly grow out of hand (imagine 50 types like this for buttons, menus etc.).

## SelectDoodad experiment #1

Let's see whether opening it up a bit for Selenide's Condition can help.

```java
class SelectDoodad {

    SelenideElement root;

    SelectDoodad shouldBeVisible(Condition condition) {
        root.shouldBe(condition);
        return this;
    }

    SelectDoodad shouldBeEnabled(Condition condition) {
        root.$(".some").shouldBe(condition);
        return this;
    }

    SelectDoodad shouldHaveSelected(Condition condition) {
        root.$(".text").shouldHave(condition);
        return this;
    }
}
```

Less methods, which sounds good. But the user might do this:

```java
SelectDoodad x;
x.shouldBeVisible(text("A"));
```

We could say that he probably won't because of the methodname. Let's see a more serious problem:

```java
SelectDoodad x;
x.shouldHaveSelected(value("A"));
```

The above line is perfectly fine at compile time.

To be able to assert on the selected value, the user must know about the implementation.

## SelectDoodad experiment #2

Let's just focus on solving the text issue.

How about manually providing overloads for concrete types of `Condition` like `Text` and `ExactText` etc.?

```java
class SelectDoodad {
    
    SelenideElement root;

    SelectDoodad shouldHaveSelected(Text condition) {
        return shouldHaveSelected(condition);
    }

    SelectDoodad shouldHaveSelected(ExactText condition) {
        return shouldHaveSelected(condition);
    }

    private SelectDoodad shouldHaveSelected(Condition condition) {
        root.$(".text").shouldHave(condition);
        return this;
    }
}
```

A bit bulky, but a more concerning problem is that we get a compile time error on this:

```java
SelectDoodad e;
e.shouldHaveSelected(text("A"));
```

This is because the static factory method returns the superclasses type `Condition` hiding away the concrete type `Text`:

```java
class Condition {

    public static Condition text(String text) {
        return new Text(text);
    }
}
```

The root cause is that predicates, value suppliers, and error message generation are bound together in a `Condition`.

## SelectDoodad experiment #3

Let's bring a new abstraction `MustString` and `MustBool`. They are basically factories.

```java

public interface MustBool {

    Condition visible();

    Condition disabled();
}

public interface MustString {

    Condition text();

    Condition value();
}
```

Add a utility class which is responsible for instantiating them with a catchy name `Must`:

```java
@UtilityClass
public class Must {

    // you could cache it into a static
    public static MustBool be(boolean expected) {
        return new MustBool() {
            @Override
            public Condition visible() {
                return expected ? Condition.visible : Condition.hidden;
            }

            @Override
            public Condition disabled() {
                return expected ? Condition.disabled : Condition.enabled;
            }
        };
    }

    public static MustString be(String expected) {
        return new MustString() {
            @Override
            public Condition text() {
                return Condition.text(expected);
            }

            @Override
            public Condition value() {
                return Condition.value(expected);
            }
        };
    }

    public static MustString beExact(String expected) {
        return new MustString() {
            @Override
            public Condition text() {
                return Condition.exactText(expected);
            }

            @Override
            public Condition value() {
                return Condition.exactValue(expected);
            }
        };
    }
}
```

In the SelectDoodad we can do:

```java
class SelectDoodad {
    
    SelenideElement root;

    public SelectDoodad visible(MustBool mustBool) {
        root.shouldHave(mustBool.visible());
        return this;
    }

    public SelectDoodad selected(MustString mustString) {
        root.$(".text").shouldHave(mustString.text());
        return this;
    }
}
```

So usage becomes:

```java
SelectDoodad e;
e.visible(Must.be(true));
e.selected(Must.be("A"));
```

A bit clunky, but it reintroduced types and also decoupled predicate from the value source without touching `Condition` or creating a new subclass of it.

The decoupling means that what the user gives to the doodad is a set of possibilities, and the doodad simply picks one of them according to its internal knowledge.

To understand what I mean, let's imagine that we have a variant of the doodad which stores the `selected` (whatever that is) in **value** attribute instead of **text**.

In this case your two different doodads will look like this:

```java
class SelectDoodad {
    
    SelenideElement root;

    public SelectDoodad selected(MustString mustString) {
        root.$(".foo")
            .shouldHave(
                // pick text
                mustString.text());
        return this;
    }
}

class SelectDoodis {
    
    SelenideElement root;

    public SelectDoodis selected(MustString mustString) {
        root.$(".bar")
            .shouldHave(
                // pick value
                mustString.value());
        return this;
    }
}
```

Note how, from the users perspective the two things accept the same thing:

```java
SelectDoodad a;
SelectDoodis b;
a.selected(Must.be("A"));
b.selected(Must.be("B"));
```

Even though this is clumsy, at least it hides away from the user the implementation details inherent in Selenide's `Condition`s.

Selenide's way of conciseness is a blessing when it comes to elements and basic ui. When it comes to large-scale, enterprisey forms with a lot of custom doodads it falls short.

> In the defense of Selenide: chaotic UI which was not meant to be tested is understandably hard and clunky to test.

Due to how Selenide blocks composition, we must implement matrices like this:

| property  | eq    | contains  | eqCaseSensitive   | containsCaseSensitive   |
| --- | --- | --- | --- | --- |
| text      | ? | ? | ? | ? |
| value     | ? | ? | ? | ? |
| attribute | ? | ? | ? | ? |

Which is somewhat related to the [Expression problem](https://wiki.c2.com/?ExpressionProblem) if you want to create a neat and concise `Condition` subclass for everything.

## Polarity, contravariant functor

The awkwardness - I think - comes from the fact that `Predicate` like things have negative polarity.

Imagine we have a functor `Str` acting over `String`s (awkward example I know):

```java
@RequiredArgsConstructor(staticName = "str")
public class Str {

    @Getter
    private final String val;

    public Str fmap(Function<String, String> f) {
        return str(f.apply(val));
    }

    private static final Pattern REGEX_SPACES = Pattern.compile("[\\s\\n\\r ]+");

    public static Function<String, String> ifNull(String ifNull) {
        return s -> s == null ? ifNull : s;
    }

    public static Function<String, String> trim() {
        return String::trim;
    }

    public static Function<String, String> lower() {
        return s -> s.toLowerCase(Locale.ROOT);
    }

    public static Function<String, String> reduceMultipleSpaces() {
        return s -> REGEX_SPACES.matcher(s).replaceAll(" ");
    }

    public static Function<String, Boolean> contains(String substring) {
        return s -> s.contains(substring);
    }
}

```

We could use it like this:
```java
final String expected = Str.str("sad")
                .fmap(Str.ifNull(""))
                .fmap(Str.trim())
                .fmap(Str.lower())
                .fmap(Str.reduceMultipleSpaces())
                .getVal();
```

We have a value of type `String` and simply applying functions on it.

We're adding things `after` it:

```
initial
initial -> ifNulled
initial -> ifNulled -> trimmed
initial -> ifNulled -> trimmed -> lowered
initial -> ifNulled -> trimmed -> lowered -> reduced
```

But in a predicate we cannot build `after` it. Instead we can only add things `before` it:

```
contains
reduce -> contains
lower -> reduce -> contains
trim -> lower -> reduce -> contains
```

You probably already see, that - via this approach - we can even incorporate `getText` or `getAttribute` aka 'backtrack' to the point when we decide which getter of the `WebElement` should be called:

```
contains
reduce -> contains
lower -> reduce -> contains
trim -> lower -> reduce -> contains
getText -> trim -> lower -> reduce -> contains
```

To more clearly show what I mean, I will add the types to the above snippet:

```
contains :: Predicate<String>
reduce -> contains :: Predicate<String>
lower -> reduce -> contains :: Predicate<String>
trim -> lower -> reduce -> contains :: Predicate<String>
getText -> trim -> lower -> reduce -> contains :: Predicate<WebElement>
```

The same process of construction can be applied to `WebElement` attributes:

```
contains :: Predicate<String>
reduce -> contains :: Predicate<String>
lower -> reduce -> contains :: Predicate<String>
trim -> lower -> reduce -> contains :: Predicate<String>
getAttribute("foo") -> trim -> lower -> reduce -> contains :: Predicate<WebElement>
```

But Selenide was not created this way, so it is kind of hassle to implement it.

Is there a route with less work?

## Get

If you don't want fork Selenide and rewrite the `Condition` related part of the codebase, there's a simple solution. You can always call `get` like methods on `SelenideElement`.

Selenide is - understandably - against `get`.

By `get` I mean directly acquiring a value from Selenium/Selenium and making an assertion on it.

This defeats the whole purpose of the do-while, neat `should` and concise API.

But, you can do it, and noone is stopping you from doing that. Just make absolutely sure, that the 'blackbox' won't change its state. If you can't guarantee that the state won't change, then you are bound to fail or have flakes that you must accept and prepare for.

If you get the text out of an element, you can use hamcrest, assertj or anything else.

Selenide is also cool enough to support screenshots on other kinds of errors thrown by these asserting libraries by `ScreenShooterExtension`.
