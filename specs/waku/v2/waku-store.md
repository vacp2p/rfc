---
title: Waku
version: 2.0.0-beta2
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>, Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Abstract](#abstract)
  - [Security Requirements](#security-requirements)
  - [Adversarial Model](#adversarial-model)
- [Wire Specification](#wire-specification)
  - [Protobuf](#protobuf)
    - [Index](#index)
    - [PagingInfo](#paginginfo)
    - [HistoryQuery](#historyquery)
    - [HistoryResponse](#historyresponse)
  - [Security Analysis](#security-analysis)
  - [Future Work](#future-work)
- [Changelog](#changelog)
    - [2.0.0-beta2](#200-beta2)
    - [2.0.0-beta1](#200-beta1)
- [Copyright](#copyright)

# Abstract

This specification explains the Waku Store protocol which enables querying of messages received through relay protocol and stored by other nodes. It also supports pagination for more efficient querying of historical messages. 

**Protocol identifier***: `/vac/waku/store/2.0.0-beta2`

## Security Requirements

## Adversarial Model 

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

To perform pagination, each `WakuMessage` stored at a node running the store protocol is associated with a unique `Index` that encapsulates the following parts. 
- `digest`:  a sequence of bytes representing the hash of a `WakuMessage`.
- `receivedTime`: the UNIX time at which the waku message is received by the node running the store protocol.

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

## Security Analysis

## Future Work

# Changelog


### 2.0.0-beta2 
Released [2020-11-05](https://github.com/vacp2p/specs/commit/edc90625ffb5ce84cc6eb6ec4ec1a99385fad125)
- Added pagination support.
  
### 2.0.0-beta1
Released [2020-10-06](https://github.com/vacp2p/specs/commit/75b4c39e7945eb71ad3f9a0a62b99cff5dac42cf)
- Initial draft version. 

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
