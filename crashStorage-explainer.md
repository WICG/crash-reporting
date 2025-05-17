# Crash Storage API Explainer

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

The Crash Storage API aims to fill this gap by allowing applications to record relevant data
throughout the lifetime of a user's session, and is sent to developers only after a crash is
encountered.

### Proposal

The Crash Storage API (currently proposed as `window.crashStorage`) is an extension of the Crash
Reporting API, offering developers a web-exposed key-value store to record arbitrary application
state that gets attached to the
[`CrashReportBody`](https://wicg.github.io/crash-reporting/#crashreportbody)
that gets sent to the developer endpoint. As a web application engages with the web platform in
different ways throughout the user's session, this API lets developers track what actions or state
in their app might be causing a crash.

By providing a dedicated storage interface that gets attached to crash reports, developers can
capture essential information leading up to a crash. This significantly increases the utility of the
Crash Reporting API, giving developers the opportunity to debug each crash they encounter with more
precision.

### Detailed design

The interface of choice for arbitrary key-value storage on the web platform is the
[`Storage` interface](https://html.spec.whatwg.org/multipage/webstorage.html#the-storage-interface),
and the Crash Storage API reuses this generic frontend for the crash-specific storage backend.

**Scoping**

At a high level, the scoping and lifetime of data in the Crash Storage API is the same as session
storage, in that it is
[scoped to the traversable navigable](https://storage.spec.whatwg.org/#traversable-navigable-storage-shed),
as opposed to the user agent's storage shed, like `localStorage`. This proposal entails creating a new
[registered storage endpoint](https://storage.spec.whatwg.org/#registered-storage-endpoints).

**Refresh persistence**

One difference between `crashStorage` and `sessionStorage` is that while `sessionStorage` data
persists across page refreshes in a traversable navigable, `crashStorage` data does not need this
level of persistence, and may in fact benefit from being more ephemeral than `sessionStorage`.
However the exact policy we land on is TBD.

**Which crash reports get access to `crashStorage` data?**

[Issue #24](https://github.com/WICG/crash-reporting/issues/24) poses an open question relating to
the scope of `crashStorage` data, and asks: which Documents actually send crash reports, when a
process hosting multiple same-origin Documents crashes? Because it is not always possible to
determine which Document in a process caused a given crash, the running idea is that the Crash
Reporting API should specify that the topmost Document for a given origin should generate a
`CrashReportBody` with context from *that* document.

It is important to consider how this interplays with the scoping of `crashStorage` data. Many web
applications are structured in a way where the top-level Document is merely a thin host for a suite
of same-origin iframes that primarily drive the application. In these applications, it is more
likely that a same-origin iframe caused a crash than the top-level Document, and if a crash report
gets generated **only** for the top-level Document, it is crucial that it includes any data put in
the `crashStorage` API by iframes in the same-origin, as to not silently ignore any important
developer-provided context.

## Usage

From a JavaScript developer's perspective, the `crashStorage` API looks and feels just like
`sessionStorage` or `localStorage` (but with any aforementioned considerations above). Below is an
example of how a developer might use the `crashStorage` API` to debug a complex operation that they
suspect is leading to crashes.

```js
window.crashStorage.setItem('complex-operation-input', String(arg1 + arg2));
// If the following operation crashes, then its inputs will be sent in a `CrashReportBody` to the
default endpoint.
complexOperationThatMightCrash(arg1, arg2);
window.crashStorage.removeItem('complex-operation-input');
```

Note that because crash storage data is accessible among all same-origin Documents under a
traversable navigable, the developer might take care to prefix keys for certain common operations
that multiple Documents may perform at the same time. For example, imagine the developer suspects
that a common `fetch()` path is crashing under certain conditions, but many Documents in a page
invoke that path at different times.

To record which specific `fetch()` is happening at a given time to help narrow down the culprit, a
developer might adopt a prefixing strategy to prevent clobbering the same state in the
`crashStorage` API:

```js
// Code that runs in multiple Documents.
function fetchURL(url) {
  const prefix = `[top-level=${self === window.top}]`;
  window.crashStorage.set(`${prefix}-fetching`, url);
  const response = await fetch(url);
}
```

## User needs

The crashes that our proposal helps developers debug are not caused by faulty web
applications—rather, faulty web browser implementations—but web app developers can respond to
crashes and bugs in the platform much faster than browsers can, given complex release cycles.

Therefore, the Crash Storage API lets developers greatly improve user experience by reducing user
exposure to common crashing scenarios before they can be fixed by browser engineers independently.
This increases the overall stability of the web platform, leading to less loss of user data and poor
experience.

## Security and privacy concerns

The following are the answers to the W3C TAG's
[security and privacy self-review questionnaire](https://w3c.github.io/security-questionnaire/).

> 2.1. What information does this feature expose, and for what purposes?

Only developer-supplied information collected throughout an origin's session, and this information
is only exposed *if* the OS process hosting the origin's Document crashes.

> 2.2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

We believe so, yes.

> 2.3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

The feature does not expose any PII. However it is possible for developers to inject PII that they
collect from their app, into the `crashStorage` API, and thus into crash report bodies.

> 2.4. How do the features in your specification deal with sensitive information?

All data inserted into the `crashStorage` API is treated the same, and the data can only come from
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

Like `sessionStorage`, storage in the `crashStorage` API is scoped to an origin under a traversable
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

TODO.

[^1]: This may be changing soon; see
https://github.com/WICG/crash-reporting/issues/24.
