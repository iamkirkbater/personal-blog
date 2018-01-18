---
title: "On Writing Quality Tests - The Why"
date: 2018-01-09T19:20:00-05:00
draft: false
---

Quality tests are key to building quality software.  But what makes a quality test?  How do I know that the test I’m writing is adequate?  Does it cover enough, or am I trying to do too much?  What is the benefit of writing tests in the first place? When I started on my journey into the world of automated testing, these were the hardest questions to get a straight answer for.  Let’s dive into the last question: Why?

First, let’s focus on what I mean when I say testing. When I'm talking about testing I'm talking about writing a suite of both unit and integration tests that can be ran in an automated environment.  Any software product goes through some sort of QA testing, however this is typically a manual process.  Unit tests are written alongside the code you write that test only that code.  In a perfect world you would be writing your tests before you write your code &mdash; utilizing a method called Test Driven Development &mdash;  but the best first step you can take is just writing any tests at all!

For those who want some numbers as to why testing is important a paper with case studies from both Microsoft and IBM reported defects in software were reduced 40-90% with just a 15-35% increase in initial development time.  It’s important to note that the increase in development time was quickly recovered with the lack of maintenance time.  You can find a quick overview of the paper (and a link to the paper itself) here: [Empirical Studies Show Test Driven Development Improves Quality](https://www.infoq.com/news/2009/03/TDD-Improves-Quality).

My personal favorite reason for testing (and in my opinion the most important) is preventing regressions.  A regression is something you once had working only to fail at a later date.  Have you ever fixed a bug and then later that bug pops back up forcing you to fix it again?  Or you’ve fixed a bug in one part of the code only to inadvertently create one or two in a different feature. Those are regressions and are a maintenance cost referenced in the paper above.

Tests verify our code is working as we expect it to and we are also clearly stating what we expect.  In fact, when you’re working with a new code base one of the best places to start figuring out what specific methods and classes do and how they integrate together is to read the tests.  

With a good test suite you also enable other developers to be confident that the code they are contributing to your project is free of defects.  For example I am in the process of adding methods to another team’s API, but there are no tests written for that API.  I am essentially throwing code up and _hoping_ that it doesn't break anything because there is no way to effectively prove that I am not.  The key word here is _effectively_.  Someone &mdash; probably multiple people, actually &mdash; has to sit down and manually test every code path.

When vetting code that others have contributed to my projects, I don't accept pull requests that have new features without tests in them.  Prove to me that the code you wrote works, and that you haven't broken anything that has already been proven to be working.  I know this because I have a suite of tests for the code we already have, if they all pass then nothing has been broken with the code changes.

A good test suite is also key to constructing a good build pipeline.  A good build pipeline will enable you to start running Continuous Integration and eventually even Continuous Delivery/Deployment instead of manual deployments.  At a minimum, a build pipeline will allow you to just run your tests without having to manually pull the code down and do it yourself, saving you the extra steps.  By taking manual procedures out of the picture and automating them, we are able to reduce the amount of problems that can arise due to human error.  The less I have to touch, the less I can screw up.

By having an automated test suite, and a good build pipeline, a regression would never make it into past a pull request, let alone into a staging environment or production.  If you accidentally break something that was already working and tested, you would know as soon as you run the tests, enabling you to still be in the context of what you just wrote instead of having a QA team (or even worse: the client or the end-user) find it days, weeks, or months later.  

Over the next few weeks I plan on following this post with one that builds on what makes a unit test good or bad, and then a series on writing an SDK for a fake API that we want to consume and we’re going to write our tests first.
