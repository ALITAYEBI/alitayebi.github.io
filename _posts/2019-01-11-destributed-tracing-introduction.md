---
published: false
---
## Distributed Tracing Introduction

Monolithic service architectures for large backend applications are becoming increasingly rare. The monoliths are being replaced with distributed system architectures, where the backend application is spread out (distributed) in an ecosystem of small and narrowly-focused services. These distributed services communicate with each other over the network to process requests.


### What is distributed tracing?

Distributed tracing is a process of collecting, analyzing and showing what happen to a request or transaction across all services ut touches in near realtime. It lets you see the path that a request takes as it travels through a distributed system. You can see the what is really going on with your request and helps with latency processes.

To have a better understanding let's review few difinitions:

> Distributed tracing, also called distributed request tracing, is a method used to profile and monitor applications, especially those built using a microservices architecture. Distributed tracing helps pinpoint where failures occur and what causes poor performance.    - [OpenTracing](https://opentracing.io/docs/overview/what-is-tracing/)

And also:

> Distributed tracing is the process of tracking the activity resulting from a request to an application. With this feature, you can:
> * Trace the path of a request as it travels across a complex system
> * Discover the latency of the components along that path
> * Know which component in the path is creating a bottleneck    - [NewRelic](https://docs.newrelic.com/docs/apm/distributed-tracing/getting-started/introduction-distributed-tracing#definition)

You can easily find a bunch of other definitions around the net. However, the what almost all of them can agree on is that distributed tracing is fundamentally about tracking and analyzing requests as they bounce around distributed architectures as a whole. This is in contrast to traditional monitoring that focuses on each service as an individual in isolation and where specific request details are lost in favor of aggregate metrics.

I don't want you to get distracted by nuts and bults.So, just keep in mind that distributed tracing is conceptually pretty straightforward: **it’s about understanding how a request is processed as it hops from service to service**.

### Where it's come from?

In monolithic web applications, logging frameworks provide enough capabilities to do a basic root-cause analysis when something fails. A developer just needs to place log statements in the code. Information like "context" (usually "thread") and "timestamp" are automatically added to the log entry, making it easier to understand the execution of a given request and correlate the entries.

```
Thread-1 2018-09-03T15:52:54+02:00 Request started
Thread-2 2018-09-03T15:52:55+02:00 Charging credit card x321
Thread-1 2018-09-03T15:52:55+02:00 Order submitted
Thread-1 2018-09-03T15:52:56+02:00 Charging credit card x123
Thread-1 2018-09-03T15:52:57+02:00 Changing order status
Thread-1 2018-09-03T15:52:58+02:00 Dispatching event to inventory
Thread-1 2018-09-03T15:52:59+02:00 Request finished
```

We can safely say that the second log entry above is not related to the other entries, as it's being executed in a different thread.

In microservices architectures, logging alone fails to deliver the complete picture. Is this service the first one in the call chain? And what happened at the inventory service (where we apparently dispatched an event)?

A common strategy to answer this question is creating an identifier at the very first building block of our transaction and propagating this identifier across all the calls, probably by sending it as an HTTP header whenever a remote call is made.

In a central log collector, we could then see entries like the ones below. Note how we could log the correlation ID (the first column in our example), so we know that the second entry is not related to the other entries.

```
abc123 Order     2018-09-03T15:52:58+02:00 Dispatching event to inventory
def456 Order     2018-09-03T15:52:58+02:00 Dispatching event to inventory
abc123 Inventory 2018-09-03T15:52:59+02:00 Received `order-submitted` event
abc123 Inventory 2018-09-03T15:53:00+02:00 Checking inventory status
abc123 Inventory 2018-09-03T15:53:01+02:00 Updating inventory
abc123 Inventory 2018-09-03T15:53:02+02:00 Preparing order manifest
```

This technique is one of the concepts at the core of any modern distributed tracing solution, but it's not really new; correlating log entries is decades old, probably as old as "distributed systems" itself.

What sets distributed tracing apart from regular logging is that the data structure that holds tracing data is more specialized, so we can also identify causality. Looking at the log entries above, it's hard to tell if the last step was caused by the previous entry, if they were performed concurrently, or if they share the same caller. Having a dedicated data structure also allows distributed tracing to record not only a message in a single point in time but also the start and end time of a given procedure.

![distributed-trace](https://www.jaegertracing.io/img/trace-detail-ss.png "distributed trace") 

### Why You Want Distributed Tracing?

Here are a few of the questions that distributed tracing can answer quickly and easily. That might otherwise be a nightmare to answer in a distributed system architecture:

- **What services did a request pass through?** Both for individual requests and for the distributed architecture as a whole (service maps).
- **Where are the bottlenecks?** How long did each hop take? Again, distributed tracing answers this for individual requests and helps point out general patterns and intermittent anomalies between services in aggregate.
- **How much time is lost due to network lag?** during communication between services (as opposed to in-service work)
- **What occurred in each service for a given request?** tagging service log messages with the given request’s Trace ID allows finding all log messages associated with a particular request across all services it went through (when combined with a log aggregation and search tool).

![What happen to my request?](https://cdn-images-1.medium.com/max/1000/1*iaVQRRi2upK4X-E-WwrWIg.png "What happen to my request") 

Distributed tracing is especially helpful on difficult-to-reproduce or intermittent problems. Without distributed tracing, the info might be lost forever or be so difficult to unearth that you’d never find it. When it’s 3:00 a.m. and an alert is going off, and you’re on-call, being able to use distributed tracing to quickly point to the not-my-service culprit is invaluable. Distributed tracing can often let you go back to bed in a few short minutes or at least point you in the right direction for why the problem is in your service. This saves you hours of guesswork, following red herrings, and painful debugging.

**Distributed tracing is extremely useful even for a single service where upstream and downstream haven’t implemented distributed tracing**. You’ll still be able to answer the question of how much time was spent in your service vs. waiting for outbound calls to complete. If an outbound call is especially slow, then you’ll know where to point the finger when someone asks why their request was laggy.

### Trace Anatomy

In order to discuss the core concepts for how distributed tracing works, we first need to define some common nomenclature and explain the anatomy of a trace. Note that distributed tracing has been around for a long time, so if you research distributed tracing you might find other tools and schemes that use different names. The concepts, however, are usually very similar:

1. A **Trace** covers the entire request across all services it touches. It consists of all the Spans for the request.
2. A **Span** is a logical chunk of work in a given Trace.
3. **Spans have parent-child relationships**. This is a very important concept that is easy to miss if you’re new to distributed tracing. Let's highlight a few details:
+ A “child span” has one span that is its “parent.”
+ A “parent span” can have multiple “child spans.”
+ This parent-child relationship allows you to create a “trace tree” out of all the spans for a request.
+ The trace tree always has one span that does not have a parent — this is the **“root span.”**

![Trace Tree?](https://cdn-images-1.medium.com/max/600/1*VO9RZ-wwHUWQDEkNSzf5vA.png) | ![Trace](https://cdn-images-1.medium.com/max/600/1*Yu0bCux_sulHPy6MhT9Ytg.png)


### Resources for Implementing Distributed Tracing?
Search around once you’re ready to implement distributed tracing. You’ll find libraries and tools for distributed tracing in almost any language and stack/framework you want. Here are a few links to get you started:

- [OpenTracing](https://opentracing.io/)
- [Jaeger](https://www.jaegertracing.io/)
- [Zipkin](https://zipkin.io/)
- [OpenCensus](https://opencensus.io)
- [NewRelic](https://docs.newrelic.com/docs/apm/distributed-tracing)

### The Last Word
Distributed tracing is critical to operating, maintaining, and debugging services in a distributed systems architecture. Distributed tracing directly translates into providing the best possible experience for consumers and developers alike. It provides deep insight into individual services as well as into how they communicate and work together as a whole. Distributed tracing makes the lives of everyone — from product owners, to prod support, to service developers — much easier and less frustrating.

