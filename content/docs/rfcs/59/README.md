---
slug: 59
title: 59/STATUS-URL-DATA
name: Status URL Data
status: raw
category: Standards Track
tags: waku-application
editor: Felicio Mununga <felicio@status.im>
contributors:
  - Aaryamann Challani <aaryamann@status.im>
---

# Abstract

This document specifies serialization, compression, and encoding techniques used to transmit data within URLs in the context of Status protocols.

# Motivation

When sharing URLs, link previews often expose metadata to the websites behind those links.
To reduce reliance on external servers for providing appropriate link previews, this specification proposes a standard method for encoding data within URLs.

# Terminology

- Community: Refer to [56/STATUS-COMMUNITIES](/spec/56)
- Channel: Refer to terminology in [56/STATUS-COMMUNITIES](/spec/56)
- User: Refer to terminology in [56/STATUS-COMMUNITIES](/spec/56)
- Shard Refer to terminology in [51/WAKU2-RELAY-SHARDING](/spec/51)

# Wire Format

```protobuf
syntax = "proto3";

message Community {
    // Display name of the community
    string display_name = 1;
    // Description of the community
    string description = 2;
    // Number of members in the community
    uint32 members_count = 3;
    // Color of the community title
    string color = 4;
    // List of tag indices
    repeated uint32 tag_indices = 5;
}

message Channel {
    // Display name of the channel
    string display_name = 1;
    // Description of the channel
    string description = 2;
    // Emoji of the channel
    string emoji = 3;
    // Color of the channel title
    string color = 4;
    // Community the channel belongs to
    Community community = 5;
    // UUID of the channel
    string uuid = 6;
}

message User {
    // Display name of the user
    string display_name = 1;
    // Description of the user
    string description = 2;
    // Color of the user title
    string color = 3;
}

message URLData {
    // Community, Channel, or User
    bytes content = 1;
    uint32 shard_cluster = 2;
    uint32 shard_index = 3;
}
```

# Implementation

The above wire format describes the data encoded in the URL.
The data MUST be serialized, compressed, and encoded using the following standards:

## Encoding

- [Base64url](https://datatracker.ietf.org/doc/html/rfc4648)

## Compression

- [Brotli](https://datatracker.ietf.org/doc/html/rfc7932)

## Serialization

- [Protocol buffers version 3](https://protobuf.dev/reference/protobuf/proto3-spec/)

## Implementation Pseudocode

### Encoding

Encoding the URL MUST be done in the following order:

```
raw_data = {User | Channel | Community}
serialized_data = protobuf_serialize(raw_data)
compressed_data = brotli_compress(serialized_data)
encoded_url_data = base64url_encode(compressed_data)
```

The `encoded_url_data` is then used to generate a signature using the private key.

### Decoding

Decoding the URL MUST be done in the following order:

```
url_data = base64url_decode(encoded_url_data)
decompressed_data = brotli_decompress(url_data)
deserialized_data = protobuf_deserialize(decompressed_data)
raw_data = deserialized_data.content
```

The `raw_data` is then used to construct the appropriate data structure (User, Channel, or Community).

## Example

- See <https://github.com/status-im/status-web/pull/345/files>

<!-- # (Further Optional Sections) -->

# Discussions

- See <https://github.com/status-im/status-web/issues/327>

# Proof of concept

- See <https://github.com/felicio/status-web/blob/825262c4f07a68501478116c7382862607a5544e/packages/status-js/src/utils/encode-url-data.compare.test.ts#L4>

<!-- # Security Considerations -->

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [Proposal Google Sheet](https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit?usp=sharing)
2. [Base64url](https://datatracker.ietf.org/doc/html/rfc4648)
3. [Brotli](https://datatracker.ietf.org/doc/html/rfc7932)
4. [Protocol buffers version 3](https://protobuf.dev/reference/protobuf/proto3-spec/)

<!-- ## informative

A list of additional references. -->
