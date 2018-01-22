---
title: "Unit tests: The Bad and the Ugly"
date: "2018-01-27"
draft: true
description: "We touched on what makes a test good, now let's cover what makes a test bad or ugly."
---
Now that we know what makes a unit test good, let's jump into what makes a test bad as well as some common cases where certain practices can mask problems.
<--!more-->

## What makes a BAD test?
In my opinion a test is only BAD if it’s reporting something as passing when it is in fact failing. (Something something [Volkswagen.js](https://github.com/auchenberg/volkswagen)). When tests show that they pass when they’re actually failing might as well just not run them at all. When errors go unchecked, you introduce uncertainty, and a lack of faith in the test suite.

Again, this is my opinion, but I think the only thing that makes a test _BAD_ is being able to build false assumptions because it’s not doing what it is supposed to, but saying it is.

With that said, there are still some anti-patterns that you should look out for. These things do not specifically make a test _bad_ on their own, but these can help mask problems or make your life a lot harder when it comes time to change your code later on.

### Extraneous Output from your test suite.
One of my biggest pet peeves when running tests is seeing a bunch of extraneous info being logged to the console. The problem with these is that they can often mask an actual problem, like I mentioned in the section on “bad tests” where a NodeJS `Unhandled Promise Rejection Error` could be lost among the many `console.log()` statements of a codebase. Without these log statements you should be able to pinpoint any problems like that because there should be no extra output hiding the problem.

The best option to mitigate this is to use a mature logging library and environment variables. A library like Winston in NodeJS will allow you to set up these transports and then you can check if `process.env.NODE_ENV` is set to `test` and remove the console transport, or disable all transports with a different env flag. In Python in my `package/__init__.py` file I check to see if an env var called `LOGGING` is set, defaulting to `True`. In my `test/__init__.py` file I set this var to `False` if it's not set already. This allows me to toggle logging on and off during the running of my tests using the same env var.

However you decide to turn off logging to the console, if you need to test that your module is logging something when the test runs, you should mock out the logger and perform assertions against that mock that it is trying to log to the correct level and/or message.

### Mixing Unit Tests and Integration Tests
The final point I want to make is that there is a key difference between Unit tests and Integration tests. Now, it seems that everyone likes to define these things a little differently, so I will first go over what I consider a Unit test vs an Integration test.

A Unit test is something that is tested completely in isolation. Every extra dependency is mocked, faked, or stubbed, and because of that every code path is able to be tested.

An Integration test is where you take all of your code and just mock external dependencies like network requests.

For example if you had module A that relies on module B, and Module B uses Module C to get data out of an API. In a Unit Test you would mock out Module B so that it would return whatever you want it to, forcing you down every code path in Module A.  In an Integration Test you would just mock the network request, but all of the other code runs as it would normally.

Both types of tests are absolutely necessary to be able to have full confidence in your code, otherwise you could end up in a situation where you have 100% test coverage unit test wise, but when you run both pieces of code together things just don’t work out quite right.

{{% video src="https://zippy.gfycat.com/HotOrangeCoypu.webm" type="video/webm" autoplay="true" loop="true" %}}
{{% tweet 687672086152753152 %}}
{{% youtube 0GypdsJulKE %}}

A good test suite should have a healthy mix of both integration tests and unit tests, however those tests should be separate. The problem is when you do a little of each your tests become brittle, meaning small changes break your tests. This leads to maintenance problems which leads to not wanting to bother with tests.

However, a good set of integration tests _should_ break when the implementation needs to change. Now, this sounds like it contradicts what I just said about not wanting to break tests. The problem arises when you go change something “small” and now you have to go through and re-write a bunch of your old tests because they all relied on the implementation. Generally in a unit test you should not have to worry about the specific implementation, as you should be *controlling* what is going to be sent back to begin with.
