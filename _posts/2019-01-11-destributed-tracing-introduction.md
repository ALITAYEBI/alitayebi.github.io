---
published: false
---
## Destributed Tracing Introduction

### What is destributed tracing?

Destributed tracing is a process of collecting and showing transaction graphs in near realtime. It lets you see the path that a request takes as it travels through a distributed system. You can see the what is really going on with your request and helps with latency processes.

To have a better understanding let's review few difinitions:

> Distributed tracing, also called distributed request tracing, is a method used to profile and monitor applications, especially those built using a microservices architecture. Distributed tracing helps pinpoint where failures occur and what causes poor performance.    - [OpenTracing](https://opentracing.io/docs/overview/what-is-tracing/)

And also:

> Distributed tracing is the process of tracking the activity resulting from a request to an application. With this feature, you can:
> * Trace the path of a request as it travels across a complex system
> * Discover the latency of the components along that path
> * Know which component in the path is creating a bottleneck    - [NewRelic](https://docs.newrelic.com/docs/apm/distributed-tracing/getting-started/introduction-distributed-tracing#definition)

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

![distributed-trace](https://www.jaegertracing.io/img/trace-detail-ss.png "Logo Title Text 2") 
