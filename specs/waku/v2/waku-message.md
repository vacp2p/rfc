---
title: Waku Message
version: 2.0.0-beta1
status: Draft
authors: Oskar Thorén <oskar@status.im>, Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [WakuMessage](#wakumessage)
  - [Protobuf](#protobuf)
  - [Payload encryption](#payload-encryption)
    - [Version 0](#version-0)
    - [Version 1 (not yet implemented in Waku v2)](#version-1-not-yet-implemented-in-waku-v2)
- [Differences from Whisper / Waku v1 envelopes](#differences-from-whisper--waku-v1-envelopes)
- [Security Consideration](#security-consideration)
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

# WakuMessage

A `WakuMessage` is what is being passed around by the other protocols, such as WakuRelay, WakuStore, and WakuFilter.

The `payload` field SHOULD contain whatever payload is being sent. See section below on payload encryption.

The `contentTopic` field SHOULD be filled out to allow for content-based filtering. See [Waku Filter spec](waku-filter.md) for details.

The `version` field MAY be filled out to allow for various types of payload encryption. Omitting it means the version is 0.

## Protobuf

```protobuf
message WakuMessage {
  optional bytes payload = 1;
  optional fixed32 contentTopic = 2;
  optional uint32 version = 3;
}
```

## Payload encryption

Payload encryption depends on the `version` field.

### Version 0

This indicates that the payload SHOULD be either unencrypted or that encryption is done at a separate layer outside of Waku.

### Version 1 (not yet implemented in Waku v2)

This indicates that payloads MUST be encrypted using [Waku v1 envelope data
format spec](../v1/envelope-data-format.md).

This provides for asymmetric and symmetric encryption. Key agreement is out of band. It also provides an encrypted signature and padding for some form of unlinkability.

# Differences from Whisper / Waku v1 envelopes

In Whisper and Waku v1, an envelope contains the following fields: `expiry, ttl,
topic, data, nonce`.

Since Waku v2 is using libp2p PubSub, some of these fields can be dropped. The previous `topic`
field corresponds to `contentTopic`. The previous `data` field corresponds to the `payload` field.

# Security Consideration

In Waku v2, the confidentiality, integrity, and authenticity of the data must be addressed at the `WakuMessage` level.  That is, the `payload` shall be encrypted or signed properly to meet the application-specific privacy needs.  Notice that due to the anonymity considerations these security features are not supported by the `WakuRelay` protocol and are delegated to the upper levels.

# Changelog

Initial release on [2020-10-22](https://github.com/vacp2p/specs/pull/222/commits/dbab6c7084b414b62818150588266abedd09315f).

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
