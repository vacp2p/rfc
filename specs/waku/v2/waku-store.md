---
title: Waku
version: 2.0.0-beta1
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Wire Specification](#wire-specification)
  * [Protobuf](#protobuf)
- [Changelog](#changelog)
- [Copyright](#copyright)

# Abstract

`WakuStore` is a protocol to enable querying of messages received through relay protocol and stored by other nodes.

**Protocol identifier***: `/vac/waku/store/2.0.0-beta1`

# Wire Specification

Peers communicate with each other using a request / response API. The messages sent are Protobuf RPC messages.

## Protobuf

```protobuf
message HistoryQuery {
  repeated string topics = 2;
}

message HistoryResponse {
  repeated WakuMessage messages = 2;
}

message HistoryRPC {
  string request_id = 1;
  HistoryQuery query = 2;
  HistoryResponse response = 3;
}
```

##### HistoryRPC

A node MUST send all History messages (`HistoryQuery`, `HistoryResponse`) wrapped inside a
`HistoryRPC`.

The `request_id` MUST be a uniquely generated string, the `HistoryResponse` and `HistoryQuery` `request_id` MUST match. The `HistoryResponse` message SHOULD contain any messages found
whose `topic` can be found in the `HistoryQuery` `topics` array. Any message whose `topic` does not match MUST NOT be included.

##### HistoryQuery

RPC call to query historical messages.

The `topics` field MUST indicate the list of topics to query.

##### HistoryResponse

RPC call to respond to a HistoryQuery call.

The `messages` field MUST contain the messages found, these are [`WakuMessage`] types as defined in the corresponding [specification](./waku-message.md).

# Changelog

2.0.0-beta1
Initial draft version. Released 2020-10-05 <!-- @TODO LINK -->

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
