---
title: "On Writing Quality Tests - The What"
date: 2018-01-09T19:20:00-05:00
draft: false
---

Now that we’ve covered “why” in the last post, let’s get into what makes a unit test “good”. Good unit tests cover all the code, each test should test one thing and one thing only, and they are completely independent of one another.  Bad tests report failing tests as passing.  We'll also dive into some common test mistakes that can muddy the waters between a good test and a bad one.

### Test Coverage
First, you should always aim for 100% coverage.  It’s not always possible, though. There are some who would argue that 100% is the only acceptable number you can aim for.  I am not one of them but I am in the camp that if you can that you should strive to hit that mark.

Each project should set a minimum acceptable level for tests.  I personally shoot for a _minimum_ of 80%. This is enforced in my build system. Take caution when setting a minimum: some will use that as the maximum amount of work they need to do.

The key to writing good tests is that you want to travel every code path.  You want to hit every if block, every else block, every switch-case, and every catch block. We use mocks that return specific responses which should be able to coerce the program down different paths.

This is the core principle of unit testing.  Every code path _needs_ to be ran through at least once otherwise we are not confident that it works properly.

### Test Minimalism
Good tests are minimal.  You should treat your tests like first-class code. If your tests are unreadable and hard to follow it’s going to be harder to maintain.  Nobody likes code that’s hard to maintain.

Just like with your application code, you also want your tests to follow programming best-practices.  DRY (Don’t Repeat Yourself) is one of the things I see that can make a test suite excruciatingly hard to read through.

Somewhat related: you should only be checking what needs to be checked for _that test case_.  If you already checked for good output in one case you _shouldn’t_ have to check for it in every other test where you test for your edge cases.  Just check what’s needed to validate the edge cases.

With tests being minimal, you should also avoid logic in your tests themselves. There’s a good chance that if you’re writing if statements in your tests you should be writing another test, forcing it down a separate path in each test.

### Test Independence
Every test should be 100% independent of other tests.  I can not repeat myself enough on this point.  If one test relies on the state of the previous test those assertions might as well be in the first test. This ties in with Test Organization and Test Minimalism.  Can you see the trend that all of these topics tend to tie into each other?

While a test should not depend on the state of any test that comes before it there should also be no remnants of a test left over.  You need to ensure that you clean up after every test. If you mock something out in one test, make sure you restore that mock before you finish the test.  If you mock something out in the setup task of a grouping of tests make sure you tear it down in the teardown task.  By making sure that your tests don’t “leak” you can ensure that the results of running a single assertion will remain the same as if you ran the entire test suite.

## What makes a BAD test?
In my opinion a test is only BAD if it’s reporting something as passing when it is in fact failing.  (Something something [Volkswagen.js](https://github.com/auchenberg/volkswagen)). When tests show that they pass when they’re actually failing might as well just not run them at all.  When errors go unchecked, you introduce uncertainty, and a lack of faith in the test suite.

Again, this is my opinion, but I think the only thing that makes a test _BAD_ is being able to build false assumptions because it’s not doing what it is supposed to, but saying it is.

With that said, there are still some anti-patterns that you should look out for. These things do not specifically make a test _bad_ on their own, but these can help mask problems or make your life a lot harder when it comes time to change your code later on.

### Extraneous Output from your test suite.
One of my biggest pet peeves when running tests is seeing a bunch of extraneous info being logged to the console. The problem with these is that they can often mask an actual problem, like I mentioned in the section on “bad tests” where a NodeJS `Unhandled Promise Rejection Error` could be lost among the many console.log() statements of a codebase.  Without these log statements, you should be able to pinpoint any problems like that because there should be no extra output.

The best option to mitigate this is to use a mature logging library.  A library like Winston will allow you to set up these transports and then you can detect if your `NODE_ENV` is test and remove the console transport, or disable all transports alltogether with a different env flag.

However you decide to turn off logging to the console, if you need to test that it’s logging something when the test runs, you can mock out the logger and performing your assertions that it is trying to log to the correct level there.

### Mixing Unit Tests and Integration Tests
The final point I want to make is that there is a key difference between Unit tests and Integration tests.  Now, it seems that everyone likes to define these things a little differently, so I will first go over what I consider a Unit test vs an Integration test.

A Unit test is something that is tested completely in isolation.  Every extra dependency is mocked, faked, or stubbed, and because of that every code path is able to be tested.

An Integration test is where you take all of your code and just mock external dependencies like network requests.

For example if you had module A that relies on module B, and Module B uses Module C to get data out of an API. In a Unit Test you would mock out Module B so that it would return whatever you want it to, forcing you down every code path in Module A.   In an Integration Test you would just mock the network request, but all of the other code runs as it would normally.

Both types of tests are absolutely necessary to be able to have full confidence in your code, otherwise you could end up in a situation where you have 100% test coverage unit test wise, but when you run both pieces of code together things just don’t work out quite right.

https://zippy.gfycat.com/HotOrangeCoypu.webm. https://twitter.com/thepracticaldev/status/687672086152753152?lang=en  [Unit Test vs Integration Test - YouTube](https://www.youtube.com/watch?v=0GypdsJulKE)

A good test suite should have a healthy mix of both integration tests and unit tests, however those tests should be separate.  The problem is when you do a little of each your tests become brittle, meaning small changes break your tests.  This leads to maintenance problems which leads to not wanting to bother with tests.

However, a good set of integration tests _should_ break when the implementation needs to change.  Now, this sounds like it contradicts what I just said about not wanting to break tests.  The problem arises when you go change something “small” and now you have to go through and re-write a bunch of your old tests because they all relied on the implementation.  Generally in a unit test you should not have to worry about the specific implementation, as you should be *controlling* what is going to be sent back to begin with.
