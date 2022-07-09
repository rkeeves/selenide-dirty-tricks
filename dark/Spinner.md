# That spinner could work though... Right?

Most ui libraries offer a way for the user to know whether the page is doing some processing.

This is the spinner thing we're all familiar with.

Can we use the spinner to stabilize all the tests?

## Overview

You can try it.

My personal opinion is that the main is challenge is 'making correct predictions about the future based on past data'. The spinner by itself won't help.

You need something which gets flipped by the app, and flopped back by you. But that has its own problems.

## The problem

Let's imagine we have three things.

`#pushme` must be pushed for ajax, it updates `#panel`.

`#panel-toggle` must be toggled to open/close `#panel`.

`#panel` has a text.

So in code it looks like something like this:

```java
$("#pushme").click();
// ajax
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

The above code results in failure because:
1. pushme gets clicked, ajax fires
2. panel-toggle gets clicked
3. we start waiting for panel to have text
4. ajax comes back, shuts back the panel
5. we check whether panel has text, but because panel is hidden the **visible** text what Selenium returns will never be `done`

Quite an asynchronous pickle!

But there's a thing which can help.

## Cop arrives on the scene

Imagine we have this spinner like thing on the ui, which signals to the user whether there's some ajax in process.

Now this is a game changer!

It acts like a cop which tells us when we can proceed!

Let's ask the devs to have a constant id on it. Let's call it `cop`.

By id, Selenium will find it insanely fast too!

And it is simple in Selenide:

```java
$("cop")
```

Neat! And concise!

## Cop experiment #0

We could write an easy and concise check for it:

```java
$("#pushme").click();
$("#cop").shouldNotBe(visible);
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

You click the button then you wait for the spinner to be not visible.

What could possibly go wrong?

What if the spinner is animated and when your `$("#cop").shouldNotBe(visible)` runs, the thing is not yet visible? Selenide would finish the do-while too fast and click on the toggle too fast.

Hmh.

## Cop experiment #1

What if we wait for it to become visible, then disappear?

```java
$("#pushme").click();
$("#cop").shouldBe(visible).shouldNotBe(visible);
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

What could possibly go wrong?

What if the spinner gets displayed and hidden before you hit `$("#cop").shouldBe(visible)`. Your test would throw because it didn't have the chance to witness the appearance of `cop`.

Hmh.

## Cop experiment #2

What if we wait a bit instead of checking visibility?

```java
$("#pushme").click();
Selenide.sleep(400L);
$("#cop").shouldNotBe(visible);
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

What could possibly go wrong?

Are you sure 400L is enough? Let's make it 500L?

The company servers are slow, make it 1000L. But then it might wait too much.

Let's make it 100L. But that's too short.

Hmh.

## Cop experiment #3

Let's be smart and create a Condition!

A Condition which only returns ACCEPT if it first sees the spinner visible and in a subsequent polling hidden.
    
But..-when we think of it...it's just an overcomplicated version of experiment #1.

## Cop experiment #4

Scrap it. Let's create a custom command which is essentially clicks then queries the text and retry this complex action involving.

But then...you are pushing a **toggler** and waiting for an animated panel to show up as a repeatable action.

Yeah...it sounds like asking for trouble.

## Cop experiment #5

Allright. Let's forget about `cop`.

We'll use ajax against itself:

```java
$("#pushme").click();
$("#panel-toggle").click();
$("#panel").shouldNotBe(visible);
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

We'll first click on the toggle to open the panel. Then we'll tell whether the update happened or not by the panel getting hidden. And then we'll do the actual thing.

This is still the `cop` approach, but this time `panel` plays the role of `cop`.

## Cop experiment #6

Game changer: we'll ask the devs to add a little almost invisible, always present text which tells the state.

No. It is still the same thing, but now with a `string` state variable instead of a `boolean`.

## Past vs. present

Remember the chicken?

```java
/* PRE     */                    /* POST    */
/* UNKNOWN */ $("#foo")          /* UNKNOWN */
/* UNKNOWN */ .shouldBe(visible) /* UNKNOWN */
/* UNKNOWN */ .click();          /* UNKNOWN */
```

You can get data only about the past.

When I say:

    A simple $().shouldBe(visible) is enough.

I actually mean:

    Claim: A simple $().shouldBe(visible) is enough.
    Assumption:
    visible won't change until further interactions are committed by you.
    Caveat:
    If this assumption cannot be guaranteed, then my claim does not hold. In such a case I understand your pain, dear stranger. I hope you'll be able to resolve the issue somehow!

This assumption is a pretty big assumption though (Feel free to change `visible` to `text` or `hidden` or whatever state variable). In some apps it holds, in some apps it does not. In some ui libraries it makes sense, in others it does not. On some pages it holds, on some pages it doesn't.

In most cases the assumption's validity can be guaranteed. But we're here because of an edge-case, in which the assumption's validity cannot be guaranteed.

## Cop experiment #7

Let's add an imaginary modification.

When the spinner goes visible, then it remains visible until the user clicks on it.

The spinner blocks the UI and all arbitrary js code is blocked from executing until the spinner becomes hidden again.

In this case our code becomes something like this:

```java
$("#pushme").click();
$("#cop").shouldBe(visible);
$("#cop").click();
$("#panel-toggle").click();
$("#panel").shouldHave(text("done"));
```

The spinner remains visible and an interaction BY YOU is needed to unblock the app.

Until such interaction occurs, all the state variables remain the same.

This is qualitatively different from the above experiments, because it allows you to 'predict' the future.

This solution has many problems though.

The first is that management won't be happy to hear that the devs are developing something for the testers instead of writing code for the customer.

The second problem is that you assert that ajax will happen.
This will make writing generic wrappers classes for widgets hard.

Imagine you have a `Gadget` which on click either does ajax, or it does not.

To support both cases, you either need an `if` or its equivalent via OOP (like using a decorator, or other fancy way). No matter what solution you end up with, your code must be able to make a distinction.

In one case it has to wait and click the spinner, while in the other it should not anticipate it.

So in summary: this solution is qualitatively different from the others and in theory it can solve your problem. In reality though, it has its own problems.
