---
slug: 55
title: 55/STATUS-1TO1-CHAT
name: Status 1-to-1 Chat
status: draft
category: Standards Track
tags: waku-application
description: A chat protocol to send public and private messages to a single recipient by the Status app.
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

# Terminology

- **Participant**: A participant is a user that is able to send and receive messages.
- **1-to-1 chat**: A chat between two participants.
- **Public chat**: A chat where any participant can join and read messages.
- **Private chat**: A chat where only invited participants can join and read messages.
- **Group chat**: A chat where multiple select participants can join and read messages.
- **Group admin**: A participant that is able to add/remove participants from a group chat.

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

## Negotiation of a 1:1 chat amongst multiple participants (group chat)

A small, private group chat can be constructed by having multiple participants negotiate a 1:1 chat amongst each other.
Each participant MUST maintain a session with all other participants in the group chat.
This allows for a group chat to be created with a small number of participants.

However, this method does not scale as the number of participants increases, for the following reasons -
1. The number of messages sent over the network increases with the number of participants.
2. Handling the X3DH key exchange for each participant is computationally expensive.

The above issues are addressed in [56/STATUS-COMMUNITIES](/spec/56/), with other trade-offs.

### Flow

The following flow describes how a group chat is created and maintained.

#### Membership Update Flow

Membership updates have the following wire format:

```protobuf
message MembershipUpdateMessage {
  // The chat id of the private group chat
  // derived in the following way:
  // chat_id = hex(chat_creator_public_key) + "-" + random_uuid
  // This chat_id MUST be validated by all participants
  string chat_id = 1;
  // A list of events for this group chat, first 65 bytes are the signature, then is a 
  // protobuf encoded MembershipUpdateEvent
  repeated bytes events = 2;
  oneof chat_entity {
    // An optional chat message
    ChatMessage message = 3;
    // An optional reaction to a message
    EmojiReaction emoji_reaction = 4; 
  }
}
```

Note that in `events`, the first element is the signature, and all other elements after are encoded `MembershipUpdateEvent`'s.

where `MembershipUpdateEvent` is defined as follows:

```protobuf
message MembershipUpdateEvent {
  // Lamport timestamp of the event
  uint64 clock = 1;
  // Optional list of public keys of the targets of the action
  repeated string members = 2;
  // Name of the chat for the CHAT_CREATED/NAME_CHANGED event types
  string name = 3;
  // The type of the event
  EventType type = 4;
  // Color of the chat for the CHAT_CREATED/COLOR_CHANGED event types
  string color = 5;
  // Chat image
  bytes image = 6;

  enum EventType {
    UNKNOWN = 0;
    CHAT_CREATED = 1; // See [CHAT_CREATED](#chat-created)
    NAME_CHANGED = 2; // See [NAME_CHANGED](#name-changed)
    MEMBERS_ADDED = 3; // See [MEMBERS_ADDED](#members-added)
    MEMBER_JOINED = 4; // See [MEMBER_JOINED](#member-joined)
    MEMBER_REMOVED = 5; // See [MEMBER_REMOVED](#member-removed)
    ADMINS_ADDED = 6; // See [ADMINS_ADDED](#admins-added)
    ADMIN_REMOVED = 7; // See [ADMIN_REMOVED](#admin-removed)
    COLOR_CHANGED = 8; // See [COLOR_CHANGED](#color-changed)
    IMAGE_CHANGED = 9; // See [IMAGE_CHANGED](#image-changed)
  }
}
```
<!-- Note: I don't like defining wire formats which are out of the scope of the rfc this way. Should explore alternatives -->
Note that the definitions for `ChatMessage` and `EmojiReaction` can be found in [chat_message.proto](https://github.com/status-im/status-go/blob/5fd9e93e9c298ed087e6716d857a3951dbfb3c1e/protocol/protobuf/chat_message.proto#L1) and [emoji_reaction.proto](https://github.com/status-im/status-go/blob/5fd9e93e9c298ed087e6716d857a3951dbfb3c1e/protocol/protobuf/emoji_reaction.proto).

##### Chat Created

When creating a group chat, this is the first event that MUST be sent. 
Any event with a clock value lower than this MUST be discarded. 
Upon receiving this event a client MUST validate the `chat_id` provided with the update and create a chat with identified by `chat_id`.

By default, the creator of the group chat is the only group admin.

##### Name Changed

To change the name of the group chat, group admins MUST use a `NAME_CHANGED` event.
Upon receiving this event a client MUST validate the `chat_id` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored. 
If the event is valid the chat name SHOULD be changed according to the provided message.

##### Members Added

To add members to the chat, group admins MUST use a `MEMBERS_ADDED` event. 
Upon receiving this event a participant MUST validate the `chat_id` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored. 
If the event is valid, a participant MUST update the list of members of the chat who have not joined, adding the members received. 

##### Member Joined

To signal the intent to start receiving messages from a given chat, new participants MUST use a `MEMBER_JOINED` event.
Upon receiving this event a participant MUST validate the `chat_id` provided with the updates. 
If the event is valid a participant MUST add the new participant to the list of participants stored locally. 
Any message sent to the group chat MUST now include the new participant.

##### Member Removed

There are two ways in which a member MAY be removed from a group chat:
- A member MAY leave the chat by sending a `MEMBER_REMOVED` event, with the `members` field containing their own public key.
- An admin MAY remove a member by sending a `MEMBER_REMOVED` event, with the `members` field containing the public key of the member to be removed.

Each participant MUST validate the `chat_id` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored.
If the event is valid, a participant MUST update the local list of members accordingly.

##### Admins Added

To promote participants to group admin, group admins MUST use an `ADMINS_ADDED` event.
Upon receiving this event, a participant MUST validate the `chat_id` provided with the updates, MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored. 
If the event is valid, a participant MUST update the list of admins of the chat accordingly.

##### Admin Removed

Group admins MUST NOT be able to remove other group admins.
An admin MAY remove themselves by sending an `ADMIN_REMOVED` event, with the `members` field containing their own public key.
Each participant MUST validate the `chat_id` provided with the updates and MUST ensure the author of the event is an admin of the chat, otherwise the event MUST be ignored.
If the event is valid, a participant MUST update the list of admins of the chat accordingly.

##### Color Changed

To change the text color of the group chat name, group admins MUST use a `COLOR_CHANGED` event.

##### Image Changed

To change the display image of the group chat, group admins MUST use an `IMAGE_CHANGED` event.

# Security Considerations

1. Inherits the security considerations of the key-exchange mechanism used, e.g., [53/WAKU2-X3DH](/spec/53/) or [35/WAKU2-NOISE](/spec/35/)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [53/WAKU2-X3DH](/spec/53/)
2. [35/WAKU2-NOISE](/spec/35/)
3. [65/STATUS-ACCOUNT](/spec/65/)
4. [54/WAKU2-X3DH-SESSIONS](/spec/54/)
5. [37/WAKU2-NOISE-SESSIONS](/spec/37/)
6. [56/STATUS-COMMUNITIES](/spec/56/)
7. [chat_message.proto](https://github.com/status-im/status-go/blob/5fd9e93e9c298ed087e6716d857a3951dbfb3c1e/protocol/protobuf/chat_message.proto#L1)
8. [emoji_reaction.proto](https://github.com/status-im/status-go/blob/5fd9e93e9c298ed087e6716d857a3951dbfb3c1e/protocol/protobuf/emoji_reaction.proto)
