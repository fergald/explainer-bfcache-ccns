# API to evict from BFCache when cookies change

## Authors:

* [Fergal Daly](mailto:fergal@chromium.org)
* [Kenji Baheux](mailto:kbx@chromium.org)

with much feedback from

* [Rakina Zata Amni](mailto:rakina@chromium.org)
* [Domenic Denicola](mailto:domenic@chromium.org)
* [Kentaro Hara](mailto:haraken@chromium.org)

**Status: Public, requesting feedback.

# Participate

* [Issue tracker](https://github.com/fergald/explainer-bfcache-ccns/issues)

We are particularly interested in feedback on

* the functionality of evicting on cookie change vs the alternative of providing an explicit API to evict
* moving to caching documents with `Cache-Control: No-Store`
* where this will go wrong that we have missed
   * risky cases we have missed
   * risky cases that are much more common than we think
   * cases that are impossible to mitigate

# Overview

Documents with a [`Cache-Control: no-store`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) header (_CCNS_)
are blocked from entering BFCache in practice on all browsers.
Chrome measurements indicate that this prevents about 17% of all history navigations
from being BFCached.
This proposal aims to move us to a place where most CCNS documents are cached by default
and CCNS prevents caching of only about 1% of history navigations.

We propose 2 alternative APIs that give page authors the ability to evict pages from BFCache,
one based on monitoring cookies,
the other a simple, explicit evict API.
With these available
we believe we can safely make a [follow-on change][follow-on-proposal]
to allow CCNS documents into BFCache by monitoring cookies
(plus some other exceptions).

# Introduction

## New API

Documents with a CCNS header are allowed, by spec, to enter BFCache
but in practice all browsers prohibit this
because these documents may contain sensitive content
that should no longer be available,
e.g. after a change in login status.

We propose an API that would give documents more control over when they are evicted from BFCache
by allowing them to declare
that certain events
(change/delete/expire specified cookies)
will trigger eviction.
If documents are evicted when important state changes
then it becomes safe to allow them to enter BFCache
even if CCNS has been set.

While this API could increase evictions
(since all it does is add a new way to cause evictions),
if it allows caching of documents with CCNS,
it could greatly increase the number of documents that get into BFCache in the first place.

## Follow-on: Cache CCNS by default

This is the key change.
With this API in place, we propose

[follow-on changes][follow-on-proposal] to fully spec how BFCache interacts with CCNS
such that in most cases,
CCNS documents will enter BFCache and be evicted on cookie changes
- the API-specified cookies
or if none are specified, any HTTPS-only cookie change.

## Impact

In Chrome’s measurements,
CCNS is the single largest reason blocking documents from being BFCached on mobile
(18% blocked by only this reason
) and the second largest reason on desktop (7% blocked by only this reason).
However, in Chrome's experiments,
less than 1% of documents with CCNS saw a HTTPS-only-cookie change during the 3m30s BFCaching window.
On Chrome mobile,
if all CCNS documents were cacheable
and only evicted on HTTPS-only-cookie change,
then we would see an increase in the BFCache hit-rate
of about 14 percentage points
(3% of all navigations would become BFCache navigations).

When a document is restored from BFCache,
it is generally presented to the user almost instantly
(10s or low 100s of ms).
Without BFCache the HTML/script must be parsed and run,
possibly including network and server delays.
This is a huge performance improvement,
a great user experience
and has been shown to increase engagement
and task completion rates on sites that make their pages compatible
([Yahoo Japan blog post][yahoo-japan-blog-post],
Google internal measurements)

## Example usage

A site with an auth cookie name 'SID' wants to ensure that pages are evicted from BFCache if this cookie is deleted or changed.

```js
backForwardCacheController.evictionCookies = ['SID'];
```

# Background

## BFCache

BFCache is a "cache" in browsers
such that when the user navigates away from a top-level document,
the browser may preserve that document in a frozen state
and if the user traverses history back to that document's history entry,
the browser may just unfreeze the preserved document.
There are many things that prevent browsers from doing this with a given document
(e.g. use of certain APIs,
too much time having passed since navigating away).

[This web.dev article](https://web.dev/bfcache) has more information on BFCache.

## Current state

Despite the name,
BFCache is not a cache in the HTTP sense,
so the CCNS header does not apply to BFCache,
in theory.
A browser implementing BFCache does not store web resources (HTTP responses) for later use,
like an HTTP cache would.
Instead,
a browser implementing BFCache freezes previous web documents in memory,
and restores them when a user navigates back, or forward.

In practice,
all the current BFCache implementations treat CCNS as a signal
to not allow the document to enter BFCache.
This has been true for quite some time
and the web ecosystem has built an expectation about this behaviour.

Some sites also use CCNS to prevent BFCache-related problems
where e.g. shopping-cart contents have changed.
The correct solution to this is not to prevent BFCaching
but to update the UI to reflect the correct state
in the [pageshow event handler][page-show-event-handler].

The general problem is that some state has changed while the document is in BFCache
and if it were restored from BFCache,
it would be inconsistent with the other documents of this site
and inconsistent with the user’s expectations.

## Authentication and cookies

Best practice for authentication has been,
for a long time,
to hold a token in an HTTPS-only cookie
so that all requests present this token.
Deleting this token corresponds to a logout.
The same outcome applies when the token expires.
Replacing the token with a new value
might also correspond to a change in authentication state.
Conservatively, we can consider any change to be significant.

## Other cookies

Other non-auth or non-HTTPS-only cookies may also be significant to a page
and so the proposed API applies to all cookies.

## Past attempt to allow documents with CCNS into BFCache

In the past,
Webkit [experimented][webkit-experimented] with allowing documents with CCNS to enter BFCache.
This was discontinued after a serious privacy problem was reported.
A bank was using shared tablets for customers.
2 customers used the tablet in close succession.
Customer-1 logged out.
The bank staff gave the tablet to customer-2
but instead of logging in,
they went back
and were presented customer-1's logged-in account.
Customer-1's cookies had been cleared by the logout
but the document for their account
was still in BFCache.

The [follow-on proposal][follow-on-proposal] allows us to avoid repeating this problem
while extending BFCaching to many docs with CCNS.

# Goals

## New API

Allow documents to declare conditions that will cause them to be evicted from BFCache.
E.g. Evicting the document if a specific cookie is updated, deleted or expired
would ensure that a logout from the site
would never be followed by a logged-in document being restored from BFCache.

## Cache CCNS documents when the API is used

Using the API is opting in to caching with CCNS.

## Follow-on: Cache CCNS documents

With the API in place and in use,
documents with CCNS that do not use the API
would be allowed into BFCache
and evicted on any change to any HTTPS-only cookie ("secure" cookie).
This would happen some time after the API is stable.
It would need to be announced ahead of time
and perhaps also use console warnings
or other prompts to raise awareness.

# Non-goals

Provide a mechanism to unconditionally prevent documents from being BFCached.
There is [broad agreement ](https://github.com/whatwg/html/issues/5744)
that we should not provide an API that simply prevents BFCaching
as this is too easy to overuse/use incorrectly.

# Proposal: Allow BFCache eviction to depend on cookies

## Overview

We propose to add an API that allows a document to declare
that it must not be restored from BFCache
if certain cookies change while it is in BFCache.
This would be independent of CCNS headers,
however with this API in place,
browsers could start to allow documents with CCNS to be restored from BFCache
(with several caveats).

Cookies are the best practice way to store auth tokens.
For sites that use them,
changes in authentication state will be accompanied by a change in cookies.
Other cookies beyond authentication
may be relevant to whether a document should be evicted,
e.g. cookies with shopping cart contents.
Ideally, the document would react to changes in such cookies in `pageshow` event handlers
but this proposed API provides an easy way to make such a document BFCache-correct
while still often caching it.

Changes in the relevant cookies are a signal that BFCached documents should be evicted.

### **API**

The API shape in this document is very much a straw person.
At this stage,
we are more interested in feedback about the fundamental idea
than about the specifics of the API (naming, shape etc).

For example, the following JS snippet
will cause any documents from this documents origin
to be evicted if the SID cookie changes,

```js
backForwardCacheController.evictionCookies = ['SID'];
```

## Details

When `backForwardCacheController.evictionCookies` is set to a non-null value

* if the document enters BFCache,
  the browser must monitor the listed cookies
  and if any change occurs,
  evict the document

### Open Question about speccing BFCache+CCNS

It's unclear whether, with this API,
we should spec the interaction between CCNS and BFCache.
If we were to do that,
to avoid leaving undefined behaviour,
we would say that

* a document using this API and CCNS may enter BFCache
  (as long as cookies for this origin have not been disabled in the browser)
* all other CCNS documents should not

and then the follow-on proposal would loosen the "all other CCNS documents" case.

It may be preferable to just leave CCNS+BFCache unspecced
and do it all in the follow-on.

## API Goals

While the API is a straw person, there are some specific goals:

* list 1 or more cookies
* allow for an explicit signal that eviction should not be triggered
  just because cookies change

# Key scenarios

### **Scenario: Monitor a cookie**

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

### **Scenario: Behave as before with regard to BFCache**

This is the default state.
This is here for completeness
and to make it clear that there is a difference between ``null`` and ``[]``.
The distinction might be confusing.
We are open to suggestions for an alternative API
that captures the range of options we want to represent.

```js
backForwardCacheController.evictionCookies = null;
```

### **Scenario: Monitor enter BFCache monitoring no cookies**

A CCNS document wants to give the browser an explicit signal
that it can be BFCached regardless of cookie changes

```js
backForwardCacheController.evictionCookies = [];
```

### **Scenario: Programmatic flushing of BFCache**

This may be considered an abuse of the API
but absent another API to more directly provide this,
we may see sites setting a cookie
that serves no other purpose than to be a sentinel,
which, when altered,
causes any BFCached documents from the site to be evicted.

This could be used when an authentication token is held in shared storage
(e.g. IndexedDB or private filesystem - not a great idea),
e.g. a 3rd-party auth token.
If the document intentionally drops all references to the token to effectively log out,
it can also change the sentinel cookie
to ensure that any BFCached documents in the logged-in state also log out.

### **Possible Scenario: Match a class of cookies**

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

# Follow-on: Caching CCNS documents by default

This section is not exactly a proposal to change specified behaviour,
since BFCaching a document with CCNS is already permitted by the spec
and some of the details below are not speccable.
That said, after making the above API available,
we believe that we could enable BFCaching of documents with CCNS by default,
whether they have used the API or not
and could update the spec to explicitly rule out some cases,
thereby implicitly ruling in the other.
There are some risks
but there are very large performance benefits from BFCache.

We believe there should be some time
and some effort to raise awareness
between making the API above available and default BFCacheing of CCNS documents.
This will allow for more awareness and early opt-in by use of the cookie-monitoring API.

## Proposal

CCNS should not be considered a reason to stop a document entering BFCache,
regardless of whether the new API is used.

If the API is not used,
eviction is triggered on a change to _any_ HTTPS-only cookie.
This will make it impossible to repeat the bank-tablet example above.
However it does not remove all risks.

We also add several other conditions that cause the BFCache to not be used

* no BFCaching of CCNS documents
  if cookies for that origin have been disabled in the browser
  as this would lead to unconditional caching
* (possible) no BFCaching if a document has no HTTPS-only cookies
  as this would lead to unconditional caching
* no BFCaching if JS is disabled in the browser
  as the document has not had a chance to use the API
* provide a disable switch to enterprises with managed browsers
  as they often have difficult-to-update software and/or shared devices
* evict documents if the HTTP-Authentication state changes
  [ fergal to file a separate issue for this as it's an existing issue with BFCache ]

## Risks

### Legacy Logins

Some legacy systems could use tokens in the URL instead of cookies,
even when cookies are not disabled.
This should be extremely rare,
if it exists at all.

### 3rd-party Auth Tokens

A site may make RPCs to 3rd-party services directly from the browser
and bring users through 3rd-party auth flows in order to collect a token to authorise these RPCs.
The token would be held in JS
and manually attached to RPC requests.
If there was a site which does this
but has no login/cookies of its own
and also has no coordination between pages
to ensure that all pages log out of 3rd-parties if one does
then this site could surprise the user by restoring a page with sensitive content
and/or still in possession of a valid auth token
when the user thinks they have logged out
by navigating away.

We know of no sites like this
and the scenario is somewhat convoluted.

### Server-side logout

Sites may log users out on the server side
and clients may be unaware of this.
This is not a new risk,
sites that do this now will have the same problem with open tabs.
This can be solved for tabs and for BFCache
via various technologies such as periodically
(and in pageshow)
checking with the server,
or using a service worker or BroadcastChannel to notify about logouts.

### Stale information

Sites using CCNS to ensure that fresh information is downloaded every time
rather than to ensure that sensitive information is not retained
will end up showing stale information
(within the BFCache timeout).

Sites can mitigate this with a pageshow handler
that detects `event.persisted == true`
and either performs client-side updates to the page's
content, or causes a reload.

### Non-HTTPS-Only (non-secure) Cookies

Documents may have content that depends on non-secure cookies
and by only evicting on changes to secure cookies
we may restore some pages that should not be restored.
Our goal is to avoid exposing sensitive information
and we expect that to be tied to secure cookies.
The same mitigations apply as to [stale information](#stale-information).

[yahoo-japan-blog-post]: https://techblog.yahoo.co.jp/entry/2022010530253635/#:~:text=%E3%81%9D%E3%81%AE%E7%B5%90%E6%9E%9C%E3%80%81iOS%E3%81%AESafari%E3%81%A7%E3%81%AFPV/UB%E3%81%8C2%EF%BC%85%E4%BB%A5%E4%B8%8A%E5%90%91%E4%B8%8A%E3%81%99%E3%82%8B%E7%B5%90%E6%9E%9C%E3%81%8C%E5%BE%97%E3%82%89%E3%82%8C%E3%81%BE%E3%81%97%E3%81%9F%E3%80%82BFCache%E3%81%AF%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AE%E4%BD%93%E9%A8%93%E3%81%8C%E6%9C%80%E9%81%A9%E5%8C%96%E3%81%95%E3%82%8C%E3%82%8B%E3%81%9F%E3%82%81%E3%80%81PV/UB%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E6%8C%87%E6%A8%99%E3%82%82%E3%83%9D%E3%82%B8%E3%83%86%E3%82%A3%E3%83%96%E3%81%AB%E5%8B%95%E3%81%84%E3%81%9F%E3%81%AE%E3%81%A0%E3%81%A8%E6%8E%A8%E6%B8%AC%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82
[page-show-event-handler]: https://web.dev/bfcache/#update-stale-or-sensitive-data-after-bfcache-restore
[webkit-experimented]: https://github.com/whatwg/html/issues/5744#issuecomment-661997090
[follow-on-proposal]: #follow-on-caching-ccns-documents-by-default
