---
layout: post
title:  "Why Our Node 22 Upgrade Kept Killing Our Pods"
date:   2026-04-08 19:02:22 -0500
author: Cody Perry
categories: programming
tags: nodejs kubernetes debugging performance]
canonical_url: https://underthehood.meltwater.com/blog/2026/04/08/why-our-node-22-upgrade-kept-killing-our-pods/
---

As an engineer on one of Meltwater's enablement teams, I work on managing our user authentication and permissions, and making that data available to other engineering teams.

In this blog post, we will be exploring our recent experience hunting down a memory leak after bumping from Node.js 18 to 22. This post will not explain how the heap works, nor the finer details of debugging like retainer chains. There are far more expansive articles out there on those topics. Instead, we will focus on our upgrade process, root cause discovery, fixes implemented, and lessons learned.

## TL;DR

We have a Kubernetes hosted Node.js service that we bumped from 18 to 22. We observed pod restarts every 6-8 hours from exceeding the container resource limits. The memory metrics were growing consistently and never dipping. Here's a representative screenshot of the growth behavior:

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/memory-growth-over-time.png" alt="Grafana dashboard showing container memory steadily increasing from 2 GiB to over 4 GiB over several days" title="Container memory growth over time" />
<figcaption>Container memory growing steadily over time without any dips, indicating a memory leak</figcaption>
</figure>

The memory issues were caused by a number of issues, the main ones being:

- Dependency object and closure retention
- Insufficient caching logic
- Cascading effects of underlying V8 engine changes

The rest of this article will go in-depth into the many fixes implemented to address the above, as well as the lessons learned from the overall experience. The key takeaway is to monitor and alert on the core Node performance metrics, especially after runtime upgrade.

## Background

The service we will be discussing today manages user's permissions across the application, and is deployed to Kubernetes. The volume of the permissions service is around ~23 req/s.

As a team, we try our best to keep up with Node's LTS <a href="https://nodejs.org/en/about/previous-releases" target="_blank">version releases</a>. When we have end of life (EOL) versions, especially those going unsupported in AWS, we try to sync the versions across all of our services at once. The upgrade process is simple:

- Update the <a href="https://github.com/nvm-sh/nvm" target="_blank">NVM</a> version
- Run `npm install`
- Rerun all automated tests to ensure no regressions
- Ship to our staging environment
- If there are no issues, namely functional regressions, ship to production

We followed that same process for our permissions service. There were no alerts and no functional regressions. What we did not realize at the time was that we had already seen a warning sign. We had previously attempted to upgrade an authentication service to Node.js 20. After upgrading the authentication service, we began to have memory issues causing the Node process to die from failed heap allocations. We made some patches to the authentication service, but ultimately were unable to address those memory issues. Looking back, that should have been the first red flag to us, as it reuses similar code via dependencies and many of the same patterns.

## Investigation

Although there were no functional regressions, in the background, we had silent failures and regressions we were unaware of: pod restarts.

### Warning Signs

Once we noticed the pods kept restarting, it was trivial to extract the reason. The scary picture was how often it was restarting:

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/pod-restarts-kubectl.png" alt="kubectl get pods output showing permissions deployment pods with hundreds of restarts and one in CrashLoopBackOff" title="Pod restart counts" />
<figcaption>kubectl output showing pods with over 300 restarts each, and one pod in CrashLoopBackOff</figcaption>
</figure>

```bash
kubectl describe pod POD_NAME -n NAMESPACE
```

Running the above gives the reason for the restart. In our case, it was the dreaded OOMKilled error. Here is some sample output of the OOMKilled error:

```yaml
Containers:
  my-app-container:
    Container ID: docker://3f1c2e8b9a7c6d5e4f1234567890abcdef...
    Image: my-app:latest
    Port: 8080/TCP
    Host Port: 0/TCP
    State: Running
    Last State: Terminated
      Reason: OOMKilled
      Exit Code: 137
    Ready: True
    Restart Count: 3
```

Just because your pod gets an OOMKilled error does not necessarily mean you have a memory leak however. We had been doing active feature development work in this service, including introducing caching logic.

### First Steps

It was possible that this feature work increased the memory footprint naturally to exceed the defined resource limits. Our first step was to increase the pod limits and monitor, but they kept running out of memory. Here is another snapshot of the OOM behavior with a restart in between:

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/oom-behavior-with-restart.png" alt="Grafana dashboard showing container memory climbing from around 200 MiB to over 350 MiB with periodic drops from pod restarts" title="OOM behavior with pod restarts" />
<figcaption>Memory climbing steadily across pods, with visible drops from OOM-triggered restarts</figcaption>
</figure>

Having never investigated an out of memory issue in Node before, we started with some simple tasks:

- Cleaning up problematic logic we found in the authentication service duplicated in the permissions service
- Bump all dependencies in case of incompatibility
- Upgrade past Node 22
- Downgrade to Node 18

After each of these changes, we monitored the memory, but it kept increasing over time after deployment (yes, even when we downgraded back to Node 18).

This led us down two paths: what changed in Node and when did this start? The former was harder to discover but the latter was easy. The memory issue started after we upgraded the service to Node 22, and an internal dependency (which also bumped it to Node 22). This explained why the downgrade to Node 18 failed, and it was not possible for us to downgrade both the library and service (due to underlying AWS requirements). We validated our assumptions about Node against two other services, one on Node 18 and one on 22. The service with 18 had no memory issues, and the other Node 22 service had the same problem as permissions.

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/memory-growth-over-time.png" alt="Grafana dashboard showing container memory steadily increasing from 2 GiB to over 4 GiB over several days" title="Consistent memory growth pattern" />
<figcaption>The same consistent memory growth pattern observed across affected services</figcaption>
</figure>

For the changes in Node 20 and 22 specifically, we spent time researching GitHub issues and changelogs. Ultimately, we discovered that two main changes occurred after Node 18. First, heap sizes were now computed differently inside of V8, resulting in lower spaces depending on your configured settings (including the defaults). Second, the way async closure retention was handled was updated in V8 to improve performance but could cause retention if you do not explicitly clean up resources (previously those closures would auto resolve and get cleaned up by GC).

This was definitely **not** a memory leak in the runtime, but the two threads identified underlying changes that had effects on our performance and we did not know why.

## Root Cause Discovery

Now that we knew we had a problem, we had to first learn how to debug a Node process effectively.

### Learning

All of our services run within Docker, so we added the following two changes:

```dockerfile
EXPOSE 9229
```

```bash
node --inspect=0.0.0.0 server.js
```

The first exposes the websocket debug port on the container, the second enables the Node inspector on the process using port 9229. Please note, it is **not** recommended to use `0.0.0.0`. This allows traffic from **any** location. We accepted this risk as we were running locally. Do not do this in production.

Once the inspector is running, you can connect to your Node process using Chrome's DevTools by going to `chrome://inspect` and then selecting your process. This allows you to start observing performance and memory.

In parallel to trying out the different memory options, we implemented a memory logger to see what specifically was growing. On a 5 minute interval, we would log from the process and examine what was growing.

```javascript
process.memoryUsage()
```

The first thing we discovered was growing array buffers.

We then spent time gaining understanding of the different views in DevTools and how to interpret them.

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/devtools-heap-snapshot-comparison.png" alt="Chrome DevTools summary comparison view showing over 2 megabytes of string allocations between two heap snapshots" title="DevTools heap snapshot comparison" />
<figcaption>The DevTools summary comparison view showing over 2 MB of string allocations between two snapshots taken 10 minutes apart</figcaption>
</figure>

For example, in the summary comparison view, you can run a difference against two heap snapshots. Here you can see that between two snapshots around 10 minutes apart, we allocated over 2 megabytes of strings, many of which were unexpectedly duplicated.

Once that investigative foundation was there, we could continue digging into what was different over time between snapshots.

### Steps Forward and Back

#### Duplicate Strings

Our first observation was an ever increasing amount of strings being created that were never cleaned up (over 1MB every 10 minutes). The key insight was these strings were highly duplicated, which is unexpected. Effectively we were caching authorization information every minute or so in the background. It would be expected that information would be cleaned up after refresh, but was not because the requests and responses themselves were being retained. To fix this, we cleaned up the fetch logic and caching in our library and bumped the version of the dependency in the permissions service.

#### Duplicate Requests

We used a library called <a href="https://github.com/forwardemail/superagent" target="_blank">superagent</a> for making requests to other APIs. We had been using it for a long time without problems. When running the service with no traffic, we saw an ever increasing number of request objects that were never cleared. To address this, we rewrote this logic with Node's native <a href="https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API" target="_blank">fetch</a>, and also ensured that closures were handled and closed properly. The previous logic led to closures being retained similar to the duplicate strings. Because they never truly resolved, the garbage collector never deallocated them.

#### Code Cleanup

Outside of the above, there were two other problems. First, there were <a href="https://en.wikipedia.org/wiki/Singleton_pattern" target="_blank">singletons</a> that were not actually singletons. Second, insufficient cache cleanup logic.

The problematic singleton was the AWS S3 SDK. We ended up creating multiple of these unexpectedly, which all held their own connections, array buffers, and more resources that were never garbage collected. We enforced a true singleton by creating the client in the constructor, and only exposing instances through a build function which reused an instance:

```javascript
constructor() {
  _s3Client = new S3Client({
    region: configuration.get('AWS_DEFAULT_REGION'),
    accessKeyId: configuration.get('AWS_ACCESS_KEY_ID'),
    secretAccessKey: configuration.get('AWS_SECRET_ACCESS_KEY')
  });
}

static build() {
  if (!_instance) {
    _instance = new S3ClientWrapper();
  }
  return _instance;
}
```

And here is a sample of how we introduced an interval to run a cache eviction function for an in-memory cache, addressing the second issue:

```javascript
_startCacheEviction() {
  _cacheEvictIntervalId = setInterval(() => {
    const removed = cache.evictExpiredEntries();
    if (removed > 0) {
      logger.info('CACHE_EVICT', {
        removed,
        entries: cache.size()
      });
    }
  }, 300000);

  _cacheEvictIntervalId.unref();
}
```

## Fixes

As you can see so far, unfortunately, there was no silver bullet here. We had found many issues, and made a multitude of fixes. Those fixes did improve the memory growth to be at a much smaller rate. We prematurely got excited: we fixed the issue!

The observant of you will notice all of this was done *locally*. There was no real volume to the service. While we fixed issues, they were only *part* of the overall problem. In order to find the other culprits, we needed to replicate the behavior of a real environment.

### Staging Debug

Because we could not replicate the exact pattern of volume in staging, we decided to enable the debugging in that environment. Our staging environment is inaccessible from the internet, so we felt safe in enabling remote debugging there. This opened a huge door for us, as we could observe the service in realtime.

With Chrome DevTools, we started grabbing heap snapshots and observing as usual. However a new problem arose; once the pod got over a certain threshold of memory (roughly ~300MB), connecting the debugger and grabbing a heap snapshot crashed the pod because of their overall size. This limited our timeframe to pull valuable snapshots.

### Further Fixes

#### Open Telemetry

Now that we had more realistic snapshots, our investigation led us to a large <a href="https://developer.chrome.com/docs/devtools/memory-problems/get-started#objects_retaining_tree" target="_blank">retainer chain</a>:

```
GC Root
└─ HTTPParser
   └─ resource_symbol
      └─ MockHttpSocket.requestParser
         (chunk-N4ZZFE24.js:283)
         └─ bound_this (native_bind)
            └─ MockHttpSocket._httpMessage
               └─ ClientRequest._events
                  (node:_http_client:190)
                  └─ error listener
                     └─ Array[1]
                        └─ contextWrapper()
                           (AbstractAsyncHooksContextManager.js:45)
                           └─ Context
                              (http-transport-utils.js:81)
                              └─ onDone Context
                                 └─ Context
                                    (http-exporter-transport.js:30)
                                    └─ Context.data
                                       └─ Uint8Array
                                          └─ ArrayBuffer
```

Examining the above, the http-exporter-transport and AbstractAsyncHooksContextManager point to OpenTelemetry, which we use for observability. Looking at the delta between multiple heap snapshots, we noticed that the number of spans and data related to them kept growing without being freed, even though the volume of the service was not changing.

This seemed like a similar problem to superagent, where something in the HTTP request layer is causing retention unexpectedly. To fix this, we switched the protocol from http/json to gRPC. That was a trivial change, thanks to an internal team running the OpenTelemetry collector with both HTTP and gRPC support in our Kubernetes cluster. After that protocol change, we saw the memory behave much better, and the behavior of allocations/frees was more balanced between snapshots.

#### Semi Space Size

Although we continued to see the memory fluctuate up and down more (indicating healthier garbage collection), it was still growing and expanded the out of memory lifespan to about 26 hours. We continued to investigate and stumbled upon <a href="https://deezer.io/node-js-20-upgrade-a-journey-through-unexpected-heap-issues-with-kubernetes-27ae3d325646" target="_blank">this great blog post</a>. The blog outlined a similar experience as us, and outlined a key change they made to replicate Node 18's behavior:

```bash
node --max-semi-space-size=16 server.js
```

This sets the semi space size of your process to 16MiB, which was similar to the old defaults in Node 18 before the V8 changes. You can learn more <a href="https://github.com/nodejs/node/blob/main/doc/api/cli.md#--max-semi-space-sizesize-in-mib" target="_blank">here</a> and the associated GitHub <a href="https://github.com/nodejs/node/issues/55487" target="_blank">issue</a>.

After all of the high level fixes we implemented, this is the after picture of the pod's memory:

<figure style="margin: 2em 0;">
<img src="/images/2026-04-08-why-our-node-22-upgrade-kept-killing-our-pods/memory-after-fixes.png" alt="Grafana dashboard showing container memory fluctuating healthily within bounds after all fixes were applied" title="Memory behavior after fixes" />
<figcaption>Healthy memory behavior after applying all fixes, with regular fluctuations indicating proper garbage collection</figcaption>
</figure>

We have continued to monitor and the memory fluctuates regularly which is expected behavior. There are still some increases that need to be investigated, with our current theory being that there is further tuning of the Node options to better optimize the garbage collection to keep the memory in a good state.

## Lessons

A lot of lessons were learned throughout this experience, and it is hard to encompass them all. I will attempt to break them down into building blocks to outline the key components.

### Domain Knowledge

Node is an extremely powerful runtime that can handle very high volume with little code. Understanding the fundamentals helps to inform everything else. This is not just about the event loop. It extends to how a Node process lives and dies. Traditional metrics can lead you astray compared to other languages and runtimes. Having the knowledge of how the Node heap is calculated, partitioned, and managed gives valuable insight into *how* you write your code.

Layer on top of this the dependencies you use to build your application, and you get many moving pieces that increase the complexity in understanding what is happening at runtime. While following the <a href="https://en.wikipedia.org/wiki/Don%27t_repeat_yourself" target="_blank">DRY</a> principle is worthwhile in some cases, too many dependencies is another. Superagent, for example, has nice features, but it also introduced issues for us that were hard to follow. Keep your dependencies small to reduce complexity, and to better understand what's happening under the hood.

Please note, this is not to say any of these dependencies have memory leaks or to accuse them of our problems. Open source is a fundamental resource to how we build software, and stems from volunteer work and passion.

### Patterns

Once the domain knowledge is in place, it helps to inform the patterns you use to build and scale your software. Invest time in building team best practices that are easy to follow, and are memory safe. In our case, this would have been using singletons correctly (plus understanding the libraries we use better) and implementing caches that are managed effectively in production.

Some of this should come naturally in the pull request review process. But the team cannot catch everything. Part of our learnings on this topic is to do a better job of sharing knowledge and using tools to help review for bad practices.

### Process

Once all the code is written, it has been reviewed, and is shipped, is that it? Our typical process said so. We of course QA'd work and validated functionality worked as expected. As you read this post however, you may have noticed breakdowns in process as well. For example, why was the application not load tested before shipping to production? That is a valid criticism and would have exposed these issues much earlier.

We need to evolve our process as a team. Our scale is ever increasing, and these issues will only become more prevalent. This includes reflection on this particular experience, as well as continuing to make improvements to make our lives easier when problems arise.

### Monitoring

Another question that might have been raised is how did we not know about the pod restarts? Should we not have had some alerting setup? These are valid questions, and monitoring lessons we will take going forward. We should have metrics for the common Node performance indicators like heap statistics, event loop lag, and garbage collection performance. Even if you do not explicitly alert on restarts or these metrics, you get instant insight into your service in real time.

## Takeaways

Littered throughout this post are a variety of takeaways. The most important of which are:

- **Avoid a moving target.** Active feature development while debugging forces careful production deployment coordination. Freeze changes where possible while investigating.
- **Monitor and alert on key Node performance metrics.** Always set resource limits, and have dashboards for heap usage, event loop lag, and garbage collection performance.
- **Understand Node's memory model.** Knowing how the heap is calculated, partitioned, and tuned gives you a head start when things go wrong.
- **Strengthen your development lifecycle.** Load test before major changes, introduce tooling and review standards to catch potential pitfalls, and limit dependencies where possible.
- **Follow Node's best practices.** Leverage singletons properly, clean up object and closure references when finished with them, and schedule in-memory cache eviction with `setInterval()`.

## Thank You

Now that we have reached the end, I want to thank you for reading so far, and letting me share our story with you. I hope it has provided valuable insights, or at the very least to learn from our mistakes.