---
title: "Telemetry in Go - Readers"
date: 2023-07-26T08:00:00-05:00
draft: false
---

I was struggling to understand when to use the two main types of readers that come within the OpenTelemetry Go package, and wanted to share what I've learned. This is the intro post to a series where I'm going to cover some telemetry things in Go, ending with creating a plug-and-play telemetry add-on for the [cobra-cli](https://cobra.dev/).

First, this blog post assumes a _slight_ bit of knowledge on your part. I'm writing this post assuming that you already know about the different types of metrics that one could collect with OpenTelemetry. I'm not going to get into the different types of instrumentation that's available like Counters vs Gauges vs Histograms, there's _plenty_ of [posts](https://www.timescale.com/blog/a-deep-dive-into-open-telemetry-metrics/) [on the](https://uptrace.dev/opentelemetry/go-metrics.html) [internet](https://grafana.com/docs/opentelemetry/instrumentation/go/manual-instrumentation/) that already go into that, and I don't want to just add yet another blog post that just goes over how to start [pushing](https://docs.honeycomb.io/getting-data-in/opentelemetry/go-distro/) [metrics](https://uptrace.dev/opentelemetry/go-metrics.html) [into](https://docs.datadoghq.com/tracing/trace_collection/otel_instrumentation/go/) [a specific](https://signoz.io/opentelemetry/go/) [backend](https://www.dynatrace.com/support/help/extend-dynatrace/opentelemetry/walkthroughs/go). 

The goal I originally set out to accomplish was to add some _extremely basic_ telemetry to a Cobra application. (Though this turned out to be a _bit_ more complex than I had originally intended, so I needed to break this up into smaller posts). _Most_ of the example blog posts I could find at the time of my research focus on either setting up Traces/Spans or setting up something automagically in a web application, so when the tutorial on "here's how to set up metric collection" consistently ended with "and then just add this middleware to your gin app" I was left most displeased. So I'm here to dive a bit into the parts that nobody else has covered yet. I'm by no means an expert in this field, I just more-or-less want to share what I've learned so that hopefully someone else doesn't have to spend the time diving in to the details.

# What is a Reader

So there are two types of "Readers" that I'm going to be talking about. A Reader in OpenTelemetry is essentially something that runs locally within your application that aggregates the given metrics your application is programmed to output.  The Reader's job is only to aggregate them, not to send them off, which is the job of the Exporter. We can see this with the [Reader interface](https://github.com/open-telemetry/opentelemetry-go/blob/64e76f8be45c9f4df85344249fad0fb72cff1230/sdk/metric/reader.go#L54) where we see that the only public methods are RegisterProducer, Collect, ForceFlush, and Shutdown. You can read about what the various methods should do in the code comments, the open-telemetry maintainers have done a very good job of adding them.

So what are the two types of Readers that we're going to focus on? Specifically, I want to dive into the difference between a PeriodicReader and a ManualReader. A PeriodicReader, as the name suggests, performs a read (calls the Collect function) at a provided interval. With a Manual Reader the Collect function needs to be manually called.

## Periodic Reader

Most of the code examples I was able to find during my research all used the PeriodicReader implementation, which is probably what you want to do for any given long-lived server-side application. It makes sense to read and then pass your metrics to your exporter at a given interval while your application hums along in those cases.

However, in my case, I don't have a long-lived application. Some of the commands in the CLI that I'm attempting to instrument run for sub 1 second, so having a PeriodicReader implementation that collects every 1s doesn't really make sense in my use case. However, it's worthy to note that it's not exactly a problem that the PeriodicReader doesn't hit it's interval, I noticed in my tests that it always still seemed to get the metrics from those short-lived runs. My [ed-chu-ma-kate'd guess](https://github.com/open-telemetry/opentelemetry-go/blob/088ac8e179bd30ee39c81278010f8d3b45ba45be/sdk/metric/periodic_reader.go#L320) is that the Exporter I was using - when it receives a Shutdown command from a `defer`'d function at the end of the command invocation - will run a last `Collect` and export the metrics before gracefully exiting.

While this _works_ it just _felt wrong_ to rely on. What if my command runs for _more_ than 1s and it `Collects` and then exports that multiple times? I mean, it's probably not a _real_ problem because I'd assume that either the Reader would have dumped the metric and wouldn't send it again, or that the server I'm sending these metrics to would be able to intelligently dedupe any duplicate metrics.

The other side of this argument, though, is what happens if my command hangs and runs indefinitely, and then the end-user has to SIGKILL the application? If we only collect manually once at the end of our CLI application run then we're going to miss that data. The `Reader` interface allows us to also `ForceFlush` the reader, which will also make an on-demand collect and export in the case of the PeriodicReader. With all of this in mind, let's take a look at the ManualReader anyway.

## Manual Reader

From the name alone, this is a Reader implementation that _I_, the maintainer, have to tell when to act. This sounds like it's exactly what I need in a command-line application to collect some simple metrics. However, there wasn't really much content on the internet on how to actually use something like this, aside from [the unit tests](https://github.com/open-telemetry/opentelemetry-go/blob/64e76f8be45c9f4df85344249fad0fb72cff1230/sdk/metric/manual_reader_test.go) which don't _really_ go too deep into how to actually use it.

At least, that's how I first understood it. Looking back on it now, those unit tests show _exactly_ how to use the Manual Reader - the piece that wasn't clicking for me was that the unit tests show that the expected output is an [empty error with no metrics](https://github.com/open-telemetry/opentelemetry-go/blob/64e76f8be45c9f4df85344249fad0fb72cff1230/sdk/metric/manual_reader_test.go#L94). What I was missing here is that while in this unit test we're expecting an empty error AND an empty struct, its OUR job as the consumer to pass the empty struct pointer to the `Collect` function to be filled, or more succinctly - when they call it a Manual Reader - it's _entirely_ manual. Where I thought this empty struct was just being compared for the sake of the unit test, it's actually what we use to populate our metric data, then we can pass that to our exporter. This has since been updated in the test code to be more clear.

Now the question really comes down to - When would you use a Manual Reader vs a Periodic Reader? Originally I thought that because I only needed to send the metrics once at the end of the execution that the Manual Reader would be a better fit, but I was actually convinced by one of the maintainers that a PeriodicReader is probably the best bet here for the sole reason of if you have to SIGTERM a command execution that at least _some_ of your emitted metrics will already be sent to the exporter.

One example of when you'd actually want to use a Manual Reader would be if you want to roll your own logic around reading, like if you have specific needs of _when_ to send metrics. This is utilized by the Prometheus exporter, a pull-based metric exporter, which takes advantage of a manual reader to grab all available metrics when the Prometheus server asks for them.

# Code Examples

Let's walk through some code examples I created as part of a way to iterate through a few working examples. I first started with an [example provided within the opentelemetry-go-contrib repository](https://github.com/open-telemetry/opentelemetry-go-contrib/blob/instrumentation/runtime/example/v0.42.0/instrumentation/runtime/example/main.go). This example sets up a Periodic Reader that reads some runtime metrics every second and "exports" them to stdout.

Iterating on that example, let's replace the Periodic Reader implementation with a Manual reader implementation, and instead of having it cycle through let's just output once and exit.

{{< gist iamkirkbater 70d1bf28ac504f91dff9223a208e1a73 iteration-1.go >}}

We'll notice that there are a few small changes.  First is that we initiate a new ManualReader, but then I also change here to use a Counter. I'm specifically looking for my MVP (minimum viable product) to only care about how often individual commands were called, so I wanted to test this specific metric and see what it looked like (and like I mentioned earlier, not loop but just count it once).

The last, and most important change is the last few lines of code at the bottom, where I create an empty struct of ResourceMetrics then `Collect` the data into that struct from the reader, and then call upon the exporter to `Export` the collected metric data.

It might not seem like a large change, but piecing together this last part took me a few hours, and it was the biggest "aha" moment I've had in a long time. I ran the program, and saw the metrics exported on the screen, and I don't think I've been happier to see JSON printed out on my terminal.

## Abstraction

Iterating on the previous example one more time, I wanted to see how "troublesome" it would be to abstract away some of the setup and teardown. Remember, my goal is to get this code idea integrated into a Cobra CLI application, so we'll definitely need to figure out how to abstract the startup and teardown portions to make sure they're getting called everywhere correctly.

{{< gist iamkirkbater 70d1bf28ac504f91dff9223a208e1a73 iteration-2.go >}}
([view the diff here](https://github.com/iamkirkbater/opentelemetry-go-examples/commit/b2d4789d3f6ff1b9b9537562ecf4004a41964391))

Going through the diff, we abstract the metricReader to be accessible throughout the entire file, and we pull out the setup functionality and closeout functionality to separate functions. The one pattern here that I borrowed from some other examples that I had been reading was to return the Shutdown functionality as a function from the `setup_metrics` function, which will allow me to find a place in the root file to do both the setup and shutdown functionality of metrics in the same place by deferring the returned function, resulting in a less-leaky abstraction.

```go
func setup_metrics() func() {
  provider := metricsdk.NewMeterProvider(
    metricsdk.WithResource(res),
    metricsdk.WithReader(metricReader),
  )
  otel.SetMeterProvider(provider)
  return func() {
    err := provider.Shutdown(context.Background())
    if err != nil {
      log.Fatal(err)
    }
  }
}

...

func main() {
  shutdown := setup_metrics()
  defer shutdown()
}
```

With this abstracted, I now have a working example in which I can continue to iterate on in order to get this integrated into a Cobra application.

Stay tuned for the next article where we'll start building an add-on for Cobra, which will be available to be pulled into your cobra applications!
