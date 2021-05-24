---
slug: 19
title: 19/WAKU2-LIGHTPUSH
name: Waku v2 Light Push
status: draft
editor: Oskar Thorén <oskar@status.im>
contributors:
---

**Protocol identifier**: `/vac/waku/lightpush/2.0.0-alpha1`

# Motivation and goals

Light nodes with short connection windows and limited bandwidth wish to publish
messages into the Waku network. Additionally, there is sometimes a need for
confirmation that a message has been received "by the network" (here, at least
one node).

`19/WAKU2-LIGHTPUSH` is a request/response protocol for this.

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

Nodes that respond to `PushRequests` MUST relay this via [11/WAKU2-RELAY](/spec/11) protocol on the specified `pubsub_topic`.
If they are unable to do so for some reason, they SHOULD return an error code in `PushResponse`.

## Security considerations

Since this can introduce amplification factor, it is RECOMMENDED for the node relaying to the rest of the network to extra precaution. This can be done by various forms of rate limiting, or by using [18/WAKU2-SWAP](/spec/18) to account for the service provided.

Note that the above is currently not fully implemented.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
