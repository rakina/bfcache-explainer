# BFCache Explainer
Authors: [Rakina Zata Amni](https://github.com/rakina), [Domenic Denicola](https://github.com/domenic), [Fergal Daly](https://github.com/fergald) - Google

## What is BFCache?
BFCache (Back/Forward Cache) is a user agent feature that allows user agents to keep a document alive in the back-forward cache after the user navigates away from it. Later on, if the user navigates back to that document, the user agent can use the previously cached document instead of re-loading a new document from scratch.


### Behavior of cached document

For the most part, the cached document is kept "frozen” at the state it was in when the user navigated away from it. This is done by not running script and task queues on the document until the user navigates back to the document again.

Also, while the document is cached, the "outside world” (other documents, service workers, etc) should not notice that the cached document is alive. The cached document should be as "invisible” as possible: it should not be noticed by, interact with, or get any updates from the outside world. The last part is especially important since as far as the user knows, the document is already "gone”, and should not be receiving potentially sensitive information such as geolocation updates while in the bfcache, even though the user might navigate back to it later on.


### Eviction

Even if a document was initially kept alive after navigation, it might get evicted from the back-forward cache and discarded. This can happen due to various reasons (e.g. memory pressure, usage of an API that tries to interact with a bfcached document, the document has been in bfcache for too long, etc). The exact policy depends on the user agent’s implementation, but there might be room for standardization here for APIs that have complicated interactions with bfcache, such as APIs that establish connections between the bfcached documents and the "outside world” (in fact, WebSockets is already [specified](https://html.spec.whatwg.org/#unloading-documents:concept-document-salvageable-7) to do eviction)


### Eligibility

Similar to eviction policy, the exact way the user agent determines if a document should or should not be kept alive in the back-forward cache differs across implementations.


## Motivation

Allowing the user agent to cache a document and serve it for future navigations gives a better and faster browsing experience for the user. This means it is in web authors’ best interest to make their web pages work well with the back-forward cache, to allow more of their page loads to be served from the back-forward cache. This means it is in the spec authors’ best interest to make their API work well with the back-forward cache.

By providing good bfcache-related guidelines for new API authors to consider and improving existing specifications’ support for back-forward cache, we can make the platform’s support for back-forward cache better.


## Current problems



*   **User agent implementation varies quite a lot especially in eligibility/ineligibility/eviction policy, as they’re not specified**
    *   This makes it hard for web authors to predict whether their pages can be bfcached. While it's acceptable for some behaviour to be unspecifieded, currently, for any given decision, it's unclear if it's unspecified because implementations are free to choose or because it has just never been considered. This makes it hard to write WPTs.
    *   Example: Previously some user agents support pages with "unload” handlers but others didn’t. It was [recently specified](https://github.com/whatwg/html/pull/5889).
*   **Many existing APIs were not written with BFCache in mind, in particular that:**
    *   Documents are not always discarded after navigation, and may retain state after navigations.
    *   A document might transition into an "inactive” state but not be discarded, and should essentially be treated as not there (not show up in various APIs, not receive updates including deferred ones)
*   **It’s hard to retrofit APIs to work well with bfcache, especially when considering compatibility**
    *   We want to avoid this problem for new APIs by incorporating BFCache into the API design guidelines and review pipeline, so that new APIs support bfcache by design (or at least explicitly specify their interaction with bfcache if it's non-trivial)
    *   For existing APIs, support is added on a case-per-case basis, to ensure it won’t break existing usage. Ideally we would want owners of the individual APIs to take the lead here.


## Proposed solutions


### Add BFCache consideration to Design Principles & TAG Review

We’re proposing additions to [Web Platform Design Principles](https://w3ctag.github.io/design-principles/) and [Self-Review Questionnaire: Security and Privacy: Questions to Consider](https://www.w3.org/TR/security-privacy-questionnaire/), so that API spec authors will write their specifications with BFCache in mind. If possible, we want these points to also be integrated into TAG Reviews of new APIs.

Main goal: **new APIs should either have support for bfcache, or explicitly disallow bfcache**. Otherwise, we’ll continue the trend of "might or might not work depending on the user agent’s policy”.

What to add to [Web Platform Design Principles](https://w3ctag.github.io/design-principles/):



*   **Add support for back-forward cache/non-”fully active” documents by following these patterns:**
    *   **Gate actions with "fully active” checks**
        *   When performing actions that might update the state of a document, be aware that the document might not be "fully active” and is considered as "non-existent” from the user’s perspective. This means they should not receive updates or perform actions.
        *   In many cases, anything that happens while the document is not "fully active” should be treated as if it never happened, and not "queue” the updates and send them after the document becomes fully activated.
        *   If it makes sense to send the latest state update (and only the latest state, not a queue of state changes) to the document after it becomes "fully active” again, consider the "listen for changes” pattern below.
        *   **Examples:**
            *   APIs that periodically do actions e.g. send information updates, like Geolocation API’s "[watch position](https://w3c.github.io/geolocation-api/#watchposition-method)” should not send updates if the document is no longer fully active. It also should not queue those updates to arrive later, and only resume sending updates when the document becomes active with the latest location.
    *   **Listen for changes to "fully active” status**
        *   When a document goes from "fully active” to non-”fully active”, it should be treated similarly to the way discarded documents are treated.
        *   The document must not retain exclusive access to shared resources and must ensure that no new requests are issued and that connections that allow for new incoming requests are terminated (connections may stay open to complete existing ongoing requests, and later update the document with the result when it gets restored).  When a document goes from non-”fully active” to "fully active” again, it can restore connections if appropriate.
        *   While web authors can manually release the resources/sever connections from within the "pagehide” event and restore them from the "pageshow” event themselves, doing this automatically from the API design would reduce the bfcache-support overhead
        *   **Examples:**
            *   APIs that hold non-exclusive resources may be able release the resource when the document becomes not fully active, and re-acquire them when it becomes "fully active” again (Screen Wake Lock API is already [doing](https://w3c.github.io/screen-wake-lock/#handling-document-loss-of-full-activity) the first part).
                *   Note that this might not be appropriate for all types of resources, e.g. if an exclusive lock is held, we cannot just release it and reacquire when 'fully active" since another page could then take that lock. If there is an API to signal to the page that this has happened, it may be acceptable but beware that if the only time this happens is with BFCache, then it's likely many pages are not prepared for it. If it is not possible to support BFCache, follow the "Discard non-’fully active’ documents for situations that can’t be supported” pattern described below.
            *   APIs that create live connections can pause/close the connection and possibly resume/reopen it later.
        *   Additionally, when a document becomes "fully active” again, it can be useful to update it with the current state of the world, if anything has changed while it is in the non-”fully active” state.
        *   Care needs to be taken with events that occurred while in the BFCache. When not ”fully active”, for some cases, all events should be dropped, in others the latest state should be delivered in a single event, in others it may be appropriate to queue events or deliver a combined event. The correct approach is case by case and should consider privacy, correctnes, performance and ergonomics. Examples:
            *   The [gamepadconnected](https://w3c.github.io/gamepad/#event-gamepadconnected) event can be sent to a document that becomes "fully active” again if a gamepad is connected while the document is not "fully active”. If the gamepad was repeatedly connected and disconnected, only the final connected event should be delivered.
            *   For geolocation or other physical sensors no information about what happened while not-”fully active” should be delivered, the events should simply resume from when the document became ”fully active”
            *   For network connections or streams, the data received while not-”fully active” should be delivered when ”fully active” but whereas a stream might have created many events with a small amount of data each, it could be delivered as smaller number of events with more data in each
    *   **Omit non-”fully active” documents from APIs that span multiple documents**
        *   Non-”fully active” documents should not be observable and so APIs should treat them as if they no longer exist. They should not be visible to the "outside world” through document-spanning APIs (e.g. [clients.matchAll()](https://w3c.github.io/ServiceWorker/#clients-matchall), window.opener)
        *   Note: This should be rare since cross-document-spanning APIs are themselves relatively rare.
        *   Examples:
            *   [BroadcastChannel](https://html.spec.whatwg.org/multipage/web-messaging.html#broadcasting-to-other-browsing-contexts:fully-active), which checks for "fully active”.
            *   clients.matchAll()'s [spec](https://w3c.github.io/ServiceWorker/#clients-matchall) currently does not distinguish between "fully active” and non-"fully active” clients but correct implementations should only return "fully active” clients
    *   **Discard non-”fully active” documents for situations that can’t be supported**
        *   If supporting non-"Document/fully active" documents is not possible for certain cases, explicitly specify it by discarding the document if the situation happens after the user navigated away, or setting the document's [salvageable](https://html.spec.whatwg.org/multipage/browsing-the-web.html#concept-document-salvageable) bit to false if the situation happens before or during the navigation away from the document, to cause it to be automatically discarded after navigation.
        *   Note: this should be rare and probably should only be used when retrofitting old APIs, as new APIs should always strive to work well with back-forward cache.
        *   Example:
            *   WebSockets [sets the salvageable bit to false](https://html.spec.whatwg.org/#unloading-documents:concept-document-salvageable-7) during unload.
            *   [clients.claim()](https://w3c.github.io/ServiceWorker/#clients-claim) should not wait for non-”fully active” clients, instead it should cause the non-”fully active” client documents to be discarded
*   **Be aware that per-document state/data might persist after navigation**
    *   Example: [sticky user activation](https://html.spec.whatwg.org/multipage/interaction.html#sticky-activation) is [currently specified to stay "forever”](https://github.com/whatwg/html/issues/6588), as it persists across navigations as long as we keep using the same document.
    *   APIs that do not want to keep their state/data after navigation & restore should proactively watch for navigations/lifecycle changes (see the "patterns list” above).

What to add to [Self-Review Questionnaire: Security and Privacy: Questions to Consider](https://www.w3.org/TR/security-privacy-questionnaire/):



*   **How does your feature interact with non-”fully active” documents?**
    *   A document might stay around in a non-”fully active” state even after the user navigated away from it, to possibly be reused when the user navigates back to the entry holding it (see [BFCache explainer](https://github.com/rakina/bfcache-explainer/)).
    *   From the user’s perspective, the non-”fully active” document is already discarded and thus should not get state updates, especially privacy-sensitive updates (e.g. geolocation information).
    *   When persisting things in the document/creating things tied to a document’s lifetime, be aware that the document might stay around and be reused after navigations.
    *   For more details, see guidelines in [Web Platform Design Principles](https://w3ctag.github.io/design-principles/)


### Make it easy for current/future specifications to add support for BFCache

Since the current specification is quite subtle, it’s easy for API spec authors to miss adding support for back-forward cache. Some support we can add:



*   Add hooks to watch for lifecycle changes, so that APIs can prepare for transition from "fully active” to non-”fully active” and vice versa
*   Add hooks to evict/make document ineligible for back-forward cache, etc. i.e. a way to express that the document would be unsalvageable and should not be reused.

For more details, see ["spec details”](#spec-details) section 


## Open questions



*   **What should we be careful/aware about when retrofitting existing APIs to support back/forward cache?**
    *   We already provide general guidelines in our proposals above, however, it might be tricky for existing APIs to add back-forward cache support without changing some behavior that might break some use cases (e.g. tearing down connections/releasing resources might be unexpected). How should we draw the line between breaking compatibility to achieve support vs extending the API to allow support vs giving up and specifying ineligibility?
        *   From a browser vendor perspective, we would like to add back-forward cache support to as many existing APIs as possible.
    *   Are there existing APIs that must be updated to support back-forward cache? How should we ensure they get updated?
        *   Privacy-sensitive APIs must be updated to ensure private information won’t leak to non-”fully active” documents.
        *   Document-spanning APIs must be updated to not include non-”fully active” documents
*   **When should we specify bfcache ineligibility vs not? Which cases should be put under "user agent discretion” and which cases should be specified?**
    *   Example: WebSockets is already [specifying bfcache-ineligibility](https://html.spec.whatwg.org/#unloading-documents:concept-document-salvageable-7)


## Spec details

Back-forward cache is specified through these concepts:



*   [Salvageable](https://html.spec.whatwg.org/multipage/browsing-the-web.html#concept-document-salvageable)
*   [Fully active](https://html.spec.whatwg.org/multipage/browsers.html#fully-active)
*   Browsing context’s [active window](https://html.spec.whatwg.org/multipage/browsers.html#active-window) & [active document](https://html.spec.whatwg.org/multipage/browsers.html#active-document)
*   [Windows](https://html.spec.whatwg.org/multipage/window-object.html#concept-document-window) & [associated documents](https://html.spec.whatwg.org/multipage/window-object.html#concept-document-window)
*   Session history entry’s [document](https://html.spec.whatwg.org/multipage/history.html#she-document) 
*   "[Frozenness](https://wicg.github.io/page-lifecycle/#document-frozenness)” (defined in the Page Lifecycle spec)

See [meta-bug](https://github.com/whatwg/html/issues/5880) for back-forward cache spec issues (not complete, certain issues mentioned in this explainer doesn’t have issues filed yet) 


### Putting a document into bfcache

When we navigate away from a document, we run the "[unloading a Document](https://html.spec.whatwg.org/multipage/browsing-the-web.html#unload-a-document)” steps. If it is eligible for bfcache (the "salvageable” bit is true), we won’t unload/discard the document. Instead, the old document stops being the active document of the browsing context and "fully active” becomes false. The session history entry "keeps” the [document](https://html.spec.whatwg.org/multipage/history.html#she-document) around. 


### Restoring a document from bfcache

When we navigate back to a document by [traversing session history](https://html.spec.whatwg.org/multipage/browsing-the-web.html#traverse-the-history), if the session history entry’s document is still around, we will use it instead of creating a new one.


#### Proposed changes



*   Check for the "salvageable” bit also when deciding to reuse the session history entry’s document


### Ensuring document in bfcache is "invisible” to the outside world

When a document is in bfcache, it should practically be "invisible”. "Invisible” here means that the "outside world” (e.g. other documents) should never notice that a non-"fully active” document is still alive, and should not be able to interact with non-”fully active” documents (e.g. send messages, wait for them to run).

Currently, this relies on the "[fully active](https://html.spec.whatwg.org/multipage/browsers.html#fully-active)” concept, for example:



*   Timers only count the amount of time that passes [while a document is fully active](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html#timers:fully-active).
*   Some [rendering steps](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model:fully-active) check if a document is fully active
*   [postMessage](https://html.spec.whatwg.org/multipage/web-messaging.html#broadcasting-to-other-browsing-contexts:fully-active) is only sent to fully active documents

However, some specifications do not use the "fully active” concept, and instead uses the "page visibility” concept, e.g. Geolocation API’s [watchPosition()](https://w3c.github.io/geolocation-api/#ref-for-dfn-request-position-3) will trigger a repeating call of "[request position](https://w3c.github.io/geolocation-api/#dfn-request-position)”, which is gated by step 4’s "Wait for document to become visible”.


#### Proposed changes


*   Maybe update all specifications that gate things behind the "visible” check to also check for "fully active”? (For example, the Screen Wake Lock API listens for loss of both [visibility](https://w3c.github.io/screen-wake-lock/#handling-document-loss-of-visibility) and [full activity](https://w3c.github.io/screen-wake-lock/#handling-document-loss-of-full-activity))


### Preserving state of the document when bfcached & after being restored

The general idea of bfcache is to keep a document & restore it as it was before, so it shouldn’t do anything while not "fully active”. The spec says we should not [run script](https://html.spec.whatwg.org/multipage/webappapis.html#calling-scripts:fully-active) nor [tasks](https://html.spec.whatwg.org/multipage/webappapis.html#concept-task-runnable) for non-”fully active” documents”

We do update the "fully active” and "visibility” states of the document when it gets into/restored from bfcache, so APIs that listen to these changes can change some states of the document (e.g. [release its Wake Lock](https://w3c.github.io/screen-wake-lock/#handling-document-loss-of-full-activity)).

For the most part, though, we preserve the state/data associated with a document after restoring it from bfcache. This might be unexpected to API authors that expect some states to start from a default value after navigation, e.g. [user activation](https://html.spec.whatwg.org/multipage/interaction.html#last-activation-timestamp) and various [user-activation-gated APIs](https://html.spec.whatwg.org/multipage/interaction.html#user-activation-gated-apis) that depend on it.


#### Open questions



*   Is it OK to keep things like [sticky user activation](https://html.spec.whatwg.org/multipage/interaction.html#sticky-activation) around even after navigations, if we restore a previously user-activated document from bfcache? (see [spec issue](https://github.com/whatwg/html/issues/6588))


### Evicting document from bfcache

When a non-fully active document might disrupt things in the "outside world” (e.g. clients.claim() wants to wait for a document in bfcache), or the user agent decides that it wants to (e.g. the document has been in bfcache for too long), a document can be evicted from bfcache.

This is specified through the "[discard](https://html.spec.whatwg.org/multipage/window-object.html#discard-a-document)” steps. If a document is not fully active, it can be [discarded](https://html.spec.whatwg.org/multipage/history.html#the-session-history-of-browsing-contexts:discard-a-document). This can happen when [navigating/unloading the document](https://html.spec.whatwg.org/multipage/browsing-the-web.html#unloading-documents:discard-a-document) (if salvageable is false), when a [browsing context is discarded](https://html.spec.whatwg.org/multipage/window-object.html#garbage-collection-and-browsing-contexts:discard-a-document), and also [whenever the user agent decides to discard the document](https://html.spec.whatwg.org/#the-session-history-of-browsing-contexts:discard-a-document-2).

We’re expecting to add more callers of discard in various cases, e.g. calling clients.claim() should trigger eviction for service worker clients that are in the back-forward cache.


#### Proposed changes



*   Add a clear note that user agents might discard a non-”fully active” document at any time.
*   Add a "make a document not salvageable” step/algorithm that sets the "salvageable” bit to false and also allows the user agent to discard the document any time after.
*   Call the "make a document not salvageable” step from clients.claim(), etc.


## Related proposals

We are proposing new APIs related to BFCache, but these probably should be covered by separate TAG reviews.


### Opt-in Header

 https://github.com/nyaxt/bfcache-opt-in-header


### Opt-out API

We [proposed](https://github.com/whatwg/html/issues/5744) to add a way to explicitly opt-out of BFCache in this thread. However, the shape of the API is still under discussion.


### WPT APIs & Policy

We are currently [proposing](https://github.com/web-platform-tests/wpt/issues/16359#issuecomment-795004780) policies & APIs for running tests with back-forward cache.
