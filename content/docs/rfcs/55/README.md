---
slug: 55
title: 55/STATUS-1TO1-CHAT
name: Status 1-to-1 Chat
status: draft
category: Standards Track
tags: waku-application
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
- Andrea Piana <andreap@status.im>
- Pedro Pombeiro <pedro@status.im>
- Corey Petty <corey@status.im>
- Oskar Thorén <oskar@status.im>
- Dean Eigenmann <dean@status.im>
---

# Abstract

This specification describes how the Status 1-to-1 chat protocol is implemented on top of the Waku v2 protocol. 
This protocol can be used to send messages to a single recipient.

# Background

This document describes how 2 peers communicate with each other to send messages in a 1-to-1 chat, with privacy and authenticity guarantees.

# Specification

## Overview

This protocol MAY use any key-exchange mechanism previously discussed -

1. [53/WAKU2-X3DH](/spec/53/) 
2. [35/WAKU2-NOISE](/spec/35/)

This protocol can provide end-to-end encryption to give peers a strong degree of privacy and security. 
Public chat messages are publicly readable by anyone since there's no permission model for who is participating in a public chat.

## Flow

### Negotiation of a 1:1 chat

There are two phases in the initial negotiation of a 1:1 chat:
1. **Identity verification** (e.g., face-to-face contact exchange through QR code, Identicon matching).
A QR code serves two purposes simultaneously - identity verification and initial key material retrieval;
1. **Asynchronous initial key exchange**

For more information on account generation and trust establishment, see [2/ACCOUNT](https://specs.status.im/spec/2)

### Post Negotiation

After the peers have shared their public key material, a 1:1 chat can be established using the methods described in the key-exchange protocols mentioned above.

### Session management

The 1:1 chat is made robust by having sessions between peers.
It is handled by the key-exchange protocol used. For example,

1. [53/WAKU2-X3DH](/spec/53/), the session management is described in [54/WAKU2-X3DH-SESSIONS](/spec/54/)

2. [35/WAKU2-NOISE](/spec/35/), the session management is described in [37/WAKU2-NOISE-SESSIONS](/spec/37/)

# Security Considerations

1. Inherits the security considerations of the key-exchange mechanism used, e.g., [53/WAKU2-X3DH](/spec/53/) or [35/WAKU2-NOISE](/spec/35/)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [2/ACCOUNT](https://specs.status.im/spec/2)
2. [53/WAKU2-X3DH](/spec/53/)
3. [35/WAKU2-NOISE](/spec/35/)
4. [10/WAKU2](/spec/10/)