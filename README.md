# Selenide dirty tricks

This repository peeks under the hood of Selenide. It also details some dirty hacks you can pull off. This repository was written with Cunningham's Law in mind.

## Overview

Selenide is wrapper library above Selenium (written in Java).

Selenide offers many 'nice to have' features over raw Selenium. An example usage of Selenide looks like this:

```java
class SomeTest {

    @Test
    void test() {
        open("x");
        $("#a")
            .$(".b")
            .$$("li")
            .shouldBe(visible).click();
    }
}
```

I thought that a somewhat deeper dive into the internal workings of Selenide would be handy, because Selenide does much more heavy lifting than one might think.

There's the [Light Side](light/LIGHT.md) which deals with vanilla Selenide experience. It tries to cover some mechanism, like driver management, command handling and others.

There's the [Dark Side](dark/DARK.md), which builds on `light side`.
It first explains why I think Selenide has nothing to do with tests being `stable` or `flaky`, and cannot `stabilize` tests (Don't get me wrong, it is a good tool, but not a future-telling crystal ball.).
Then I detail dirty tricks you can pull off if your AUT is an uncontrollable mess. I'll try to mention some of these hacks' costs, and weaknesses. Although I must say: If the AUT is so unstable that you need these tricks, then it is probably just the tip of the pain-iceberg.

Thanks for reading!
