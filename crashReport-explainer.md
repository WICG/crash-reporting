# Crash Report Storage API Explainer

## Participate

You can participate in the evolution of this proposal by filing issues against the Crash Reporting
API specification, at https://github.com/WICG/crash-reporting/issues/new.

## Introduction

The [Crash Reporting API](https://github.com/wicg/crash-reporting) defines a non-web-exposed
mechanism for the user agent to send JSON reports to a developer endpoint, when the OS process
hosting a Document crashes. These JSON diagnostic reports include some context about the page
(see https://github.com/WICG/crash-reporting/pull/23), and are sent to the `default` reporting
endpoint specified by the developer[^1]. See
[*the Reporting API > Enabling reporting*](https://github.com/w3c/reporting/blob/main/EXPLAINER.md#enabling-reporting)
for more.

But these reports are missing important application context that web app developers are in a
position to supply, which could greatly help narrow down the cause of crashes.

## Motivation

Web applications often encounter unexpected crashes in the process hosting their web content,
and currently there is no easy, reliable way for web developers to access diagnostic information
that their application might have collected right before a crash, making post-crash analysis
extremely fraught.

The best that developers can do today is constantly send relevant application state back to their
server—possibly through a mechanism like Web Sockets trying to represent a live session—and estimate
when sessions were terminated abruptly due to an OS process crash, and analyze any recorded
application state *in only those cases*.

The *web-exposed* Crash Report API aims to fill this gap by allowing applications to record relevant
data throughout the lifetime of a user's session, and is sent to developers only after a crash is
encountered.

### Proposal

The Crash Report API (currently proposed as `window.crashReport`) is an extension of the Crash
Reporting API, offering developers a web-exposed key-value store to record arbitrary application
state that gets attached to the
[`CrashReportBody`](https://wicg.github.io/crash-reporting/#crashreportbody)
that gets sent to the developer endpoint. As a web application engages with the web platform in
different ways throughout the user's session, this API lets developers track what actions or state
in their app might be causing a crash.

By providing a dedicated interface whose backing store gets attached to crash reports, developers
can capture essential information leading up to a crash. This significantly increases the utility of
the server-destined crash reports, giving developers the opportunity to debug each crash they
encounter with more precision.

### Detailed design

We propose a straightforward `CrashReportStorage` Web IDL interface, with a key/value setter, and
removal method:

```js
[Exposed=Window]
interface CrashReportStorage {
  void set(DOMString key, DOMString value);
  void remove(DOMString key);
};
```

Both `set()` and `remove()` are synchronous, and expected to be implemented either by a backing blob
of shared memory that spans between the crashing process and the process that reports on its crash,
a synchronous IPC call, or some other equivalently reliable mechanism.

> [!NOTE]
> Note that while synchronous storage APIs are discouraged on the web platform, this API is not a
> storage API. The backing store's scope is restricted to the current `Document`, there is no
> getter, and for this API to be maximally useful and ergonomic, it must be a suitable one-line
> drop-in, in a potentially-crashy synchronous block of code. If the API were Promise-based and
> therefore asynchronous, it becomes invasive to the surrounding code that it helps debug, by
> introducing asynchronicity that may alter the application's ability to reproduce the suspected
> crash.

**Scoping**

The scope of key-value map backing the `CrashReportStorage` interface is Document-bound. Because it
is 1:1 with a Document, storage partitioning considerations that are relevant for other traditional
"storage" APIs are not necessary—this API is just additional Document state.

**Refresh persistence**

One difference between `crashReport` and `sessionStorage` is that while `sessionStorage` data
persists across page refreshes in a traversable navigable, `crashReport` data does not need this
level of persistence, and may in fact benefit from being more ephemeral than `sessionStorage`.
However the exact policy we land on is TBD.

**Which crash reports get access to `crashReport` data?**

[Issue #24](https://github.com/WICG/crash-reporting/issues/24) poses an open question relating to
the scope of `crashReport` data, and asks: which Documents actually send crash reports, when a
process hosting multiple same-origin Documents crashes? Because it is not always possible to
determine which Document in a process caused a given crash, the running idea is that the Crash
Reporting API should specify that the topmost Document for a given origin should generate a
`CrashReportBody` with context from *that* document.

It is important to consider how this interplays with the scoping of `crashReport` data. Many web
applications are structured in a way where the top-level Document is merely a thin host for a suite
of same-origin iframes that primarily drive the application. In these applications, it is more
likely that a same-origin iframe caused a crash than the top-level Document, and if a crash report
gets generated **only** for the top-level Document, it is crucial that it includes any data put in
the `crashReport` API by iframes in the same-origin, as to not silently ignore any important
developer-provided context.

## Usage

From a JavaScript developer's perspective, the `crashReport` API looks and feels just like
`sessionStorage` or `localStorage` (but with any aforementioned considerations above). Below is an
example of how a developer might use the `crashReport` API` to debug a complex operation that they
suspect is leading to crashes.

```js
window.crashReport.set('complex-operation-input', String(arg1 + arg2));
// If the following operation crashes, then its inputs will be sent in a `CrashReportBody` to the
default endpoint.
complexOperationThatMightCrash(arg1, arg2);
window.crashReport.remove('complex-operation-input');
```

Note that because crash storage data is accessible among all same-origin Documents under a
traversable navigable, the developer might take care to prefix keys for certain common operations
that multiple Documents may perform at the same time. For example, imagine the developer suspects
that a common `fetch()` path is crashing under certain conditions, but many Documents in a page
invoke that path at different times.

To record which specific `fetch()` is happening at a given time to help narrow down the culprit, a
developer might adopt a prefixing strategy to prevent clobbering the same state in the
`crashReport` API:

```js
// Code that runs in multiple Documents.
function fetchURL(url) {
  const prefix = `[top-level=${self === window.top}]`;
  window.crashReport.set(`${prefix}-fetching`, url);
  const response = await fetch(url);
}
```

## Alternatives considered

Initially we proposed this API as a new storage API, inheriting from the
[`Storage` interface](https://html.spec.whatwg.org/multipage/webstorage.html#the-storage-interface),
as this is the interface of choice for arbitrary key-value storage on the web platform.

Under this proposal, the scoping and lifetime of data in the Crash Storage API is the same as
session storage, in that it is
[scoped to the traversable navigable](https://storage.spec.whatwg.org/#traversable-navigable-storage-shed),
as opposed to the user agent's storage shed, like `localStorage`. This version of the proposal
entails creating a new
[registered storage endpoint](https://storage.spec.whatwg.org/#registered-storage-endpoints).

After consulting with storage experts, it didn't make sense to treat this API as a traditional
"storage" API, since it didn't have the same scoping and quota requirements, and isn't intended to
have a "getter" to retrieve values—it's a simple one-way dumping ground for diagnostic data that the
browser internals care about.

## User needs

The crashes that our proposal helps developers debug are not caused by faulty web
applications—rather, faulty web browser implementations—but web app developers can respond to
crashes and bugs in the platform much faster than browsers can, given complex release cycles.

Therefore, the Crash Report API lets developers greatly improve user experience by reducing user
exposure to common crashing scenarios before they can be fixed by browser engineers independently.
This increases the overall stability of the web platform, leading to less loss of user data and poor
experience.

## Security and privacy concerns

One possible vector for concern about this API is that it gives malicious websites the ability to
refine exploits and make them more reliable in the wild. Example: assume an attacker owns both
Evil.com and EvilIframe.com, and tricks a user into visiting Evil.com. Evil.com can host
EvilIframe.com in an iframe, which can try out various exploits that often crash the hosting
renderer process when they fail.

In browsers with site isolation, Evil.com can "retry" the exploit in EvilIframe.com over and over
again. And if EvilIframe.com uses the `window.crashReport` API to record details about the
parameters of its exploit attempts, it becomes easier for EvilIframe.com to quickly adjust its
parameters on the fly and create a more robust, reliable exploit with the help of this API. Without
this API, Evil.com might not have insight into what specifically caused the iframe crash, and
therefore cannot direct future attempts at the same exploit.

While the `window.crashReport` API might help facilitate this scenario, it's likely that Evil.com
and EvilIframe.com can already collude in an equivalent way to get the same results. Specific
parameters of EvilIframe's exploit attempts can already be communicated asynchronously back to its
own servers via WebSockets, or to Evil.com itself via `postMessage()`, and the event of a crash can
also be observed indirectly, with some effort, through these tools.

Therefore, this API doesn't expose any information that isn't already available without some effort.
The security concern is that the same ergonomic benefits that allow legitimate developers to debug
their own crashes may also extend to attackers, and their ability to create more robust exploits.

----

The following are the answers to the W3C TAG's
[security and privacy self-review questionnaire](https://w3c.github.io/security-questionnaire/).

> 2.1. What information does this feature expose, and for what purposes?

Only developer-supplied information collected throughout an origin's session, and this information
is only exposed *if* the OS process hosting the origin's Document crashes.

> 2.2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

We believe so, yes.

> 2.3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

The feature does not expose any PII. However it is possible for developers to inject PII that they
collect from their app, into the `crashReport` API, and thus into crash report bodies.

> 2.4. How do the features in your specification deal with sensitive information?

All data inserted into the `crashReport` API is treated the same, and the data can only come from
what the web app developer already has access to.

> 2.5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

Only what developers choose to collect and expose to themselves in crash reports.

> 2.6. Do the features in your specification introduce state that persists across browsing sessions?

No; the lifetime of data matches at most `sessionStorage`, which is generally ephemeral.

No; as stated earlier in the explainer, the lifetime of the data persists no further than
`sessionStorage`.

> 2.7. Do the features in your specification expose information about the underlying platform to origins?

No.

> 2.8. Does this specification allow an origin to send data to the underlying platform?

No.

> 2.9. Do features in this specification enable access to device sensors?

No.

> 2.10. Do features in this specification enable new script execution/loading mechanisms?

No.

> 2.11. Do features in this specification allow an origin to access other devices?

No.

> 2.12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

> 2.13. What temporary identifiers do the features in this specification create or expose to the web?

This proposal does not itself create any unique identifiers, nor expose any to the web. Developers
that use the API, however, are able to create and store their own should they choose, just as they
can with other storage or even networking APIs.

> 2.14. How does this specification distinguish between behavior in first-party and third-party contexts?

Like `sessionStorage`, storage in the `crashReport` API is scoped to an origin under a traversable
navigable. Therefore, while there is no distinction in how the API *behaves* in first-party or
third-party contexts, data is isolated along this boundary, per the security model of the web
platform.

> 2.15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

No expected difference.

> 2.16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

TODO: Not yet.

> 2.17. Do features in your specification enable origins to downgrade default security protections?

No.

> 2.18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

TODO.

> 2.19. What happens when a document that uses your feature gets disconnected?

TODO.

> 2.20. Does your spec define when and how new kinds of errors should be raised?

TODO.

> 2.21. Does your feature allow sites to learn about the user’s use of assistive technology?

No.

[^1]: This may be changing soon; see
https://github.com/WICG/crash-reporting/issues/24.
