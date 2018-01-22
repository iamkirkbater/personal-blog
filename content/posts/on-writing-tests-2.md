---
title: "What makes a unit test Good?"
date: "2018-01-19"
draft: false
description: "In the last post we covered the why, now let's jump into what makes a unit test good."
images: "/img/posts/on-testing-2.jpg"
---

Now that we’ve covered “why” in the [last post](/posts/2018/01/on-writing-quality-tests---the-why/), let’s get into what makes a unit test “good”. Good unit tests cover all the code, each test should test one thing and one thing only, and they are completely independent of one another.
<!--more-->

Let's first start with some definitions because everyone in the testing world seems to have different meanings for the same words.

* **Assertion**: A way to tell your test runner what behavior you expect.
* **Test**: A grouping of assertions that all together prove that the test works as we expect.
* **Group**: A logical grouping of tests.
* **Test Suite**: A combination of groups and/or tests that prove a module, library, or application behaves as expected.
* **Fixture**: A set of fake data of "known" information.  This can be known good or known bad info based on which code path you want to take.
* **Mock**: When you depend on functionality of another object we create a mock of that object. This allows us to explicitly return whatever we want from it, and is what allows us to force our code down different paths.

### Test Coverage
You should always aim for 100% coverage. It's not always possible mostly due to methods or variables being protected or private. There are some who would argue that 100% code coverage is the only acceptable percentage you can aim for. I am not one of them but I am in the camp that if you can get 100% coverage you should, just don't sacrifice coding best practices for it.

Each project should set a minimum acceptable coverage level for tests. I personally shoot for a _minimum_ of 80%. This is enforced in my build system. Take caution when setting a minimum: some will use that as the maximum amount of work they need to do.

{{% aside class="warning" %}}A common complaint against testing is that achieving a high code-coverage level on a legacy code base is extremely hard to do.  In these cases, some _good_ tests are better than no tests at all. When I speak about setting enforceable goals, I'm referring to greenfield projects. For such legacy projects you can add tests as you add features and raise the minimum level to be enforced as you add more tests.{{% /aside %}}

The key to writing good tests is that you want to travel every code path. You want to hit every if block, every else block, every switch-case, and every catch block. We use mocks that return specific responses which should be able to coerce the program down different paths, and make assertions against those responses based on what we expect to happen.

This is one of the core principle of unit testing. With that said, though, just running through the code alone won't prove that your code works. You need to be making the correct assertions as you go, as well. The primary point of coverage is that you don't want to regress as your code grows. You don't want to be at 85% and then introduce a new feature and drop down to 70%.

### Clarity through Minimalism
Good tests are clear. You should treat your tests like first-class code. If your tests are unreadable and hard to follow they're going to be harder to maintain. Nobody likes code that’s hard to maintain.

Just like with your application code, you also want your tests to follow programming best-practices. Not practicing DRY (Don’t Repeat Yourself) is one of the things I see that can make a test suite excruciatingly hard to read through. I want to repeat myself that the entire point of making them clear is to make them easy to read and maintainable. Use

{{% aside class="info" %}}Somewhat related: you should only be checking what needs to be checked for _that test case_. If you already checked for good output in one case you _shouldn’t_ have to check for it in every other test where you test for your edge cases. Just check what’s needed to validate the edge cases.{{% /aside %}}

I personally believe the best way to be clear about what is happening in your code is to be minimal. I'm a firm believer that each unit test should be testing for only one thing. This will usually need many assertions to prove that the one thing is working. For example if you're writing a test that an object is created correctly, you would have one test to check that it's created correctly with an assertion for each member of that object.

I feel it's better to have many small tests than a few large ones. This way it's easier to debug tests that fail unexpectedly and pinpoint what exactly is wrong in your code.

With tests being minimal, you should also avoid logic in your tests themselves. There’s a good chance that when you’re writing `if` statements in your tests you should be writing another test instead, forcing it down separate paths in each test.

To help keep your tests clear, most (if not all) testing libraries will have some sort of set up and tear down functionality. Where that functionality is located differs from library and language. Consult your specific library's documentation for more information.

### Organization
In any software project your code needs to be organized. Test code is no exception. Keeping your tests organized will help with maintenance and just like your application logic it will be easier to modify.

I personally have my own "style" when it comes to tests. I prefer to group all of my failure cases at the top of my test grouping, and then I add all of my other assertions below that. For example if I want to test that a method should throw an error under conditions X and Y, but should resolve under Z, I first test for X and Y then I test for Z. It's just a personal preference, really, but I've found that I have more success cases than failure cases over time so I can just add new success cases to the end of the grouping.

Let's talk about grouping tests. Just like the failure -> success organization of the individual tests, this is mostly just personal preference on how to actually execute. Consistency is the key factor to this.  It doesn't matter what way you want to organize your tests: just be consistent.

I have a single test file that tests a single code file. I then have multiple tests for each method, so I group all of those tests together. If a method has a significantly complex functionality I might even have more groupings inside of those main groups. At the end of the day I want to be able to quickly navigate my test suite and pinpoint where a problem is.

For example if I have a file: `src/some/dir/aws.js` I would have a test file `test/some/dir/awsTest.js`. Then, inside of `awsTest.js` I would be testing every method of `aws.js`, and it might look something like this:

```JavaScript
describe('Testing aws.js', () => {
  describe('Testing myFirstMethod', () => {
    it('runs the test', () => Promise.resolve());
  });

  describe('Testing myComplexMethod', () => {
    describe('Logical grouping of individual tests here', () => {
      it('runs the test', () => Promise.resolve());
      it('runs the test', () => Promise.resolve());
      it('runs the test', () => Promise.resolve());
    });

    describe('Another grouping of tests', () => {
      it('runs the test', () => Promise.resolve());
      it('runs the test', () => Promise.resolve());
    });
  });
});
```

If there is one takeaway I want you to have from this section it's that you need to be consistent. Just like with your application code being organized will help contributors find the tests they're looking for when they want to add or modify a feature. Most of your test organization will boil down to personal preference. However you decide to organize your test suite: **be consistent**.

### Independence
Every test should be 100% independent of other tests. I can not repeat myself enough on this point. If one test relies on the state of the previous test, then the assertions from the second might as well be in the first test.

Tying back to test clarity you should utilize your setup and teardown functions to do things like reset your test database, restore your mocks, or otherwise reset your fixtures back to default values. If one test modifies a fixture and that fixture isn't restored back to it's original state subsequent tests may have unpredictable results &mdash; the opposite of what we want.

By making sure that your tests don’t “leak” into subsequent tests you can ensure that the results of running a single test will remain the same as if you ran the entire test suite. You need to ensure that you clean up after every test. If you mock something out in only one test make sure you restore that mock before you finish the test.

If the same thing needs to be mocked for every test, consider using your testing library's setup functionality to mock it out and tear down to restore it.  Otherwise if you modify the data of your test object in the first test it will carry over into subsequent tests causing inconsistent test results. Some languages and frameworks even randomize the order in which your tests are run to help guarantee this.

## Final Thoughts
At the end of the day we want our test code to be just as maintainable as our application code. The easiest way to do this is by keeping the tests clear and organized. We also _need_ to have an adequate amount of coverage of our code base, and we absolutely do not want any state to be persisted between tests.  This is the foundation of testing, and by following these principles you should alleviate most of your testing woes.

In my next post I'll dive into what makes a test bad, and what can muddy the waters of a good test that can cause or mask problems down the line. I'm also planning on creating a series of posts where we create a library to talk to an API using TDD so we can hopefully tie all these points together. [Follow me on Twitter](https://twitter.com/KirkBater) to be notified when the next post is ready!
