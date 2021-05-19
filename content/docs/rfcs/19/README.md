---
slug: 19
title: 19/WAKU2-LIGHTPUSH
name: Waku v2 Light Push
status: raw
editor: Oskar Thorén <oskar@status.im>
contributors:
---

**Protocol identifier**: `/vac/waku/lightpush/2.0.0-alpha1`

# Motivation and goals

Light nodes with short connection windows and limited bandwidth wish to publish messages into the Waku network.
Additionally, there sometimes is a need for confirmation that a message has been received "by the network".

`19/LIGHTPUSH` is a request/reply protocol for this.

# Payloads

```protobuf
syntax = "proto3";

message PushRequest {
    string pubsub_topic = 1;
    WakuMessage message = 2;
}

message PushResponse {
    bool is_success = 1;
    // Error messages, etc
    string info = 2;
}

message PushRPC {
    string request_id = 1;
    PushRequest request = 2;
    PushResponse response = 3;
}
```

## Message relaying

Nodes that respond to `PushRequests` MUST relay this via `RELAY` protocol on the specified `pubsub_topic`.
If they are unable to do so for some reason, they SHOULD return an error code in `PushResponse`.

## Notes

<!--
TODO: Check current message confirmation setup
-->

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

