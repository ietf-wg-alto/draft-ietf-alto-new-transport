---
docname: draft-ietf-alto-new-transport-latest
title: "The ALTO Transport Information Publication Service"
abbrev: "ALTO TIPS"
category: std
date: {DATE}

ipr: trust200902
area: Transport Area
workgroup: ALTO

stand_alone: yes
pi:
  strict: yes
  comments: yes
  inline: yes
  editing: no
  toc: yes
  tocompact: yes
  tocdepth: 3
  iprnotified: no
  sortrefs: yes
  symrefs: yes
  compact: yes
  subcompact: no

author:
  -
    ins: R. Schott
    name: Roland Schott
    street: Ida-Rhodes-Straße 2
    city: Darmstadt
    code: 64295
    country: Germany
    org: Deutsche Telekom
    email: Roland.Schott@telekom.de

  -
    ins: Y. R. Yang
    name: Yang Richard Yang
    street: 51 Prospect Street
    city: New Haven
    code: CT
    country: USA
    org: Yale University
    email: yry@cs.yale.edu

  -
    ins: K. Gao
    name: Kai Gao
    street: "No.24 South Section 1, Yihuan Road"
    city: Chengdu
    code: 610000
    country: China
    org: Sichuan University
    email: kaigao@scu.edu.cn

  -
    ins: L. Delwiche
    name: Lauren Delwiche
    street: 51 Prospect Street
    city: New Haven
    code: 06520
    country: USA
    org: Yale University
    email: lauren.delwiche@yale.edu

  -
    ins: L. Keller
    name: Lachlan Keller
    street: 51 Prospect Street
    city: New Haven
    code: 06520
    country: USA
    org: Yale University
    email: lachlan.keller@yale.edu

normative:
  RFC2119:
  RFC7285:
  RFC8259:
  RFC8174:
  RFC8895:
  RFC9112:
  RFC9113:
  RFC9114:

informative:
  RFC9205:
  IANA-Media-Type:
    title: Media Types
    target: https://www.iana.org/assignments/media-types/media-types.xhtml
    date: 2023-06

--- abstract

The ALTO Protocol (RFC 7285) leverages HTTP/1.1 and is designed for the simple,
sequential request-reply use case, in which an ALTO client requests a
sequence of information resources, and the server responds with the complete
content of each resource one at a time.

ALTO incremental updates using Server-Sent Events (SSE) (RFC 8895) defines a
multiplexing protocol on top of HTTP/1.x, so that an ALTO server can
incrementally push resource updates to clients whenever monitored network
information resources change, allowing the clients to monitor multiple resources
at the same time. However, HTTP/2 and later versions already support concurrent,
non-blocking transport of multiple streams in the same HTTP connection.

To take advantage of newer HTTP features, this document introduces the ALTO
Transport Information Publication Service (TIPS). TIPS uses an incremental
RESTful design to give an ALTO client the new capability to explicitly,
concurrently (non-blocking) request (pull) specific incremental updates using
native HTTP/2 or HTTP/3, while still functioning for HTTP/1.1.

--- middle

# Introduction {#intro}

Application-Layer Traffic Optimization (ALTO) provides means for network
applications to obtain network status information. So far, two transport
protocols have been designed:

1. The ALTO base protocol {{RFC7285}}, which is designed for the simple use case
   in which an ALTO client requests a network information resource, and the
   server sends the complete content of the requested information (if any)
   resource to the client.

2. ALTO incremental updates using Server-Sent Events (ALTO/SSE) {{RFC8895}},
   which is designed for an ALTO client to indicate to the server that it wants
   to receive updates for a set of resources, and the server can then
   concurrently, and incrementally push updates to that client whenever
   monitored resources change.

Both protocols are designed for HTTP/1.1 {{RFC9112}}, but HTTP/2 {{RFC9113}}
and HTTP/3 {{RFC9114}}
can support HTTP/1.1 workflows. However, HTTP/2 and HTTP/3 provide features that
can improve on certain properties of ALTO and ALTO/SSE.

- First, consider the ALTO base protocol, which is designed to transfer only
  complete information resources. A client can run the base protocol on top of
  HTTP/2 or HTTP/3 to request multiple information resources in concurrent
  streams, but each request must be for a complete information resource: there is
  no capability it transmit incremental updates. Hence, there can be large
  overhead when the client already has an information resource and then there are
  small changes to the resource.

- Next, consider ALTO/SSE {{RFC8895}}. Although ALTO/SSE can transfer
  incremental updates, it introduces a customized multiplexing protocol on top
  of HTTP, assuming a total-order message channel from the server to the client.
  The multiplexing design does not provide naming (i.e., a resource identifier)
  to individual incremental updates. Such a design cannot use concurrent data
  streams available in HTTP/2 and HTTP/3, because both cases require a resource
  identifier. Additionally, ALTO/SSE is a push-only protocol, which denies the
  client flexibility in choosing how and when it receives updates.

To mitigate these concerns, this document introduces a new ALTO service, called
the Transport Information Publication Service (TIPS). TIPS uses an incremental
RESTful design to provide an ALTO client with a new capability to explicitly,
concurrently issue non-blocking requests for specific incremental updates using
native HTTP/2 or HTTP/3, while still functioning for HTTP/1.1.

Despite the benefits, however, ALTO/SSE {{RFC8895}}, which solves a similar
problem, has its own advantagess. First, SSE is a mature technique with a
well-established ecosystem that can simplify development. Second, SSE
does not require multiple connections to receive updates for multiple objects
over HTTP/1.1.

HTTP/2 {{RFC9113}} and HTTP/3 {{RFC9114}}
also specify server push, which might enhance TIPS. While push
is currently not widely implemented, we provide a non-normative
specification of push-mode TIPS as an alternative design in an appendix.

Specifically, this document specifies:

-  Extensions to the ALTO Protocol for dynamic subscription and efficient
   uniform update delivery of an incrementally changing network information
   resource.

-  A new resource type that indicates the TIPS updates graph model for a
   resource.

-  URI patterns to fetch the snapshots or incremental updates.

{{sec-bcp-http}} discusses to what extent the TIPS design adheres to the Best
Current Practices for building protocols with HTTP {{RFC9205}}.

## Requirements Language

{::boilerplate bcp14-tagged}

## Notations

This document uses the same syntax and notations as introduced in
{{Section 8.2 of RFC7285}} to specify the extensions to existing ALTO resources and services.

# TIPS Overview {#overview}

## Transport Requirements {#requirements}

The ALTO Protocol and its extensions support two transport mechanisms:
First, a client can directly request an ALTO resource and obtain a complete
snapshot of that ALTO resource, as specified in the base protocol {{RFC7285}};
Second, a client can subscribe to incremental changes of one or multiple ALTO
resources using the incremental update extension {{RFC8895}}, and a server pushes
the updates to the client through Server Sent Events (SSE).

However, the current transport mechanisms are not optimized for storing,
transmitting, and processing (incremental) updates of ALTO information
resources. Specifically, the new transport mechanism must satisfy the following
requirements:

Incremental updates:
: Incremental updates can reduce both the data storage on an ALTO server and the
  transmission time of the updates, especially when the change of an ALTO
  resource is minor. The base protocol does not support incremental updates and
  the current incremental update mechanism in {{RFC8895}} has limitations (as
  discussed below).

Concurrent, non-blocking update transmission:
: When a client needs to receive and apply multiple incremental updates, it is
  desired to transmit the updates concurrently to fully utilize the bandwidth
  and to reduce head-of-line blocking. The ALTO incremental update extension
  {{RFC8895}}, unfortunately, does not satisfy this requirement -- even though
  the updates can be multiplexed by the server to avoid head-of-line blocking
  between multiple resources, the updates are delivered sequentially and can
  suffer from head-of-line blocking inside the connection, for example, when
  there is a packet loss.

Prefetching updates:
: Prefetching updates can reduce the time to send the request, making it
  possible to achieve sub-RTT transmission of ALTO incremental updates. In
  {{RFC8895}}, this requirement is fulfilled using server-sent event (SSE) and
  is still desired in the ALTO new transport.

Backward compatibility:
: While some of the previous requirements are offered by HTTP/2 {{RFC9113}} and
  HTTP/3 {{RFC9114}}, it is desired that the ALTO new transport mechanism can
  work with HTTP/1.1 as many development tools and current ALTO implementations
  are based on HTTP/1.1.

The ALTO new transport specified in this document satisfies all the design
requirements and hence improves the efficiency of continuous dissemination of
ALTO information. The key idea is to introduce a unified data model to describe
the changes (snapshots and incremental updates) of an ALTO resource, referred to
as a TIPS view. Along with the data model, this document also specifies a
unified naming for the snapshots and incremental updates, independent of the
HTTP version. Thus, these updates can be concurrently requested. Prefetching is
realized using long polling.

This document assumes the deployment model discussed in  {{sec-dep-model}}.

## TIPS Terminology {#terminology}

In addition to the terms defined in {{RFC7285}}, this document uses the following terms:

Transport Information Publication Service (TIPS):
: Is a new type of ALTO service, as specified in this document, to enable a
  uniform transport mechanism for updates of an incrementally changing ALTO
  network information resource.

Network information resource:
: Is a piece of retrievable information about network state, per {{RFC7285}}.

TIPS view (tv):
: Is defined in this document to be the container of incremental transport
  information about the network information resource. Though the TIPS view may
  include other transport information, it has two basic components: updates
  graph (ug) and receiver set (rs).

Updates graph (ug):
: Is a directed, acyclic graph whose nodes represent the set of versions of an
  information resource, and edges the set of update items to compute these
  versions. An ALTO map service (e.g., Cost Map, Network Map) may need only a
  single updates graph. A dynamic network information service (e.g., Filtered
  Cost Map) may create an updates graph (within a new TIPS view) for each unique
  request.

Version:
: Represents a historical content of an information resource. For an information
  resource, each version is associated with and uniquely identified by a
  monotonically and consecutively increased sequence number. We use the term
  "version s" to refer to the version associated with sequence number s.

Start sequence number (start-seq):
: Is the smallest non-zero sequence number in an updates graph.

End sequence number (end-seq):
: Is the largest sequence number in an updates graph.

Snapshot:
: Is a full replacement of a resource and is contained within an updates graph.

Incremental update:
: Is a partial replacement of a resource contained within an updates graph,
  codified in this document as a JSON Merge Patch or JSON Patch. An incremental
  update is mandatory if the source version (i) and target version (j) are
  consecutive, i.e., i + 1 = j, and optional or a shortcut otherwise. Mandatory
  incremental updates are always in an updates graph, while optional/shortcut
  incremental updates may or may not be included in an updates graph.

Update item:
: Refers to the content on an edge of the updates graph, which can be either a
  snapshot or incremental update. An update item can be considered as a pair
  (op, data) where op denotes whether the item is an incremental update or a
  snapshot, and data is the content of the item.

ID#i-#j:
: Denotes the update item on a specific edge in the updates graph to transition
  from version i to version j, where i and j are the sequence numbers of the
  source node and the target node of the edge, respectively.

Receiver set (rs):
: Contains the set of clients who have requested to receive server push updates.
  This term is not used in the normative specification.


~~~~ drawing
                                   +-------------+
    +-----------+ +--------------+ |  Dynamic    | +-----------+
    |  Routing  | | Provisioning | |  Network    | | External  |
    | Protocols | |    Policy    | | Information | | Interface |
    +-----------+ +--------------+ +-------------+ +-----------+
          |              |                |              |
+----------------------------------------------------------------------+
| ALTO Server                                                          |
| +------------------------------------------------------------------+ |
| |                                              Network Information | |
| | +-------------+                         +-------------+          | |
| | | Information |                         | Information |          | |
| | | Resource #1 |                         | Resource #2 |          | |
| | +-------------+                         +-------------+          | |
| +-----|--------------------------------------/-------\-------------+ |
|       |                                     /         \              |
| +-----|------------------------------------/-----------\-----------+ |
| |     |       Transport Information       /             \          | |
| | +--------+                     +--------+        +--------+      | |
| | |  tv1   |----+          +-----|  tv2   |        |  tv3   |---+  | |
| | +--------+    |          |     +--------+        +--------+   |  | |
| |     |         |          |           |             |          |  | |
| | +--------+ +--------+ +--------+ +--------+ +--------+ +--------+| |
| | | tv1/ug | | tv1/rs | | tv2/ug | | tv2/rs | | tv3/ug | | tv3/rs || |
| | +--------+ +--------+ +--------+ +--------+ +--------+ +--------+| |
| +----|\---------/\---------|---------/---------------|-------------+ |
|      | \       /  \        |        /                |               |
+------|--\-----/----\-------|-------/-----------------|---------------+
       |   \   /      \      |      /                  |
       |    +-/-----+  \     |     /                   |
       |     /       \  \    |    /  A single          |   A single
     ==|====/==     ==\==\===|===/== HTTP/2 or /3    ==|== HTTP/1.1
       |   /           \  \  |  /    connection        |   connection
   +----------+       +----------+                 +----------+
   | Client 1 |       | Client 2 |                 | Client 3 |
   +----------+       +----------+                 +----------+

tvi   = TIPS view i
tvi/ug = incremental updates graph associated with tvi
tvi/rs = receiver set of tvi (for server push)
~~~~
{: #fig-overview artwork-align="center" title="Overview of ALTO TIPS"}

{{fig-overview}} shows an example illustrating an overview of the ALTO TIPS
service. The server provides the TIPS service of two information resources (#1
and #2) where we assume #1 is an ALTO map service, and #2 is a filterable
service. There are 3 ALTO clients (Client 1, Client 2, and Client 3) that are
connected to the ALTO server. Each client maintains a single HTTP connection
with the ALTO server and uses the TIPS view to retrieve updates (see the
arguments in {{single-http}}). Specifically, a TIPS view (tv1) is created for
the map service #1, and is shared by multiple clients. For the filtering service
#2, two different TIPS view (tv2 and tv3) are created upon different client
requests with different filter sets.

# TIPS Updates Graph

In order to provide incremental updates for a resource, an ALTO server creates
an updates graph, which is a directed, acyclic graph that contains a sequence of
incremental updates and snapshots (collectively called update items) of a
network information resource.

## Basic Data Model of Updates Graph {#data-model}

For each resource (e.g., a cost map, a network map), the incremental updates and
snapshots can be represented using the following directed acyclic graph model,
where the server tracks the change of the resource maps with version IDs that are
assigned sequentially (i.e., incremented by 1 each time):

-  Each node in the graph is a version of the resource, where a tag identifies
   content of the version (tag is valid only within the scope of resource).
   Version 0 is reserved as the initial state (empty/null).

-  Each edge is an update item. In particular, edge from i to j is the update
   item to transit from version i to version j.

-  Version is path independent (different paths arrive at the same version/node
   has the same content)

A concrete example is as shown in {{fig-ug}}. There are 7 nodes in the graph,
representing 7 different versions of the resource. Edges in the figure represent
the updates from the source version to the target version. Thick lines represent
mandatory incremental updates (e.g., ID103-104), dotted lines represent optional
incremental updates (e.g., ID103-105), and thin lines represent snapshots (e.g.,
ID0-103). Note that node content is path independent: the content of node v can
be obtained by applying the updates from any path that ends at v. For example,
assume the latest version is 105 and a client already has version 103. We say
the base version of the client is 103 as it serves as a base upon which
incremental updates can be applied. The target version 105 can either be
directly fetched as a snapshot, computed incrementally by applying the
incremental updates between 103 and 104, then 104 and 105, or if the optional
update from 103 to 105 exists, computed incrementally by taking the "shortcut"
path from 103 to 105.

~~~~ drawing
                                                        +======+
                                                  ------|  0   |
                                                 /      +======+
                                        ID0-101 /        |   |
                                              |/__       |   |
                                       +======+          |   |
                       tag: 3421097 -> | 101  |          |   |
                                       +======+          |   |
                               ID101-102  ||             |   |
                                          \/             |   |
                                       +======+          |   |
                       tag: 6431234 -> | 102  |          |   |
                                       +======+          |   |
                               ID102-103  ||             |   |
                                          \/             |   |
                                       +======+          /   |
    +--------------+   tag: 0881080 -> | 103  |<--------/    |
    | Base Version |   =======>        +======+ ID0-103      |
    +--------------+             103-104  ||    ..           |
                                          \/     ..          |
                                       +======+  ..          |
                       tag: 6452654 -> | 104  |  .. ID103    |
                                       +======+  .. -105     |
                               ID104-105  ||     ..          | ID0-105
                                          \/   |._           /
                                       +======+             /
                       tag: 7838392 -> | 105  |<-----------/
                                       +======+
                               ID105-106  ||
                                          \/
                                       +======+
                       tag: 6470983 -> | 106  |
                                       +======+
~~~~
{: #fig-ug artwork-align="center" title="TIPS Model Example"}

## Resource Location Schema {#schema}

Update items are exposed as HTTP resources and the URLs of these items, which we
call resource location schema, follow specific patterns. To access each
individual update in an updates graph, consider the model represented as a
"virtual" file system (adjacency list), contained within the root of a TIPS view
URI (see {{open-resp}} for the definition of tips-view-uri). For example,
assuming that the update graph of a TIPS view is as shown in {{fig-ug}}, the
location schema of this TIPS view will have the format as in {{fig-ug-schema}}.

~~~~ drawing
  <tips-view-uri>  // relative URI to a TIPS view
    |_ ug    // updates graph
    |  |_ 0
    |  |  |_ 101    // full 101 snapshot
    |  |  |_ 103
    |  |  \_ 105
    |  |_ 101
    |  |  \_ 102    // 101 -> 102 incremental update
    |  |_ 102
    |  |  \_ 103
    |  |_ 103
    |  |  |_ 104
    |  |  \_ 105    // optional shortcut 103 -> 105 incr. update
    |  |_ 104
    |  |  \_ 105
    |  \_ 105
    |     \_ 106
    \_ meta         // TIPS view meta
       \_ ...
~~~~
{: #fig-ug-schema artwork-align="center" title="Location Schema Example"}

TIPS uses this directory schema to generate template URIs which allow
clients to construct the location of incremental updates after receiving the
tips-view-uri path from the server. The generic template for the location of the
update item on the edge from node 'i' to node 'j' in the updates graph is:

~~~
    <tips-view-uri>/ug/<i>/<j>
~~~

Due to the sequential nature of the update item IDs, a client can long poll a
future update that does not yet exist (e.g., the incremental update from 106 to
107) by constructing the URI for the next edge that will be added, starting from
the sequence number of the current last node (denoted as end-seq) in the graph
to the next sequential node (with the sequence number of end-seq + 1):

~~~
    GET /<tips-view-uri>/ug/<end-seq>/<end-seq + 1>
~~~

## Updates Graph Modification Invariants

A server may change its updates graph (to compact, to add nodes,
etc.), but it must ensure that any resource state that it makes
available is reachable by clients, either directly via a snapshot
(that is, relative to 0) or indirectly by requesting an earlier
snapshot and a contiguous set of incremental updates.  Additionally,
to allow clients to proactively construct URIs for future update
items, the ID of each added node in the updates graph must increment
contiguously by 1.  More specifically, the updates graph MUST satisfy
the following invariants:

-  Continuity: At any time, let ns denote the smallest non-zero version (i.e.,
   start-seq) in the update graph and ne denote the latest version (i.e.,
   end-seq). Then any version in between ns and ne must also exist. This implies
   that the incremental update from ni to ni + 1 exists for any ns <= ni <= ne,
   and all versions in the update graph (except 0) is an integer interval
   `[ns, ne]`.

-  Feasibility: Let ns denote the start-seq in the update graph. The server must
   provide a snapshot of ns and, in other words, there is always a direct link
   to ns in the update graph.

-  "Right shift" only: Assume a server provides versions in `[n1, n2]` at time t
   and versions in `[n1', n2']` at time t'. If t' > t, then n1' >= n1 and n2' >=
   n2.

For example, consider the case that a server compacts a resource's updates graph
to conserve space, using the example model in {{data-model}}. Assume at time 0,
the server provides the versions `{101, 102, 103, 104, 105, 106}`. At time 1,
both `{103, 104, 105, 106}` and `{105, 106}` are valid sets. However, `{102,
103, 104, 105, 106}` and `{104, 105, 106}` are not valid sets as there is no
snapshot to version 102 or 104 in the update graph. Thus, there is a risk that
the right content of version 102 (in the first example) or 104 (in the second
example) cannot be obtained by a client that does not have the previous version
101 or 103, respectively.

# TIPS High Level Workflow {#workflow}

## Workflow Overview

There are two ways a client can receive updates for a resource:

1.  Client Pull;

2.  Server Push.

At a high level, an ALTO client first uses the TIPS service to indicate the
information resource(s) that the client wants to monitor. For each requested
resource, the server returns a JSON object that contains a URI, which points to
the root of a TIPS view, and a summary of the current view, which contains, at
the minimum, the start-seq and end-seq of the update graph and a
server-recommended edge to consume first.

For client pull, the TIPS view summary provides enough information for the
client to continuously pull each additional update, following the workflow in
{{fig-workflow-pull}}. Detailed specification of this mode is given in {{pull}}.
Note that in {{fig-workflow-pull}}, the update item at
`/<tips-view-uri1>/ug/<j>/<j+1>` may not yet exist, so the server holds the
request until the update becomes available (long polling).

~~~~ drawing
Client                                  TIPS
  o                                       .
  | Open persistent HTTP connection       .
  |-------------------------------------->|
  |                                       .
  | POST to create/receive a TIPS view    .
  |           for resource 1              .
  | ------------------------------------> |
  | <tips-view-uri1>, <tips-view-summary> .
  |<------------------------------------- |
  |                                       .
  | GET /<tips-view-uri1>/ug/<i>/<j>      .
  | --------------------------------------|
  | content on edge i to j                .
  |<--------------------------------------|
  |                                       .
  | GET /<tips-view-uri1>/ug/<j>/<j+1>    .
  | ------------------------------------->|
  |                                       .
  |                                       .
  | content on edge j to j+1              .
  |<--------------------------------------|
  |                                       .
  | DELETE TIPS view for resource 1       .
  |-------------------------------------> |
  |                                       .
  | Close HTTP connection                 .
  |-------------------------------------->|
  o
~~~~
{: #fig-workflow-pull artwork-align="center" title="ALTO TIPS Workflow Supporting Client Pull"}

## TIPS over a Single HTTP Connection {#single-http}

A key requirement in the current new transport extension is that a client must
interact with the ALTO server using a single persistent HTTP connection, and the
life cycle of the TIPS views are bounded to that specific connection. This
design is due to the following reasons:

The first reason is to reduce the management complexity in modern server
deployment technologies. As microservices are becoming the new trend of web
development, requests to the same service are load balanced to different
instances, even between the same source and destination addresses. However, TIPS
views are stateful information which depends on the client's input. If requests
from the same client session can be directed to different instances, the
operator of the ALTO server must implement complex mapping management or load
balancing mechanisms to make sure the requests arrive at the same server.

The second reason is to simplify the state management of a single session. If
multiple connections are associated with a single session, implementations of
ALTO servers and clients must manage the state of the connections, which
increases the complexity of both ALTO servers and clients.

Third, single persistent HTTP connection offers an implicit way of life cycle
management of TIPS views, which can be resource-consuming. Malicious users may
create TIPS views and then disconnect, to get around the limits on concurrent
TIPS views, if not implemented correctly by an ALTO server. Leaving the TIPS
views alive after the HTTP connection is closed or timed out also makes session
management complex: When a client reconnects, should it try to access the TIPS
view before the disconnection or simply start a new session? Whether and when
can the server remove the TIPS views? In the current extension, the idea is to
avoid such complexity and enforce the consensus that a session will be
automatically closed once the connection is closed or timed out.

## TIPS with Different HTTP Versions

The HTTP version of an "https" connection is negotiated between client and
server using the TLS ALPN extension, as specified in Section 3.1 of {{RFC9113}}
for HTTP/2 and Section 3.1 of {{RFC9114}} for HTTP/3. For an "http" connection,
the explicit announcement of HTTP/2 or HTTP/3 support by the server is outside
the scope of this document.

While TIPS is designed to take advantage of newer HTTP features like server push
and substreams for concurrent fetch, TIPS still functions with HTTP/1.1 for
client poll defined in {{pull}}, with the limitation that it cannot cancel any
outstanding requests or fetch resources concurrently over the same connection
due to the blocking nature of HTTP/1.1 requests. If a client only capable of
HTTP/1.1 desires to concurrently monitor multiple resources at the same time, it
should open multiple connections, one for each resource, so that an outstanding
long-poll request can be issued for each resource to monitor for new updates.
For HTTP/2 and /3, with multiplexed streams, multiple resources can be monitored
simultaneously.

## TIPS Sequence Number Management

Conceptually, the sequence number space of TIPS views satisfy the following
requirements: First, a specific client may expect to see the same
sequence number for the same version to avoid occasional disconnections.
Second, the coupling of sequence numbers should be minimized for ALTO resources
monitored by multiple clients.

Thus, the sequence number space is constant for each TIPS view per-client but is
independent across TIPS views. Note that

1. For the same TIPS resource queried by different clients, multiple TIPS views
   will be created, one for each client, whose sequence number spaces are
   independent.

2. Independence implies that ALTO clients must not assume there is a consensus of
   how different versions map to sequence numbers but does not force the
   sequence numbers to be different. See {{shared-tips-view}} for cases where an
   ALTO server may desire to use the same sequence number space across TIPS
   views.

# TIPS Information Resource Directory (IRD) Announcement {#ird}

To announce a TIPS information resource in the information resource directory
(IRD), an ALTO server MUST specify the "media-type", "capabilities" and "uses"
as follows.

## Media Type

The media type of the Transport Information Publication Service resource is
"application/alto-tips+json".

## Capabilities {#caps}

The capabilities field of TIPS is modeled on that defined in
Section 6.3 of {{RFC8895}}.

Specifically, the capabilities are defined as an object of type
TIPSCapabilities:

~~~
     object {
       IncrementalUpdateMediaTypes incremental-change-media-types;
     } TIPSCapabilities;

     object-map {
        ResourceID -> String;
     } IncrementalUpdateMediaTypes;
~~~
{: #tips-cap artwork-align="center" title="TIPSCapabilities"}

with field:

incremental-change-media-types:
:  If a TIPS can provide updates with incremental changes for a
   resource, the "incremental-change-media-types" field has an entry
   for that resource-id, and the value is the supported media types
   of the incremental change separated by commas.  For the
   implementation of this specification, this MUST be "application/
   merge-patch+json", "application/json-patch+json", or "application/
   merge-patch+json,application/json-patch+json", unless defined by a future
   extension.

   When choosing the media types to encode incremental updates for a
   resource, the server MUST consider the limitations of the
   encoding.  For example, when a JSON merge patch specifies that the
   value of a field is null, its semantics are that the field is
   removed from the target and hence the field is no longer defined
   (i.e., undefined).  This, however, may not be the intended result
   for the resource, when null and undefined have different semantics
   for the resource.  In such a case, the server MUST choose JSON
   patch over JSON merge patch if JSON patch is indicated as a
   capability of the TIPS.  If the server does not support JSON patch
   to handle such a case, the server then needs to send a full
   replacement.

## Uses

The "uses" attribute MUST be an array with the resource-ids of every
network information resource for which this TIPS can provide service.

This set may be any subset of the ALTO server's network information resources
and may include resources defined in linked IRDs. However, it is RECOMMENDED
that the ALTO server selects a set that is closed under the resource dependency
relationship. That is, if a TIPS' "uses" set includes resource R1 and resource
R1 depends on ("uses") resource R0, then the TIPS' "uses" set SHOULD include R0
as well as R1. For example, if a TIPS provides a TIPS view for a cost map, it
SHOULD also provide a TIPS view for the network map upon which that cost map
depends.

If the set is not closed, at least one resource R1 in the "uses" field of a TIPS
depends on another resource R0 which is not in the "uses" field of the same
TIPS. Thus, a client cannot receive incremental updates for R0 from the same
TIPS service. If the client observes in an update of R1 that the version tag for
R0 has changed, it must make a request to retrieve the full content of R0, which
is likely to be less efficient than receiving the incremental updates of R0.

## An Example

Extending the IRD example in Section 8.1 of {{RFC8895}}, {{ex-ird}} is the IRD of an
ALTO server supporting ALTO base protocol, ALTO/SSE, and ALTO TIPS.

~~~
    "my-network-map": {
      "uri": "https://alto.example.com/networkmap",
      "media-type": "application/alto-networkmap+json"
    },
    "my-routingcost-map": {
      "uri": "https://alto.example.com/costmap/routingcost",
      "media-type": "application/alto-costmap+json",
      "uses": ["my-networkmap"],
      "capabilities": {
        "cost-type-names": ["num-routingcost"]
      }
    },
    "my-hopcount-map": {
      "uri": "https://alto.example.com/costmap/hopcount",
      "media-type": "application/alto-costmap+json",
      "uses": ["my-networkmap"],
      "capabilities": {
        "cost-type-names": ["num-hopcount"]
      }
    },
    "my-simple-filtered-cost-map": {
      "uri": "https://alto.example.com/costmap/filtered/simple",
      "media-type": "application/alto-costmap+json",
      "accepts": "application/alto-costmapfilter+json",
      "uses": ["my-networkmap"],
      "capabilities": {
        "cost-type-names": ["num-routingcost", "num-hopcount"],
        "cost-constraints": false
      }
    },
    "update-my-costs": {
      "uri": "https://alto.example.com/updates/costs",
      "media-type": "text/event-stream",
      "accepts": "application/alto-updatestreamparams+json",
      "uses": [
          "my-network-map",
          "my-routingcost-map",
          "my-hopcount-map",
          "my-simple-filtered-cost-map"
      ],
      "capabilities": {
        "incremental-change-media-types": {
          "my-network-map": "application/json-patch+json",
          "my-routingcost-map": "application/merge-patch+json",
          "my-hopcount-map": "application/merge-patch+json"
        },
        "support-stream-control": true
      }
    },
    "update-my-costs-tips": {
      "uri": "https://alto.example.com/updates-new/costs",
      "media-type": "application/alto-tips+json",
      "accepts": "application/alto-tipsparams+json",
      "uses": [
          "my-network-map",
          "my-routingcost-map",
          "my-hopcount-map",
          "my-simple-filtered-cost-map"
      ],
      "capabilities": {
        "incremental-change-media-types": {
          "my-network-map": "application/json-patch+json",
          "my-routingcost-map": "application/merge-patch+json",
          "my-hopcount-map": "application/merge-patch+json",
          "my-simple-filtered-cost-map": "application/merge-patch+json"
        },
      }
    }
~~~
{: #ex-ird artwork-align="center" title="Example of an ALTO Server Supporting ALTO Base Protocol, ALTO/SSE, and ALTO TIPS"}

Note that it is straightforward for an ALTO server to run HTTP/2 and
support concurrent retrieval of multiple resources such as "my-
network-map" and "my-routingcost-map" using multiple HTTP/2 streams.

The resource "update-my-costs-tips" provides an ALTO TIPS based
connection, and this is indicated by the media-type "application/
alto-tips+json".

# TIPS Open/Close

Upon request, a server sends a TIPS view to a client.  This TIPS view
may be created at the time of the request or may already exist
(either because another client has an active connection to a TIPS
view for the same requested network resource or because the server
perpetually maintains a TIPS view for an often-requested resource).
The server MAY keep track of which clients have an active connection
to each TIPS view to determine whether or not it should delete a TIPS
view and its corresponding updates graph and associated data.

## Open Request {#open-req}

An ALTO client requests that the server provide a TIPS view for a given resource
by sending an HTTP POST body with the media type
"application/alto-tipsparams+json". That body contains a JSON object of type
TIPSReq, where:

~~~
    object {
       ResourceID   resource-id;
       [JSONString  tag;]
       [Object      input;]
    } TIPSReq;
~~~
{: #fig-open-req artwork-align="center" title="TIPSReq"}


with the following fields:

resource-id:
:  The resource-id of an ALTO resource and MUST be in the TIPS' "uses" list
   ({{ird}}). If a client does not support all incremental methods from the set
   announced in the server's capabilities, the client MUST NOT use the TIPS
   service.

tag:
:  If the resource-id is a GET-mode resource with a version tag (or
   "vtag"), as defined in Section 10.3 of {{RFC7285}}, and the ALTO
   client has previously retrieved a version of that resource from
   ALTO, the ALTO client MAY set the "tag" field to the tag part of
   the client's version of that resource.  The server MAY use the tag
   when calculating a recommended starting edge for the client to
   consume.  Note that the client MUST support all incremental
   methods from the set announced in the server's capabilities for
   this resource.

input:
:  If the resource is a POST-mode service that requires input, the
   ALTO client MUST set the "input" field to a JSON object with the
   parameters that the resource expects.

## Open Response {#open-resp}

The response to a valid request MUST be a JSON object of type
AddTIPSResponse, denoted as media type "application/alto-tips+json":

~~~
    object {
      JSONString        tips-view-uri;
      TIPSViewSummary   tips-view-summary;
    } AddTIPSResponse;

    object {
      UpdatesGraphSummary   updates-graph-summary;
    } TIPSViewSummary;

    object {
      JSONNumber       start-seq;
      JSONNumber       end-seq;
      StartEdgeRec     start-edge-rec;
    } UpdatesGraphSummary;

    object {
      JSONNumber       seq-i;
      JSONNumber       seq-j;
    } StartEdgeRec;
~~~
{: #fig-open-resp artwork-align="center" title="AddTIPSResponse"}

with the following fields:

tips-view-uri:
:  Relative URI to the TIPS view of a network resource, which MUST be
   unique per connection, and is de-aliased by the server to refer to
   the actual location of the TIPS view which may be shared by other
   clients.

:  When creating the URI for the TIPS view, TIPS MUST NOT use other
   properties of an HTTP request, such as cookies or the client's IP
   address, to determine the TIPS view.  Furthermore, TIPS MUST NOT
   reuse a URI for a different object in the same connection.

:  It is expected that there is an internal mechanism to map a tips-
   view-uri to the TIPS view to be accessed.  For example, TIPS may
   assign a unique, internal state id to each TIPS view instance.
   However, the exact mechanism is left to the TIPS provider.

tips-view-summary:
:  Contains an updates-graph-summary.

   The updates-graph-summary field contains the starting sequence
   number (start-seq) of the updates graph and the last sequence
   number (end-seq) that is currently available, along with a
   recommended edge to consume (start-edge-rec).  How the server
   calculates the recommended edge depends on the implementation.
   Ideally, if the client does not provide a version tag, the server
   should recommend the edge of the latest snapshot available.  If
   the client does provide a version tag, the server should calculate
   the cumulative size of the incremental updates available from that
   version onward and compare it to the size of the complete resource
   snapshot.  If the snapshot is bigger, the server should recommend
   the first incremental update edge starting from client's tagged
   version.  Else, the server should recommend the latest snapshot
   edge.

If the request has any errors, the TIPS service must return an HTTP
"400 Bad Request" to the ALTO client; the body of the response
follows the generic ALTO error response format specified in
Section 8.5.2 of {{RFC7285}}.  Hence, an example ALTO error response
has the format shown in {{ex-bad-request}}.

~~~~
    HTTP/1.1 400 Bad Request
    Content-Length: 131
    Content-Type: application/alto-error+json
    Connection: close

    {
        "meta":{
            "code":  "E_INVALID_FIELD_VALUE",
            "field": "resource-id",
            "value": "my-network-map/#"
        }
    }
~~~~
{: #ex-bad-request artwork-align="center" title="ALTO Error Example"}

Note that "field" and "value" are optional fields.  If the "value"
field exists, the "field" field MUST exist.

-  If the TIPS request does not have a "resource-id" field, the error
   code of the error message MUST be `E_MISSING_FIELD` and the "field"
   field SHOULD be "resource-id".  The TIPS service MUST NOT create
   any TIPS view.

-  If the "resource-id" field is invalid or is not associated with
   the TIPS, the error code of the error message MUST be
   `E_INVALID_FIELD_VALUE`.  The "field" field SHOULD be the full path
   of the "resource-id" field, and the "value" field SHOULD be the
   invalid resource-id.

-  If the resource is a POST-mode service that requires input, the
   client MUST set the "input" field to a JSON object with the
   parameters that that resource expects.  If the "input" field is
   missing or invalid, TIPS MUST return the same error response that
   resource would return for missing or invalid input (see
   {{RFC7285}}).

Furthermore, it is RECOMMENDED that the server uses the following HTTP codes to
indicate other errors, with the media type "application/alto-error+json".

-  429 (Too Many Requests): when the number of TIPS views open requests exceeds
   server threshold. Server may indicate when to re-try the request in the
   "Re-Try After" headers.

##  Open Example

For simplicity, assume that the ALTO server is using the Basic
authentication.  If a client with username "client1" and password
"helloalto" wants to create a TIPS view of an ALTO Cost Map resource
with resource ID "my-routingcost-map", it can send the
request depiced in {{ex-op}}.

~~~~
    POST /tips HTTP/1.1
    Host: alto.example.com
    Accept: application/alto-tips+json, application/alto-error+json
    Authorization: Basic Y2xpZW50MTpoZWxsb2FsdG8K
    Content-Type: application/alto-tipsparams+json
    Content-Length: 41

    {
      "resource-id": "my-routingcost-map"
    }
~~~~
{: #ex-op artwork-align="center" title="Open Example"}

If the operation is successful, the ALTO server returns the
message shown in {{ex-op-rep}}.

~~~~
    HTTP/1.1 200 OK
    Content-Type: application/alto-tips+json
    Content-Length: 288

    {
        "tips-view-uri": "/tips/2718281828459",
        "tips-view-summary": {
          "updates-graph-summary": {
            "start-seq": 101,
            "end-seq": 106,
            "start-edge-rec" : {
              "seq-i": 0,
              "seq-j": 105
            }
          }
        }
    }
~~~~
{: #ex-op-rep artwork-align="center" title="Response Example"}

## Close Request {#close-req}

An ALTO client can indicate it no longer desires to pull/receive updates for a
specific network resource by "deleting" the TIPS view using the returned
tips-view-uri and the HTTP DELETE method. Whether or not the server actually
deletes the TIPS view is implementation dependent. For example, an ALTO server
may maintain a set of clients that subscribe to the TIPS view of a resource: a
client that deletes the view is removed from the set, and the TIPS view is only
removed when the dependent set becomes empty. See other potential
implementations in {{shared-tips-view}}. The DELETE request MUST have the
following format:

~~~~
    DELETE /<tips-view-uri>
~~~~

The response to a valid request must be 200 if success, and the
corresponding error code if there is any error.

If the connection between the client and TIPS provider is severed
without a DELETE request having been sent, the server MUST treat it
as if the client had sent a DELETE request because the TIPS view is,
at least from the client view, per-session based.

It is RECOMMENDED that the server uses the following HTTP codes to
indicate errors, with the media type "application/alto-error+json",
regarding TIPS view close requests.

-  404 (Not Found): if the requested TIPS view does not exist or is
   closed.

# TIPS Data Transfers - Client Pull {#pull}

TIPS allows an ALTO client to retrieve the content of an update item
from the updates graph, with an update item defined as the content
(incremental update or snapshot) on an edge in the updates graph.

##  Request

The client sends an HTTP GET request, where the media type of an
update item resource MUST be the same as the "media-type" field of
the update item on the specified edge in the updates graph.

The GET request MUST have the following format:

~~~~
    GET /<tips-view-uri>/ug/<i>/<j>
~~~~

For example, consider the updates graph in {{fig-ug-schema}}. If the client
wants to query the content of the first update item (0 -> 101) whose media type
is "application/alto- costmap+json", it must send a request to
"/tips/2718281828459/ug/0/101" and set the "Accept" header to "application/alto-
costmap+json, application/alto-error+json". See {{iu-example}} for a concrete
example.

## Response

If the request is valid (`ug/<i>/<j>` exists), the response is encoded
as a JSON object whose data format is indicated by the media type.

It is possible that a client conducts proactive fetching of future updates, by
long polling updates that have not been listed in the directory yet. For
long-poll prefetch, the client must have indicated the media type which may
appear. It is RECOMMENDED that the server allows for at least the prefetch of
`<end-seq> -> <end-seq + 1>`

Hence, the server processing logic SHOULD be:

-  If `ug/<i>/<j>` exists: return content using encoding.

-  Else if `ug/<i>/<j>` pre-fetch is acceptable: put request in a
   backlog queue.

-  Else: return error.

It is RECOMMENDED that the server uses the following HTTP codes to
indicate errors, with the media type "application/alto-error+json",
regarding update item requests.

-  404 (Not Found): if the requested TIPS view does not exist or is
   closed.

-  410 (Gone): if an update has a seq that is smaller than the start-
   seq.

-  415 (Unsupported Media Type): if the media type(s) accepted by the
   client does not include the media type of the update chosen by the
   server.

-  425 (Too Early): if the seq exceeds the server prefetch window

-  429 (Too Many Requests): when the number of pending (long-poll)
   requests exceeds server threshold.  Server may indicate when to
   re-try the request in the "Re-Try After" headers.

##  Example {#iu-example}

Assume the client wants to get the contents of the update item on
edge 0 to 101.  The format of the request is shown in {{ex-get}}.

~~~~
    GET /tips/2718281828459/ug/0/101 HTTP/1.1
    Host: alto.example.com
    Accept: application/alto-costmap+json, application/alto-error+json
~~~~
{: #ex-get artwork-align="center" title="GET Example"}

The response is shown in {{ex-get-res}}.

~~~~
    HTTP/1.1 200 OK
    Content-Type: application/alto-costmap+json
    Content-Length: 50

    { ... full replacement of my-routingcost-map ... }
~~~~
{: #ex-get-res artwork-align="center" title="Response to a GET Request"}

## New Next Edge Recommendation

While intended TIPS usage is for the client to recieve a recommended
starting edge in the TIPS summary, consume that edge, then construct
all future URIs by incrementing the sequence count by 1, there may be
cases in which the client needs to request a new next edge to
consume.  For example, if a client has an open TIPS view yet has not
polled in a while, the client may requests the next logical
incremental URI but the server has compacted the updates graph so it
no longer exists.  Thus, the client must request a new next edge to
consume based on its current version of the resource.

###  Request

An ALTO client requests that the server provide a next edge
recommendation for a given TIPS view by sending an HTTP POST request
with the media type "application/alto-tipsparams+json".  The URI has
the form:

~~~~
    POST /<tips-view-uri>/ug
~~~~

The POST body have the following form, where providing the version
tag of the resource the client already has is optional:

~~~~
    object {
        [JSONString  tag;]
    } TIPSNextEdgeReq;
~~~~


###  Response

The response to a valid request MUST be a JSON object of type
UpdatesGraphSummary (defined in {{open-resp}} but reproduced in {{fig-resp}} as
well), denoted as media type "application/alto-tips+json":

~~~~
    object {
      JSONNumber       start-seq;
      JSONNumber       end-seq;
      StartEdgeRec     start-edge-rec;
    } UpdatesGraphSummary;

    object {
      JSONNumber       seq-i;
      JSONNumber       seq-j;
    } StartEdgeRec;
~~~~
{: #fig-resp artwork-align="center" title="Data Format of TIPS Response"}


It is RECOMMENDED that the server uses the following HTTP codes to
indicate errors, with the media type "application/alto-error+json",
regarding new next edge requests.

-  404 (Not Found): if the requested TIPS view does not exist or is
   closed.

-  415 (Unsupported Media Type): if the media type(s) accepted by the
   client does not include the media type `application/alto-tips+json`.

# Operation and Processing Considerations

##  Considerations for Load Balancing {#load-balancing}

TIPS allow clients to make concurrent pulls of the incremental
updates potentially through different HTTP connections.  As a
consequence, it introduces additional complexities when the ALTO
server is being load balanced -- a feature widely used to build
scalable and fault-tolerant web services.  For example, a request may
be incorrectly processed if the following two conditions both hold:

-  the backend servers are stateful, i.e., the TIPS view is created
   and stored only on a single server;

-  the ALTO server is using layer-4 load balancing, i.e., the
   requests are distributed based on the TCP 5-tuple.

Thus, additional considerations are required to enable correct load
balancing for TIPS, including:

-  Use a stateless architecture: One solution is to follow the
   stateless computing pattern: states about the TIPS view are not
   maintained by the backend servers but are stored in a distributed
   database.  Thus, concurrent requests to the same TIPS view can be
   processed on arbitrary stateless backend servers, which all
   fetches data from the same database.

-  Configure the load balancers properly: In case when the backend
   servers are stateful, the load balancers must be properly
   configured to guarantee that requests of the same TIPS view always
   arrive at the same server.  For example, an operator or a provider
   of an ALTO server may configure layer-7 load balancers that
   distribute requests based on URL or cookies.

## Considerations for Choosing Updates

When implementing TIPS, a developer should be cognizant of the
effects of update schedule, which includes both the choice of timing
(i.e., when/what to trigger an update on the updates graph) and the
choice of message format (i.e., given an update, send a full
replacement or an incremental change).  In particular, the update
schedule can have effects on both the overhead and the freshness of
information.  To minimize overhead, developers may choose to batch a
sequence of updates for resources that frequently change by
cumulative updates or a full replacement after a while.  Developers
should be cognizant that batching reduces the freshness of
information and should also consider the effect of such delays on
client behaviors.

For incremental updates, this design allows both JSON patch and JSON
merge patch for incremental changes.  JSON merge patch is clearly
superior to JSON patch for describing incremental changes to cost
maps, endpoint costs, and endpoint properties.  For these data
structures, JSON merge patch is more space efficient, as well as
simpler to apply.  There is no advantage allowing a server to use
JSON patch for those resources.

The case is not as clear for incremental changes to network maps.

First, consider small changes, such as moving a prefix from one PID
to another.  JSON patch could encode that as a simple insertion and
deletion, while JSON merge patch would have to replace the entire
array of prefixes for both PIDs.  On the other hand, to process a
JSON patch update, the ALTO client would have to retain the indexes
of the prefixes for each PID.  Logically, the prefixes in a PID are
an unordered set, not an array; aside from handling updates, a client
has no need to retain the array indexes of the prefixes.  Hence, to
take advantage of JSON patch for network maps, ALTO clients would
have to retain additional, otherwise unnecessary, data.

Second, consider more involved changes, such as removing half of the
prefixes from a PID.  JSON merge patch would send a new array for
that PID, while JSON patch would have to send a list of remove
operations and delete the prefix one by one.

Therefore, each TIPS instance may choose to encode the updates using
JSON merge patch or JSON patch based on the type of changes in
network maps.

## Considerations for Cross-Resource Dependency Scheduling

Dependent ALTO resources result in cross-resource dependencies in
TIPS.  Consider the following pair of resources, where my-cost-map
(C) is dependent on my-network-map (N).  The updates graph for each
resource is shown, along with links in between the respective updates
graphs to show dependency:

~~~~ drawing
                       +---+   +---+   +---+   +---+   +---+
  my-network-map (N)   | 0 |-->|89 |-->|90 |-->|91 |-->|92 |
                       +---+   +---+   +---+   +---+   +---+
                                 |   \       \       \
                                 |    \       \       \
                       +---+   +---+   +---+   +---+   +---+
  my-cost-map (C)      | 0 |-->|101|-->|102|-->|103|-->|104|
                       +---+   +---+   +---+   +---+   +---+
                        |_______________________|
~~~~
{: #fig-cross artwork-align="center" title="Example Dependency Model"}

In {{fig-cross}}, the cost-map versions 101 and 102 (denoted as C101 and C102)
are dependent on the network-map version 89 (denoted as N89). The cost-map
version 103 (C103) is dependent on the network-map version 90 (N90), and so on.

In pull-mode, a client can decide the order in which to receive the updates.

In push-mode, the server must decide.  Pushing order may affect how
fast the client can build a consistent view and how long the client
needs to buffer the update.

-  Example 1: The server pushes N89, N90, N91, C101, C102 in that
   order.  The client either gets no consistent view of the resources
   or it has to buffer N90 and N91.

-  Example 2: The server pushes C101, C102, C103, N89.  The client
   either gets no consistent view or it has to buffer C103.

Therefore, the server is RECOMMENDED to push updates in the ascending
order of the smallest dependent tag, e.g., {C101, C102, N89} before
{C103, N90}

## Considerations for Client Processing Updates {#client-processing}

In general, when an ALTO client receives a full replacement for a
resource, the ALTO client should replace the current version with the
new version.  When an ALTO client receives an incremental update for
a resource, the ALTO client should apply those updates to the current
version of the resource.

However, because resources can depend on other resources (e.g., cost
maps depend on network maps), an ALTO client MUST NOT use a dependent
resource if the resource on which it depends has changed.  There are
at least two ways an ALTO client can do that.  The following
paragraphs illustrate these techniques by referring to network and
cost map messages, although these techniques apply to any dependent
resources.

Note that when a network map changes, the server SHOULD send the
network map update message before sending the updates for the
dependent cost maps.

One approach is for the ALTO client to save the network map update
message in a buffer and continue to use the previous network map and
the associated cost maps until the ALTO client receives the update
messages for all dependent cost maps.  The ALTO client then applies
all network and cost map updates atomically.

Alternatively, the ALTO client MAY update the network map
immediately.  In this case, the cost maps using the network map
become invalid because they are inconsistent with the current network
map; hence, the ALTO client MUST mark each such dependent cost map as
temporarily invalid and MUST NOT use each such cost map until the
ALTO client receives a cost map update indicating that it is based on
the new network map version tag.

When implementing server push, the server SHOULD send updates for
dependent resource (i.e., the cost maps in the preceding example) in
a timely fashion.  However, if the ALTO client does not receive the
expected updates, a simple recovery method is that the ALTO client
uses client pull to request the missing update.  The ALTO client MAY
retain the version tag of the last version of any tagged resources
and search those version tags when identifying the new updates to
pull.  Although not as efficient as possible, this recovery method is
simple and reliable.

Though a server SHOULD send update items sequentially, it is possible
that a client receives the update items out of order (in the case of
a retransmitted update item or a result of concurrent fetch).  The
client MUST buffer the update items if they arrive out of order and
then apply them sequentially (based upon the sequence numbers) due to
the operation of JSON merge patch and JSON patch.

## Considerations for Updates to Filtered Cost Maps

If TIPS provides updates to a Filtered Cost Map that allows
constraint tests, then an ALTO client MAY request updates to a
Filtered Cost Map request with a constraint test.  In this case, when
a cost changes, the updates graph MUST have an update if the new
value satisfies the test.  If the new value does not, whether there
is an update depends on whether the previous value satisfied the
test.  If it did not, the updates graph SHOULD NOT have an update.
But if the previous value did, then the updates graph MUST add an
update with a "null" value to inform the ALTO client that this cost
no longer satisfies the criteria.

TIPS can avoid having to handle such a complicated behavior by
offering TIPS only for Filtered Cost Maps that do not allow
constraint tests.

## Considerations for Updates to Ordinal Mode Costs

For an ordinal mode cost map, a change to a single cost point may
require updating many other costs.  As an extreme example, suppose
the lowest cost changes to the highest cost.  For a numerical mode
cost map, only that one cost changes.  But for an ordinal mode cost
map, every cost might change.  While this document allows TIPS to
offer incremental updates for ordinal mode cost maps, TIPS
implementors should be aware that incremental updates for ordinal
costs are more complicated than for numerical costs, and that small
changes of the original cost value may result in large updates.

A TIPS implementation can avoid this complication by only offering
full replacements as updates in the updates graph for ordinal cost
maps.

## Considerations for Managing Shared TIPS Views {#shared-tips-view}

From a client's point of view, it sees only one copy of the TIPS view
for any resource.  However, on the server side, there are different
implementation options, especially for common resources (e.g.,
network map or cost map) that may be frequently queried by many
clients.  Some potential options are listed below:

-  An ALTO server creates one TIPS view of the common resource for
   each client.  When the client deletes the view, the server deletes
   the view in the server storage.

-  An ALTO server maintains one copy of the TIPS view for each common
   resource and all clients requesting the same resources use the
   same copy.  There are two ways to manage the storage for the
   shared copy:

   -  the ALTO server maintains the set of clients that subscribe to
      the TIPS view, and only removes the view from the storage when
      the set becomes empty.

   -  the TIPS view is never removed from the storage.

Developers may choose different implementation options depending on
criteria such as request frequency, available resources of the ALTO
server, the ability to scale, and programming complexity.

##  Considerations for Offering Shortcut Incremental Updates

Besides the mandatory stepwise incremental updates (from i to i+1),
an ALTO server may optionally offer shortcut incremental updates, or
simple shortcuts, between two non-consecutive versions i and i+k (k >
1).  Such shortcuts offer alternative paths in the update graph and
can potentially speed up the transmission and processing of
incremental updates, leading to faster synchronization of ALTO
information, especially when the client has limited bandwidth and
computation.  However, implementors of an ALTO server must be aware
that:

1.  Optional shortcuts may increase the size of the update graph, in
    the worst case being the square of the number of updates (i.e.,
    when a shortcut is offered for each version to all future
    versions).

2.  Optional shortcuts require additional storage on the ALTO server.

3.  Optional shortcuts may reduce concurrency when the updates do not
    overlap, e.g., when the updates apply to different parts of an
    ALTO resource.  In such a case, the total size of the original
    updates is close to the size of the shortcut, but the original
    updates can be transmitted concurrently while the shortcut is
    transmitted in a single connection.

# Security Considerations

The security considerations (Section 15 of {{RFC7285}}) of the base
protocol fully apply to this extension.  For example, the same
authenticity and integrity considerations (Section 15.1 of {{RFC7285}})
still fully apply; the same considerations for the privacy of ALTO
users (Section 15.4 of {{RFC7285}}) also still fully apply.

The additional services (addition of update read service and update
push service) provided by this extension extend the attack surface
described in Section 15.1.1 of {{RFC7285}}.  The following sub-sections discuss the
additional risks and their remedies.

## TIPS: Denial-of-Service Attacks

Allowing TIPS views enables a new class of Denial-of-Service attacks.
In particular, for the TIPS server, an ALTO client might
create an excessive number of TIPS views.

To avoid these attacks on the TIPS server, the server SHOULD choose
to limit the number of active views and reject new requests when that threshold
is reached. TIPS allows predictive fetching and the server SHOULD also choose to
limit the number of pending requests. If a new request exceeds the threshold,
the server SHOULD log the event and may return the HTTP status "429 Too many
requests".

It is important to note that the preceding approaches are not the
only possibilities.  For example, it may be possible for TIPS to use
somewhat more clever logic involving IP reputation, rate-limiting,
and compartmentalization of the overall threshold into smaller
thresholds that apply to subsets of potential clients.

## ALTO Client: Update Overloading or Instability

The availability of continuous updates, when the client indicates receiving
server push, can also cause overload for an ALTO client, in particular, an ALTO
client with limited processing capabilities. The current design does not include
any flow control mechanisms for the client to reduce the update rates from the
server. For example, TCP, HTTP/2, and QUIC provide stream and connection flow
control data limits, and PUSH stream limits, which might help prevent the client
from being overloaded. Under overloading, the client MAY choose to remove the
information resources with high update rates.

Also, under overloading, the client may no longer be able to detect
whether information is still fresh or has become stale.  In such a
case, the client should be careful in how it uses the information to
avoid stability or efficiency issues.

## Spoofed URI

An outside party that can read the TIPS response or that can observe
TIPS requests can obtain the TIPS view URI and use that to send
fraudulent "DELETE" requests, thus disabling the service for the
valid ALTO client.  This can be avoided by encrypting the requests
and responses (Section 15 of {{RFC7285}}).

# IANA Considerations

IANA is requested to register the following media types from the registry available at {{IANA-Media-Type}}:

*  application/alto-tips+json: as described in {{open-resp}};

*  application/alto-tipsparams+json: as described in {{open-req}};

> Note to the RFC Editor: Please replace This-Document with the RFC number to be assigned to this document.

##  application/alto-tips+json Media Type

Type name:
: application

Subtype name:
: alto-tips+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: Encoding considerations are identical to those specified for the
  "application/json" media type. See {{RFC8259}}.

Security considerations:
: See the Security Considerations section of This-Document.

Interoperability considerations:
: This document specifies format of conforming messages and the interpretation
  thereof.

Published specification:
: {{open-resp}} of This-Document.

Applications that use this media type:
: ALTO servers and ALTO clients either stand alone or are embedded within other
  applications.

Fragment identifier considerations:
: N/A

Additional information:
:  <br>

   Deprecated alias names for this type:
   : N/A

   Magic number(s):
   : N/A

   File extension(s):
   : This document uses the media type to refer to protocol messages and thus
      does not require a file extension.

   Macintosh file type code(s):
   : N/A

Person and email address to contact for further information:
: See Authors' Addresses section.

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See Authors' Addresses section.

Change controller:
: Internet Engineering Task Force (mailto:iesg@ietf.org).

Provisional registration?:
: No

## application/alto-tipsparams+json Media Type

Type name:
: application

Subtype name:
: alto-tipsparams+json

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: Encoding considerations are identical to those specified for the
   "application/json" media type. See {{RFC8259}}.

Security considerations:
: See the Security Considerations section of This-Document.

Interoperability considerations:
: This document specifies format of conforming messages and the interpretation
  thereof.

Published specification:
: {{open-req}} of This-Document.

Applications that use this media type:
: ALTO servers and ALTO clients either stand alone or are embedded within other
  applications.

Fragment identifier considerations:
: N/A

Additional information:
: <br>

  Deprecated alias names for this type:
  : N/A

   Magic number(s):
   : N/A

   File extension(s):
   : This document uses the media type to refer to protocol messages and thus
      does not require a file extension.

   Macintosh file type code(s):
   : N/A

Person and email address to contact for further information:
: See Authors' Addresses section.

Intended usage:
: COMMON

Restrictions on usage:
: N/A

Author:
: See Authors' Addresses section.

Change controller:
: Internet Engineering Task Force (mailto:iesg@ietf.org).

Provisional registration?:
: No

--- back

# A High Level Deployment Model {#sec-dep-model}

Conceptually, the TIPS system consists of three types of resources:

- (R1) TIPS frontend to manage (create/delete) TIPS views.

- (R2) TIPS view directory, which provides metadata (e.g., references) about the
  network resource data.

- (R3) The actual network resource data, encoded as complete ALTO network
  resources (e.g., cost map, network map) or incremental updates.

~~~~ drawing
                      +------------------------------------------------+
                      |                                                |
 +------+             |R1: Frontend/Open  R2: Directory/Meta  R3: Data |
 |      | "iget" base |     +-----+           +-----+         +-----+  |
 |      | resource 1  |     |     |           |     |         |     |  |
 |      |-------------|---->|     |           |     |         |     |  |
 |      | incremental |     |     |           |     |-------->|     |  |
 |      | transfer    |     |     |           |     |         |     |  |
 |      | resource    |     |     |           |     |         |     |  |
 |      |<------------|-----|     |           +-----+         +-----+  |
 |Client|             |     |     |                                    |
 |      | "iget" base |     |     |                                    |
 |      | resource 2  |     |     |           +-----+         +-----+  |
 |      |-------------|---->|     |           |     |         |     |  |
 |      | incremental |     |     |           |     |         |     |  |
 |      | transfer    |     |     |           |     | ------->|     |  |
 |      | resource    |     |     |           |     |         |     |  |
 |      |<------------|-----|     |           |     |         |     |  |
 +------+             |     +-----+           +-----+         +-----+  |
                      |                                                |
                      +------------------------------------------------+
~~~~
{: #fig-service-model artwork-align="center" title="Sample TIPS Deployment Model"}

Design Point: Component Resource Location

- Design 1 (Single): all the three resource types at the same, single server (accessed via
  relative reference)

- Design 2 (Flexible): all three resource types can be at their own server (accessed via
  absolute reference)

- Design 3 (Dir + Data): R2 and R3 must remain together, though R1 might not be
  on the same server

This document specifies Design 1 in order to simplify session management, though
at the expense of maximum load balancing flexibility. See {{load-balancing}} for
a discussion on load balancing considerations. Future documents may extend the
protocol to support Design 2 or Design 3.

# Conformance to "Building Protocols with HTTP" Best Current Practices {#sec-bcp-http}


This specification adheres fully to {{RFC9205}} as further elaborated below:

-  TIPS does not "redefine, refine, or overlay the semantics of
   generic protocol elements such as methods, status codes, or
   existing header fields" and instead focuses on "protocol elements
   that are specific to `[the TIPS]` application -- namely, `[its]` HTTP
   resources" ({{Section 3.1 of RFC9205}}).

-  There are no statically defined URI components ({{Section 3.2 of RFC9205}}).

-  No minimum version of HTTP is specified by TIPS which is
   recommended ({{Section 4.1 of RFC9205}}).

-  The TIPS design follows the advice that "When specifying examples of
   protocol interactions, applications should document both the
   request and response messages with complete header sections,
   preferably in HTTP/1.1 format" ({{Section 4.1 of RFC9205}}).

-  TIPS uses URI templates which is recommended ({{Section 4.2 of RFC9205}}).

-  TIPS follows the pattern that "a client will begin interacting
   with a given application server by requesting an initial document
   that contains information about that particular deployment,
   potentially including links to other relevant resources.  Doing so
   ensures that the deployment is as flexible as possible
   (potentially spanning multiple servers), allows evolution, and
   also gives the application the opportunity to tailor the
   "discovery document" to the client" ({{Section 4.4.1 of RFC9205}}).

-  TIPS uses existing HTTP schemes ({{Section 4.4.2 of RFC9205}}).

-  TIPS defines its errors "to use the most applicable status code"
   ({{Section 4.6 of RFC9205}}).

-  TIPS does not "make assumptions about the relationship between
   separate requests on a single transport connection; doing so
   breaks many of the assumptions of HTTP as a stateless protocol and
   will cause problems in interoperability, security, operability,
   and evolution" ({{Section 4.11 of RFC9205}}).  The only relationship
   between requests is that a client must make a request to first
   discover where a TIPS view of resource will be served, which is
   consistent with the URI discovery in {{Section 4.4.1 of RFC9205}}.

*  {{Section 4.14 of RFC9205}} notes that there are
   quite a few caveats with using server push, mostly because of lack
   of widespread support.  The authors have considered these
   factors and have still decided server push can be valuable in the
   TIPS use case.

# Push-mode TIPS using HTTP Server Push

In this section, we give a non-normative specification of the push-mode TIPS. It
is intended to not be part of the standard protocol extension, because of the
lack of server push support and increased protocol complexity. However,
push-mode TIPS can potentially improve performance (e.g., latency) in more
dynamic environments and use cases, with wait-free message delivery. Using
native server push also results in minimal changes to the current protocol.
Thus, a preliminary push-mode TIPS extension using native server push is
specified here as a reference for future push-mode TIPS protocol designs.

## Basic Workflow

A client that prefers server push can use the workflow as shown in
{{fig-workflow-push}}. In this case, the client indicates for server push when it
creates the TIPS view. Future updates are pushed to the client as soon as they
become available.

~~~~ drawing
Client                                  TIPS
  o                                       .
  | Open persistent HTTP connection       .
  |-------------------------------------->|
  |                                       .
  | POST to create/receive a TIPS view    .
  |      for resource 1 and add           .
  |      self to receiver set             .
  | ------------------------------------> |
  | <tips-view-uri1>, <tips-view-summary> .
  |<------------------------------------- |
  |                                       .
  | PUSH <tips-view-uri1>/ug/<i>/<j>      .
  | <-------------------------------------|
  |                                       .
  | PUSH <tips-view-uri1>/ug/<j>/<j+1>    .
  | <-------------------------------------|
  |                                       .
  | PUT to remove self from receiver      .
  |      set of resource 1                .
  |-------------------------------------> |
  |                                       .
  | Close HTTP connection                 .
  |-------------------------------------->|
  o
~~~~
{: #fig-workflow-push artwork-align="center" title="ALTO TIPS Workflow Supporting Server Push"}

## TIPS Information Resource Directory (IRD) Announcement {#ird-push}

The specifications for media type, uses, requests and responses of the push-mode
TIPS is the same as specified in {{caps}}.

### Capabilities

The capabilities field of push-mode TIPS is modeled on that defined in {{caps}}.

Specifically, the capabilities are defined as an object of type
PushTIPSCapabilities:

~~~
     object {
       [Boolean                     support-server-push;]
     } PushTIPSCapabilities: TIPSCapabilities;
~~~

with field:

support-server-push:
:  The "support-server-push" field specifies whether the given TIPS
   supports server push.  If the "support-server-push" field is TRUE,
   this TIPS will allow a client to start or stop server push.  If
   the field is FALSE or not present, this TIPS does not provide
   server push.

### Example

~~~~
    "update-my-costs-tips-with-server-push": {
      "uri": "https://alto.example.com/updates-new/costs",
      "media-type": "application/alto-tips+json",
      "accepts": "application/alto-tipsparams+json",
      "uses": [
          "my-network-map",
          "my-routingcost-map",
          "my-hopcount-map",
          "my-simple-filtered-cost-map"
      ],
      "capabilities": {
        "incremental-change-media-types": {
          "my-network-map": "application/json-patch+json",
          "my-routingcost-map": "application/merge-patch+json",
          "my-hopcount-map": "application/merge-patch+json",
          "my-simple-filtered-cost-map": "application/merge-patch+json"
        },
        "support-server-push": true
      }
    }
~~~~

## Push-mode TIPS Open/Close

### Open Request {#push-open-req}

An ALTO client requests that the server provide a TIPS view for a given resource
by sending an HTTP POST body with the media type
"application/alto-tipsparams+json". That body contains a JSON object of type
PushTIPSReq, where:

~~~
    object {
       [Boolean     server-push;]
    } PushTIPSReq: TIPSReq;
~~~

with the following field:

server-push:
:  Set to TRUE if a client desires to receive updates via server
   push.  If the value is FALSE or not present, the client does not
   accept server push updates.  See {{push}} for detailed
   specifications.

### Open Response

The push-mode TIPS requires extending the contents of `tips-view-summary` field
of AddTIPSResponse:

~~~
    object {
      [Boolean              server-push;]
    } PushTIPSViewSummary : TIPSViewSummary;
~~~

with the following field:

server-push:
:  An optional server-push boolean value which is set to TRUE if and only if the
   client indicates server push. If the client indicates server push, the
   recommended edge in the updates-graph-summary field will be the first content
   pushed.

## TIPS Data Transfer - Server Push {#push}

TIPS allows an ALTO client to receive an update item pushed by the
ALTO server.

If a client registers for server push, it should not request updates
via pull to avoid receiving the same information twice, unless the
client does not receive the expected updates (see {{client-processing}}).

###  Manage Server Push

A client starts to receive server push when it is added to the
receiver set.  A client can read the status of the push state and
remove itself from the receiver set to stop server push.

####  Start Server Push

A client can add itself explicitly to the receiver set or add itself
to the receiver set when requesting the TIPS view.  Before a client
starts receiving server push for a TIPS view, it MUST enable server
push in HTTP, i.e., following Section 8.4 of {{RFC9113}} for HTTP/2 and
Section 4.6 of {{RFC9114}} for HTTP/3.  If the client does not enable
HTTP server push, the ALTO server MUST return an ALTO error with the
`E_INVALID_FIELD_VALUE` code and set the "field" to "server-push".

Explicit add: A client can explicitly add itself in the receiver set
by using the HTTP PUT method with media type "application/alto-
tipsparams+json", where the client may optionally specify a starting
edge (next-edge) from which it would like to receive updates:

~~~~
    PUT /<tips-view-uri>/push

    object {
      Boolean     server-push;
      [NextEdge    next-edge;]
    } PushState;

    object {
      JSONNumber       seq-i;
      JSONNumber       seq-j;
    } NextEdge;
~~~~

with the following fields:

server-push:
:  Set to true if the client desires to receive server push updates.

next-edge:
:  Optional field to request a starting edge to be pushed if the
   client has pulled the updates graph directory and has calculated
   the path it desires to take.  The server MAY push this edge first
   if available.

Short cut add: When requesting a TIPS view, an ALTO client can start
server push by setting the option "server-push" field to be true
using the HTTP POST method defined in {{open-req}}.

#### Read Push State

A client can use the HTTP GET method, with accept header set to
"application/alto-tipsparams+json" to check whether server push is correctly
enabled. The requested URL is the root path of the TIPS view, appended with
"push", as shown below.

~~~~
    GET /<tips-view-uri>/push
~~~~

The server returns an JSON object with content type
"application/alto-tipsparams+json". The response MUST include only one field
"server-push". If the server push is enabled, the value of the "server-push"
field MUST be the JSONBool value "true" (without the quote marks), and otherwise
JSONBool value "false" (without the quote marks).

#### Stop Push

A client can stop receiving server push updates either explicitly or
implicitly.

Explicit stop:
:  A client stops push by using the HTTP PUT method to `/<tips-view- uri>/push`,
   with content type "application/alto-tipsparams+json" and setting server-push
   to FALSE:

Implicit stop:
:  There are two ways. First, TIPS view is connection ephemeral: the close of
   connection or stream for the TIPS view deletes the TIPS view from the view
   of the client.

   Second, the client sends a DELETE `/<tips-view-uri>` request, indicating it
   no longer is interested in the resource, which also deletes the
   client from the push receiver set if present.

Note that a client may choose to explicitly stop server push for a
resource, but may not delete the TIPS view so that it can switch
seamlessly from server push to client pull in the case that the
server push frequency is undesirable, without having to request a new
TIPS view.

### Scheduling Server Push Updates

The objective of the server is to push the latest version to the
client using the lowest cost (sum of size) of the updates.  Hence, it
is RECOMMENDED that the server computes the push path using the
following algorithm, upon each event computing a push:

-  Compute client current version (nc). During initialization, if the TIPS view
   request has a tag, find that version; otherwise nc = 0

-  Compute the shortest path from the current version to the latest version, nc,
   n1, ... ne (latest version). Note that the shortest path may not involve the
   tagged version and instead follow the edge from 0 to the latest snapshot.

-  Push `/<tips-view-uri>/ug/nc/n1`.

Note

-  Initialization: If the client specifically requests a starting
   edge to be pushed, the server MAY start with that edge even if it
   is not the shortest path.

-  Push state: the server MUST maintain the last entry pushed to the
   client (and hence per client, per connection state) and schedule
   next update push accordingly.

###  Server Push Stream Management

Let `SID_tv` denote the stream that creates the TIPS view. The server push MUST
satisfy the following requirements:

-  `PUSH_PROMISE` frames MUST be sent in stream `SID_tv` to serialize and allow
   the client to know the push order;

-  The  `PUSH_PROMISE` frame chooses a new server-selected stream ID, and the
   stream is closed after push.

-  Canceling an update pushed by the server ({{Section 8.4.2 of RFC9113}} for
   HTTP/2 or {{Section 4.6 of RFC9114}} for HTTP/3) makes the state management
   complex. Thus, a client MUST NOT cancel a schedule update.

### Examples

The examples below are for HTTP/2 and based on the example update graph in
{{data-model}}.

#### Starting Server Push

{{fig-ex-ssp}} is an example request from a client to an ALTO server which
enables server push when creating a TIPS view.

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :method = POST
        :scheme = https
        :path = /tips
        host = alto.example.com
        accept = application/alto-tips+json,
                 application/alto-error+json
        content-type = application/alto-tipsparams+json
        content-length = 67

    DATA
      - END_STREAM
      {
        "resource-id": "my-routingcost-map",
        "server-push": true
      }
~~~~
{: #fig-ex-ssp artwork-align="center" title="Example Request to Start Server Push"}

And {{fig-ex-ssp-resp}} is the response the server returns to the client. Note
that the END_STREAM bit is not set.

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :status = 200
        content-type = application/alto-tips+json
        content-length = 196

    DATA
      - END_STREAM
      {
        "tips-view-uri": "/tips/2718281828459",
        "tips-view-summary": {
          "updates-graph-summary": {
            "start-seq": 101,
            "end-seq": 106,
            "start-edge-rec" : {
              "seq-i": 0,
              "seq-j": 105
            }
          },
          "server-push": true
        }
      }
~~~~
{: #fig-ex-ssp-resp artwork-align="center" title="Example Response to a Start Server Push Request"}

#### Querying the Push State

Now assume the client queries the server whether server push is successfully
enabled. {{fig-ps-req}} is the request:

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :method = GET
        :scheme = https
        :path = /tips/2718281828459/push
        host = alto.example.com
        accept = application/alto-error+json,
                      application/alto-tipsparams+json
~~~~
{: #fig-ps-req artwork-align="center" title="Example Request to Query Push State"}

And {{fig-ps-resp}} is the response.

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :status = 200
        content-type = application/alto-tipsparams+json
        content-length = 519

    DATA
      - END_STREAM
      {
        "server-push": true
      }
~~~~
{: #fig-ps-resp artwork-align="center" title="Example Response of the Push State Request"}

#### Receiving Server Push

Below shows the example of how the server may push the updates to the client.

First, the ALTO server sends a PUSH_PROMISE in the same stream that is left open
when creating the TIPS view. As there is no direct edge from 0 to 106, the first
update is from 0 to 105, as shown in {{fig-push-1}}.

~~~~
    PUSH_PROMISE
      - END_STREAM
        Promised Stream 4
        HEADER BLOCK
        :method = GET
        :scheme = https
        :path = /tips/2718281828459/ug/0/105
        host = alto.example.com
        accept = application/alto-error+json,
                      application/alto-costmap+json
~~~~
{: #fig-push-1 artwork-align="center" title="An Example PUSH_PROMISE Frame with a Pushed Update from 0 to 105"}

Then, the content of the pushed update (a full replacement) is delivered through
stream 4, as announced in the PUSH_PROMISE frame in {{fig-push-1}}.
{{fig-push-2}} shows the content of the message.

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :status = 200
        content-type = application/alto-costmap+json
        content-length = 539

    DATA
      + END_STREAM
      {
        "meta" : {
          "dependent-vtags" : [{
              "resource-id": "my-network-map",
              "tag": "da65eca2eb7a10ce8b059740b0b2e3f8eb1d4785"
            }],
          "cost-type" : {
            "cost-mode"  : "numerical",
            "cost-metric": "routingcost"
          },
          "vtag": {
            "resource-id" : "my-routingcost-map",
            "tag" : "3ee2cb7e8d63d9fab71b9b34cbf764436315542e"
          }
        },
        "cost-map" : {
          "PID1": { "PID1": 1,  "PID2": 5,  "PID3": 10 },
          "PID2": { "PID1": 5,  "PID2": 1,  "PID3": 15 },
          "PID3": { "PID1": 20, "PID2": 15  }
        }
    }
~~~~
{: #fig-push-2 artwork-align="center" title="Content of the Pushed Update from 0 to 105"}

As the latest version has sequence number 106, the ALTO server sends another
PUSH_PROMISE in the same stream that is left open when creating the TIPS view to
transit from 105 to 106, as shown in {{fig-push-3}}.

~~~~
    PUSH_PROMISE
      - END_STREAM
        Promised Stream 6
        HEADER BLOCK
        :method = GET
        :scheme = https
        :path = /tips/2718281828459/ug/105/106
        host = alto.example.com
        accept = application/alto-error+json,
                      application/merge-patch+json
~~~~
{: #fig-push-3 artwork-align="center" title="Another Example of PUSH_PROMISE Frame with a Pushed Update from 105 to 106"}

Then, the content of the pushed update (an incremental update as a JSON merge
patch) is delivered through stream 6, as announced in the PUSH_PROMISE frame.
{{fig-push-4}} shows the content of the update message.

~~~~
    HEADERS
      + END_STREAM
      + END_HEADERS
        :status = 200
        content-type = application/merge-patch+json
        content-length = 266

    DATA
      + END_STREAM
      {
        "meta": {
            "vtag": {
              "tag": "c0ce023b8678a7b9ec00324673b98e54656d1f6d"
            }
        },
        "cost-map": {
          "PID1": {
            "PID2": 9
          },
          "PID3": {
            "PID1": null,
            "PID3": 1
          }
        }
      }
~~~~
{: #fig-push-4 artwork-align="center" title="Content of the Pushed Update from 105 to 106"}

#### Stopping Server Push

{{fig-stop-req}} is an example of explicitly stopping the server push. The client sends a
PUT request to the push state of the TIPS view and set the "server-push" value
to "false".

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :method = PUT
        :scheme = https
        :path = /tips/2718281828459/push
        host = alto.example.com
        accept = application/alto-error+json
        content-type = application/alto-tipsparams+json
        content-length = 29

    DATA
      - END_STREAM
      {
        "server-push": false
      }
~~~~
{: #fig-stop-req artwork-align="center" title="Example Request to Stop Server Push"}

The server simply returns an empty message with status code 200, to indicate
that the operation succeeds, ashown in {{fig-stop-resp}}.

~~~~
    HEADERS
      - END_STREAM
      + END_HEADERS
        :status = 200
~~~~
{: #fig-stop-resp artwork-align="center" title="Example Response of the Stop Server Push Request"}

# Acknowledgments
{:numbered="false"}

The authors of this document would like to thank Mark Nottingham and Spencer
Dawkins for providing invaluable reviews of earlier versions of this document,
Adrian Farrel, Qin Wu, and Jordi Ros Giralt for their continuous feedback, Russ
White, Donald Eastlake, Martin Thomson, Bernard Adoba, Spencer Dawkins, and
Sheng Jiang for the directorate reviews, and Martin Duke for the Area Director
Review.
