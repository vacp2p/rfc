---
title: Waku
version: 2.0.0-alpha4
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

TODO

# Abstract

TODO

### Historical message support

**Protocol identifier***: `/vac/waku/store/2.0.0-alpha6`

See `WakuStore` spec.

TODO To be elaborated on

#### Protobuf

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
`HistoryRPC`. This allows the node handler to determine how to handle a message as the Waku
Filter protocol is not a request response based protocol but instead a push based system.

The `request_id` MUST be a uniquely generated string.

##### HistoryQuery

RPC call to query historical messages.

The `uuid` field MUST indicate current request UUID, it is used to identify the corresponding response.

The `topics` field MUST indicate the list of topics to query.

##### HistoryResponse

RPC call to respond to a HistoryQuery call.

The `uuid` field MUST indicate which query is being responded to.

The `messages` field MUST contain the messages found.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
