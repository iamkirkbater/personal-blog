---
title: "What makes a unit test Good?"
date: "2018-01-19"
draft: false
description: "In the last post we covered the why, now let's jump into what makes a unit test good."
---

Now that we’ve covered “why” in the last post, let’s get into what makes a unit test “good”. Good unit tests cover all the code, each test should test one thing and one thing only, and they are completely independent of one another.
<!--more-->

### Test Coverage
You should always aim for 100% coverage. It's not always possible, though, mostly due to methods or variables being protected or private. There are some who would argue that 100% code coverage is the only acceptable percentage you can aim for. I am not one of them but I am in the camp that if you can get 100% coverage you should.

Each project should set a minimum acceptable level for tests. I personally shoot for a _minimum_ of 80%. This is enforced in my build system. Take caution when setting a minimum: some will use that as the maximum amount of work they need to do.

The key to writing good tests is that you want to travel every code path. You want to hit every if block, every else block, every switch-case, and every catch block. We use mocks that return specific responses which should be able to coerce the program down different paths.

This is the core principle of unit testing. Every code path _needs_ to be ran through at least once; otherwise we cannot be confident that it works properly.

### Clarity through Minimalism
Good tests are clear. You should treat your tests like first-class code. If your tests are unreadable and hard to follow it’s going to be harder to maintain. Nobody likes code that’s hard to maintain.

Just like with your application code, you also want your tests to follow programming best-practices. DRY (Don’t Repeat Yourself) is one of the things I see that can make a test suite excruciatingly hard to read through.

{{% aside %}}Somewhat related: you should only be checking what needs to be checked for _that test case_. If you already checked for good output in one case you _shouldn’t_ have to check for it in every other test where you test for your edge cases. Just check what’s needed to validate the edge cases.{{% /aside %}}

I personally believe the best way to be clear about what is happening in your code is to be minimal. I'm a firm believer that each test should be testing for one specific thing. I feel it's better to have many small tests than a few large ones. This way it's easier to debug tests that fail unexpectedly and pinpoint what exactly is wrong in your code.

With tests being minimal, you should also avoid logic in your tests themselves. There’s a good chance that if you’re writing if statements in your tests you should be writing another test, forcing it down a separate path in each test.

To help with this, most (if not all) testing libraries will have some sort of set up and tear down functionality.  Most that I've seen also break this down into `before`, `beforeEach`, `afterEach`, and `after` functions. The `before` and `after` run before and after the grouping of tests, respectively, where `beforeEach` and `afterEach` run before and after individual tests.

### Organization
In any software project your code needs to be organized. Test code is no exception. Keeping your tests organized will help with maintenance and just like your application logic it will be easier to modify.

I personally have my own "style" when it comes to tests. I prefer to group all of my failure cases at the top of my test grouping, and then I add all of my other assertions below that. For example, if I want to test that a method should throw an error under conditions X and Y, but should resolve under Z, I first test for X and Y then I test for Z. It's really just a personal preference, really, but I've found that I more success cases than failure cases over time so I can just add new success cases to the end of the grouping.

Let's talk about grouping tests. This, just like the failure -> success organization, is probably mostly just personal preference on how to actually execute &mdash; the key is to be consistent. First, I have a single test file that tests a single code file. For example if I have a file: `src/some/dir/aws.js` I would have a test file `test/some/dir/awsTest.js`. Then, inside of `awsTest.js` I would be testing every method of `aws.js`. I probably also have multiple tests for each method so I group all of those tests together. If a method has a significantly complex functionality I might even have more groupings inside of those. At the end of the day I want to be able to quickly navigate my test suite and pinpoint where a problem is if it arises.

### Independence
Every test should be 100% independent of other tests. I can not repeat myself enough on this point. If one test relies on the state of the previous test the assertions from the second might as well be in the first test. Tying back to test clarity you should utilize your setup and teardown functions to do things like reset your test database, restore your mocks, or otherwise reset your fake data back to default values.

By making sure that your tests don’t “leak” you can ensure that the results of running a single test will remain the same as if you ran the entire test suite. You need to ensure that you clean up after every test. If you mock something out in only one test make sure you restore that mock before you finish the test. If the same thing needs to be mocked for every test, use your testing library's `beforeEach` to mock it out and your `afterEach` to restore it.  Otherwise if you modify the data of your test object in the first test and it will carry over into subsequent tests causing inconsistent test results.

## Final Thoughts
At the end of the day we want our test code to be just as maintainable as our application code. The easiest way to do this is by keeping the tests clear, organized. We also _need_ to have an adequate amount of coverage of our code base, and we absolutely do not want any state to be persisted between tests.  This is the foundation of testing, and by following these principles you should alleviate most of your testing woes.

In my next post I'll dive into what makes a test bad, and what can muddy the waters of a good test that can cause or mask problems down the line. [Follow me on Twitter](https://twitter.com/KirkBater) to be notified when the next post is ready!
