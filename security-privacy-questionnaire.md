# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

> 01.  What information does this feature expose,
>      and for what purposes?

This feature may send a new network request, which can be visible from server.
This request is sent in order to speedup the resource load, that is the purpose of this feature.

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to implement the intended functionality?

We believe so. Technically it's possible that the server can observe the network request which is dispatched by this feature. However, it can be minimized by having a criteria and apply the feature to the only sites which meet the criteria.

> 03.  Do the features in your specification expose personal information,
>      personally-identifiable information (PII), or information derived from
>      either?

The proposal does not change how the browser handles PII.

> 04.  How do the features in your specification deal with sensitive information?

N/A

> 05.  Do the features in your specification introduce state
>      that persists across browsing sessions?

Yes. The proposal is an internal optimization to ask a user agent to remember make preload requests to main resources per ServiceWorkers.

> 06.  Do the features in your specification expose information about the
>      underlying platform to origins?

No.

> 07.  Does this specification allow an origin to send data to the underlying
>      platform?

No.

> 08.  Do features in this specification enable access to device sensors?

No.

> 09.  Do features in this specification enable new script execution/loading
>      mechanisms?

No.

> 10.  Do features in this specification allow an origin to access other devices?

No.

> 11.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No.

> 12.  What temporary identifiers do the features in this specification create or
>      expose to the web?

None.

> 13.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

This follows how a ServiceWorker fetch handler behaves.

> 14.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

It's the same in any context when there are registered service worker. However, if storage partitioning mechanism is used, this feature won't be used so much as the ServiceWorker is also under the storage partitioning.

> 15.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

No. However, the original Service Worker specification considers it in [6. Security Considerations](https://www.w3.org/TR/service-workers/#security-considerations). The proposal does not change any security requirements and privacy requirements explained there.

> 16.  Do features in your specification enable origins to downgrade default
>      security protections?

No.

> 17.  What happens when a document that uses your feature is kept alive in BFCache
>      (instead of getting destroyed) after navigation, and potentially gets reused
>      on future navigations back to the document?

The document will be reused.

> 18.  What happens when a document that uses your feature gets disconnected?

This feature is only available while handling resource loads. It could raise a network error once disconnected.

> 19.  What should this questionnaire have asked?

No.