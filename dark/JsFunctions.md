# Is there a javascript function I can use?

So how can we tell from javascript that the app is 'steady'?

## Overview

Primefaces use jquery and some custom js. Some of these can be used by your tests.

I collected below some practical examples.

These still suffer from the problems outlined in the `chicken's tale`, but you can try to use them.

For more js functions, please check out their respective documentations:

[Jquery API doc](https://api.jquery.com/)

[Primefaces js API doc](https://primefaces.github.io/primefaces/jsdocs/index.html)

## Primefaces helps the community

The devs of Primefaces were kind enough to create a selenium library.

The devs were also kind enough to provide some particularly handy js functions that you can use in client side code.

We'll walk through some of these.

## Ajax queue

You have access to a queue which is basically a bookkeeping facility for ajax request-responses.

This is a step closer to what you actually need.

This can tell you about future state if you can guarantee that the following holds:
    
**Assumption: once the queue is empty it remains empty until you willfully mutate the state via Selenide again**

If you cannot guarantee this, then this is useless just like the `#cop` from the spinner example.

For v8:

```javascript
(!window.PrimeFaces
    || !PrimeFaces.ajax
    || !PrimeFaces.ajax.Queue
    || PrimeFaces.ajax.Queue.isEmpty())
```

For v11+:

```javascript
(!window.PrimeFaces
    || !PrimeFaces.ajax
    || !PrimeFaces.ajax.Queue
    || PrimeFaces.ajax.Queue.isEmpty())
```

## Animation stopped

You don't really need this, because you have Selenide or fluent waits for it: Most animations have a tendency to mutate a state variable from X to Y, and the state variable will remain Y until further interactions are applied by you.

But here you go:

For v8:

```javascript
jQuery(':animated').length == 0
```

For v11+:

```javascript
(!window.PrimeFaces
    || !PrimeFaces.animationActive)
```

## Turn animations off

Fiddling with these can result in pretty crazy things if you don't know what you are doing. Also, it has the possibility to **break things** really.

This is dangerous!

For v8:

Strongly unadvised, because it can break Primefaces.

```javascript
jQuery.fx.off = true
```
For v11+:

In this version the devs made a custom global flag because they're nice.

A bit unadvised, because it still has the possibility to break Primefaces.

```javascript
PrimeFaces.animationEnabled = false
```
