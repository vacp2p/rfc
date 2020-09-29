---
title: Waku Message
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

TODO

# Abstract

TODO

# Motivation

TODO

## WakuMessage

A `WakuMessage` is what is being passed around by the other protocols, such as WakuRelay, WakuStore, and WakuFilter.

The `payload` field SHOULD contain whatever payload is being sent. Encryption of this field is done at a separate layer.

The `contentTopic` field SHOULD be filled out to allow for content-based filtering (see section below).


## Protobuf

```protobuf
message WakuMessage {
  optional bytes payload = 1;
  optional string contentTopic = 2;
}
```

# Differences from Whisper / Waku v1 envelopes

In Whisper and Waku v1, an envelope contains the following fields: `expiry, ttl,
topic, data, nonce`.

Since Waku v2 is using libp2p PubSub, some of these fields can be dropped. The previous `topic`
field corresponds to `contentTopic`. The previous `data` field corresponds to the `payload` field.

# Changelog

TODO

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
