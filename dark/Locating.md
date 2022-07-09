# Can I communicate that I'm testing?

When it comes to locating elements, there are a lot of good guides which detail how to get the job done.

Some companys might even ask these tricks in their interviews.

So how can we locate some 'well-buried' element in the dom?

## Selenium

Selenium has a tiny section dedicated to locator usage [best practices](https://www.selenium.dev/documentation/test_practices/encouraged/locators/).

It is well-written and worth a read.

In short it advises the use of locators in the following order (best first, worst last):

1. id: only works if unique and predictable, it is fast
2. linktext, partiallinktext: only work on link elements, otherwise good
3. css: faster than xpath
4. xpath: prone to error, but really flexible
5. tagname: prone to mismatch

## Cypress

Cypress has nothing to do with Selenium, but let's see what they say about locators.

Cypress too has a tiny section dedicated to locator usage [best practices](https://docs.cypress.io/guides/references/best-practices#Selecting-Elements).

In short it advises the use of locators in the following order (best first, worst last):

1. attribute made by the devs for the test: isolated from changes
2. linktext, partiallinktext: good, but coupled to content that may change
3. name attribute: coupled to html semantics
4. id: coupled to styling, layout, ui library etc
5. css classes: coupled to styling
6. tagname: prone to mismatch

## Being scraped or being tested

As you can see `Selenium` and `Cypress` don't really match. Not just because the technology is different.

There's a huge difference - in my opinion - in their philosophy.

Our question was:

    So how can we locate some 'well-buried' element in the dom?

`Selenium` answers this question insanely well, with technical insights too (id is performant). This is an A+ answer! Top-notch in my opinion!

`Cypress` instead asks back: Do you want to be webscraped or be tested?

Let's inspect these standpoints more closely.

## Invisible to the dev

The `Selenium` route is something I'm familiar with due to personal experience, and probably most of us.

You get an old application, with randomly generated ids, falling apart dom, and you must locate complex widgets. But the poor devs have to keep changing and updating it.

We can locate things for sure. It is sometimes a hassle, because ids cannot be used.

This is due to how the ids are generated.

Given a xhtml snippet of:

```xhtml
<form id="foo">
    <doodad id="bar"></doodad>
<form>
```

It'll be rendered into an html snippet like this:

```html
<form id="foo">
    <div id="foo:bar">
        <input id="foo:bar_more">
        <label id="foo:bar_noise">
    </div>
<form>
```

Given you don't give an explicit id

```xhtml
<form>
    <doodad id="bar"></doodad>
<form>
```

An id will be assigned from an ever increasing counter:

```xhtml
<form id="j_idt719">
    <div id="j_idt719:bar">
        <input id="j_idt719:bar_more">
        <label id="j_idt719:bar_noise">
    </div>
<form>
```

Someone probably says: 'Pah, then put explicit ids all over the place'.

But what if the container element comes from your company's pre-written tag-library, or a fragment? Aka you don't have control over that id?

Well, this is why companies hire smart testers. They are always able to figure out a locator!

But what about the developers?

To illustrate their point of view, imagine you are a dev, and you see an xhtml like this:

```xhtml
<p:outputLabel for="country" value="Country: " />
<p:selectOneMenu id="country" value="#{dropdownView.country}" style="width:150px">
    <p:ajax listener="#{dropdownView.onCountryChange}" update="city" />
    <f:selectItem itemLabel="Select Country" itemValue="" noSelectionOption="true" />
    <f:selectItems value="#{dropdownView.countries}" />
</p:selectOneMenu>
```

Now please try to answer the following questions:
- Is the `p:outputLabel` used in tests?
- Is the `p:selectOneMenu` used in tests?
- If so which properties are necessary for properly locating them?

Or in more laymen's term: Tell me what kind of change won't break the build.

You have no information.

For all you know, tests are scraping your site randomly, you have no information about how, so you can't really make non-breaking changes or somewhat informed decisions.

But it does not require any coordination between dev and tester. They can work as encapsulated units in their corporate cubicles.

## Visible to the dev

The `Selenium` route is something I'm familiar with due to personal experience, and probably most of us.

Let's try what `Cypress` advised.  We'll add an attribute `data-test` - in our clunky, deprecated EE way of course.

Once again please try to answer the following questions:
- Is the `p:outputLabel` used in tests?
- Is the `p:selectOneMenu` used in tests?

```xhtml
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:f="http://java.sun.com/jsf/core"
      xmlns:h="http://java.sun.com/jsf/html"
      xmlns:p="http://primefaces.org/ui"
      xmlns:pt="http://xmlns.jcp.org/jsf/passthrough">
...
<p:outputLabel for="country" value="Country: " />
<p:selectOneMenu id="country" pt:data-test="country" value="#{dropdownView.country}" style="width:150px">
    <p:ajax listener="#{dropdownView.onCountryChange}" update="city" />
    <f:selectItem itemLabel="Select Country" itemValue="" noSelectionOption="true" />
    <f:selectItems value="#{dropdownView.countries}" />
</p:selectOneMenu>
```

We cannot be 100% sure, but the `p:selectOneMenu` doodad seems to have some strange `pt:data-test="country"` on it. It looks like something test related so there's a possibility that it is tested.

You can't really guide 'passthrough' though. It'll be applied somewhere on the rendered html snippet, but at least it is there.

This approach means a bit more thinking from the devs, and will definitely bring some clashes between testers and devs, but it is better than not communicating intent. *(Telling someone on chat that my change will possibly break the build is a side channel, so I don't consider it communicating intent.)*

The really bad part is that if you have a six year old application, you'll have to make changes to both the application's xhtml and tests. Customers will never pay for this.

## Id does too much already

We see that the two approaches are direct opposites of each other.

The cost of using dedicated attributes is just too high.

The question naturally arises: But id is just an attribute, why can't we use it for communicating testing intent? *(Aka we want to - with minimal effort - enjoy the best of both worlds.)*

I think - think, not **know** - that `Cypress`'s advice was qualitatively different from using ids, and you cannot use ids for communicating testing intent.

My reasoning goes like this: With `data-thisisusedfortest` you communicate clearly and unambigously ONE thing, and one thing only: that the doodad attributed with `data-thisisusedfortest` is under test.

With id how can you possibly tell?

The id is used by client side js, ajax requests and responses, renderers etc. It is simply already does too many things. You have simply no grammatical way to communicate with it your special intent of testing.

## Can't be incremental and consistent

Even if find that id does too much already and we need a separate attribute on the doodad, a minimal effort way still exists.

What if we leave the old code alone, but in new features we create attributes. Everyone will be happy and things will get better. Step-by-step, incrementally.

I think - think, not **know** - that most pages are random collections of fragments and doodads. Sometimes they contain popup dialogs which have their own xhtml.

If you create a new feature you'll probably continue reusing company fragments, templates, doodads.

For example:

Imagine a new feature. A user wants to have a pop up modal dialog for filling out a form and creating a business dingus. The modal dialog is non-negotiable, because people like it and can't get enough of it.

The popup dialog's layout, css, the buttons, everything comes from some old code. You only write the 'contents' and the server side bean, while the dialog's other parts come from old code. You either cannot or do not want to touch a code from 3 years ago, which is connected into everywhere. So the dialog will be a horrible mashup of the 'new locators' and the 'old locators'.

The test will still have to use the old ways, but now, sometimes - here and there, occasionally - will use the new simplified way of location.

This will be totally inconsistent with past tests.

Also your new dingus form will be totally inconsistent with past forms.

If you can accept this inconsitency then it is okay. If you can't accept it, then that is okay too. Just know that it can and will happen.
