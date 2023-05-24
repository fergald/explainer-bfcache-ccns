# Enabling BFCache for pages that set Cache-Control: no-store. 2 linked proposals to get there.

## Authors:

* [Fergal Daly](mailto:fergal@chromium.org)
* [Kenji Baheux](mailto:kbx@chromium.org)

with much feedback from

* [Rakina Zata Amni](mailto:rakina@chromium.org)
* [Domenic Denicola](mailto:domenic@chromium.org)
* [Kentaro Hara](mailto:haraken@chromium.org)

**Status: Public, requesting feedback.

## Participate

* [Issue tracker](https://github.com/fergald/explainer-bfcache-ccns/issues)

We are particularly interested in feedback on

* the functionality of evicting on cookie change
  vs the alternative of providing an explicit API to evict
* moving to caching documents with `Cache-Control: no-store`
* where this will go wrong
   * risky cases we have missed
   * risky cases that are much more common than we think
   * cases that are impossible to mitigate

## Overview

### Problem

Documents with a [`Cache-Control: no-store`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control) header (_CCNS_)
are blocked from entering BFCache in practice on all browsers.
Chrome measurements indicate
that this prevents about 17% of history navigations on mobile
and 7% of history navigations on desktop
from being BFCached.
This is the largest single blocker of BFCache usage.

CCNS documents are blocked because
they may contain sensitive content that
was accessible when the document was first fetched
but which should now be inaccessible.
This is especially important on devices that are shared.

The goal is to unblock BFCaching of
a meaningful fraction of CCNS pages
without compromising sensitive information.

### Proposal

These proposals aims to move us from
not caching any CCNS documents to
caching most CCNS (but evicting some).
In this state, we estimate that
CCNS will cause BFCache misses for only about 1% of history navigations.

We propose to start BFCaching pages
where we believe we can avoid
exposing sensitive information
that would otherwise have been inaccessible to the user.
This policy is conservative
and will deny BFCache to many pages
that could safely be cached.

We also propose [APIs][api]
that give page authors the ability to evict pages from BFCache,
one based on monitoring cookies and storage (preferred),
the other a simple, explicit eviction API.
With either of these available
we believe browsers can safely make a [follow-on change][follow-on-proposal]
to allow most CCNS documents into BFCache by monitoring cookies
and usage of the `Authorization` header
and then evicting documents when secure cookies change.
This gives page authors finer-grained control
allowing them to avoid over-eviction.

There are some other circumastances
where we should not cache.
These are detailed below.

### Impact

In Chrome’s measurements,
CCNS is the single largest reason blocking documents from being BFCached on mobile
(18% blocked by only this reason)
and the second largest reason on desktop (7% blocked by only this reason).
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

## Background

### BFCache

BFCache is a "cache" in browsers
such that when the user navigates away from a top-level document,
the browser may preserve that document in a frozen state
and if the user traverses history back to that document's history entry,
the browser may just unfreeze the preserved document.

This provides a significant performance boost
on these navigations
at the cost of not refreshing the content.
Depending on the nature of the content,
not refreshing the content may be harmless,
inconvenient or a serious problem
(e.g. the user has logged out
but the page still contains logged-in content).

There are many things that prevent browsers from doing this with a given document
(e.g. use of certain APIs,
too much time having passed since navigating away).

[This web.dev article](https://web.dev/bfcache) has more information on BFCache.

### Sensitive information

In the context of CCNS and BFCache,
we consider sensitive information
to be information that should not be available
after the user has navigated away from the page
and causes the use of CCNS.

We do not consider data in `localStorage`
to be sensitive data in this context
because that data currently remains available
after the user has navigated away.

Data received over the network
either in the page
or in response to authenticated RPCs.
Furthermore we would expect
that if the content is senstive
that is it would be uncacheable
and include CCNS itself.

The sources of senstive data we consider are

- documents fetched as part of the page (the main frame and iframes)
  authorized by cookies
- data received via `fetch` and `XMLHttpRequest`
- data received from
  - `WebSocket`
  - `WebTransport`
  - `WebRTC`

### Authorization

#### Secure cookies

Best practice for authorization has been,
for a long time,
to hold a token in an HTTPS-only cookie
so that all requests present this token.
Deleting this cookie deauthenticates the browser
and usually corresponds to a logout.
The same outcome applies when the cookie expires.
Replacing the cookie with a new value
might also correspond to a change in authorization state
or it may just be a refresh of the token
or other neutral change.

Conservatively, we can consider any change
to be a significant authorization change
even if it does not correspond to a logout.

#### Other cookies

Other non-auth or non-HTTPS-only cookies may also be significant to a page
e.g. representing contents of shopping carts.
Changes to these may also imply that inactive pages in BFCache or prerendering
have outdated/incorrect contents.

The most conservative stance would be
to consider all cookies on CCNS documents
as potentially guarding access
to sensitive information.
We are not aware of any reasons why
sites would use insecure cookies
for this purpose
and it is very poor security practice to do so.
As a result we aim for an end-state
where only secure cookies are considered.

#### Access tokens

Authorization may also happen via 3rd parties,
or custom mechanisms
resulting in tokens that are presented in RPCs.

For example, federated sign-in
(sign-in with Google/Twitter/Facebook/etc)
provides a token that can then be presented on later RPCs.
There are sites that use this as their only form of authorization
(with no use of cookies).
Typically they are single-page apps since,
without cookies,
HTTP requests resulting from navigation
may be unauthenticated
with all sensitive information
being provided through RPCs.

##### Presentation

The OAuth [spec](https://datatracker.ietf.org/doc/html/rfc6750#section-5.3) recommends
not to pass access tokens in the URL,
to use the `Authorization` header instead.
Twitter and Facebook accept access tokens in the URL,
this practice is suggested on Stack Exchange
and Facebook's JS client library
put the tokens in the URL.
We must assume that passing authentication tokens in the URL
is common practice.
For OAuth,
the token in the URL is standardized
to be passed in the parameter `access_token`
but we cannot assume that OAuth
is the only authentication scheme.

This means that in practice,
we cannot use the absence of the `Authorization` header
as a signal that an RPC contains no sensitive information.

##### Storage

Best practice for tokens like this
is to write them to site-private storage,
e.g. LocalStorage, SessionStorage, IndexedDB.
The token is read from storage when attaching to each RPC.
This ensures that a logout occurring in another tab
or in the same tab but in unrelated code
will result in fully deauthenticating the user.

### Current interactions between BFCache and CCNS

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
to prevent the document from entering BFCache.
This has been true for quite some time
and the web ecosystem has built an expectation about this behaviour.

Some sites also use CCNS to prevent BFCache-related problems,
e.g. where shopping-cart contents have changed.
A better solution to this is not to prevent BFCaching
but to update the UI to reflect the correct state
in the [pageshow event handler][page-show-event-handler].

The general problem is that some state has changed
while the document is in BFCache
and if it were restored from BFCache,
it would be inconsistent with the other documents of this site
and inconsistent with the user’s expectations
of correctness or privacy.

### Past attempt to allow documents with CCNS into BFCache

In the past,
Webkit [experimented][webkit-experimented] with allowing documents with CCNS to enter BFCache.
This was discontinued after a serious privacy problem was reported.
A bank was using shared tablets for customers.
2 customers used the tablet in close succession.
Customer-1 logged out.
The bank staff gave the tablet to customer-2
but instead of logging in,
they went back
and were presented with a page from customer-1's logged-in account.

Customer-1's cookies had been cleared by the logout
but the document for their account
was still in BFCache.

## Detailed Proposal

Getting to the point where we can BFCache
many documents with CCNS
takes multiple steps
*and* there are multiple paths we can take.
Those paths are essentially

- caching-first (proposed)
  - [start caching some CCNS pages by default
    using the most conservative criteria](#allow-ccns-documents-to-be-bfcached-without-the-api)
  - [introduce the API](#allow-more-ccns-documents-to-be-bfcached-with-the-api)
  - [cache more CCNS pages by default
    by adding the API to the criteria](#bfcache-ccns-pages-if-https-only-cookies-dont-change)
- API-first (dropped in favour of caching-first)
  - introduce the API
  - move to caching some CCNS pages

### New API to monitor authorization impacting events

We propose an [API][api] that would give documents more control over when they are evicted from BFCache
by allowing them to declare
that certain events
(change/delete/expire specified cookies or storage keys)
will trigger eviction.
This API has some merit on its own,
as a way to avoid problems with BFCache
without blocking it entirely.
However there has been no demand for it from developers
and on its own
it would cause more BFCache evictions.

### Signals used to determine if BFCaching is allowed

- API is available - this requires that both
  - the API is exposed to JS
  - JS is not disabled by the user
- HTTP-authentication state
- enterprise policy switch
- what cookies have changed.
  This signal is only valid if cookies are not disabled
  by the user/agent for this page.
  If cookies are disabled then we consider them to always have changed.
- have RPCs occurred with CCNS responses?
  This is based on observing network responses
  made by the top-level frame
  and all subframes.
- have network connections been made
  where we have no signal about cacheability?
  - `WebSocket`
  - `WebTransport`
  - `WebRTC`

### Signals that are always used

In the below,
the following signals are always used
and can cause a CCNS page not to be restored from BFCache
regardless of any other conditions.

#### HTTP-authentication state

If HTTP-authentication state changes then
a CCNS page will not be restored from BFCache.

#### Enterprise policy switch

If enterprise policy disabled BFCaching of CCNS pages
then a CCNS page will not be restored from BFCache.
Enterprises often have difficult-to-update software
and/or shared devices.

#### Opaque network APIs

If any of

- `WebSocket`
- `WebTransport`
- `WebRTC`

APIs are used by a document with CCNS
the we are forced to assume it contains sensitive information.

### Allow CCNS documents to be BFCached without the API

If the API is not available,
either because it is not launched
or because the user has disabled JS for this page
then we take a conservative approach

A frame (main or sub) is considered a "safe" frame
if all of the follow are true:

- the frame is not CCNS or
  cookies are enabled and no HTTP-only cookies
  have changed since the document was fetched.
- no RPCs with potentially sensitive information have occurred
  (CCNS on response)
- none of the "always used" signals are present

### Timeout

Chrome's current timeout for BFCache entries is 10 minutes.
WebKit and Mozilla have longer timeouts,
up to 45 minutes.
We proprose that a CCNS page
should not be restored after 3 minutes.
This is to address the fact that

- CCNS may be used for freshness
  rather than for protecting sensitive information
- access to sensitive information
  may be revoked at the server side
  without any signal to the browser

#### Proprosed approach

We propose to allow a CCNS main frame
to be restored from BFCache
if it and all of its same-origin subframes
are considered safe.

#### More conservative approaches

##### Cookies

Instead of all HTTP-only cookies,
we could only consider all cookies.
Best practice for cookies
used to access sensitive inforation
is for them to be HTTP-only.
This is also well-established, common practice,
not just a theoretical best practice.
Even still, it's likely some sites do not follow it.
We know from previous measurements
(based on cookies alone,
ignoring RPCs)
that about 50% of CCNS have a cookie change
while in BFCache for up to 3.5 minutes
but only 5% have a HTTP-only cookie change.

##### Subframes

Instead of only subframes
which are same-origin with the main frame,
we could consider all subframes.
The risk here is that there are cross-origin subframes
that contain sensitive information
and the user could deauthenticate
from that site in another tab
(or in the same tab while the page is in BFCache)
and then be surprised to find sensitive information
visible after the page in question is restored.

Many sites disallow embedding in a subframe
specifically because they contain senstive information
and allowing embedding leads to security problems.
We are unaware of examples of this.

### Allow more CCNS documents to be BFCached with the API

We take usage of the [API][api]
as a signal that the document can be BFCached
even with the CCNS header
because it has specified the conditions
under which it should not be restored.

A CCNS page can be cached as before but

- if the API has been used to declare cookies for a document
  then changes to other cookies are not considered relevant.
- if the API has been used to declare
  where access tokens are stored
  and those tokens have not changed
  then RPCs are considered relevant.

This allows sites to

- increase their BFCache hit rate
- avoid restoring documents with information that should not be restored
- with the very small engineering overhead
  of declaring a list of cookies
  or storage locations to monitor

### BFCache CCNS pages if HTTPS-only cookies don't change

This is the ultimate combination.

With the [API][api] available,
when no cookies are declared via the API
we only consider changes to secure cookies
as preventing BFCacheing.

This would happen some time after the [API][api] is stable.
It would need to be well announced ahead of time
and perhaps also use console warnings
or other prompts to raise awareness.

There are some risks from this proposal
but there are very large performance benefits from BFCache.
We believe there should be some time
and some effort to raise awareness
between making the [API][api] available and default BFCacheing of CCNS documents.
This will allow for more awareness
and early opt-in by use of the cookie-monitoring [API][api].

## Spec changes

The interaction between BFCache and CCNS is currently unspecced.
Allowing CCNS pages into BFCache requires no changes to spec.
However, since freely fallowing CCNS pages into BFCache
has known bad consequences it would be better to clarify the interaction.

## Risks and remediations

### Legacy Logins

Some legacy systems could use tokens in the URL instead of cookies,
even when cookies are not disabled.
This should be extremely rare,
if it exists at all.

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

Sites may use CCNS to ensure that fresh information is downloaded every time
(rather than to ensure that sensitive information is not retained).
Such sites would end up showing stale information
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
These sites can use the new [API][api]
and can also use any of the mitigitations for [stale information](#stale-information).

## Security considerations

This introduces no new surfaces or features beyond existing BFCacheing.

## Privacy considerations

The concerns are divided into

1. single-user devices where the user is surprised
   by seeing (again) information that is now inaccessible
2. shared devices where one user has taken steps (e.g. logout)
   to prevent access to sensitive content by later users

We believe, that for sites following good practices
we have covered these.

For sites not following good practices,
there are mitigations described above.
These all require action by the site.

It seems unlikely that we can find a way
to ensure that sites not following good practices
will not be impacted by this change.

# TAG Security and Privacy Questionnaire

01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

> No new information is exposed by this.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

> Yes.

03.  How do the features in your specification deal with personal information,
     personally-identifiable information (PII), or information derived from
     them?

> N/A.

04.  How do the features in your specification deal with sensitive information?

>

05.  Do the features in your specification introduce new state for an origin
     that persists across browsing sessions?

> No.

06.  Do the features in your specification expose information about the
     underlying platform to origins?

> No.

07.  Does this specification allow an origin to send data to the underlying
     platform?

> No.

08.  Do features in this specification enable access to device sensors?

> No.

09.  Do features in this specification enable new script execution/loading
     mechanisms?

> No.

10.  Do features in this specification allow an origin to access other devices?

> No.

11.  Do features in this specification allow an origin some measure of control over
     a user agent's native UI?

> No.

12.  What temporary identifiers do the features in this specification create or
     expose to the web?

> None.

13.  How does this specification distinguish between behavior in first-party and
     third-party contexts?

> N/A.

14.  How do the features in this specification work in the context of a browser’s
     Private Browsing or Incognito mode?

> No difference.

15.  Does this specification have both "Security Considerations" and "Privacy
     Considerations" sections?

> Yes.

16.  Do features in your specification enable origins to downgrade default
     security protections?

> No.

17.  How does your feature handle non-"fully active" documents?

> It is only concerned with these documents.

18.  What should this questionnaire have asked?

> .


[yahoo-japan-blog-post]: https://techblog.yahoo.co.jp/entry/2022010530253635/#:~:text=%E3%81%9D%E3%81%AE%E7%B5%90%E6%9E%9C%E3%80%81iOS%E3%81%AESafari%E3%81%A7%E3%81%AFPV/UB%E3%81%8C2%EF%BC%85%E4%BB%A5%E4%B8%8A%E5%90%91%E4%B8%8A%E3%81%99%E3%82%8B%E7%B5%90%E6%9E%9C%E3%81%8C%E5%BE%97%E3%82%89%E3%82%8C%E3%81%BE%E3%81%97%E3%81%9F%E3%80%82BFCache%E3%81%AF%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E3%83%90%E3%83%83%E3%82%AF%E6%99%82%E3%81%AE%E4%BD%93%E9%A8%93%E3%81%8C%E6%9C%80%E9%81%A9%E5%8C%96%E3%81%95%E3%82%8C%E3%82%8B%E3%81%9F%E3%82%81%E3%80%81PV/UB%E3%81%AE%E3%82%88%E3%81%86%E3%81%AA%E6%8C%87%E6%A8%99%E3%82%82%E3%83%9D%E3%82%B8%E3%83%86%E3%82%A3%E3%83%96%E3%81%AB%E5%8B%95%E3%81%84%E3%81%9F%E3%81%AE%E3%81%A0%E3%81%A8%E6%8E%A8%E6%B8%AC%E3%81%95%E3%82%8C%E3%81%BE%E3%81%99%E3%80%82
[page-show-event-handler]: https://web.dev/bfcache/#update-stale-or-sensitive-data-after-bfcache-restore
[webkit-experimented]: https://github.com/whatwg/html/issues/5744#issuecomment-661997090
[follow-on-proposal]: #follow-on-caching-ccns-documents-by-default

[api]: api.md
[authorization]: api.md#authorization
[storage-partitioning]: https://blog.chromium.org/2020/01/building-more-private-web-path-towards.html
