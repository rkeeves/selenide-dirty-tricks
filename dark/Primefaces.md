# Is there a ready-to-go solution?

Customers don't pay you for writing testing libraries and frameworks.

So it is natural to use existing solutions if possible.

But is there such a thing for Primefaces?

## Overview

Primefaces has a [solution](https://github.com/primefaces/primefaces/tree/master/primefaces-selenium).

Check it out, you might like it because it solves a lot of issues. But it has its downsides too.

## Things improve, because humans learn from their mistakes

In 2011 the culture was very different. The people who nowadays preach us that *'you dum-dum junior, you should favor composition over inheritance'* were knee deep in inheritance trees, so don't think that they didn't write their fair share of vaporware. (To be honest, I personally write vaporware to this day, so don't take me too seriously either.)

Also, testing was not really that popular. *Imagine 20 years in the future, devs writing exhaustive proofs for 99% of their programs and looking back at us with disgust for our 'unit tests' and 'integration tests' etc. Yep I know... halting problem, but I didn't say 100% :)*

In [one forum post](https://forum.primefaces.org/viewtopic.php?t=16259) someone asked:

```
Looking into source repo, I only find a few test cases. Less than ten.

How is testing managed for PF?
```

And got the reply:

```
Why do you worry about test suite? Community tests PrimeFaces everyday.
```

Then he got a more serious answer:

```
PF is tested by community and PrimeFaces team manually before each release, as java based renderers and JSF in particular are not easy to test we don't have an automated test suite. There are plans for selenium integration, but that's for future.
```

The devs delivered on the promise and they created a whole [project dedicated to selenium](https://github.com/primefaces/primefaces/tree/master/primefaces-selenium).

## Primefaces Selenium

I don't really want to get into the details of this project too much, as I don't really know it deeply enough.

In short they made wrappers for some widgets, and added a lot of cool helper facilities to handle xhr, animation, page loading.

It is NOT like Selenide though. It's a bit clumsy to write, and it does not have all the 'quality of life' features that Selenide offers.

It does not:
- provide concise API
- take screenshots with smart error messages with zero effort
- support weird locators
- have a concept of polling or `should`

It does:
- proxying just like Selenide but in a different way
- heavily rely on `PageObject` pattern (which can be a curse)
- still require manual tampering with ajax
- use `if`s

A test looks like this:

```java
@Test
@Order(1)
@DisplayName("Rating: set value and cancel value using AJAX")
public void testAjax(Page page) {
    // Arrange
    Rating rating = page.ratingAjax;
    Messages messages = page.messages;
    Assertions.assertNull(rating.getValue());

    // Act - add value
    rating.setValue(4);

    // Assert - rate-event
    Assertions.assertEquals(4L, rating.getValue());
    Assertions.assertEquals("Rate Event", messages.getMessage(0).getSummary());
    Assertions.assertEquals("You rated:4", messages.getMessage(0).getDetail());

    // Act - cancel value
    rating.cancel();

    // Assert
    Assertions.assertNull(rating.getValue());
    Assertions.assertEquals("Cancel Event", messages.getMessage(0).getSummary());
    Assertions.assertEquals("Rate Reset", messages.getMessage(0).getDetail());
    assertConfiguration(rating.getWidgetConfiguration());
}
```

With a little OOP imperative massaging magic you can hide away these things under abstractions.

On the other hand it overshines Selenide in respect of modeling widgets. It models a lot of widgets!

Selenide in this regard is subpar, because it models only two things (So you have to do the modeling by hand, which is a HUGE task.):
- SelenideElement
- ElementCollection

So give it a try! It might be a better tool for you.
