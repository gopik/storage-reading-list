# What is API flow control
A mechanism that we would like to use to maintain an optimal rate with which we call APIs. Usually this is done by configuring rate limits.
Consider an image upload service that provides an API to authenticated clients to upload images and makes them available for view/download/search.
This functionality has different performance limits, like

1. Uploads per second
2. Megabytes per second

as well as variability in response like -

1. Index lag (item available for search after upload)
2. View lag (which depends on index for example)

# System limits

Ideally we want very high upload rate and low response times but there are limits to a system which limits these. Beyond these limits, many of these these things occur -

1. Uploads start failing.
2. Uploads time out.
2. Lags increase to unacceptable ranges.

Hence it's important to restrict the service usage within system limits. Usual approach is to limit it conservatively and when system usage approaches the limit, we increase the resources available to the system. The resources that define the system limits are usually following (and more) -

1. Available network bandwidth (ingress/egress)
2. Storage bandwidth
3. CPU time
4. RAM
5. Other resources that limit concurrency (maximum number of active connections to db etc)

## Bottlenecks

Usually systems are bottlenecked by one of these resources and the system capacity is determined by the bottleneck resource. Increasing other resources would not improve system capacity if it's limited due to a bottleneck resource.

# System capacity parameters

Since the system doesn't perform well when utilized near it's capacity limits, it's important to manage it's usage properly. First step is to understand
the system capacity. This is defined as maximum load the system can take such that response time stays within some maximum limit. For example, if we consider the system response time as the index lag and limit it to be under 30s, then the capacity of the system would be maximum upload rate (in requests per second or megabytes per second) such that all uploads are indexed within 30s of they getting success response from the system.

## Rate limits
This defines an API limit beyond which system becomes unresponsive. Then a usual approach is to keep the combined request rate across all clients within this limit. To ensure fairness, rate limits could be capped per client (via auth token or api key etc) but that's secondary detail.

## Concurrency limits
An alternative approach is to define the system capacity in terms of maximum concurrent requests that can make progress. If there are requests in flight equal to maximum concurrency of the system and more requests arrive, they are enqueued and kept waiting until one of the inflight request finishes. This adds to the latency of the request. What this implies is, beyond the maximum concurrency, requests spend useless time waiting in queue.

## Throughput
A closely related metric to request rate is throughput. This is the rate of work finished in the system. This is not same as request rate though. Request
rate is the rate at which requests enter the system and throughput is the rate at which requests finished. They are same as long as the number of
in flight requests is less than system concurrency. So in a way this is a property of the system where as request rate is external stimulus to the system.

# Little's Law
[Little's law](https://en.wikipedia.org/wiki/Little%27s_law#Finding_response_time) shows the relationship between system response time, throughput and concurrency. According to little's law:

```math
InflightRequestCount = AverageResponseTime * AverageThroughput
```

While the pending requests are less than maximum concurrency of the system, average response time is not impacted by request rate and throughput remains same as request rate. Once maximum concurrency is reached, throughput rate remains clamped and average response time increases as request rate increases.

# System Model

This is how a simplified model of our system will look like. Basically it is represented by a queue where requests land, and a set of resources with count same as maximum concurrency of the system. Whenever a resource is free, it takes an item from the queue and occupies the resource for entire duration of the request. It doesn't matter if request is synchronous or asynchronous, but until system's response contains the effect of the request, this resource is assumed to be occupied.

<img src="https://raw.githubusercontent.com/gopik/storage-reading-list/main/SystemModel.drawio.svg"/>

This is explained in a much clearer way in the netflix [blog](https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581) on concurrency limiting. The way we would model our image upload system is, when an upload request comes, it's added to the queue. Some request is picked up from queue and a resource is occupied till the image uploaded by this request is visible in search or some gallery etc. Once it is indexed, the resource is released.

Note it doesn't matter that upload api call itself will write the data somewhere and return a success code. But from the bigger system's perspective, it's still not complete.

Now it's clear that the requests sitting in the queue are not useful. They add to the response time and are committed from the caller's perspective.
The 2nd point is very important, say the request is in queue and system is busy. If the request now times out, the client doesn't know that it was
never seen by the system and is safe to be retried. Instead, if this request waited in client's queue, the state of the request is clear. Hence it's
important to avoid queued state (on the input side) where possible.

## Model utility
This model is a very simplistic representation of a system. Specifically, this ignores the types of requests (some might be expensive, others cheap),
system load etc. This model captures average behaviour of the system in steady state. This model is still very useful. 

For example, this model helps us understand what would happen if queue size keeps increasing over time.
Eventually more system resources will be spent in queue management instead of actually handling the request. 

### Failure scenarios
This scenario can easily happen in some failure scenarios. Let's say a downstream service is down and requests start timing out. Now the average latency of that api raises to timeout.
Using little's law, we can estimate the average queue size in this case. If timeout is 10x the typical latency of a working system, average queue
size will grow by 10x. This implies that other requests which shouldn't be impacted by this failure will get slower since they'll be behind in the queue.

### Burst scenarios
Let's say an api receives a burst of requests much higher than it can typically handle. All the remaining requests will get queued up and the queue size
will be dependent on the burst duration. Let's consider the system throughput is 100 requests per second. We get a burst of 1000 requests per second for 1 minute. The queue size at the end of one minute would be 60000 - 6000 = 56000 requests. At the average rate of 100 requests per second, this queue will
take 560 seconds to drain. Now assume the timeout for such request is 5 seconds. In 5 seconds only 500 requests would be handled and 55500 requests would
end up timing out. This means, out of this 60000 requests 55500 will any way timeout (client doesn't care about the result and results in wasted work) and need to be responded to by the service. And for approximately 10 minutes the service remains unavailable. This could have been prevented if the calls were rejected upfront.


# Request Limiting

Request limiting is done to avoid exceeding system capacity which would reduce the availability of the system. This is either done proactively where we know the system limits end ensure that it's operated well within it's limits. Another approach is where the limiting is done reactively when we can figure out that we are operating in a region near system capacity.

### Rate limits

With rate limits, for each request we check if accepting this request would exceed a rate that is specified by average number of requests in a given time duration. If so, either the request is rejected, or buffered until some time is elapsed. This is a static configuration that is set either by proper load testing or by trial and error (where we have a configured request limit and adjust the resources until system can work under such limit). Usually rate limits are configured as granular as required, like per API rate limits or per client or per API/client combination.

### Concurrency Limits

With concurrency limits, we limit how many in progress requests are allowed in the system. If a request comes and in progress limit is reached, the request is either dropped or we wait until some time passes hoping some pending requests finish. These limits need to be identified by load testing or trial and error. 

This method is more robust to changes to system performance compared to the rate limit approach. This provides for automatic back pressure by keeping pending request count limited. Whereas in case of rate limit, if system throughput reduces, the backlog of requests will keep increasing until system
recovers.

# Adaptive Limiting

This is an approach of request limiting that adapts the limits based on system response. We estimate the limits based on current system performance. For example rate limits can be adjust dynamically based on the latencies. Similarly concurrency limits can be decided based on current system performance parameters.

For example, as explained in **AWS** [loadshedding](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) article, the system tracks certain performance parameters and based on this, it might reject certain requests.

In an adaptive system, we need to deal with following issues -

1. **Error in adaptation** - Since we are estimating the limits based on observable parameters, there will be some estimation error.
2. **Adaptation lag** -  We react to observed parameters, our adaptation impacts the system, which in turn causes further adaptation. This feedback loop takes time to stabilize.
3. **Instability** - Since this is a [closed loop](https://en.wikipedia.org/wiki/Control_theory#Closed-loop_transfer_function) feedback system, incorrect adaptation might worsen the system performance.

Based on above issues, it's safer to limit concurrency compared to rate. Let's see why:

Error in adaptation: Error in concurrency estimation results in additional requests getting queued up, but the queue size remains constant (bounded). In case of error in rate estimation, queue size keeps increasing at the rate of the difference in estimated rate and actual rate.

Adaptation Lag: In case of concurrency estimation lag, a fixed size queue builds up during the adaptation. But once estimated, this queue gets consumed and brought to 0. In case of rate limit lag, any queue built during the lag remains as is even after we found the correct rate. To drain the queue, the rate must be compensated negatively.

Instability: This issue remains in both limits. This causes the estimate to oscillate around the true value and depend on the actual algorithms used for estimation.

# Adaptive concurrency limits
If we are able to estimate the concurrency limits of the system, we get the following advantages:
1. No need to pre configure any hard coded limits.
2. No need to revisit as the system evolves.
3. Any transient changes in system parameters provide feedback to the callers.

The objective of adaptive limits is to optimize the system utilization and avoid overloading where system performance worsens. This provides a mechanism of "backpressure" to the clients to change the request flow based on observed performance parameters. This means, the request flow can then be managed at the client itself and we get the benefit of requests waiting in client output queue instead of system input queue.

With this another interesting problem appears, what happens when there are many clients to this system that are doing this adaptation. This needs another property of the adaptation - Fairness. The clients should be able to independently adapt using the algorithm and eventually end up using fair share of system resources also ensuring system utilization is maximized.

This problem appears in other domains and has been solved. An equivalent problem has been solved in the TCP congestion control and congestion avoidance.
In case of TCP flow control, the client is the tcp client, requests are packets and the system is the network of routers through which the packet flows.
The capacity of the system is identified by the bottleneck link in the network. The objective of the flow control is to maximize the resource usage (bandwidth) without bringing system down (queue overflows). The observed parameters of the system are packet loss and rtt. When system is under heavy load, there'll be lot of packet loss and delays in packet delivery.

From api flow control perspective, bandwidth maps to api resource consumption. This shows up in the response latency (note: this is not exactly equal to response latency). Hence bandwidth from TCP flow control maps to response latency instead of the rtt.

## Model of the system parameters for adaptation

Let's consider various parameters of the model that'll help with adaptation. Some parameters are stimulus parameters and some are response.
We can control stimulus parameters, some of the responses which may or may not be observable. The observed parameters will be used for the feedback
about system load and then we update the stimulus parameters.

1. Throughput (R) - Rate at which requests finish. We consider successful requests for this rate.
2. Queue Length (Q) - Number of requests waiting in queue, can't be observed.
3. Processing time (T) - Time taken to handle a request once it started processing (doesn't include queue wait time)
4. In flight requests (P) - Number of requests that are pending response. This is observable and can be controlled.
5. Request rate - The rate at which requests come to the system, this is observable and can be controlled, this impacts inflight requests.
6. Queue wait time (W) - Time a request is waiting before it is taken up for processing.
7. Available concurrency (C) - Maximum number of requests that can be processed. Can't be observed or controlled.
8. Observed latency (L) - Complete time taken for a request to finish. This is observed parameter.

All these parameters are not independent. Here's how they are related:

* **P = Q + C** Number of inflight requests is sum of requests being handled by the system and requests waiting in queue.
* **C = R * T** Available concurrency is product of throughput and processing time.
* **L = W + T** Observed latency is sum of queue wait time and processing time.
* **W = Q/R** Average wait time in queue is requests in queue divided by rate of completion of requests.

The direct feedback parameter is Q (number of requests in queue) which is not observable. But it impacts latency, so can be estimated by
latency changes approximately. This can be indirectly controlled (approximately) by number of requests in flight.

### Fairness
When system receives requests, according the model, they first enter the queue and handled in FIFO manner. At any given time t, if more than one
request arrive, they are added to the queue in arbitrary order. Hence, if the resource is free, any request is equally likely to get it.

This means, if different clients are sending requests at different rate, the clients are likely to get resources proportional to the request rate (assuming all requests are identical). If requests are not identical, they end up getting resources proportional to the both request rate and processing
time of request rate. This means slower requests are more likely to occupy the resources compared to faster resources if they both are sent at the
same rate.

To ensure fairness, the metric used is total resource usage being shared equally by all clients. Hence C (concurrency) which is proportional to the product of resource usage time (T) and throughout (R) should be assigned fairly to all clients.

## Adaptive limiting control algorithms

The objective of the limiting algorithm is to ensure utilize system resources as efficiently possible and use it fairly between different clients
The limiting algorithms are designed to work with wide ranges of system parameters but they may or may not work well in different ranges. Some algorithms
may work well when the average observed latency is high and others might work well when it's low. Usually they result in not able efficiently utilizing system resources or resource split between clients is not very fair. The best algorithm for a particular situation needs simulation of system model and application of a particular algorithm for a given model parameter ranges.

Here's are a few control system algorithms used in practice. These algorithms discover the optimal rate or pending request count. Typically they have different states, for example in one state they might be probing the capacity of the system. They keep looking for feedback signals and correct their estimate of system capacity. 

### AIMD

AIMD stands for **additive increase** and **multiplicative decrease**. Increase/Decrease refers to the load put by the client on the system. The feedback signal is a boolean signal. If the signal is off, load is increased additively. Say the load refers to in flight requests. While the feedback signal is off, every iteration increases the load by a fixed quantity. When the feedback signal is on, the requests per second will be reduced by a fixed proportion.

Say additive quantity is 10 reqs, and multiplicative factor is 0.5. Here's how the client will react.

1. No signal, 10 reqs in flight.
2. No signal, 20 reqs.
3. No signal, 30 reqs.
4. Load signal, 15 reqs inflight. Don't send more requests until pending request comes down to 15.
5. Load signal, 7.5 reqs.
6. No load, 17.5 reqs.

If different clients of the same system use the same additive quantity and same reduction factor, then those clients will converge to an equal resource usage. See [AIMD video](https://www.youtube.com/watch?v=LFyGWEJqU-c) for demonstration.

Note: The request count is weighted by the cost of the request since all requests are not equal. This will be assumed where wherever request count (or rate) is mentioned.

### Netflix/EnvoyProxy gradient concurrency limit algorithm

In both these algorithms, the parameter that's controlled is pending requests in flight. The signal for system load is a function of latency. The idea is we estimate a **no load** latency of requests and assume this as true cost of processing without queuing (**T** described above). If the observed latency
is much higher than no load latency, the requests are throttled proportional to the ratio of observed latency to no load latency. This also has a mechanism to keep probing for higher request rate if observed latency is close to no load latency.

**gradient** = some function of (sample latency / exponentially weighted average of latencies)

**pending request count** = Some factor * **gradient**

It most likely doesn't guarantee fairness between multiple clients. AIMD video claims that AIMD is only function that guarantees fairness, any other increase/decrease doesn't. This falls under MIMD (multiplicative increase multiplicative decrease).

**Netflix** has this logic implemented as a library which is available here - https://github.com/Netflix/concurrency-limits. Gradient based limiting is one of the strategies available.

**Envoy proxy** has concurrency limiting as part of proxy functionality. While netflix implementation is aimed at the providing this as client library, envoy proxy is implemented as server side filter. Envoy proxy implementation limits the total number of pending requests across all hosts in a cluster to a specific limit which is estimated adaptively.

### Google's BBR

Google's BBR algorithm is a congestion avoidance algorithm that aims to probe the bandwidth and has outstanding packets that are just sufficient to fill the "pipe" (if we consider the network as a pipe of certain length which is defined by the smallest time a packet takes to reach destination) but not be queued anywhere. The idea is, network has a bottleneck link and that defines the maximum bandwidth of the the network. BBR algorithm tries to estimate following parameters -

1. Bandwidth - How fat the pipe is.
2. Round trip time - How long the pipe is.

Once these 2 are computed, BBR aims to keep (Bw * Rtt) packets in flight. To estimate RTT, **Q** from the system parameters should be as small as possible. To estimate Bw (**C** from system parameters), the queue should be non empty (otherwise we haven't reached system limits). This means both can't be estimated at the same time. Hence BBR algorithm runs in different phases -

1. Start Up phase - Quickly ramp up packet send rate to estimate the capacity of the network.
2. Drain - During start up probe, we must have queued up additional packets.
3. Bw Probe - A unit gain filter that probes by slightly increasing the packet rate (and compensates if currently running at max capacity)
4. rtt probe - Probe the rtt once in a while (which requires inflight requests to be close to 0)

Allow (Bw * rtt) packets in flight.

This can be mapped to api flow control by mapping Bw to the to the cost of each request (**R*T**). rtt probe can be used to estimate the average processing time of each request.

Why do we map Bw to **R*T** and not **R**? The reason is, **R** is really a packet rate and not data rate. So it favours large packets if all flows were given the same packet rate. **T** is proportional to packet size, hence Bw really maps to **R*T** and not just **R**.

Is this flow control fair to multiple clients running the same algorithm? I am not very sure if it is fair or not. In network flows, BBR claims to be fair (at least the v2). This [article](https://www.sciencedirect.com/science/article/pii/S2405959520301296) might provide insights.

# Conclusion
We explored how to model a system and system capacity. We looked at different parameters that are important to understand the current load on the system.
We looked at fixed and adaptive limits to keep system load under control. We also looked at how do we ensure fairness between independent clients that are all trying to maximize the utilization of the system and some algorithms on how to actually do an adaptation.


# References
* https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/adaptive_concurrency_filter
* https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581
* https://www.youtube.com/watch?v=LFyGWEJqU-c - How AIMD works.
* https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/ - Amazon article on system behaviour on overload.
