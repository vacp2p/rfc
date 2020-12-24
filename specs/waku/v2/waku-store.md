---
title: WakuStore
version: 2.0.0-beta2
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>, Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Design Requirements](#design-requirements)
- [Security Consideration](#security-consideration)
  - [Terminology](#terminology)
- [Adversarial Model](#adversarial-model)
- [Wire Specification](#wire-specification)
  - [Protobuf](#protobuf)
    - [Index](#index)
    - [PagingInfo](#paginginfo)
    - [HistoryQuery](#historyquery)
    - [HistoryResponse](#historyresponse)
- [Future Work](#future-work)
- [Changelog](#changelog)
    - [Next](#next)
    - [2.0.0-beta2](#200-beta2)
    - [2.0.0-beta1](#200-beta1)
- [Copyright](#copyright)

# Abstract

This specification explains the Waku `WakuStore` protocol which enables querying of messages received through relay protocol and stored by other nodes. It also supports pagination for more efficient querying of historical messages. 

**Protocol identifier***: `/vac/waku/store/2.0.0-beta2`


# Design Requirements
Nodes willing to provide storage service in `WakuStore` protocol are required to be *highly available* and in specific have a *high uptime* to consistently receive and store network messages. The high uptime requirement makes sure that no message is missed out a complete and intact view of the message history is delivered to the querying nodes. Nevertheless, in case that storage provider nodes cannot afford high availability, the querying nodes may retrieve the historical messages from multiple sources to achieve a full and intact view of the past.

# Security Consideration
The main security consideration to be taken into account while using `WakuStore` is to notice that the querying nodes have to reveal their topics of interest to the queried nodes hence compromising their privacy. 

## Terminology
The term Personally identifiable information (PII) refers to any piece of data that can be used to uniquely identify a user. For example, the signature verification key, and the hash of one's static IP address are unique for each user and hence count as PII.

# Adversarial Model
Any peer running the `WakuStore` protocol i.e., both the querying node and the queried node are considered as an adversary. Furthermore, we consider the adversary as a passive entity that attempts to collect information from other peers to conduct an attack but it does so without violating protocol definitions and instructions. For example, under the passive adversarial model, no malicious node hides or lies about the history of messages as it is against the description of the `WakuStore` protocol. 

The following are not considered as part of the adversarial model:
- An adversary with a global view of all the peers and their connections.
- An adversary that can eavesdrop on communication links between arbitrary pairs of peers (unless the adversary is one end of the communication). In specific, the communication channels are assumed to be secure.


# Wire Specification
Peers communicate with each other using a request / response API. The messages sent are Protobuf RPC messages. The followings are the specifications of the Protobuf messages. 

## Protobuf

```protobuf
message Index {
  bytes digest = 1;
  float receivedTime = 2;
}

message PagingInfo {
  int64 pageSize = 1;
  Index cursor = 2;
  enum Direction {
    FORWARD = 0;
    BACKWARD = 1;
  }
  Direction direction = 3;
}

message HistoryQuery {
  repeated string topics = 2;
  optional PagingInfo pagingInfo = 3; // used for pagination
}

message HistoryResponse {
  repeated WakuMessage messages = 2;
  optional PagingInfo pagingInfo = 3; // used for pagination
}

message HistoryRPC {
  string request_id = 1;
  HistoryQuery query = 2;
  HistoryResponse response = 3;
}
```

### Index

To perform pagination, each `WakuMessage` stored at a node running the `WakuStore` protocol is associated with a unique `Index` that encapsulates the following parts. 
- `digest`:  a sequence of bytes representing the hash of a `WakuMessage`.
- `receivedTime`: the UNIX time at which the waku message is received by the node running the `WakuStore` protocol.

### PagingInfo

`PagingInfo` holds the information required for pagination. It consists of the following components.
- `pageSize`: A positive integer indicating the number of queried `WakuMessage`s in a `HistoryQuery` (or retrieved  `WakuMessage`s in a `HistoryResponse`).
- `cursor`: holds the `Index` of a `WakuMessage`.
- `direction`: indicates the direction of paging which can be either `FORWARD` or `BACKWARD`.

### HistoryQuery

RPC call to query historical messages.

- The `topics` field MUST indicate the list of topics to query.
- `PagingInfo` holds the information required for pagination.  Its `pageSize` field indicates the number of  `WakuMessage`s to be included in the corresponding `HistoryResponse`. If the `pageSize` is zero then no pagination is required. If the `pageSize` exceeds a threshold then the threshold value shall be used instead. In the forward pagination request, the `messages` field of the `HistoryResponse` shall contain at maximum the `pageSize` amount of waku messages whose `Index` values are larger than the given `cursor` (and vise versa for the backward pagination). Note that the `cursor` of a `HistoryQuery` may be empty (e.g., for the initial query), as such, and depending on whether the  `direction` is `BACKWARD` or `FORWARD`  the last or the first `pageSize` waku messages shall be returned, respectively.
The queried node MAY sort the `WakuMessage`s based on their `Index`, where the `receivedTime` constitutes the most significant part and the `digest` comes next, and then perform pagination on the sorted result. As such, the retrieved page contains an ordered list of `WakuMessage`s from the oldest message to the most recent one.

### HistoryResponse

RPC call to respond to a HistoryQuery call.
- The `messages` field MUST contain the messages found, these are [`WakuMessage`] types as defined in the corresponding [specification](./waku-message.md).
- `PagingInfo`  holds the paging information based on which the querying node can resume its further history queries. The `pageSize` indicates the number of returned waku messages (i.e., the number of messages included in the `messages` field of `HistoryResponse`). The `direction` is the same direction as in the corresponding `HistoryQuery`. In the forward pagination, the `cursor` holds the `Index` of the last message in the `HistoryResponse` `messages` (and the first message in the backward paging). The requester shall embed the returned  `cursor` inside its next `HistoryQuery` to retrieve the next page of the waku messages.  The  `cursor` obtained from one node SHOULD NOT be used in a request to another node because the result MAY be different.

# Future Work

- **Anonymous query**: This feature guarantees that nodes can anonymously query historical messages from other nodes (i.e., without disclosing the exact topics of waku messages they are interested in).  As such, no adversary in the `WakuStore` protocol would be able to learn which peer is interested in which topics of waku message. The current version of the `WakuStore` protocol does not provide anonymity for historical queries as the querying node needs to directly connect to another node in the `WakuStore` protocol and explicitly disclose the topics of its interest to retrieve the corresponding messages. However, one can consider preserving anonymity through one of the following ways: 
  - By hiding the source of the request i.e., anonymous communication. That is the querying node shall hide all its PII in its history request e.g., its IP address. This can happen by the utilization of a proxy server or by using Tor. Note that the current structure of historical requests does not embody any piece of PII, otherwise, such data fields must be treated carefully to achieve query anonymity. <!-- TODO: if nodes have to disclose their PeerIDs (e.g., for authentication purposes) when connecting to other nodes in the store protocol, then Tor does not preserve anonymity since it only helps in hiding the IP. So, the PeerId usage in switches must be investigated further. Depending on how PeerId is used, one may be able to link between a querying node and its queried topics despite hiding the IP address-->. 
  - By deploying secure 2-party computations in which the querying node obtains the historical messages of a certain topic whereas the queried node learns nothing about the query. Examples of such 2PC protocols are secure one-way Private Set Intersections (PSI). <!-- TODO: add a reference for PSIs? --> <!-- TODO: more techniques to be included -->. 
<!-- TODO: Censorship resistant: this is about a node that hides the historical messages from other nodes. This attack is not included in the specs since it does not fit the passive adversarial model (the attacker needs to deviate from the store protocol).-->
# Changelog

### Next 
- Added the initial threat model and security analysis.

### 2.0.0-beta2 
Released [2020-11-05](https://github.com/vacp2p/specs/commit/edc90625ffb5ce84cc6eb6ec4ec1a99385fad125)
- Added pagination support.
  
### 2.0.0-beta1
Released [2020-10-06](https://github.com/vacp2p/specs/commit/75b4c39e7945eb71ad3f9a0a62b99cff5dac42cf)
- Initial draft version. 

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
