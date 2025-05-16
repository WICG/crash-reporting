# Crash Storage API Explainer

## Participate

You can participate the evolution of this proposal by filing issues against the Crash Reporting API
specification, at https://github.com/WICG/crash-reporting/issues/new.

## Introduction

The [Crash Reporting API](https://github.com/wicg/crash-reporting) defines a non-web-exposed
mechanism for the user agent to send JSON reports to a developer endpoint, when the OS process
hosting web content crashes. These JSON diagnostic reports include some context about the page
(see https://github.com/WICG/crash-reporting/pull/23), and are sent to the `default` reporting
endpoint specified by the developer. See
[*the Reporting API > Enabling reporting*](https://github.com/w3c/reporting/blob/main/EXPLAINER.md#enabling-reporting)
for more.

### Proposal

The Crash Storage API (currently proposed as `window.crashStorage`) is an extension of the Crash
Reporting API, offering developers a web-exposed key-value store to record arbitrary application
state that gets attached to a
[`CrashReportBody`](https://wicg.github.io/crash-reporting/#crashreportbody). As the application
engages with the web platform in different ways throughout the user's session, this API lets
developers surgically track what specific actions in their app might be causing a crash.

By providing a dedicated storage interface that gets attached to crash reports, developers can
capture essential information leading up to a crash. This significantly bolsters the utility of the
pre-existing Crash Reporting API, giving developers the opportunity to debug each crash they
encounter with more precision.

### Detailed design

The interface of choice for arbitrary key-value storage on the web platform is the
[`Storage` interface](https://html.spec.whatwg.org/multipage/webstorage.html#the-storage-interface),
and the Crash Storage API reuses this generic frontend for the crash-specific storage backend.

**Scoping**

At a high level, the scoping and lifetime of the Crash Storage API is the same as session storage,
in that it is
[scoped to the traversable navigable](https://storage.spec.whatwg.org/#traversable-navigable-storage-shed),
as opposed to the user agent's storage shed, like `localStorage`. This proposal entails creating a new
[registered storage endpoint](https://storage.spec.whatwg.org/#registered-storage-endpoints).

**Refresh persistence**

One difference between `crashStorage` and `sessionStorage` is that while `sessionStorage` data
persists across refreshes in a traversable navigable, `crashStorage` data does not need this level
of persistence, however the exact policy we land on is TBD.

**Which reports get access to `crashStorage` data**

There is an
[open question related to the scope of `crashStorage` data](https://github.com/WICG/crash-reporting/issues/25),
that asks: which Documents actually send crash reports when a process hosting multiple same-origin
Documents crashes? Because it is not always possible to determine which Document in a process caused
a crash, the running idea is that the Crash Reporting API should specify that the topmost Document
for a given origin should generate a `CrashReportBody` with context from *that* document, and send
it to the developer endpoint.

It is important to consider how this interplays with the scoping of `crashStorage` data. Many web
applications are structured in a way where the top-level Document is merely a thin host for a suite
of iframes, many of which are same-origin with the top-level Document, and are the primary drivers
of the application. In these web applications, it is more likely that a same-origin iframe caused a
crash than the top-level Document, and if a crash report body gets generated for the top-level
Document, it is crucial that it includes any developer-provided context for the entire origin, such
as any application state stored in the `crashStorage` API.

## Usage

From a JavaScript developer's perspective, the `crashStorage` API looks and feels just like
`sessionStorage` or `localStorage`, with the aforementioned scoping and persistence considerations
above. The intended use of the API looks like this:

```js
window.crashStorage.setItem('complex-operation-input', String(arg1 + arg2));
// If the following operation crashes, its input state will be sent in a `CrashReportBody` to the
default endpoint.
complexOperationThatMightCrash(arg1, arg2);
window.crashStorage.removeItem('complex-operation-input');
```

Note that because crash storage data is stored among all same-origin Documents under a traversable
navigable, the developer might take care to prefix certain keys for common operations that multiple
Documents are doing at the same time. For example, imagine the developer suspects that `fetch()` is
crashes under certain conditions, but multiple same-origin Documents in a page call `fetch()` at
different times.

To record which specific `fetch()` is happening at a given time, to help narrow down any crashes, a
developer might adopt some prefixing strategy to prevent clobbering the same state in the
`crashStorage` API:

```js
// Code that runs in multiple Documents.
function fetchURL(url) {
  const prefix = `[top-level=${self === window.top}]`;
  window.crashStorage.set(`${prefix}-fetching`, url);
  const response = await fetch(url);
}
```

## Motivation

Modern web applications often encounter unexpected crashes in the process hosting their web content,
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

## User needs

TODO.

## Security and privacy concerns

TODO.
