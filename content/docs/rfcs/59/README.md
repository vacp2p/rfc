---
slug: 59
title: 59/STATUS-URL-DATA
name: Status URL Data
status: draft
category: Standards Track
tags: waku-application
editor: Felicio Mununga <felicio@status.im>
contributors:
---

# Abstract

This document describes serialization, compression and encoding of data within URLs, so the app links could carry enough information to help users with instant previewing and verifying even prior clicking or taking an action on the content.

# Background / Rationale / Motivation

## Requirements

### Product

- Encode data within URL until Waku response is instant and considered reliable enough for link previewing and visiting of the onboarding website. Or until a product roadmap for alternative decentralized storage is finalized.
- A verify state while loading onboarding website until encoded data is matched against Waku response to mitigate identity attacks
- Accept only `[A-Za-z0-9_-.\u0020]` character class for textual fields
- Shortest result possible

### Related scope

#### Features

- Onboarding website
- Link preview
- Link sharing
- Deep linking
- Routing and navigation

#### App entities

- Community
- Channel
- User

<!-- # Theory / Semantics

A standard track RFC in `stable` status MUST feature this section.
A standard track RFC in `raw` or `draft` status SHOULD feature this section.
This section SHOULD explain in detail how the proposed protocol works.
It may touch on the wire format where necessary for the explanation.
This section MAY also specify endpoint behaviour when receiving specific messages, e.g. the behaviour of certain caches etc. -->

# Wire Format Specification / Syntax

## Data

### Fields

```protobuf
syntax = "proto3";

message Community {
 string display_name = 1;
 string description = 2;
 uint32 members_count = 3;
 string color = 4;
 repeated uint32 tag_indices = 5;
}

message Channel {
 string display_name = 1;
 string description = 2;
 string emoji = 3;
 string color = 4;
 Community community = 5;
 string uuid = 6;
}

message User {
 string display_name = 1;
 string description = 2;
 string color = 3;
}

message URLData {
 // Community, Channel, or User
 bytes content = 1;
}

// Field on CommunityDescription, CommunityChat and ContactCodeAdvertisement
message URLParams {
 string encoded_url_data = 1;
 // Signature of encoded URL data
 string encoded_signature = 2;
}
```

# Implementation Suggestions

## Example

- See <https://github.com/status-im/status-web/pull/345/files>

## Encoding

- Base64url

## Compression

- Brotli

## Serialization

- Protocol buffers

<!-- # (Further Optional Sections) -->

# Proposals

- See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit?usp=sharing> for all
- See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit#gid=1895477181> for final two

# Discussions

- See <https://github.com/status-im/status-web/issues/327>

# Proof of concept

- See <https://github.com/felicio/status-web/blob/825262c4f07a68501478116c7382862607a5544e/packages/status-js/src/utils/encode-url-data.compare.test.ts#L4>

<!-- # Security/Privacy Considerations

A standard track RFC in `stable` status MUST feature this section.
A standard track RFC in `raw` or `draft` status SHOULD feature this section.
Informational RFCs (in any state) may feature this section.
If there are none, this section MUST explicitly state that fact.
This section MAY contain additional relevant information, e.g. an explanation as to why there are no security consideration for the respective document. -->

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- [60/STATUS-URL-SCHEME](/spec/60/)

<!-- ## informative

A list of additional references. -->

# Footnotes

- This specification prefers maintainability, extensibility, project familiarity and backward compatibility.
- Implementation and proposals are most effective for mid to max field lengths, not min.
- Mind that some social platforms or IMs visually trim the URLs, so they would actually not be displayed in full length. See <https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit#gid=1260088614>.
- Without sharing public keys with hosting servers some data like profile photos or banners cannot be included in the previews
