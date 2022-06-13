# API to evict from BFCache when cookies change


## Authors:



*   Fergal Daly ([fergal@chromium.org](mailto:fergal@chromium.org))
*   Kenji Baheux‎ ([kbx@chromium.org](mailto:kbx@chromium.org))

**Status: Draft for discussion**


## Participate

*   [Issue tracker](https://github.com/fergald/explainer-bfcache-ccns/issues)

We are particularly interested in feedback on

*   the functionality of evicting on cookie change
*   follow-on of defaulting to caching documents with Cache-Control: No-Store

## Introduction

Documents with a [Cache-Control: no-store](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) header (_CCNS_) are allowed, by spec,
to enter BFCache (i.e. to remain salvageable after navigating away)
but in practice all browsers prohibit this
because these documents may contain sensitive content
that should no longer be available,
e.g. after a change in login status.

We propose an API that would give documents more control over when they are evicted
by allowing them to declare that certain events
(change/delete specified cookies)
will cause eviction from BFCache.
If documents are evicted when important state changes
then it becomes safe to allow them to enter BFCache
even if CCNS has been set.

While this API could increase evictions
(since all it does is add a new way to cause evictions),
if it makes it safe to cache documents with CCNS
it could greatly increase the number of documents
that get into BFCache in the first place.

### Impact

In Chrome’s measurements,
CCNS is the single largest reason blocking documents from being BFCached on mobile
(18% blocked by only this reason)
and the second largest reason on desktop
(7% blocked by only this reason).
However, in Chrome's experiments,
less than 1% of documents with CCNS saw a HTTP-only-cookie change
during the 3m30s BFCaching window.
On Chrome mobile,
if all CCNS documents were cacheable
and only evicted on HTTP-only-cookie change,
then we would see an increase in the BFCache hit-rate
of about 14 percentage points
(3% of all navigations would become BFCache navigations).

## Background

### Current state

Despite the name,
BFCache is not a cache in the HTTP sense,
so the CCNS header does not apply to BFCache,
in theory.
A browser implementing BFCache does not store web resources
(HTTP responses)
for later use,
like an HTTP cache would.
Instead, a browser implementing BFCache
keeps the live state of web documents in memory,
and unfreezes them when a user navigates back, or forward.

In practice,
all the current BFCache implementations
treat CCNS as a signal to not allow the document to enter BFCache.
This has been true for quite some time
and the web ecosystem has built an expectation about this behaviour.

Some sites also use CCNS to prevent BFCache-related problems
where e.g. shopping-cart contents have changed.
The correct solution to this is not to prevent BFCaching
but to update the UI
to reflect the correct state in the pageshow event handler.
Doing this by blocking BFCache hurts performance.

The general problem is that
some state has changed while the document is in BFCache
and if it were restored from BFCache,
it would be inconsistent with the other documents of this site and inconsistent with the user’s expectations
(including privacy and security expectations).

### Authentication and cookies

Best practice for authentication has been,
for a long time,
to hold a token in an HTTP-only cookie
so that all requests present this token.
Deleting this token corresponds to a logout.
Replacing the token with a new value
might also correspond to a change in authentication state.
The browser cannot understand the details of cookie changes,
so conservatively, we consider any change to be significant.

### Past attempt to allow documents with CCNS into BFCache

In the past, [Webkit experimented](https://github.com/whatwg/html/issues/5744#issuecomment-661997090) with
allowing documents with CCNS to enter BFCache.
This was discontinued
because a user going back after logout
would lead to the previous document being displayed,
still with the logged-in content.
This is a serious risk shared devices
(and still a problem on unshared devices).
Our proposal addresses this case.

## Goals

### New API

Allow documents to declare that
they should be evicted from BFCache
(i.e. salvageable documents will be marked unsalvageable)
if certain cookies change.

### Cache CCNS documents when the API is used

A document's use of the API,
is taken as a signal
that it should be allowed to enter BFCache,
even if CCNS is present

### Follow-on: Cache CCNS documents

Clear a path for the [follow-on change](#follow-on-caching-ccns-documents-by-default) of
allowing documents with CCNS
to be BFCached by default.

## Non-goals

We do not want to
provide an API
to unconditionally prevent documents from being BFCached.


## User research

TODO - integrate what we have

[If any user research has been conducted to inform the design choices presented discuss the process and findings. We strongly encourage that API designers consider conducting user research to verify that their designs meet user needs and iterate on them, though we understand this is not always feasible.]

## Proposal: Allow BFCache eviction to depend on cookies

### Overview

We propose to add an API
that allows a document to declare that
it should not be restored from BFCache
(marked as unsalvageable)
if certain cookies change
while it is in BFCache
(not active but still salvageable).
This would be independent of CCNS headers,
however with this API in place,
browsers could start to allow documents with CCNS
to be restored from BFCache
(with several caveats).

### **API**

This API shape here is very much a straw man.
At this stage,
we are more interested in feedback
about the fundamental idea
than about the specifics of the API
(naming, shape etc).

The only function of the API
is to set a list of cookie names
that should be used for eviction.
This could be done with assignment
or by passing an array as an argument.

For example, to evict documents if the SID cookie changes

```
document.BackForwardCacheController.evictionCookie = ['SID'];
```

### Details

When a document has set `document.BackForwardCacheController.evictionCookie` and that document is in BFCache
(not active but salvageable),
the browser should monitor the listed cookies
and if any change occurs,
evict the document
(mark the document as not salvageable).

The following is not a change to spec
since CCNS is not currently a reason to prevent BFCaching.
If a document has used the API,
that is a signal that the document
can safely enter BFCache despite CCNS.
One caveat is that if the user has disabled cookies,
this API cannot work as intended.
In that case it is best to continue
blocking the document from entering BFCache.

While the API is a straw man,
there are some specific goals:

*   list 1 or more cookies to trigger eviction
*   allow for an explicit signal that no cookies should trigger eviction

## Key scenarios

#### **Scenario: Monitor a cookie**

1. Site a.com uses a cookie named "SID" to store an auth token for logged-in users.
1. The user goes to a.com/foo.
1. This runs the following JS.
   ```
   document.BackForwardCacheController.evictionCookie = ['SID'];
   ```
1. The user navigates to a.com/bar.
1. The user logs out from a.com (no navigation occurs).
1. The user navigates back to a.com/foo.

This results in a.com/foo being evicted
(marked as unsalvageable)
at step 5
and step 6 is a fresh load of a.com/foo
instead of restoring a.com/foo
as it was while logged in.

#### **Scenario: Monitor no cookies**

A document wants to give the browser an explicit signal
that it can be BFCached regardless of cookie changes

```
document.BackForwardCacheController.evictionCookie = [];
```

#### **Scenario: Programmatic flushing of BFCache**

This may be considered an abuse of the API but
absent another API to more directly provide this,
we may see sites setting a cookie
that serves no other purpose than to be a sentinel,
which, when altered,
causes any BFCached documents from the site to be evicted.

This could be used when an authentication token is held in shared storage
(not good practice),
e.g. a 3rd-party auth token.
If the document intentionally drops all references to the token
to effectively log out,
it can also change the sentinel cookie
to ensure that any BFCached documents in the logged-in state are evicted.

#### **Possible Scenario: Match a class of cookies**

The straw-man API does not support this
and it's unclear that it's needed
(certainly in V1)
but a site may want to say
"evict if any HTTP-only cookies change" or
"evict if any cookies on this path change".

Until there is a clear use case,
this will remain just a note on a possible extension of the API.

## Considered alternatives

### Header

Sites could also specify the same information in an HTTP header.

The choice to use a JS API rather than a header
is currently somewhat arbitrary.
We haven’t thought of major advantages to either,
although JS seems more flexible and
avoids parsing and serialisation concerns
(which also avoids difficulties in extending it in the future).
It may be that we would want both.
We will spend more time on this after validating the basic idea.

# Follow-on: Caching CCNS documents by default

This section is not a proposal to change specified behaviour,
since BFCaching a document with CCNS
is already permitted by the spec
and some of the details below are not speccable.
That said, after making this API available,
we believe that we could enable BFCaching of documents with CCNS by default,
whether they have used the API or not.
There are some risks but there are very large performance benefits.

## Proposal

CCNS should not be considered a reason to mark a document as unsalveagable,
stopping it entering BFCache,
regardless of whether the new API is used.

If the API is not used,
eviction is triggered on a change to _any_ HTTP-only cookie.
We also add several other conditions when the API is not used

* no BFCaching if a document has no HTTP-only cookies at all or
  cookies have been disabled as this would lead to unconditional caching
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
This should be extremely rare, if it exists at all.

### 3rd-party Auth Tokens

A site may make RPCs to 3rd-party services directly from the browser
and bring users through 3rd-party auth flows
in order to collect a token to authorise these RPCs.
The token would be held in JS and
manually attached to RPC requests.
If there was a site which does this
but has no login/cookies of its own
and also has no coordination between pages
to ensure that all pages log out of 3rd-parties if one does
then this site could surprise the user
by restoring a page with sensitive content
and/or still in possession of a valid auth token
when the user thinks they have logged out.

We know of no sites like this and the scenario is somewhat convoluted.

### Stale information

Sites using CCNS
to ensure that fresh information is downloaded every time
rather than to ensure that sensitive information is not retained
will end up showing stale information
(within the BFCache timeout).

Sites can mitigate this with a pageshow handler
that detects `event.persisted == true`
and causes a reload.
