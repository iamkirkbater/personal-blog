---
title: "Throwing Errors with Controller-Runtime fake"
date: "2020-06-18"
draft: false
description: "A quick guide on how you can throw errors with k8s Controller Runtime Fake"
images: "/img/posts/on-testing-2.jpg"
---

Recently we've been running into a race condition where a specific error is being thrown from Controller Runtime, however trying to replicate that behavior in a unit test has proven to be quite tricky.
<!--more-->

### Backstory
Let's set the stage here with some backstory. If you're not interested, skip to the [TL;DR](#tldr)

I'm on a team that's responsible for preparing AWS accounts to have a kubernetes cluster installed on them. We run this as a hosted solution for our customers. Essentially, the goal of our operator is to create an AWS account, ensure that we have a few users installed, and then test that we can initialize instances in the account, then we say the account is ready and another team's product comes in to install kubernetes.

And then when a cluster is deleted, our operator will prepare the account to be reused.  We do a quick spot-check to make sure that there's no residual infrastructure left that's going to just sit idle and burn money. Then we update the account inside kubernetes saying that it's "Ready" to be reused again. 

If we have an issue during this reuse prep stage, we mark the Account as "Failed", so that it can't be used again.

### The Issue
As we've scaled into the thousands of accounts that we manage, we've started seeing an uptick on the amount of "Failed" accounts.  The main problem is Failed accounts are toil. They require manual intervention to return back to a "Ready" state.

One of the biggest problems we were finding is that it seemed like the majority of the accounts were fine, but just marked as a "Failed" state. There was no clear indication of why it had failed (and at the same time we were having an issue with our logging being seemingly randomly truncated, but that's another story).

After a bit of debugging _(read: grabbing relevant logs over the course of a weekend when an account would fail and explicitly saving them to a file)_ we found that most of the errors were coming from another process modifying the Account CR during the process of when we would be running our checks, which caused kubernetes to throw a `Conflict` type of error.

### TL;DR
Long story short, the uninstall and reuse was working as expected, but when we went to save the state of the Account CR the kubernetes api would respond with a Conflict error, and then we would put the account into a "Failed" state.

### The Fix
The actual fix was relatively easy and straightforward, we just needed to catch the Conflict error and return early before explicitly failing the account, which requeues the object on the Reconcile loop and then the object will be processed again and updated appropriately.  

```Go
if errors.IsConflict(err) {
	myLogger.Info("Account CR Modified during CR reset.")
	return reconcile.Result{}, err
}
```

The problem we ran into was: _How in the hell do we test this?!?!?_

Enter the controller-runtime `fake` library.[^1] This is the library that `operator-sdk`[^2] uses to communicate with the kubernetes API, which is what our `aws-account-operator`[^3] is running.

The fake is an in-memory kubernetes API, so when you run your unit tests you run them against this instead of a real API.  For 90% of the errors you'll try to catch `fake` can replicate them easily. Errors like "Not Found" or "Already Exists" are simple enough, you just either don't populate that object or put one in before you run your test, respectively.

However, when trying to do something like create a Conflict error, this starts to get tricky.  A conflict can be thrown when the following happens: process A gets the object from kubernetes, then process B gets the same object and modifies it and saves it while process A is still running, then process A finishes and attempts to save the original object. This flow will throw the following error: `the object has been modified; please apply your changes to the latest version and try again`.

If you were to try to simulate this with the `fake` library, you'd have to create the object, run a delayed process to update that object in a goroutine, and then run the reconcile code. You'd probably have to add a `sleep` function, and play around with it to get the timing of the sleep right so that it forces the error, but if you run it on a faster or slower machine you might have different results. **THIS RESULTS IN A FLAKY TEST**

We Don't Like Flaky Tests. Say it again: We Don't Like Flaky Tests.

After many hours of searching for a way to mock this, I settled on attempting to use GoMock and Mockgen. The problem with this approach is that it's an all-or-nothing approach. I can either use the `fake` client and get the kubernetes behavior, or I can use the `mock` client and explicitly mock out each call to the client. Where this approach fell short is that this is an older codebase and the testing is lacking; which is one of the reasons why we're adding tests as we go.

With the fact that this codebase doesn't have the greatest coverage level, and was not explicitly designed with unit testing in mind, it's harder to test because functions are doing a bit too much, or they're longer than they need to be.  With that in mind, we would have had to mock out at least 10 separate calls, in order.  This leads to a brittle test, because what happens if we change the order of what we call later during a refactor, or get rid of something.  This test would then fail and need to be updated.

Oh the pitfalls of having well tested code.  But I digress...

I ended up getting a tip from a co-worker to take a look at the [kubernetes/test-infra](https://github.com/kubernetes/test-infra/blob/12a936787a32c4028dbf0849dd80c3692e50fea3/prow/deck/jobs/jobs_test.go#L510) project.  And this was exactly what we were looking to do.  

```Go
type possiblyErroringFakeCtrlRuntimeClient struct {
	client.Client
	shouldError bool
}

func (p *possiblyErroringFakeCtrlRuntimeClient) Update(
	ctx context.Context,
	acc runtime.Object) error {
	if p.shouldError {
		return fixtures.Conflict
	}
	return p.Client.Update(ctx, acc)
}
```

What's happening here is we're creating a fake client struct, and then overriding the specific function we need to throw an error when directed to. If we don't need to hit that specific error the client passes through the request to `fake`. In our case we only need to throw one type of error once so the example above is pretty trivial, however I'm sure you can see how you can extend this functionality for multiple and more complex use-cases.

The test itself looks something like this. We use Ginkgo, so it might not be what you're used to with Table Driven tests.

```Go
var _ = Describe("AccountClaim", func() {
	var (
		name         = "testAccountClaim"
		namespace    = "myAccountClaimNamespace"
		accountClaim *awsv1alpha1.AccountClaim
		r            *ReconcileAccountClaim
		fakeClient   client.Client
	)

	apis.AddToScheme(scheme.Scheme)

	BeforeEach(func() {
		region := awsv1alpha1.AwsRegions{
			Name: "us-east-1",
		}
		accountClaim = &awsv1alpha1.AccountClaim{
			ObjectMeta: metav1.ObjectMeta{
				Name:      name,
				Namespace: namespace,
			},
			Spec: awsv1alpha1.AccountClaimSpec{
				LegalEntity: awsv1alpha1.LegalEntity{
					Name: "LegalCorp. Inc.",
					ID:   "abcdefg123456",
				},
				AccountLink: "osd-creds-mgmt-aaabbb",
				Aws: awsv1alpha1.Aws{
					Regions: []awsv1alpha1.AwsRegions{region},
				},
			},
		}
	})
	
    Context("Reconcile", func() {
		It("should retry on a conflict error", func() {
			accountClaim.DeletionTimestamp = &metav1.Time{Time: time.Now()}
			accountClaim.SetFinalizers(append(accountClaim.GetFinalizers(), "finalizer.aws.managed.openshift.io"))

			account := &awsv1alpha1.Account{
				ObjectMeta: metav1.ObjectMeta{
					Name:      "osd-creds-mgmt-aaabbb",
					Namespace: "aws-account-operator",
				},
				Spec: awsv1alpha1.AccountSpec{
					LegalEntity: awsv1alpha1.LegalEntity{
						Name: "LegalCorp. Inc.",
						ID:   "abcdefg123456",
					},
				},
			}

			objs := []runtime.Object{accountClaim, account}
			fakeClient = fake.NewFakeClient(objs...)
			cl := &possiblyErroringFakeCtrlRuntimeClient{
				fakeClient,
				true,
			}
			r = &ReconcileAccountClaim{client: cl, scheme: scheme.Scheme}
			req := reconcile.Request{
				NamespacedName: types.NamespacedName{
					Name:      name,
					Namespace: namespace,
				},
			}

			_, err := r.Reconcile(req)

			Expect(err).To(HaveOccurred())
			Expect(err.Error()).To(Equal("Account CR Modified during CR reset. Conflict"))
		})
	})
})
```

So basically I prepare the objects that I need, and load them into the `possiblyErroringFakeCtrlRuntimeClient`, while setting "shouldError" on the client to true, so that it throws the conflict error. I then run the Reconcile function, and check the error, to make sure that it occurred and that it's the type of error that I'm expecting to see.

### Follow Up
I added a couple `fmt.Println()`'s in to the Reconciler code to make sure that I'm exiting where I think I'm supposed to, and since I am doing that and throwing the correct error, I can now take that "probably a bit too large and hard to test" function out; abstract out the functionality to a smaller, easier to test function; and then run this test again to make sure that it's still throwing the correct error, while adding more tests to that smaller function.

This improves the testability of our codebase, while also following the Red->Green->Refactor methodology!

Until Next Time!

[^1]: https://godoc.org/sigs.k8s.io/controller-runtime/pkg/client/fake
[^2]: https://sdk.operatorframework.io/
[^3]: https://github.com/openshift/aws-account-operator
