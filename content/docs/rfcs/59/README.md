---
slug: 59
title: 59/STATUS-URL-DATA
name: Status URL Data
status: raw
category: Standards Track
tags: waku-application
editor: Felicio Mununga <felicio@status.im>
contributors:
---

# Abstract

This document presents a comprehensive overview of the serialization, compression, and encoding techniques used to transmit data within URLs. 
The primary objective is to enable app links to contain sufficient information, allowing users to conveniently preview and verify the content before taking any actions or clicking on the link. 
By employing these methods, users can make informed decisions about engaging with the content, enhancing their overall browsing experience.

# Background / Rationale / Motivation

## Requirements

### Product

- The process of encoding data within URLs should continue until the Waku response becomes sufficiently instantaneous and reliable for link previewing and visiting the onboarding website. Alternatively, this process can be halted once a product roadmap for an alternative decentralized storage solution is finalized.

- During the loading of the onboarding website, a verification state should be implemented to compare the encoded data against the Waku response. This measure aims to mitigate identity attacks and ensure the integrity of the data.

- Only characters from the [A-Za-z0-9_-.\u0020] character class should be accepted for textual fields. This restriction promotes compatibility and security in the handling of textual data.

- Strive for the shortest possible result, optimizing the encoding process to minimize the overall length of the encoded data while maintaining its intended functionality and integrity.
### Related scope

#### Features

- Onboarding website
- Link preview
- Link sharing
- Deep linking
- Routing and navigation

#### App entities

- Community: Refer to [56/STATUS-COMMUNITIES](/spec/56)
- Channel: Refer to terminology in [56/STATUS-COMMUNITIES](/spec/56)
- User: Refer to terminology in [56/STATUS-COMMUNITIES](/spec/56)

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

- [Proposal Google Sheet](https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit?usp=sharing)

<!-- ## informative

A list of additional references. -->

# Footnotes

- This specification prioritizes maintainability, extensibility, familiarity with existing projects, and backward compatibility. 
These factors are considered essential for the long-term viability and success of the implementation.

- When it comes to implementation and proposals, it is most effective to optimize for field lengths ranging from moderate to maximum. 
Focusing on this range allows for better efficiency and effectiveness in handling the data, rather than solely optimizing for minimum field lengths.

- It is important to keep in mind that certain social platforms or instant messaging services may visually trim URLs, resulting in the URLs not being displayed in their full length.
For more information and specific examples, please refer to the following link: [URL Trimming Examples](https://docs.google.com/spreadsheets/d/1JD4kp0aUm90piUZ7FgM_c2NGe2PdN8BFB11wmt5UZIY/edit#gid=1260088614).

- Note that without sharing public keys with hosting servers, it is not possible to include certain data, such as profile photos or banners, in the previews. 
This limitation arises due to the need for authentication and secure access to the required data.