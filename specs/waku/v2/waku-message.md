---
title: Waku Message
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
    - [WakuMessage](#wakumessage)
    - [Protobuf](#protobuf)
    - [Payload encryption](#payload-encryption)
        - [Version 0](#version-0)
        - [Version 1](#version-1)
- [Differences from Whisper / Waku v1 envelopes](#differences-from-whisper--waku-v1-envelopes)
- [Changelog](#changelog)
- [Copyright](#copyright)

# Abstract

This specification provides a way to encapsulate messages sent over Waku with specific information security goals.

# Motivation

When using Waku to send messages over Waku there are multiple concerns:
- We may have a separate encryption layer as part of our application
- We may want to provide efficient routing for resource restricted devices
- We may want to provide compatibility with Waku v1 envelopes
- We may want payloads to be encrypted by default
- We may want to provide unlinkability for metadata protection

This specification attempts to provide for these various requirements.

## WakuMessage

A `WakuMessage` is what is being passed around by the other protocols, such as WakuRelay, WakuStore, and WakuFilter.

The `payload` field SHOULD contain whatever payload is being sent. See section below on payload encryption.

The `contentTopic` field SHOULD be filled out to allow for content-based filtering. See [Waku Filter spec](waku-filter.md) for details.

The `version` field MAY be filled out to allow for various types of payload encryption. Omitting it means the version is 0.

## Protobuf

```protobuf
message WakuMessage {
  optional bytes payload = 1;
  optional string contentTopic = 2;
  optional string version = 3;
}
```

## Payload encryption

Payload encryption depends on the `version` field.

### Version 0

This indicates that the payload SHOULD be either unencrypted or that encryption is done at a separate layer outside of Waku.

### Version 1

This indicates that payloads MUST be encrypted using [Waku v1 envelope data
format spec](../v1/envelope-data-format.md).

This provides for asymmetric and symmetric encryption. Key agreement is out of band. It also provides an encrypted signature and padding for some form of unlinkability.

# Differences from Whisper / Waku v1 envelopes

In Whisper and Waku v1, an envelope contains the following fields: `expiry, ttl,
topic, data, nonce`.

Since Waku v2 is using libp2p PubSub, some of these fields can be dropped. The previous `topic`
field corresponds to `contentTopic`. The previous `data` field corresponds to the `payload` field.

# Changelog

Initial release on [](TODO).

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
