---
slug: 22
title: 22/CHAT2
name: Waku v2 Chat
status: raw
editor: Franck Royer <franck@status.im>
contributors:
   - Hanno Cornelius <hanno@status.im>
---

**Content Topic**: `dingpu`.

This specification explains the Waku v2 chat protocol.
This protocol is mainly used to enable showcase and dogfooding of the Waku v2 protocol.
Each Waku v2 implementation currently supports the chat protocol:
[nim-waku](https://github.com/status-im/nim-waku/blob/master/examples/v2/chat2.nim),
js-waku ([NodeJS](https://github.com/status-im/js-waku/tree/main/examples/cli-chat) and [web](https://github.com/status-im/js-waku/tree/main/examples/web-chat))
and [go-waku](https://github.com/status-im/go-waku/tree/master/examples/chat2).
There are no relation between this protocol and the protocol used by the Status app for chat purposes.

# Design

The chat protocol enables sending and receiving messages in a unique chat room.
The messages MUST NOT be encrypted.

The `contentTopic` MUST be set to `dingpu`.

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
