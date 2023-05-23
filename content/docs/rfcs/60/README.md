---
slug: 60
title: 60/STATUS-URL-SCHEME
name: Status URL Scheme
status: draft
category: Standards Track
tags: waku-application
editor: Felicio Mununga <felicio@status.im>
contributors:
---

# Abstract

This document describes URL scheme for previewing and deep linking content as well as for triggering actions.

# Background / Rationale / Motivation

## Requirements

### Related scope

#### Features

- Onboarding website
- Link preview
- Link sharing
- Deep linking
- Routing and navigation
- Payment requests
- Chat creation

# Wire Format Specification / Syntax

## Schemes

- Internal `status-app://`
- External `https://` (i.e. univers/deep links)

## Paths

| Name | Url | Description |
| ----- | ---- | ---- |
| User profile | `/u/<encoded_data>#<encoded_signature_and_user_chat_key>` | Preview/Open user profile |
| | `/u#<user_chat_key>` | |
| | `/u#<ens_name>` | |
| Community | `/c/< encoded_data >#<encoded_signature_and_community_chat_key>` | Preview/Open community |
| | `/c#<community_chat_key>` | |
| Community channel | `/cc/< encoded_data >#< encoded_signature_and_community_chat_key >`| Preview/Open community channel |
| | `/cc/<channel_uuid>#<community_chat_key>` | |

<!-- # Security/Privacy Considerations

A standard track RFC in `stable` status MUST feature this section.
A standard track RFC in `raw` or `draft` status SHOULD feature this section.
Informational RFCs (in any state) may feature this section.
If there are none, this section MUST explicitly state that fact.
This section MAY contain additional relevant information, e.g. an explanation as to why there are no security consideration for the respective document. -->

# Discussions

- See <https://github.com/status-im/specs/pull/159>
- See <https://github.com/status-im/status-web/issues/327>

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- [59/STATUS-URL-DATA](/spec/59/)
