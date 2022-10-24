---
title: "A Preconnect Hint for SVCB/HTTPS RR"
category: exp

docname: draft-pardue-httpbis-preconnect-hint-svc-param-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "LPardue/draft-pardue-httpbis-related-origins"
  latest: "https://LPardue.github.io/draft-pardue-httpbis-preconnect-hint-svc-param/draft-pardue-httpbis-preconnect-hint-svc-param.html"

author:
 -
    fullname: Lucas Pardue
    email: lucaspardue.24.7@gmail.com

normative:

informative:


--- abstract

HTTP resources from one origin often have relationships to resources on other
origins. This document outlines how a "preconnectHint" SvcParamKey for SVCB,
could be used to provide an early indication of origins that are related to the
current origin. Clients could use this information to opportunistically prepare
and open connections, with the expectation that they will be used to fetch
related resources. This mechanism provides information from a source that is not
the authenticated origin, which could cause the client to perform actions that
no other party would ask them to do; privacy considerations due to this are
visited.


--- middle

# Introduction

HTTP resources from one origin often have relationships to resources on other
origins. For example, when a user agent loads the main HTML document of a
webpage from one one origin, it can discover critical dependencies at additional
first-party or third-party origins. In order to fetch these resources,
additional connections might be required. The additional time required to
establish new connections can inflate the perceived latency of rendering or
using resources.

In some cases, a server may be authoritative for several origins. HTTP/2
{{?RFC9113}} and HTTP/3 {{?RFC9114}} allow clients to coalesce different origins
onto the same connection when certain conditions are met. The ORIGIN frame
({{?RFC8336}} and {{?I-D.ietf-httpbis-origin-h3}}) enhances this further. These
techniques can help minimize perceived latency by eliminating connection setup
time entirely, but they apply only for the subset of origins that can safely be
coalesced.

The 103 (Early Hints) status code {{?RFC8297}} can convey hints about resource
relationships. A server can emit interim responses to help the client start
making preparations, such as creating new connections. This is especially useful
when the origin might take time to generate the final response. Where related
resources reside on third-party origins, this technique helps minimize perceived
latency by providing the client with information as early as possible in the
HTTP message exchange. Thus it fills a capability gap left by coalescing.
However, the technique is dependent on the timing of message exchanges, which
might might make it difficult to achieve performance gains in practice.

It is common for a web page to fetch resources from a fairly stable set of
additional origins, even if the specific resource paths are more volatile. For
example, a page may be designed to load scripts from third-party resource CDNs,
or images from content sub-domains under the same top-level origin. User agents
can only learn of these relationships once a resource has been fetched. This
"run-time learning" of stable relationships delays the time at which a client
discovers it might need to create additional connections. Between fetching a
resource and learning its dependencies, here is an opportunity to reduce
performance delays caused by the delayed run-time learning of these
relationships, even in light of the techniques described above.

This document defines the "preconnectHint" SvcParamKey for SVCB
{{!I-D.ietf-dnsop-svcb-https}}, to indicate origins that are related to the
current origin. Clients that query HTTPS RRs can use the information contained
in this parameter to opportunistically prepare and open connections, with the
expectation that they will be used to fetch related resources. This could
include chaining DNS queries for the hinted origins. Preconnect origins might
satisfy the conditions required for HTTP connection coalescing, in which case
the optimization could still reduce the delay before clients perform
coalescing-related checks; for example, certificate fetching or validation.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The preconnectHint SvcParamKey {#preconnectHint-param}

SVCB and HTTPS Resource Records can include the `preconnectHint` SvcParamKey to
indicate related origins. Its presentation format value is a comma-separated
list of `<domain-name>` ({{Section 5.1 of !RFC1035}}).

For example, consider the resource at `https://example.org/index.html` that has
the dependencies `https://www.example.com/style.css`, and
`https://scripts.example.org/script.js`. As an
HTTPS RR, the relationship between example.org and other origins could be
represented as:

~~~
example.org IN HTTPS 1 . alpn=h2,h3 preconnectHint=www.example.com,scripts.example.org
~~~

A common pattern in HTTP is to use 3xx redirect status codes to move from `www`
subdomains to domain apexes. Building on the previous example, if
`www.example.com` were intended to be redirected to `example.com`, this could be
represented in an additional RR as:

~~~
www.example.com IN HTTPS 1 . alpn=h2,h2 preconnectHint=example.com
~~~

Clients that learn about preconnect origins MAY use this information to make
whatever preparations they deem fit. For instance, they could use it to perform
the same connection coalescing authority checks ({{RFC9113}} and {{RFC9114}})
that would otherwise happen later in a connection lifecycle. Similarly, they
could use this information to help maintain a connection pool and, where needed,
proactively create or keep open connections to those origins in anticipation of
being used.

There are new privacy considerations to make when using any preconnect origin
information provided via the DNS. The records are presented by resolvers that
act on behalf of, but are not authoritative for, the origin. As such, the
resolvers could present unauthenticated information that could cause a client to
take actions that the authenticated origin would not. This could be abused to
leak information about the client. A malicious resolver could also abuse this
mechanism unbeknownst to the origin.

The `preconnectHint` SvcParamKey is a hint and could contain stale, incorrect or
superfluous information. A client SHOULD implement checks and heuristics that
limit state or resource commitment based on this information. For example, a
client could track how often preconnect origins are matched to related
resources. Notably, the scope of relationships is at the origin level, not any
other component that might later comprise a URI that is to be fetched. As such,
constraining the set of values in the `preconnectHint` parameter, to those that
are most likely to be used by a client, can help avoid commitment of resources
that might subsequently go unused.

Knowledge of HTTP resource relationships might be restricted to authorized
clients. Exposing those origins as a `preconnectHint` to unauthorised DNS
clients could leak confidential or sensitive information. Therefore, the
`preconnectHint` SvcParmKey SHOULD NOT contain origins that relate to
information that would otherwise only be accessible to authorized clients.

# Security and Privacy Considerations

Information about origin relationships is typically presented by the
authenticated origin itself. Delegating this information to an unauthenticated
and untrusted DNS resolver provides opportunities to manipulate client
behaviour, which could risk privacy problems; see {{preconnectHint-param}}.

The preconnectHint parameter reveals information about the relationships
of resources hosted on a server. While this information is typically already
available to any client that visits the server, some resources may only be
discoverable by authorized clients. Guidance for managing this is given in
{{preconnectHint-param}}.

# IANA Considerations

TBD


--- back

# Acknowledgments
{:numbered="false"}

This document is based on one of the options described by Barry Pollard's ["Even
earlier connection
hints"](https://docs.google.com/document/d/1ApILvaFpZPGx6NkqhPvlqni6kTaWfcY6xSEuXg8XdiA/edit#heading=h.fxvrme9c9xpm)
proposal presented to the Web Perf WG at TPAC 2022.

Ben Schwartz suggested the name `preconnectHint`.

Thanks to Chris Wood for a providing insight into the privacy implications of
this design.
