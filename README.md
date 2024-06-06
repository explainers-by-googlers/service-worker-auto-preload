# Explainer for ServiceWorkerAutoPreload

This proposal is an early design sketch by the Chrome loading team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Chrome loading team

## Participate
- https://github.com/explainers-by-googlers/service-worker-auto-preload/issues
- Discussion Forum https://github.com/WICG/proposals/issues/155

## Table of Contents [if the explainer is longer than one printed page]

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [How it works](#how-it-works)
  - [Fetch handler fallback](#fetch-handler-fallback)
  - [Redirect](#redirect)
- [Rollout plan](#rollout-plan)
- [Opt-out](#opt-out)
- [Alternative considered](#alternative-considered)
  - [BestEffortServiceWorker](#besteffortserviceworker)
  - [Auto preload for subresources](#auto-preload-for-subresources)
- [FAQ](#faq)
  - [How is it different from the Navigation Preload API?](#how-is-it-different-from-the-navigation-preload-api)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

ServiceWorker is a popular API involved in about [15%](https://chromestatus.com/metrics/feature/timeline/popularity/990) of page loads. This API offers many advanced features such as fetch handling, push notification, background sync, etc. The ServiceWorker’s fetch handling capability allows developers to intercept and control the resource fetch requests via “fetch” event handler. This enables web apps to be reliable even while offline and delivers faster loading experiences by using its own CacheStorage. However, the cost of starting up a ServiceWorker can be non-negligible. ServiceWorker fetch handlers are invoked during the critical path for navigation and subresource loading. Developers are sometimes using fetch handlers to speed up page loading but the fetch handlers may slow down page loading if the ServiceWorker is not running, the cache access is slower than network (e.g., 5G network), or the resource was not cached. In the worst case, it can take up to hundreds of milliseconds to bootstrap the ServiceWorker itself and process fetch event handlers.

Fetch handlers are used for various reasons but there are many cases where developers use them with the intention of serving resources faster than the network. To fulfill their intention, this document proposes the introduction of “ServiceWorkerAutoPreload”, a new loading strategy for ServiceWorker.

ServiceWorkerAutoPreload is a mode where the browser issues a network request (i.e. a regular request which may result in a HTTP cache hit or an actual fetch) in parallel with the ServiceWorker bootstrap, and consumes the network request result inside the fetch handler.

This new behavior introduced by ServiceWorkerAutoPreload acts something like “auto [NavigationPreload](https://w3c.github.io/ServiceWorker/#navigationpreloadmanager)”. NavigationPreload starts a dedicated network request and the fetch handler at the same time, and resolves it as [event.preloadResponse](https://w3c.github.io/ServiceWorker/#dom-fetchevent-preloadresponse) inside the fetch handler. Similarly ServiceWorkerAutoPreload starts those two processes at the same time, but the network request is resolved with fetch(event.request) inside the fetch handler. Unlike NavigationPreload, ServiceWorkerAutoPreload doesn’t require any additional code changes.

ServiceWorkerAutoPreload also uses its response when the browser issues a fallback network request. Without ServiceWorkerAutoPreload, the browser issues a new network request after receiving the fallback result from the fetch handler.


## Goals

Offset the ServiceWorker bootstrap latency and fetch handler costs on websites without any behavior changes.

The only cost is the server side cost to respond to the network requests, which may not be used if the fetch handler returns a result from the disk cache. This cost can be mitigated by applying ServiceWorkerAutoPreload only for websites that meet the [eligibility criteria](#rollout-plan).

## How it works

ServiceWorkerAutoPreload issues the network request (we use the term “auto preload network request” in this explainer) and invokes the fetch handler which may involve the bootstrap process at the same time for GET main resource requests with [eligible service workers](#rollout-plan).

The network request is consumed in the fetch handler when it has fetch(event.request). At that time, the fetch handler doesn’t issue a new network request. Instead, the request is resolved with the response of the auto preload network request, which ServiceWorkerAutoPreload already triggered. This means the network request is triggered outside of the fetch handler, but the result is consumed by the fetch handler. Even when ServiceWorkerAutoPreload is enabled, the browser always respects the result from the fetch handler. It doesn’t use the response from the auto preload network request as it is without fetch handler interceptions. So developers can assume the fetch handler is always invoked, and the result of the fetch handler won’t be changed. The response is consistent whether ServiceWorkerAutoPreload is enabled or not.

When the fetch handler responds without calling fetch(e.request), the auto preload network request is simply discarded. When the request is modified with [Request.clone()]([url](https://developer.mozilla.org/ja/docs/Web/API/Request/clone)), or the new [request]([url](https://developer.mozilla.org/ja/docs/Web/API/Request)) is created inside the fetch handler, a new network request is dispatched and the auto preload network request will be discarded, too.

For example, the following fetch handler will benefit from ServiceWorkerAutoPreload.

```js
self.addEventListener('fetch', (event) => {
  // This fetch() doesn't invoke a new network request. Instead, resolved with the response from the auto preload network request.
  event.respondWith(fetch(event.request));
});
```

If the fetch handler brings a different response from fetch(event.request), or the fetched response is modified in the fetch handler, the auto preload network request is just discarded. 

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(new Response('Custom response'));
});
```

As an optimization, when the fetch handler has served content but the auto preload network request is still running, the browser can cancel the auto preload network request to save the network / server resources.

The auto preload network request will only be used to fulfill fetch()s equal to event.request. Other fetch()s (e.g., sending beacons for measurement) should just work normally.

```js
onfetch = (e) => {
  // Call fetch() to send a beacon request for the analysis purpose. This initiates a new network request as usual.
  fetch("/path-for-analyze");

  e.respondWith(
    (async () => {
      // This will not issue a second fetch. The response will come from the auto preload network request.
      const resp = await fetch(e.request);
      return resp;
    })()
  );
};
```

### Fetch handler fallback

It’s possible that the ServiceWorker fetch handler doesn’t respond by calling [respondWith()](https://www.w3.org/TR/service-workers/#dom-fetchevent-respondwith). In that case a fallback request is triggered from the browser after receiving the fetch handler result. Without ServiceWorkerAutoPreload, this behavior has a performance penalty because the browser has to wait for the fetch handler completion, or even the entire ServiceWorker bootstrap process if the ServiceWorker is not started yet. ServiceWorkerAutoPreload optimizes this behavior by dispatching the auto preload network request before starting the ServiceWorker and executing the fetch handler, and use the response as a fallback response if the fetch handler doesn’t respond.

### Redirect

Any requests/responses including redirects should be intercepted by the fetch handler. Handling redirects in fetch() is a complicated process, it depends on the [redirect mode](https://fetch.spec.whatwg.org/#concept-request-redirect-mode) that each request has, and the fetch handler may intercept the response with [opaque-redirect filtered response](https://fetch.spec.whatwg.org/#concept-filtered-response-opaque-redirect). When the browser receives a redirect response from the auto preload network request, the browser lets the fetch handler handle the remaining process so that it can intercept all requests and responses created by the fetch() standard process.

## Rollout plan

ServiceWorkerAutoPreload is specified as an optional optimization that the browser can apply at its choosing. Because while ServiceWorkerAutoPreload can provide performance improvements, it’s observable via the server as additional requests. However, it can be mostly not observable since the browser limits this optimization only to ServiceWorkers in which the fetch handler returns the response always consistent with the network request.

As an update to the [ServiceWorker spec](https://w3c.github.io/ServiceWorker), we propose adding the new step in [Handle Fetch](https://w3c.github.io/ServiceWorker/#handle-fetch), which creates a new [request](https://fetch.spec.whatwg.org/#concept-request) and [fetch](https://fetch.spec.whatwg.org/#concept-fetch), and puts the response to the map with the [request](https://fetch.spec.whatwg.org/#concept-request) as a key (this is similar to what the [Static Routing API](https://github.com/WICG/service-worker-static-routing-api) does with the [race response map](https://w3c.github.io/ServiceWorker/#serviceworkerglobalscope-race-response-map)).

We (Google Chrome team) plan to enable this optimization automatically when sites meet an eligibility criteria. The eligibility criteria is that higher rates of fetch handler results are fallback. We may also consider including the pass-through case. The criteria may be revised in the future to behave more smartly. For the experiment purpose before full launch, we plan to launch this feature to all sites under the limited traffic.

## Opt-out

Opting out of the experiment can be done via the [Static Routing API](https://github.com/WICG/service-worker-static-routing-api). By registering the router info that matches all requests, and asking them to go to the fetch handler, it works as the opt-out signal from ServiceWorkerAutoPreload.

```js
self.addEventListener('install', e => {
  e.addRoutes({
    condition: {
      urlPattern: new URLPattern({})
    },
    source: "fetch-event"
  });
});
```

## Alternative considered

### BestEffortServiceWorker

[BestEffortServiceWorker](https://docs.google.com/document/d/1h2BSBJnU5vD-xbj1O-wniMtXMyBrQSE29-Uk_BBLxL4/edit#heading=h.7nki9mck5t64) is the idea we had explored before to improve the performance at scale. BestEffortServiceWorker was a similar optimization in terms of dispatching a network request and invoke the fetch handler at the same time, but BestEffortServiceWorker was designed to have the race between the network request and the fetch handler, and use the response whichever is faster. However, we realized this was challenging to apply because it may bring inconsistent results depending on the race result. That could happen when the fetch handler modifies requests or responses, BestEffortServiceWorker just runs the race between the network request and the fetch handler and uses the result that comes faster, it doesn’t guarantee that those responses are the same. This could bring some inconsistent behaviors to the ecosystem.

Unlike BestEffortServiceWorker, ServiceWorkerAutoPreload guarantees the response consistency by respecting the fetch handler result. But still it’s beneficial by dispatching a network request earlier.

### Auto preload for subresources

Not only for the main resource, the ServiceWorkerAutoPreload mechanism can be applied to subresource network requests as well, and potentially that may bring more performance benefits. However, we primarily focus on the main resource preloading, because 1) when handling subresources, ServiceWorkers are already running in most cases. Assuming the fetch handler interception cost is already minimal, 2) issuing auto preload network requests for subresources could increase the memory usage, and that may cause OOM.

## FAQ

### How is it different from the Navigation Preload API?

The [navigation preload API](https://w3c.github.io/ServiceWorker/#navigationpreloadmanager) is the API that dispatches a preload request for main resource fetch in parallel with the ServiceWorker bootstrap. The preload request is resolved in the fetch handler, with the dedicated promise called event.preloadResponse. NavigationPreload is enabled by developers via [PreloadManager.enable()](https://w3c.github.io/ServiceWorker/#dom-navigationpreloadmanager-enable). Also, developers can distinguish preload requests from regular requests by using a [header](https://w3c.github.io/ServiceWorker/#service-worker-registration-navigation-preload-header-value) added to the preload request.

Both NavigationPreload and ServiceWorkerAutoPreload are trying to solve the same problem, which is minimizing the cost of ServiceWorker bootstrap. However, after years have passed since NavigationPreload was initially introduced, we still see many websites face this problem. As the usage of NavigationPreload is [lower](https://chromestatus.com/metrics/feature/timeline/popularity/1803) compared to the entire [sites having ServiceWorker](https://chromestatus.com/metrics/feature/timeline/popularity/990), we think the browser shouldn’t ask developers to do some work, instead, this problem should be solved by the browser itself. With ServiceWorkerAutoPreload, this can be solved without changing ServiceWorker scripts on the developer side. Because the auto preload network request is dispatched automatically, and it’s resolved in the fetch handler as the regular response of fetch().

As tradeoffs, first, the server may receive extra requests. Second, unlike NavigationPreload, developers can’t distinguish between the auto preload network request and the regular request. This is the limitation to guarantee the consistent result with the result when ServiceWorkerAutoPreload is not enabled. If that can be distinguished by the header or something in the request, that means it’s technically a different resource, and it’s possible to serve a different content from the regular request. And it may not result in the HTTP cache hit and bring more requests to the server.

ServiceWorkerAutoPreload won’t be enabled if the site has already enabled NavigationPreload. NavigationPreload should be the priority because developers are allowed to serve different content as a navigation preload response, or modify the response inside the fetch handler. For the consistency and minimizing the duplicated requests, ServiceWorkerAutoPreload does nothing in that case.

## Stakeholder Feedback / Opposition

We heard from developers and partners that they often use ServiceWorker to improve the page loading speed, but sometimes the startup process taking too long time.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- chikamune
- domenic
- horo-t
- xharaken
- yoshisatoyanagisawa
