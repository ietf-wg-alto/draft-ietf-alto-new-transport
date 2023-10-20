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
    ins: K. Gao
    name: Kai Gao
    street: "No.24 South Section 1, Yihuan Road"
    city: Chengdu
    code: 610000
    country: China
    org: Sichuan University
    email: kaigao@scu.edu.cn

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
  RFC3986:

informative:
  RFC9205:
  IANA-Media-Type:
    title: Media Types
    target: https://www.iana.org/assignments/media-types/media-types.xhtml
    date: 2023-06

--- abstract

The ALTO Protocol (RFC 7285) leverages HTTP/1.1 and is designed for the simple,
sequential request-reply use case, in which an ALTO client requests a
sequence of information resources and the server responds with the complete
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
   to receive updates for a set of resources and the server can then
   concurrently and incrementally push updates to that client whenever
   monitored resources change.

Both protocols are designed for HTTP/1.1 {{RFC9112}}, but HTTP/2 {{RFC9113}} and
HTTP/3 {{RFC9114}} can support HTTP/1.1 workflows. However, HTTP/2 and HTTP/3
provide features that can improve certain properties of ALTO and ALTO/SSE.

- First, consider the ALTO base protocol, which is designed to transfer only
  complete information resources. A client can run the base protocol on top of
  HTTP/2 or HTTP/3 to request multiple information resources in concurrent
  streams, but each request must be for a complete information resource: there is
  no capability it transmits incremental updates. Hence, there can be a large
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

To mitigate these concerns, this document introduces a new ALTO service called
the Transport Information Publication Service (TIPS). TIPS uses an incremental
RESTful design to provide an ALTO client with a new capability to explicitly,
concurrently issue non-blocking requests for specific incremental updates using
native HTTP/2 or HTTP/3, while still functioning for HTTP/1.1.

Despite the benefits, however, ALTO/SSE {{RFC8895}}, which solves a similar
problem, has its advantages. First, SSE is a mature technique with a
well-established ecosystem that can simplify development. Second, SSE does not
allow multiple connections to receive updates for multiple objects over
HTTP/1.1.

HTTP/2 {{RFC9113}} and HTTP/3 {{RFC9114}} also specify server push, which might
enhance TIPS. We discuss push-mode TIPS as an alternative design in {{push}}.

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
  information about the network information resource. The TIPS view has one
  basic component, updates graph (ug), but may include other transport
  information.

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
  monotonically and consecutively increased sequence number. This document uses the term
  "version s" to refer to the version associated with sequence number "s".

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
  snapshot or an incremental update. An update item can be considered as a pair
  (op, data) where op denotes whether the item is an incremental update or a
  snapshot, and data is the content of the item.

ID#i-#j:
: Denotes the update item on a specific edge in the updates graph to transition
  from version i to version j, where i and j are the sequence numbers of the
  source node and the target node of the edge, respectively.

~~~~ drawing
                                   +-------------+
    +-----------+ +--------------+ |  Dynamic    | +-----------+
    |  Routing  | | Provisioning | |  Network    | | External  |
    | Protocols | |    Policy    | | Information | | Interface |
    +-----------+ +--------------+ +-------------+ +-----------+
          |              |                |              |
+-----------------------------------------------------------------+
| ALTO Server                                                     |
| +-------------------------------------------------------------+ |
| |                                         Network Information | |
| | +-------------+                         +-------------+     | |
| | | Information |                         | Information |     | |
| | | Resource #1 |                         | Resource #2 |     | |
| | +-------------+                         +-------------+     | |
| +-----|--------------------------------------/-------\--------+ |
|       |                                     /         \         |
| +-----|------------------------------------/-----------\------+ |
| |     |       Transport Information       /             \     | |
| | +--------+                     +--------+        +--------+ | |
| | |  tv1   |                     |  tv2   |        |  tv3   | | |
| | +--------+                     +--------+        +--------+ | |
| |     |                          /                     |      | |
| | +--------+            +--------+                 +--------+ | |
| | | tv1/ug |            | tv2/ug |                 | tv3/ug | | |
| | +--------+            +--------+                 +--------+ | |
| +----|----\----------------|-------------------------|--------+ |
|      |     \               |                         |          |
+------|------\--------------|-------------------------|----------+
       |       +------+      |                         |
       |               \     |                         |
   +----------+       +----------+                 +----------+
   | Client 1 |       | Client 2 |                 | Client 3 |
   +----------+       +----------+                 +----------+

tvi   = TIPS view i
tvi/ug = incremental updates graph associated with tvi
~~~~
{: #fig-overview artwork-align="center" title="Overview of ALTO TIPS"}

{{fig-overview}} shows an example illustrating an overview of the ALTO TIPS
service. The server provides the TIPS service of two information resources (#1
and #2) where #1 is an ALTO map service, and #2 is a filterable
service. There are 3 ALTO clients (Client 1, Client 2, and Client 3) that are
connected to the ALTO server.

Each client uses the TIPS view to retrieve updates. Specifically, a TIPS view
(tv1) is created for the map service #1, and is shared by multiple clients. For
the filtering service #2, two different TIPS views (tv2 and tv3) are created upon
different client requests with different filter sets.

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

-  Each node in the graph is a version of the resource, where a tag identifies the
   content of the version (tag is valid only within the scope of resource).
   Version 0 is reserved as the initial state (empty/null).

-  Each edge is an update item. In particular, the edge from i to j is the update
   item to transit from version i to version j.

-  Version is path-independent (different paths arrive at the same version/node
   has the same content)

A concrete example is shown in {{fig-ug}}. There are 7 nodes in the graph,
representing 7 different versions of the resource. Edges in the figure represent
the updates from the source version to the target version. Thick lines represent
mandatory incremental updates (e.g., ID103-104), dotted lines represent optional
incremental updates (e.g., ID103-105), and thin lines represent snapshots (e.g.,
ID0-103). Note that node content is path independent: the content of node v can
be obtained by applying the updates from any path that ends at v. For example,
assume the latest version is 105 and a client already has version 103. The base
version of the client is 103 as it serves as a base upon which incremental
updates can be applied. The target version 105 can either be directly fetched as
a snapshot, computed incrementally by applying the incremental updates between
103 and 104, then 104 and 105, or if the optional update from 103 to 105 exists,
computed incrementally by taking the "shortcut" path from 103 to 105.

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

# TIPS Workflow and Resource Location Schema {#workflow}

## Workflow {#workflow-overview}

At a high level, an ALTO client first uses the TIPS service (denoted as TIPS-F
and F is for frontend) to indicate the information resource(s) that the client
wants to monitor. For each requested resource, the server returns a JSON object
that contains a URI, which points to the root of a TIPS view (denoted as
TIPS-V), and a summary of the current view, which contains the information to
correctly interact with the current view. With the URI to the root of a TIPS
view, clients can construct URIs (see {{schema}}) to fetch incremental updates.

An example workflow is shown in {{fig-workflow-pull}}. After the TIPS-F
service receives the request from the client to monitor the updates of an ALTO
resource, it creates a TIPS view service and returns the corresponding
information to the client. The URI points to that specific TIPS-V instance and
the summary contains the start-seq and end-seq of the update graph, and a
server-recommended edge to consume first, e.g., from i to j.

An ALTO client can then continuously pull each additional update with the
information. For example, the client in {{fig-workflow-pull}} first fetches the
update from i to j, and then from j to j+1. Note that the update item at
`<tips-view-uri>/ug/<j>/<j+1>` may not yet exist, so the server holds the
request until the update becomes available (long polling).

A server MAY close a TIPS view at any time, e.g., under high system load or due
to client inactivity. In the event that a TIPS view is closed, an edge request
will receive error code 404 in response, and the client will have to request a
new TIPS view URI.

If resources allow, servers SHOULD avoid closing TIPS views that have active
polling edge requests or have recently served responses until clients have had a
reasonable interval to request the next update.

~~~~ drawing
Client                                 TIPS-F           TIPS-V
  o                                       .                .
  | POST to create/receive a TIPS view    .  Create TIPS   .
  |           for resource 1              .      View      .
  |-------------------------------------> |.-.-.-.-.-.-.-> |
  | <tips-view-uri>, <tips-view-summary>  .                |
  | <-------------------------------------| <-.-.-.-.-.-.-.|
  |                                                        .
  | GET /<tips-view-path>/ug/<i>/<j>                       .
  |------------------------------------------------------> |
  | content on edge i to j                                 |
  | <------------------------------------------------------|
  |                                                        .
  | GET /<tips-view-path>/ug/<j>/<j+1>                     .
  |------------------------------------------------------> |
  .                                                        .
  .                                                        .
  | content on edge j to j+1                               |
  | <------------------------------------------------------|
  |                                                        .
  o                                                        .
                                                           .
                                         TIPS View Closed  o
~~~~
{: #fig-workflow-pull artwork-align="center" title="ALTO TIPS Workflow Supporting Client Pull"}

## Resource Location Schema {#schema}

The resource location schema defines how a client constructs URI to fetch
incremental updates.

To access each update in an updates graph, consider the model
represented as a "virtual" file system (adjacency list), contained within the
root of a TIPS view URI (see {{open-resp}} for the definition of tips-view-uri).
For example, assuming that the update graph of a TIPS view is as shown in
{{fig-ug}}, the location schema of this TIPS view will have the format as in
{{fig-ug-schema}}.

~~~~ drawing
  <tips-view-path>  // root path to a TIPS view
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
    \_ ...
~~~~
{: #fig-ug-schema artwork-align="center" title="Location Schema Example"}

TIPS uses this directory schema to generate template URIs which allow
clients to construct the location of incremental updates after receiving the
tips-view-uri from the server. The generic template for the location of the
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
    <tips-view-uri>/ug/<end-seq>/<end-seq + 1>
~~~

Incremental updates of a TIPS view are read-only. Thus, they are fetched using
the HTTP GET method.

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
{: #tips-cap artwork-align="left" title="TIPSCapabilities"}

with field:

incremental-change-media-types:
:  If a TIPS can provide updates with incremental changes for a
   resource, the "incremental-change-media-types" field has an entry
   for that resource-id, and the value is the supported media types
   of incremental changes, separated by commas.  For the
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
R0 has changed, it must request the full content of R0, which is likely to be
less efficient than receiving the incremental updates of R0.

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
{: #ex-ird artwork-align="left" title="Example of an ALTO Server Supporting ALTO Base Protocol, ALTO/SSE, and ALTO TIPS"}

Note that it is straightforward for an ALTO server to run HTTP/2 and
support concurrent retrieval of multiple resources such as "my-
network-map" and "my-routingcost-map" using multiple HTTP/2 streams.

The resource "update-my-costs-tips" provides an ALTO TIPS service, and this is
indicated by the media-type "application/ alto-tips+json".

# TIPS Management

Upon request, a server sends a TIPS view to a client. This TIPS view may be
created at the time of the request or may already exist (either because another
client has already created a TIPS view for the same requested network resource
or because the server perpetually maintains a TIPS view for an often-requested
resource).

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
{: #fig-open-req artwork-align="left" title="TIPSReq"}


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
      URI               tips-view-uri;
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
{: #fig-open-resp artwork-align="left" title="AddTIPSResponse"}

with the following fields:

tips-view-uri:
:  URI to the requested TIPS view. The value of this field MUST have the
   following format:

   ~~~
       scheme "://" tips-view-host "/" tips-view-path

       tips-view-host = host [ ":" port]
       tips-view-path = path
   ~~~

   where scheme MUST be "http" or "https" unless specified by a future
   extension, and host, port and path are as specified in Sections 3.2.2, 3.2.3,
   and 3.3 in {{RFC3986}}.

   A server SHOULD NOT use properties that are not included in the request body
   to determine the URI of a TIPS view, such as cookies or the client's IP
   address. Furthermore, TIPS MUST NOT reuse a URI for a different TIPS view
   (different resources or different request bodies).

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
   the first incremental update edge starting from the client's tagged
   version.  Otherwise, the server should recommend the latest snapshot
   edge.

If the request has any errors, the TIPS service MUST return an HTTP
"400 Bad Request" to the ALTO client; the body of the response
follows the generic ALTO error response format specified in
Section 8.5.2 of {{RFC7285}}.  Hence, an example ALTO error response
has the format shown in {{ex-bad-request}}.

~~~~
    HTTP/1.1 400 Bad Request
    Content-Length: 131
    Content-Type: application/alto-error+json

    {
        "meta":{
            "code":  "E_INVALID_FIELD_VALUE",
            "field": "resource-id",
            "value": "my-network-map/#"
        }
    }
~~~~
{: #ex-bad-request artwork-align="left" title="ALTO Error Example"}

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

- 429 (Too Many Requests): when the number of TIPS views open requests exceeds
  the server threshold. The server may indicate when to re-try the request in
  the "Re-Try After" headers.

##  Open Example

For simplicity, assume that the ALTO server is using the Basic
authentication.  If a client with username "client1" and password
"helloalto" wants to create a TIPS view of an ALTO Cost Map resource
with resource ID "my-routingcost-map", it can send the
request depicted in {{ex-op}}.

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
{: #ex-op artwork-align="left" title="Open Example"}

If the operation is successful, the ALTO server returns the
message shown in {{ex-op-rep}}.

~~~~
    HTTP/1.1 200 OK
    Content-Type: application/alto-tips+json
    Content-Length: 258

    {
      "tips-view-uri": "https://alto.example.com/tips/2718281828459",
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
{: #ex-op-rep artwork-align="left" title="Response Example"}

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
    GET /<tips-view-path>/ug/<i>/<j>
    HOST: <tips-view-host>
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

A client may conduct proactive fetching of future updates, by
long polling updates that have not been listed in the directory yet. For
long-poll prefetch, the client must have indicated the media type that may
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
   closed by the server.

-  410 (Gone): if an update has a seq that is smaller than the start-
   seq.

-  415 (Unsupported Media Type): if the media type(s) accepted by the
   client does not include the media type of the update chosen by the
   server.

-  425 (Too Early): if the seq exceeds the server prefetch window

-  429 (Too Many Requests): when the number of pending (long-poll)
   requests exceeds the server threshold. The server may indicate when to re-try
   the request in the "Re-Try After" headers.

##  Example {#iu-example}

Assume the client wants to get the contents of the update item on
edge 0 to 101.  The format of the request is shown in {{ex-get}}.

~~~~
    GET /tips/2718281828459/ug/0/101 HTTP/1.1
    Host: alto.example.com
    Accept: application/alto-costmap+json, \
              application/alto-error+json
~~~~
{: #ex-get artwork-align="left" title="GET Example"}

The response is shown in {{ex-get-res}}.

~~~~
    HTTP/1.1 200 OK
    Content-Type: application/alto-costmap+json
    Content-Length: 50

    { ... full replacement of my-routingcost-map ... }
~~~~
{: #ex-get-res artwork-align="left" title="Response to a GET Request"}

## New Next Edge Recommendation

While intended TIPS usage is for the client to receive a recommended
starting edge in the TIPS summary, consume that edge, then construct
all future URIs by incrementing the sequence count by 1, there may be
cases in which the client needs to request a new next edge to
consume.  For example, if a client has an open TIPS view yet has not
polled in a while, the client may request the next logical
incremental URI but the server has compacted the updates graph so it
no longer exists.  Thus, the client must request a new next edge to
consume based on its current version of the resource.

###  Request

An ALTO client requests that the server provide a next edge recommendation for a
given TIPS view by sending an HTTP POST request with the media type
"application/alto-tipsparams+json". The request has the form:

~~~~
    POST /<tips-view-path>/ug HTTP/1.1
    HOST: <tips-view-host>
~~~~

The POST body has the following form, where providing the version
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
{: #fig-resp artwork-align="left" title="UpdatesGraphSummary"}


It is RECOMMENDED that the server uses the following HTTP codes to
indicate errors, with the media type "application/alto-error+json",
regarding new next edge requests.

-  404 (Not Found): if the requested TIPS view does not exist or is
   closed by the server.

-  415 (Unsupported Media Type): if the media type(s) accepted by the
   client does not include the media type `application/alto-tips+json`.

# Operation and Processing Considerations

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
simpler to apply.  There is no advantage in allowing a server to use
JSON patch for those resources.

The case is not as clear for incremental changes to network maps.

First, consider small changes, such as moving a prefix from one PID
to another.  JSON patch could encode that as a simple insertion and
deletion, while JSON merge patch would have to replace the entire
array of prefixes for both PIDs.  On the other hand, to process a
JSON patch update, the ALTO client would have to retain the indexes
of the prefixes for each PID.  Logically, the prefixes in a PID are
an unordered set, not an array; aside from handling updates, a client
does not need to retain the array indexes of the prefixes.  Hence, to
take advantage of JSON patch for network maps, ALTO clients would
have to retain additional, otherwise unnecessary, data.

Second, consider more involved changes, such as removing half of the
prefixes from a PID.  JSON merge patch would send a new array for
that PID, while JSON patch would have to send a list of remove
operations and delete the prefix one by one.

Therefore, each TIPS instance may choose to encode the updates using
JSON merge patch or JSON patch based on the type of changes in
network maps.

## Considerations for Load Balancing {#load-balancing}

There are two levels of load balancing in TIPS. The first level is to balance
the load of TIPS views for different clients, and the second is to balance the
load of incremental updates.

Load balancing of TIPS views can be achieved either at the application layer or
at the infrastructure layer. For example, an ALTO server may set
`<tips-view-host>` to different subdomains to distribute TIPS views, or simply
use the same host of the TIPS service and rely on load balancers to distribute
the load.

TIPS allows a client to make concurrent pulls of incremental updates for the
same TIPS view potentially through different HTTP connections. As a consequence,
it introduces additional complexities when the ALTO server is being load
balanced. For example, a request may be directed to a wrong backend server and
get incorrectly processed if the following two conditions both hold:

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
   distribute requests based on the tips-view-path component in the URI.

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

Thus, the client must decide the order in which to receive and apply the
updates. The order may affect how fast the client can build a consistent view
and how long the client needs to buffer the update.

-  Example 1: The client requests N89, N90, N91, C101, C102 in that
   order.  The client either gets no consistent view of the resources
   or has to buffer N90 and N91.

-  Example 2: The client requests C101, C102, C103, N89.  The client
   either gets no consistent view or has to buffer C103.

Therefore, the client is RECOMMENDED to request and process updates in the
ascending order of the smallest dependent tag, e.g., {C101, C102, N89} before
{C103, N90}

## Considerations for Client Processing Updates {#client-processing}

In general, when an ALTO client receives a full replacement for a
resource, the ALTO client should replace the current version with the
new version.  When an ALTO client receives an incremental update for
a resource, the ALTO client should apply those updates to the current
version of the resource.

However, because resources can depend on other resources (e.g., cost
maps depend on network maps), an ALTO client must not use a dependent
resource if the resource on which it depends has changed.  There are
at least two ways an ALTO client can do that.  The following
paragraphs illustrate these techniques by referring to network and
cost map messages, although these techniques apply to any dependent
resources.

Note that when a network map changes, the server should send the
network map update message before sending the updates for the
dependent cost maps.

One approach is for the ALTO client to save the network map update
message in a buffer and continue to use the previous network map and
the associated cost maps until the ALTO client receives the update
messages for all dependent cost maps.  The ALTO client then applies
all network and cost map updates atomically.

Alternatively, the ALTO client may update the network map
immediately.  In this case, the cost maps using the network map
become invalid because they are inconsistent with the current network
map; hence, the ALTO client must mark each such dependent cost map as
temporarily invalid and must not use each such cost map until the
ALTO client receives a cost map update indicating that it is based on
the new network map version tag.

Though a server should send update items sequentially, it is possible that a
client receives the update items out of order (in the case of a retransmitted
update item or a result of concurrent fetch). The client must buffer the update
items if they arrive out of order and then apply them sequentially (based on
the sequence numbers) due to the operation of JSON merge patch and JSON patch.

## Considerations for Updates to Filtered Cost Maps

If TIPS provides updates to a Filtered Cost Map that allows
constraint tests, then an ALTO client may request updates to a
Filtered Cost Map request with a constraint test.  In this case, when
a cost changes, the updates graph MUST have an update if the new
value satisfies the test.  If the new value does not, whether there
is an update depends on whether the previous value satisfies the
test.  If it did not, the updates graph SHOULD NOT have an update.
But if the previous value did, then the updates graph MUST add an
update with a "null" value to inform the ALTO client that this cost
no longer satisfies the criteria.

TIPS can avoid having to handle such complicated behavior by
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
   each client.

-  An ALTO server maintains one copy of the TIPS view for each common
   resource and all clients requesting the same resources use the
   same copy.  There are two ways to manage the storage for the
   shared copy:

   - the ALTO server maintains the set of clients that have sent a polling
     request to the TIPS view, and only removes the view from the storage when
     the set becomes empty and no client immediately issues a new edge request;

   - the TIPS view is never removed from the storage.

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

Allowing TIPS views enables new classes of Denial-of-Service attacks. In
particular, for the TIPS server, one or multiple malicious ALTO clients might
create an excessive number of TIPS views, to exhaust the server resource and/or
to block normal users from the accessing the service.

To avoid such attacks, the server SHOULD choose to limit the number of active
views and reject new requests when that threshold is reached. TIPS allows
predictive fetching and the server SHOULD also choose to limit the number of
pending requests. If a new request exceeds the threshold, the server SHOULD log
the event and may return the HTTP status "429 Too many requests".

It is important to note that the preceding approaches are not the only
possibilities. For example, it may be possible for TIPS to use somewhat more
clever logic involving TIPS view eviction policies, IP reputation,
rate-limiting, and compartmentalization of the overall threshold into smaller
thresholds that apply to subsets of potential clients. If service availability
is a concern, ALTO clients may establish service level agreements with the ALTO
server.

## ALTO Client: Update Overloading or Instability

The availability of continuous updates can also cause overload for an ALTO
client, in particular, an ALTO client with limited processing capabilities. The
current design does not include any flow control mechanisms for the client to
reduce the update rates from the server. For example, TCP, HTTP/2 and QUIC
provide stream and connection flow control data limits, which might help prevent
the client from being overloaded. Under overloading, the client MAY choose to
remove the information resources with high update rates.

Also, under overloading, the client may no longer be able to detect
whether information is still fresh or has become stale.  In such a
case, the client should be careful in how it uses the information to
avoid stability or efficiency issues.

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
: This document specifies the format of conforming messages and the interpretation
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
: This document specifies the format of conforming messages and the interpretation
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

# A High-Level Deployment Model {#sec-dep-model}

Conceptually, the TIPS system consists of three types of resources:

- (R1) TIPS frontend to create TIPS views.

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
 |      |<------------|-----------------------|     |         |     |  |
 |Client|             |     |     |           +-----+         +-----+  |
 |      | "iget" base |     |     |                                    |
 |      | resource 2  |     |     |           +-----+         +-----+  |
 |      |-------------|---->|     |           |     |         |     |  |
 |      | incremental |     |     |           |     |         |     |  |
 |      | transfer    |     +-----+           |     | ------->|     |  |
 |      | resource    |                       |     |         |     |  |
 |      |<------------|-----------------------|     |         |     |  |
 +------+             |                       +-----+         +-----+  |
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

This document supports Design 1 and Design 3. For Design 1, the TIPS service
simply needs to always use the same host for the TIPS views. For Design 3, the
TIPS service can set tips-view-host to a different server. Note that the
deployment flexibility is at the logical level, as these services
can be distinguished by different paths and potentially be routed to different
physical servers by layer-7 load balancing. See {{load-balancing}} for a
discussion on load balancing considerations. Future documents may extend the
protocol to support Design 2.

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
   also allows the application to tailor the "discovery document" to the client"
   ({{Section 4.4.1 of RFC9205}}).

-  TIPS uses existing HTTP schemes ({{Section 4.4.2 of RFC9205}}).

-  TIPS defines its errors "to use the most applicable status code"
   ({{Section 4.6 of RFC9205}}).

-  TIPS does not "make assumptions about the relationship between
   separate requests on a single transport connection; doing so
   breaks many of the assumptions of HTTP as a stateless protocol and
   will cause problems in interoperability, security, operability,
   and evolution" ({{Section 4.11 of RFC9205}}).  The only relationship
   between requests is that a client must first discover where a TIPS view of
   a resource will be served, which is consistent with the URI discovery in
   {{Section 4.4.1 of RFC9205}}.

# Push-mode TIPS using HTTP Server Push {#push}

TIPS allows ALTO clients to subscribe to incremental updates of an ALTO
resource, and the specification in this document is based on the current best
practice of building such a service using native HTTP. Earlier versions of this
document had investigated the possibility of enabling push-mode TIPS, i.e., by
taking advantage of the server push feature in HTTP/2 and HTTP/3.

In the ideal case, push-mode TIPS can potentially improve performance (e.g.,
latency) in more dynamic environments and use cases, with wait-free message
delivery. Using native server push also results in minimal changes to the
current protocol. While not adopted due to the lack of server push support and
increased protocol complexity, push-mode TIPS remains a potential direction of
protocol improvement.

# Persistent HTTP Connections

Previous versions of this document use persistent HTTP connections to detect the
liveness of clients. This design, however, does not conform well with the best
current practice of HTTP. For example, if an ALTO client is accessing a TIPS
view over an HTTP proxy, the connection is not established directly between the
ALTO client and the ALTO server, but between the ALTO client and the proxy and
between the proxy and the ALTO server. Thus, using persistent connections may
not correctly detect the right liveness state.

# Acknowledgments
{:numbered="false"}

The authors of this document would like to thank Mark Nottingham and Spencer
Dawkins for providing invaluable reviews of earlier versions of this document,
Adrian Farrel, Qin Wu, and Jordi Ros Giralt for their continuous feedback, Russ
White, Donald Eastlake, Martin Thomson, Bernard Adoba, Spencer Dawkins, and
Sheng Jiang for the directorate reviews, Martin Duke for the Area Director
review, and Mohamed Boucadair for shepherding the document.
