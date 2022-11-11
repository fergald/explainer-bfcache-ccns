# Proposal: give sites the ability to evict documents from BFCache when cookies or storage keys change

## Context

This proposal is a piece of a larger proposal
to safely enable BFCaching of pages
with the `Cache-Control: no-store` (_CCNS_) header.
For full details on that,
see the accompanying [explainer][ccns-explainer].

## Overview

We propose to add an API that allows a document to declare
that it must not be restored from BFCache
if certain cookies or storage keys change while it is in BFCache.

For example, the following JS snippet
will cause any documents from this document's origin
which are currently in BFCache,
to be evicted if the 'SID' cookie changes,

```js
backForwardCacheController.evictionSignals.cookies = ['SID'];
```

Similarly, the following JS snippet
will cause any documents from this document's origin
which are currently in BFCache,
to be evicted
if the value of the key 'authToken' in session storage changes,

```js
backForwardCacheController.evictionSignals.sessionStorage = ['authToken'];
```

## Background

### Authentication

#### Secure cookies

Best practice for authentication has been,
for a long time,
to hold a token in an HTTPS-only cookie
so that all requests present this token.
Deleting this cookie deauthenticates the browser
and usually corresponds to a logout.
The same outcome applies when the cookie expires.
Replacing the cookie with a new value
might also correspond to a change in authentication state
or it may just be a refresh of the token
or other neutral change.

Conservatively, we can consider any change
to be a significant authentication change
even if it does not correspond to a logout.

#### Other cookies

Other non-auth or non-HTTPS-only cookies may also be significant to a page
e.g. representing contents of shopping carts.
Changes to these may also imply that pages in BFCache
have outdated/incorrect contents.

#### `Authentication` header and tokens

Authentication may also happen via 3rd parties,
resulting in tokens that are presented
via the `Authentication` header.
For example, federated sign-in
(sign-in with Google/Twitter/Facebook/etc)
provides a token that can then be presented on later RPCs.
There are sites that use this as their only form of authentication
(with no use of cookies).
Typically they are single-page apps since,
without cookies,
HTTP requests resulting from navigation
would be unauthenticated.

Best practice for tokens like this
is to write them to site-private storage,
e.g. LocalStorage, SessionStorage, IndexedDB.
The token is read from storage when attaching to each RPC.
This ensures that a logout occurring in another tab
or in the same tab but in unrelated code
will result in fully deauthenticating the user.

Any use of the `Authentication` header
indicates that the page is now probably displaying information
that could only be acquired with authentication.

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

Allow sites to declare that certain state changes
in cookies or storage
should cause the site's documents to be evicted from BFCache.

When authentication state changes,
a site using cookies for authentication
will see a change in cookies.
A site using the `Authentication` header
should see a change in stored token(s).

Evicting when cookies or storage changes strikes a balance
between impact and ergonomics for site developers.
Declaring a set of relevant cookies or storage keys is low overhead.
It ensures that documents are not restored from BFCache
after authentication state changes.

Evicting when cookies change
causes a relatively low rate of eviction.
Chrome's analysis shows that fewer than 50% of documents
currently blocked from BFCache by the CCNS header
have any cookie change within 3.5 minutes of entering BFCache.
Fewer than 5% have any [secure cookie][secure-cookie] change
in that time.

We do not have any statistics
on the frequency of changes to stored authentication tokens
since we cannot know which stored items
are authentication tokens.
We will measure how often the `Authentication` header
is seen on pages with CCNS
and what impact it would have
if we were to continue to block cacheing
of CCNS pages
when we see that header used.

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

The API presents a `inactiveDocumentController.invalidationSignals` object
which has fields for registering cookies and other storage mechanisms:

- cookies
- SessionStorage
- LocalStorage
- IndexedDB

For cookies, SessionStorage and LocalStorage,
items to monitor for changes
are identified by a single key.
For IndexedDB, a pair of database name and store name
identify the item to be monitored.
It is unclear if this is sufficient for all uses of IndexedDB for token storage.
It assumes that tokens will be stored in a store dedicated to token storage.
If the store is also used for other data
then we will over-evict.

For example, the following JS snippet
will cause any documents from this document's origin
to be evicted if the SID cookie changes
or if there is a change to the 'tokens' store
in the 'auth' database of IndexedDB.

```js
backForwardCacheController.evictionSignals.cookies = ['SID'];
backForwardCacheController.evictionSignals.indexedDB = [['auth', 'tokens']];
```

### Details

When `backForwardCacheController.evictionSignals.[some field]` is set to a list

* if the document enters BFCache,
  the browser must monitor the listed cookies or keys
  (the list may be empty)
  and if any change occurs in those cookies,
  (modify/delete/expire),
  or storage keys (delete/set),
  the document should be evicted from BFCache.

When `backForwardCacheController.evictionSignals.cookies` is left unset
or set to `null` or `undefined`.

* no cookie-based eviction should occur.
* the browser can consider the API to _have not_ been used for cookies.

When the `localStorage`, `sessionStorage` and `indexedDB` fields
have all been left unset or set to `null` or `undefined`.

* no storage-based eviction should occur.
* the browser can consider the API to _have not_ been used for tokens.

### API Goals

While the API is a straw person, there are some specific goals:

* list 1 or more cookies/keys
* distinguish between the API having been used or not
* allow for an explicit signal that eviction should not be triggered
  just because cookies/keys change

### API Non-goals

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
backForwardCacheController.evictionSignals.cookies = ['SID'];
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
backForwardCacheController.evictionSignals.cookies = null;
```

#### **Scenario: Explicitly monitor no cookies**

A document wants to signal
that it can be BFCached regardless of cookie changes.
This use case makes sense
in the context of the [broader proposal][ccns-explainer]
to allow BFCaching of documents with the CCNS header.

```js
backForwardCacheController.evictionSignals.cookies = [];
```

#### **Scenario: Programmatic flushing of BFCache**

This may be considered an abuse of the API
but absent another API to more directly provide this,
we may see sites setting a cookie or other item
that serves no other purpose than to be a sentinel,
which, when altered,
causes BFCached documents from the site to be evicted.

#### **Possible Scenario: Match a class of cookies**

The API does not support this
and it's unclear that it's needed
(certainly in V1)
but a site may want to say something like,
"evict if any HTTPS-only cookies change"
or "evict if any cookies on this path change".

Until there is a clear use case,
this will remain just a note on a possible extension of the API.

## Considered alternatives

### Header

Sites could also specify the same information in an HTTP header.

The choice to use a JS API rather than a header is currently somewhat arbitrary.

* JS seems more flexible
  and avoids parsing and serialisation concerns if we make changes in the future.
* A header centralises control
  and prevents random JS libraries from altering BFCache+CCNS behaviour

It may be that we would want both.
We will spend more time on this after validating the basic idea.

### Cookie Metadata

Instead of requiring pages to list the cookies which matter,
cookies could declare themselves to be significant.

This would mean that all pages of the site would be evicted
when the cookies changes
without any code changes.
This is probably overly broad
but it makes it impossible to have a problem
whereby a page is forgotten.

### Manual Clearing

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

## Security considerations

This just adds another way for a page to be evicted from BFCache.
It's conceivable that a.com could decrease b.com's BFCache hit rate,
by causing b.com's cookies to change more frequently than normal.
E.g. by repeatedly navigating an iframe to a URL of b.com's
that changes cookies even without any user-interaction
(e.g. /logout).
b.com can mitigate this attack by
- listing specific cookies (as recommended)
- preventing embedding of URLs that change cookies without interaction
  (already a good idea)

## Privacy considerations

This gives no new information to sites.
If a page is evicted due to this API
it means that a cookie for that site has been deleted/altered/expired.
It cannot discover that this happens
until the user returns to that page in a history traversal.
At that point the site can discover the state of all of its cookies.

## TAG Security and Privacy Questionnaire

01.  What information might this feature expose to Web sites or other parties,
     and for what purposes is that exposure necessary?

> None.

02.  Do features in your specification expose the minimum amount of information
     necessary to enable their intended uses?

> Yes.

03.  How do the features in your specification deal with personal information,
     personally-identifiable information (PII), or information derived from
     them?

> N/A.

04.  How do the features in your specification deal with sensitive information?

> N/A.

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


[ccns-explainer]: README.md
[ccns-advice]: https://codingshower.com/disable-bfcache/
[secure-cookie]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#restrict_access_to_cookies
