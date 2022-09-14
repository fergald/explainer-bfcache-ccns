# Proposal: give sites the ability to evict documents from BFCache when cookies change

## Context

This proposal is a piece of a larger proposal
to safely enable BFCaching of pages
with the `Cache-Control: no-store` (_CCNS_) header.
For full details on that,
see the accompanying [explainer][ccns-explainer].

## Overview

We propose to add an API that allows a document to declare
that it must not be restored from BFCache
if certain cookies change while it is in BFCache.

For example, the following JS snippet
will cause any documents from this document's origin
which are currently in BFCache,
to be evicted if the SID cookie changes,

```js
backForwardCacheController.evictionCookies = ['SID'];
```

## Motivation

While this API could be useful on its own,
the main motivation for it is as a prerequisite
for safely enabling BFCaching of pages
with the `Cache-Control: no-store` (_CCNS_) header.
CCNS is one of the largest blockers of BFCache usage
and unblocking it would provide a large performance benefit to users.

## Problem we are solving

When a document is in BFCache for some time
and is then restored
and made visible again,
the contents of the document may be out of date
or worse, the document may contain sensitive information
that the current user should not have access to,
e.g. if a logout occurred while the document was in BFCache.

Our goal is to make it easy for sites to avoid these problems
while still allowing and maximising usage of BFCache.

## Existing solutions

### Block BFCache

One simple solution is to prevent documents from being BFCached.
We know that some sites are using CCNS headers to do this
and [advice to do so][ccns-advice] can be found online.
BFCache is a significant performance win for users,
so completely preventing its use,
just in case
is an overly broad solution and hurts users.

### Respond to `pageshow` events

A more complicated solution
is to implement `pageshow` event handlers
that ensure that the document's contents are updated
when the document is restored from BFCache.
This provides the best user experience
but updating a site to do this is non-trivial.

## Proposed solution

Allow sites to declare that certain cookies
should cause the site's documents to be evicted from BFCache.

Cookies are the best practice way to store authentication tokens.
For sites that use them,
a change in authentication state will be accompanied by a change in cookies.
Other cookies beyond authentication
may also be relevant to whether a document should be evicted,
e.g. cookies for shopping cart contents.

Evicting when cookies change strikes a balance
between impact and ease of implementation.
Declaring a set of relevant cookies is low overhead.
Evicting when cookies change
ensures that documents are not restored from BFCache
after authentication state changes.
Evicting when cookies change
causes a relatively low rate of eviction.
Chrome's analysis shows that fewer than 50% of documents
currently blocked from BFCache by the CCNS header
have any cookie change within 3.5 minutes of entering BFCache.
Fewer than 5% have any [secure cookie][secure-cookie] change
in that time.

### Interaction with CCNS header

As mentioned above,
this API could be useful on its own.
However it is being proposed as part of a [broader proposal][ccns-explainer]
to allow BFCaching of documents with the CCNS header.
In that context,
use of this API would be taken as a signal
that a document with CCNS may enter BFCache.
This would allow sites to almost entirely eliminate CCNS blocking
while still ensuring that
documents are not restored after authentication state has changed.

## **API**

The API shape in this document is very much a straw person.
At this stage,
we are more interested in feedback about the fundamental idea
than about the specifics of the API (naming, shape etc).

For example, the following JS snippet
will cause any documents from this document's origin
to be evicted if the SID cookie changes,

```js
backForwardCacheController.evictionCookies = ['SID'];
```

### Details

When `backForwardCacheController.evictionCookies` is set to a list

* if the document enters BFCache,
  the browser must monitor the listed cookies
  (the list may be empty)
  and if any change occurs in those cookies,
  (modify/delete/expire),
  the document should be evicted from BFCache.
* the browser can consider the API to _have_ been used.

When `backForwardCacheController.evictionCookies` is left unset
or set to `null` or `undefined`.

* no cookie-based eviction should occur.
* the browser can consider the API to _have not_ been used.

### API Goals

While the API is a straw person, there are some specific goals:

* list 1 or more cookies
* distinguish between the API having been used or not
* allow for an explicit signal that eviction should not be triggered
  just because cookies change

# API Non-goals

Provide a mechanism to unconditionally prevent documents from being BFCached.
There is [broad agreement ](https://github.com/whatwg/html/issues/5744)
that we should not provide an API that simply prevents BFCaching
as this is too easy to overuse/use incorrectly.

### Key scenarios

#### **Scenario: Monitor cookies**

1. Site a.com uses a cookie named "SID" to store an auth token for logged-in users.
2. The user goes to a.com/foo.
3. This runs the following JS.
```js
backForwardCacheController.evictionCookies = ['SID'];
```
4. The user navigates to a.com/bar.
5. The user logs out from a.com (no navigation occurs).
6. The user traverses back to a.com/foo.

This results in a.com/foo being evicted at step 5,
so step 6 is a fresh load of a.com/foo
instead of restoring a.com/foo as it was while logged in.

#### **Scenario: Behave as before with regard to BFCache**

This is the default state.
This is here for completeness
and to make it clear that there is a difference between ``null`` and ``[]``.
The distinction might be confusing.
We are open to suggestions for an alternative API
that captures the range of options we want to represent.

This use case makes sense
in the context of the [broader proposal][ccns-explainer]
to allow BFCaching of documents with the CCNS header.

```js
backForwardCacheController.evictionCookies = null;
```

#### **Scenario: Explicitly monitor no cookies**

A document wants to signal
that it can be BFCached regardless of cookie changes.
This use case makes sense
in the context of the [broader proposal][ccns-explainer]
to allow BFCaching of documents with the CCNS header.

```js
backForwardCacheController.evictionCookies = [];
```

#### **Scenario: Programmatic flushing of BFCache**

This may be considered an abuse of the API
but absent another API to more directly provide this,
we may see sites setting a cookie
that serves no other purpose than to be a sentinel,
which, when altered,
causes BFCached documents from the site to be evicted.

This could be used when an authentication token is held in shared storage.
Holding a token in IndexedDB or private filesystem
is not best practice but may occur,
e.g. a 3rd-party auth token.
If the document intentionally drops all references to the token
to effectively log out,
it can also change the sentinel cookie
to ensure that any BFCached documents in the logged-in state
are not restored from BFCache.

#### **Possible Scenario: Match a class of cookies**

The API does not support this
and it's unclear that it's needed
(certainly in V1)
but a site may want to say something like,
"evict if any HTTPS-only cookies change"
or "evict if any cookies on this path change".

Until there is a clear use case,
this will remain just a note on a possible extension of the API.

# Considered alternatives

## Header

Sites could also specify the same information in an HTTP header.

The choice to use a JS API rather than a header is currently somewhat arbitrary.

* JS seems more flexible
  and avoids parsing and serialisation concerns if we make changes in the future.
* A header centralises control
  and prevents random JS libraries from altering BFCache+CCNS behaviour

It may be that we would want both.
We will spend more time on this after validating the basic idea.

## Manual Clearing

Allow sites to clear their same-site BFCache entries,
e.g. `Clear-Site-Data: "cache"` (or adding a new value like `"bfcache"`)
or adding an explicit JS API.
An explicit API has been [discussed before](https://github.com/whatwg/html/issues/5744).
This could solve the logout-while-cached problem,
however it has some downsides

* by giving an explicit API
  we risk over-use of that API.
  Our user research and discussions with some sites
  indicates that CCNS is often mis-understood and overused.
  If we provide another easy way to continue to do the same
  it may be used thoughtlessly.
  Until a blanket evict is confirmed to be really needed for specific situations,
  it's preferable to leave the opportunity for the ecosystem
  to push back with concrete scenarios
  for the cases where the current proposal might not be enough in its current shape.
* it provides no signal that can be used to opt in to BFCacheing of CCNS pages
  and so, no CCNS pages could be BFCached until we implement the follow-on.
  This means that until the follow-on,
  this API would only increase evictions.
  It also means that sites with CCNS pages
  cannot effectively trial it before the follow-on
  - their CCNS pages will not be cached.
* with the cookie API,
  sites can lower their eviction rate by listing a small set of cookies,
  which will prevent over-eviction in the follow-on
* pages that use the cookie API will see no change when the follow-on lands
* the cookie API is set-and-forget
  it does not require ensuring that every path that leads to a significant state change
  also clears BFCache

## Triggering on something other than cookies

If there are sites which store authentication tokens in IndexedDB or private filesystem,
we could perhaps evict on changes to certain keys or files.
We have not proposed this since we are unaware of any concrete examples of this.
It's unclear that we _should_ support them if they exist
as a HTTPS-only cookie is considered best practice for storing auth tokens,
however nothing prevents this API from being extended to monitor other stores.

[ccns-explainer]: README.md
[ccns-advice]: https://codingshower.com/disable-bfcache/
[secure-cookie]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#restrict_access_to_cookies
