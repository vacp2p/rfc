---
slug: 22
title: 22/TOY-CHAT
name: Waku v2 Chat
status: raw
category: application
editor: Franck Royer <franck@status.im>
contributors:
   - Hanno Cornelius <hanno@status.im>
---

**Content Topic**: `/waku/2/huilong/proto`.

This specification explains a toy chat example using Waku v2.
This protocol is mainly used to:

1. Dogfood Waku v2,
2. Show an example of how to use Waku v2.

Currently, all main Waku v2 implementations support the toy chat protocol:
[nim-waku](https://github.com/status-im/nim-waku/blob/master/examples/v2/chat2.nim),
js-waku ([NodeJS](https://github.com/status-im/js-waku/tree/main/examples/cli-chat) and [web](https://github.com/status-im/js-waku/tree/main/examples/web-chat))
and [go-waku](https://github.com/status-im/go-waku/tree/master/examples/chat2).

Note that this is completely separate from the protocol the Status app is using for its chat functionality.

# Design

The chat protocol enables sending and receiving messages in a chat room.
There is currently only one chat room, which is tied to the content topic.
The messages SHOULD NOT be encrypted.

The `contentTopic` MUST be set to `/waku/2/huilong/proto`.

# Payloads

```protobuf
syntax = "proto3";

message Chat2Message {
   uint64 timestamp = 1;
   string nick = 2;
   bytes payload = 3;
}
```

- `timestamp`: The time at which the message was sent, in Unix Epoch seconds,
- `nick`: The nickname of the user sending the message,
- `payload`: The text of the messages, UTF-8 encoded.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
