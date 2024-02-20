---
slug: 13
title: 13/WAKU2-STORE
name: Waku v2 Store
status: draft
tags: waku-core
editor: Sanaz Taheri <sanaz@status.im>
contributors:
  - Dean Eigenmann <dean@status.im>
  - Oskar Thorén <oskarth@titanproxy.com>
  - Aaryamann Challani <aaryamann@status.im>
---

# Abstract
This specification explains the `13/WAKU2-STORE` protocol which enables querying of messages received through the relay protocol and 
stored by other nodes. 
It also supports pagination for more efficient querying of historical messages. 

**Protocol identifier***: `/vac/waku/store-query/3.0.0`

## Terminology
The term PII, Personally Identifiable Information, 
refers to any piece of data that can be used to uniquely identify a user. 
For example, the signature verification key, and 
the hash of one's static IP address are unique for each user and hence count as PII.

# Design Requirements
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, 
“RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt).

Nodes willing to provide the storage service using `13/WAKU2-STORE` protocol,
SHOULD provide a complete and full view of message history.
As such, they are required to be *highly available* and 
specifically have a *high uptime* to consistently receive and store network messages. 
The high uptime requirement makes sure that no message is missed out hence a complete and 
intact view of the message history is delivered to the querying nodes.
Nevertheless, in case storage provider nodes cannot afford high availability, 
the querying nodes may retrieve the historical messages from multiple sources to achieve a full and intact view of the past.

The concept of `ephemeral` messages introduced in [`14/WAKU2-MESSAGE`](/spec/14) affects `13/WAKU2-STORE` as well.
Nodes running `13/WAKU2-STORE` SHOULD support `ephemeral` messages as specified in [14/WAKU2-MESSAGE](/spec/14).
Nodes running `13/WAKU2-STORE` SHOULD NOT store messages with the `ephemeral` flag set to `true`.

# Adversarial Model
Any peer running the `13/WAKU2-STORE` protocol, i.e. 
both the querying node and the queried node, are considered as an adversary. 
Furthermore, 
we currently consider the adversary as a passive entity that attempts to collect information from other peers to conduct an attack but 
it does so without violating protocol definitions and instructions. 
As we evolve the protocol, 
further adversarial models will be considered.
For example, under the passive adversarial model, 
no malicious node hides or 
lies about the history of messages as it is against the description of the `13/WAKU2-STORE` protocol. 

The following are not considered as part of the adversarial model:
- An adversary with a global view of all the peers and their connections.
- An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). 
In specific, the communication channels are assumed to be secure.

# Wire Specification

## Payloads

```protobuf
syntax = "proto3";

// Protocol identifier: /vac/waku/store-query/3.0.0
package waku.store.v3;

import "waku/message/v1/message.proto";

message WakuMessageKeyValue {
  optional bytes message_hash = 1; // Globally unique key for a Waku Message
  optional waku.message.v1.WakuMessage message = 2; // Full message content as value
}

message StoreQueryRequest {
  string request_id = 1;
  bool include_data = 2; // Response should include full message content
  
  // Filter criteria for content-filtered queries
  optional string pubsub_topic = 10;
  repeated string content_topics = 11;
  optional int64 time_start = 12;
  optional int64 time_end = 13;

  // List of key criteria for lookup queries
  repeated bytes message_hashes = 20 // Message hashes (keys) to lookup
  
  // Pagination info. 50 Reserved
  optional bytes pagination_cursor = 51; // Message hash (key) from where to start query (exclusive)
  bool pagination_forward = 52;
  optional uint64 pagination_limit = 53;
}

message StoreQueryResponse {
  string request_id = 1;

  optional uint32 status_code = 10;
  optional string status_desc = 11;

  repeated WakuMessageKeyValue messages = 20;

  optional bytes pagination_cursor = 51;
}
```
## General store query concepts

### Waku message key-value pairs

The store query protocol operates as a query protocol for a key-value store of historical Waku messages,
with each entry having a [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/) as value
and [deterministic message hash](https://rfc.vac.dev/spec/14/#deterministic-message-hashing) as key.
The store can be queried to return either a set of keys or a set of key-value pairs.
Within the store query protocol, Waku message keys and values MUST be represented in a `WakuMessageKeyValue` message.
This message MUST contain the deterministic `message_hash` as key.
It MAY contain the full `WakuMessage` as value in the `message` field,
depending on the use case as set out below.

### Waku message store eligibility

In order for a Waku message to be eligible for storage:
- it MUST be a _valid_ [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/).
- the `timestamp` field MUST be populated with the Unix epoch time at which the message was generated in nanoseconds.
If at the time of storage the `timestamp` deviates by more than 20 seconds
either into the past or the future when compared to the store node’s internal clock,
the store node MAY reject the message.
- the `ephemeral` field MUST be set to `false`.

### Waku message sorting

The key-value entries in the store MUST be time-sorted by the `WakuMessage` `timestamp` attribute.
Where two or more key-value entries have identical `timestamps`,
the entries MUST be further sorted by the natural order of their message hash keys.
Within the context of traversing over key-value entries in the store,
_"forward"_ indicates traversing the entries in ascending order,
whereas _"backward"_ indicates traversing the entries in descending order.

### Pagination

If a large number of entries in the store service node match the query criteria provided in a `StoreQueryRequest`,
the client MAY make use of pagination
in a chain of store query request and response transactions
to retrieve the full response in smaller batches termed _"pages"_.
Pagination can be performed either in [a _forward_ or _backward_ direction](#waku-message-sorting).

A store query client MAY indicate the maximum number of matching entries it wants in the `StoreQueryResponse`,
by setting the page size limit in the `pagination_limit` field.
Note that a store service node MAY enforce its own limit
if the `pagination_limit` is unset
or larger than the service node's internal page size limit.

A `StoreQueryResponse` with a populated `pagination_cursor` indicates that more stored entries match the query than included in the response.

A `StoreQueryResponse` without a populated `pagination_cursor` indicates that
there are no more matching entries in the store.

The client MAY request the next page of entries from the store service node
by populating a subsequent `StoreQueryRequest` with the `pagination_cursor` received in the `StoreQueryResponse`.
All other fields and query criteria MUST be the same as in the preceding `StoreQueryRequest`.

A `StoreQueryRequest` without a populated `pagination_cursor` indicates that
the client wants to retrieve the "first page" of the stored entries matching the query.

## Store Query Request

A client node MUST send all historical message queries within a `StoreQueryRequest` message.
This request MUST contain a `request_id`.
The `request_id` MUST be a uniquely generated string.

If the store query client requires the store service node to include Waku message values in the query response,
it MUST set `include_data` to `true`.
If the store query client requires the store service node to return only message hash keys in the query response,
it SHOULD set `include_data` to `false`.
By default, therefore, the store service node assumes `include_data` to be `false`.

A store query client MAY include query filter criteria in the `StoreQueryRequest`.
There are two types of filter use cases:
1. Content filtered queries and
2. Message hash lookup queries

### Content filtered queries

A store query client MAY request the store service node to filter historical entries by a content filter.
Such a client MAY create a filter on content topic, on time range or on both.

To filter on content topic, the client MUST populate _both_ the `pubsub_topic` _and_ `content_topics` field.
The client MUST NOT populate either `pubsub_topic` or `content_topics` and leave the other unset.
Both fields MUST either be set or unset.
A mixed content topic filter with just one of either `pubsub_topic` or `content_topics` set, SHOULD be regarded as an invalid request.

To filter on time range, the client MUST set `time_start`, `time_end` or both.
Each `time_` field should contain a Unix epoch timestamp in nanoseconds.
An unset `time_start` SHOULD be interpreted as "from the oldest stored entry".
An unset `time_end` SHOULD be interpreted as "up to the youngest stored entry".

If any of the content filter fields are set,
namely `pubsub_topic`, `content_topic`, `time_start`, or `time_end`,
the client MUST NOT set the `message_hashes` field.

### Message hash lookup queries

A store query client MAY request the store service node to filter historical entries by one or more matching message hash keys.
This type of query acts as a "lookup" against a message hash key or set of keys already known to the client.

In order to perform a lookup query, the store query client MUST populate the `message_hashes` field with the list of message hash keys it wants to lookup in the store service node.

If the `message_hashes` field is set,
the client MUST NOT set any of the content filter fields,
namely `pubsub_topic`, `content_topic`, `time_start`, or `time_end`.

### Presence queries

A presence query is a special type of lookup query that allows a client to check for the presence of one or more messages in the store service node,
without retrieving the full contents (values) of the messages.
This can, for example, be used as part of a reliability mechanism,
whereby store query clients verify that previously published messages have been successfully stored.

In order to perform a presence query,
the store query client MUST populate the `message_hashes` field in the `StoreQueryRequest` with the list of message hashes
for which it wants to verify presence in the store service node.
The `include_data` property MUST be set to `false`.
The client SHOULD interpret every `message_hash` returned in the `messages` field of the `StoreQueryResponse` as present in the store.
The client SHOULD assume that all other message hashes included in the original `StoreQueryRequest` but not in the `StoreQueryResponse` is not present in the store.

### Pagination info

The store query client MAY include a message hash as `pagination_cursor`,
to indicate at which key-value entry a store service node SHOULD start the query.
The `pagination_cursor` is treated as exclusive
and the corresponding entry will not be included in subsequent store query responses.

For forward queries, only messages following (see [sorting](#waku-message-sorting)) the one indexed at `pagination_cursor` will be returned.
For backward queries, only messages preceding (see [sorting](#waku-message-sorting)) the one indexed at `pagination_cursor` will be returned.

If the store query client requires the store service node to perform a forward query,
it MUST set `pagination_forward` to `true`.
If the store query client requires the store service node to perform a backward query,
it SHOULD set `pagination_forward` to `false`.
By default, therefore, the store service node assumes pagination to be backward.

A store query client MAY indicate the maximum number of matching entries it wants in the `StoreQueryResponse`,
by setting the page size limit in the `pagination_limit` field.
Note that a store service node MAY enforce its own limit
if the `pagination_limit` is unset
or larger than the service node's internal page size limit.

See [pagination](#pagination) for more on how the pagination info is used in store transactions.

## Store Query Response

In response to any `StoreQueryRequest`,
a store service node SHOULD respond with a `StoreQueryResponse` with a `requestId` matching that of the request.
This response MUST contain a `status_code` indicating if the request was successful or not.
Successful status codes are in the `2xx` range.
Client nodes SHOULD consider all other status codes as error codes and assume that the requested operation had failed.
In addition, the store service node MAY choose to provide a more detailed status description in the `status_desc` field.

### Filter matching

For [content filtered queries](#content-filtered-queries), an entry in the store service node matches the filter criteria in a `StoreQueryRequest` if each of the following conditions are met:
- its `content_topic` is in the request `content_topics` set
and it was published on a matching `pubsub_topic` OR the request `content_topics` and `pubsub_topic` fields are unset
- its `timestamp` is _larger or equal_ than the request `start_time` OR the request `start_time` is unset
- its `timestamp` is _smaller_ than the request `end_time` OR the request `end_time` is unset

Note that for content filtered queries, `start_time` is treated as _inclusive_ and `end_time` is treated as _exclusive_.

For [message hash lookup queries](#message-hash-lookup-queries), an entry in the store service node matches the filter criteria if its `message_hash` is in the request `message_hashes` set.

The store service node SHOULD respond with an error code and discard the request
if the store query request contains both content filter criteria and message hashes.

### Populating response messages

The store service node SHOULD populate the `messages` field in the response
only with entries matching the filter criteria provided in the corresponding request.
Regardless of whether the response is to a _forward_ or _backward_ query,
the `messages`field in the response MUST be ordered in a forward direction
according to the [message sorting rules](#waku-message-sorting).

If the corresponding `StoreQueryRequest` has `include_data` set to true,
the service node SHOULD populate both the `message_hash` and `message` for each entry in the response.
In all other cases, the store service node SHOULD populate only the `message_hash` field for each entry in the response.

### Paginating the response

The response SHOULD NOT contain more `messages` than the `pagination_limit` provided in the corresponding `StoreQueryRequest`.
It is RECOMMENDED that the store node defines its own maximum page size internally.
If the `pagination_limit` in the request is unset,
or exceeds this internal maximum page size,
the store service node SHOULD ignore the `pagination_limit` field and apply its own internal maximum page size.

In response to a _forward_ `StoreQueryRequest`:
- if the `pagination_cursor` is set,
  the store service node SHOULD populate the `messages` field
  with matching entries following the `pagination_cursor` (exclusive).
- if the `pagination_cursor` is unset,
  the store service node SHOULD populate the `messages` field
  with matching entries from the first entry in the store.
- if there are still more matching entries in the store
  after the maximum page size is reached while populating the response,
  the store service node SHOULD populate the `pagination_cursor` in the `StoreQueryResponse`
  with the message hash key of the _last_ entry _included_ in the response.

In response to a _backward_ `StoreQueryRequest`:
- if the `pagination_cursor` is set,
  the store service node SHOULD populate the `messages` field
  with matching entries preceding the `pagination_cursor` (exclusive).
- if the `pagination_cursor` is unset,
  the store service node SHOULD populate the `messages` field
  with matching entries from the last entry in the store.
- if there are still more matching entries in the store
  after the maximum page size is reached while populating the response,
  the store service node SHOULD populate the `pagination_cursor` in the `StoreQueryResponse`
  with the message hash key of the _first_ entry _included_ in the response.

# Security Consideration

The main security consideration to take into account while using this protocol is that a querying node have to reveal their content filters of interest to the queried node, hence potentially compromising their privacy.

# Future Work

- **Anonymous query**: This feature guarantees that nodes can anonymously query historical messages from other nodes i.e.,
without disclosing the exact topics of [14/WAKU2-MESSAGE](/spec/14) they are interested in.  
As such, no adversary in the `13/WAKU2-STORE` protocol would be able to learn which peer is interested in which content filters i.e.,
content topics of [14/WAKU2-MESSAGE](/spec/14). 
The current version of the `13/WAKU2-STORE` protocol does not provide anonymity for historical queries,
as the querying node needs to directly connect to another node in the `13/WAKU2-STORE` protocol and
explicitly disclose the content filters of its interest to retrieve the corresponding messages. 
However, one can consider preserving anonymity through one of the following ways: 
  - By hiding the source of the request i.e., anonymous communication.
  That is the querying node shall hide all its PII in its history request e.g., its IP address.
  This can happen by the utilization of a proxy server or by using Tor. 
  Note that the current structure of historical requests does not embody any piece of PII, otherwise,
  such data fields must be treated carefully to achieve query anonymity. 
  <!-- TODO: if nodes have to disclose their PeerIDs (e.g., for authentication purposes) when connecting to other nodes in the store protocol, then Tor does not preserve anonymity since it only helps in hiding the IP. So, the PeerId usage in switches must be investigated further. Depending on how PeerId is used, one may be able to link between a querying node and its queried topics despite hiding the IP address--> 
  - By deploying secure 2-party computations in which the querying node obtains the historical messages of a certain topic,
  the queried node learns nothing about the query. 
  Examples of such 2PC protocols are secure one-way Private Set Intersections (PSI). 
  <!-- TODO: add a reference for PSIs? --> <!-- TODO: more techniques to be included --> 
<!-- TODO: Censorship resistant: this is about a node that hides the historical messages from other nodes. This attack is not included in the specs since it does not fit the passive adversarial model (the attacker needs to deviate from the store protocol).-->

- **Robust and verifiable timestamps**: Messages timestamp is a way to show that the message existed prior to some point in time.
However, the lack of timestamp verifiability can create room for a range of attacks,
including injecting messages with invalid timestamps pointing to the far future.   
To better understand the attack,
consider a store node whose current clock shows `2021-01-01 00:00:30` (and assume all the other nodes have a synchronized clocks +-20seconds).
The store node already has a list of messages,
 `(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20)`,
that are sorted based on their timestamp.  
An attacker sends a message with an arbitrary large timestamp e.g.,
10 hours ahead of the correct clock `(m',2021-01-01 10:00:30)`. 
The store node places `m'` at the end of the list,
`(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20), (m',2021-01-01 10:00:30)`. 
Now another message arrives with a valid timestamp e.g.,
`(m11, 2021-01-01 00:00:45)`.
However, since its timestamp precedes the malicious message `m'`,
it gets placed before `m'` in the list i.e.,
`(m1,2021-01-01 00:00:00), (m2,2021-01-01 00:00:01), ..., (m10:2021-01-01 00:00:20), (m11, 2021-01-01 00:00:45), (m',2021-01-01 10:00:30)`.
In fact, for the next 10 hours,
`m'` will always be considered as the most recent message and
served as the last message to the querying nodes irrespective of how many other messages arrive afterward. 

A robust and verifiable timestamp allows the receiver of a message to verify that a message has been generated prior to the claimed timestamp. 
One solution is the use of [open timestamps](https://opentimestamps.org/) e.g.,
block height in Blockchain-based timestamps. 
That is, messages contain the most recent block height perceived by their senders at the time of message generation. 
This proves accuracy within a range of minutes (e.g., in Bitcoin blockchain) or 
seconds (e.g., in Ethereum 2.0) from the time of origination. 

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References
1. [14/WAKU2-MESSAGE](/spec/14)
2. [protocol buffers v3](https://developers.google.com/protocol-buffers/)
3. [11/WAKU2-RELAY](/spec/11)
4. [Open timestamps](https://opentimestamps.org/) 

