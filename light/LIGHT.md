# Light side

I broke down Selenide feature-wise into smaller sections. I call these sections the `light side`, because in these I just write about Selenide and don't tell you about anything else.

Each section has explanation text and heavily modified and simplified code snippets.

Check out the original source, because it could've changed. Also I edited a lot of code out, to keep things as simple as possible. Keep in minf that the code snippets presented here are **NOT** the actual code, just a heavily watered down version.

Here's the `light side`, the vanilla, intended way of things:

- [How can I have a SelenideElement if nothing implements it?](DynamicProxy.md)
- [Where is the behavior?](CommandsSingleton.md)
- [How does it find the right command?](CommandLookup.md)
- [How does it poll?](Polling.md)
- [What's lazy?](Lazy.md)
- [How does it find Y inside X?](RelativeSearch.md)
- [How does a condition work?](Condition.md)
- [How does it know when to take screenshot?](Screenshot.md)
- [How does it take screenshots?](Photographer.md)
- [Where's the driver?](Driver.md)
- [How do I add listeners to the driver?](WebDriverListeners.md)
- [What is inject?](SPI.md)
