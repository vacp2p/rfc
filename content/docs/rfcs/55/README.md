---
slug: 55
title: 55/STATUS-COMMUNITIES
name: Status Communities that run on Waku v2
status: raw
category: Standards Track
tags: waku-application
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
---

# Abstract

This document describes the design of Status Communities for Waku v2, allowing for multiple users to communicate in a group chat. 
This is a key feature for the Status messaging app. 

# Background and Motivation

Large group chats enable communities to communicate.
This would require channels, which are subject-based.
The messages in a channel are broadcasted to all the users in the channel.

A regular group chat between two or more peers reduces to a 1:1 chat between each peer and the other peers.
One mechanism for 1:1 chats is described in [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/).
However, this method does not scale as the number of peers increases, for the following reasons -
1. The number of messages sent over the network increases as the number of peers increases.
2. Handling the X3DH key exchange for each peer is computationally expensive.

Having multicast channels reduces the overhead of a regular group chat.
Additionally, if all the peers have a shared key, then the number of messages sent over the network is reduced to one per message.

# Design Requirements

Due to the nature of communities, the following requirements are necessary for the design of communities  -

1. The Community owner is trusted.
2. The Community owner can add or remove peers from the Community.
This extends to banning and kicking peers.
3. The Community owner can add, edit and remove channels.
4. The peers in the Community can send/receive messages to the channels which they have access to.
5. Communities may be encrypted (private) or unencrypted (public).
6. A Community is uniquely identified by a public key.
7. The public key of the Community is shared out of band.
8. The metadata of the Community can be found by listening on a content topic derived from the public key of the Community.
9. Peers part of a Community run their own Waku nodes.
Light peers may not be able to run their own Waku node.

# Design

## Cryptographic Primitives

The following cryptographic primitives are used in the design -

- X3DH
- Single Ratchet 
    - The single ratchet is used to encrypt the messages sent to the Community.
    - The single ratchet is re-keyed when a peer is added/removed from the Community.

## Wire format

<!--   
The wire format is described first to give an overview of the protocol.
It is referenced in the flow of community creation and community management.
More or less an intersection of https://github.com/status-im/specs/blob/403b5ce316a270565023fc6a1f8dec138819f4b0/docs/raw/organisation-channels.md and https://github.com/status-im/status-go/blob/6072bd17ab1e5d9fc42cf844fcb8ad18aa07760c/protocol/protobuf/communities.proto,

-->

```protobuf
syntax = "proto3";

message IdentityImage {
  // payload is a context based payload for the profile image data,
  // context is determined by the `source_type`
  bytes payload = 1;
  // source_type signals the image payload source
  SourceType source_type = 2;
  // image_type signals the image type and method of parsing the payload
  ImageType image_type = 3;
  // encryption_keys is a list of encrypted keys that can be used to decrypt an encrypted payload
  repeated bytes encryption_keys = 4;
  // encrypted signals the encryption state of the payload, default is false.
  bool encrypted = 5;
  // SourceType are the predefined types of image source allowed
  enum SourceType {
    UNKNOWN_SOURCE_TYPE = 0;

    // RAW_PAYLOAD image byte data
    RAW_PAYLOAD = 1;

    // ENS_AVATAR uses the ENS record's resolver get-text-data.avatar data
    // The `payload` field will be ignored if ENS_AVATAR is selected
    // The application will read and parse the ENS avatar data as image payload data, URLs will be ignored
    // The parent `ChatMessageIdentity` must have a valid `ens_name` set
    ENS_AVATAR = 2;
  }
}

// SocialLinks represents social link assosiated with given chat identity (personal/community)
message SocialLink {
  // Type of the social link
  string text = 1;
  // URL of the social link
  string url = 2;
}
// ChatIdentity represents identity of a community/chat
message ChatIdentity {
  // Lamport timestamp of the message
  uint64 clock = 1;
  // ens_name is the valid ENS name associated with the chat key
  string ens_name = 2;
  // images is a string indexed mapping of images associated with an identity
  map<string, IdentityImage> images = 3;
  // display name is the user set identity
  string display_name = 4;
  // description is the user set description
  string description = 5;
  string color = 6;
  string emoji = 7;
  repeated SocialLink social_links = 8;
  // first known message timestamp in seconds (valid only for community chats for now)
  // 0 - unknown
  // 1 - no messages
  uint32 first_message_timestamp = 9;
}

message Grant {
  // Community ID (The public key of the community)
  bytes community_id = 1;
  // The member ID (The public key of the member)
  bytes member_id = 2;
  // The chat for which the grant is given
  string chat_id = 3;
  // The Lamport timestamp of the grant
  uint64 clock = 4;
}

message CommunityMember {
  // The roles a community member MAY have
  enum Roles {
    UNKNOWN_ROLE = 0;
    ROLE_ALL = 1;
    ROLE_MANAGE_USERS = 2;
    ROLE_MODERATE_CONTENT = 3;
  }
  repeated Roles roles = 1;
}

message CommunityPermissions {
  // The type of access a community MAY have
  enum Access {
    UNKNOWN_ACCESS = 0;
    NO_MEMBERSHIP = 1;
    INVITATION_ONLY = 2;
    ON_REQUEST = 3;
  }

  // If the community should be available only to ens users
  bool ens_only = 1;
  // If the community is private
  bool private = 2;
  Access access = 3;
}

message CommunityAdminSettings {
  // If the Community admin may pin messages
  bool pin_message_all_members_enabled = 1;
}

message CommunityChat {
  // A map of members in the community to their roles in a chat
  map<string,CommunityMember> members = 1;
  // The permissions of the chat
  CommunityPermissions permissions = 2;
  // The metadata of the chat
  ChatIdentity identity = 3;
  // The category of the chat
  string category_id = 4;
  // The position of chat in the display
  int32 position = 5;
}

message CommunityCategory {
  // The category id 
  string category_id = 1;
  // The name of the category
  string name = 2;
  // The position of the category in the display
  int32 position = 3;
}

message CommunityInvitation {
  // Encrypted/unencrypted community description
  bytes community_description = 1;
  // The grant offered by the community
  bytes grant = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the community
  bytes public_key = 4;
}

message CommunityRequestToJoin {
  // The Lamport timestamp of the request  
  uint64 clock = 1;
  // The ENS name of the requester
  string ens_name = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the community
  bytes community_id = 4;
  // The display name of the requester
  string display_name = 5;
}

message CommunityCancelRequestToJoin {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The ENS name of the requester
  string ens_name = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the community
  bytes community_id = 4;
  // The display name of the requester
  string display_name = 5;
  // Magnet uri for community history protocol
  string magnet_uri = 6;
}

message CommunityRequestToJoinResponse {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The community description
  CommunityDescription community = 2;
  // If the request was accepted
  bool accepted = 3;
  // The grant offered by the community
  bytes grant = 4;
  // The community public key
  bytes community_id = 5;
}

message CommunityRequestToLeave {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The community public key
  bytes community_id = 2;
}

message CommunityDescription {
  // The Lamport timestamp of the message
  uint64 clock = 1;
  // A mapping of members in the community to their roles
  map<string,CommunityMember> members = 2;
  // The permissions of the Community
  CommunityPermissions permissions = 3;
  // The metadata of the Community
  ChatIdentity identity = 5;
  // A mapping of chats to their details
  map<string,CommunityChat> chats = 6;
  // A list of banned members
  repeated string ban_list = 7;
  // A mapping of categories to their details
  map<string,CommunityCategory> categories = 8;
  // The admin settings of the Community
  CommunityAdminSettings admin_settings = 10;
  // If the community is encrypted
  bool encrypted = 13;
  // The list of tags
  repeated string tags = 14;
}
```

Note: The usage of the clock is described in the [Clock](#clock) section.

## Community Management

The flows for Community management are as described below.

### Community Creation Flow

1. The Community owner generates a public/private key pair.
2. The Community owner configures the Community metadata, according to the wire format "CommunityDescription".
3. The Community owner publishes the Community metadata on a content topic derived from the public key of the Community. 
he Community metadata SHOULD be encrypted with the public key of the Community. <!-- TODO: Verify this-->
The Community metadata MAY be sent during fixed intervals, to ensure that the Community metadata is available to peers.
The Community metadata SHOULD be sent every time the Community metadata is updated.
4. The Community owner MAY advertise the Community out of band, by sharing the public key of the Community on other mediums of communication.

### Community Join Flow (peer requests to join a Community)

1. A peer generates a public/private key pair.
2. The peer requests to join a Community by sending a "CommunityRequestToJoin" message to the Community.
At this point, the peer MAY send a "CommunityCancelRequestToJoin" message to cancel the request.
3. The Community owner MAY accept or reject the request.
4. If the request is accepted, the Community owner sends a "CommunityRequestToJoinResponse" message to the peer.
5. The Community owner then adds the member to the Community metadata, and publishes the updated Community metadata.

### Community Join Flow (peer is invited to join a Community)

1. The peer is invited to join a Community by the Community owner, by sending a "CommunityInvitation" message.
2. A peer generates a public/private key pair.
3. The peer decrypts the "CommunityInvitation" message, and verifies the signature.
4. The peer requests to join a Community by sending a "CommunityRequestToJoin" message to the Community.
5. The Community owner MAY accept or reject the request.
6. If the request is accepted, the Community owner sends a "CommunityRequestToJoinResponse" message to the peer.
7. The Community owner then adds the member to the Community metadata, and publishes the updated Community metadata.

### Community Leave Flow

1. A peer requests to leave a Community by sending a "CommunityRequestToLeave" message to the Community.
2. The Community owner MAY accept or reject the request.
3. If the request is accepted, the Community owner removes the member from the Community metadata, and publishes the updated Community metadata.

### Community Ban Flow

1. The Community owner adds a member to the ban list, revokes their grants, and publishes the updated Community metadata.

## Waku protocols 

The following Waku protocols SHOULD be used to implement Status Communities -

1. [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/) - To send and receive messages
2. [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/) - To encrypt and decrypt messages
3. [53/WAKU2-X3DH-SESSIONS](https://rfc.vac.dev/spec/54/) - To handle session keys
4. [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/) - To wrap community messages in a Waku message
5. [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) - To store and retrieve messages for offline devices


The following Waku protocols MAY be used to implement Status Communities -

1. [12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/) - Content filtering for resource restricted devices
2. [35/WAKU2-NOISE](https://rfc.vac.dev/spec/35/) - As a replacement to [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/)

## Backups

<!--TODO -->

## Clock

The clock used in the wire format refers to the Lamport timestamp of the message.
The Lamport timestamp is a logical clock that is used to determine the order of events in a distributed system.
This allows ordering of messages in an asynchronous network where messages may be received out of order.

# Security Considerations

1. The Community owner is a single point of failure. If the Community owner is compromised, the Community is compromised.

2. Follows the same security considerations as the [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/) protocol.

# Future work

1. To scale and optimize the Community management, the Community metadata should be stored on a decentralized storage system, and only the references to the Community metadata should be broadcasted. The following document describes this method in more detail - [Optimizing the `CommunityDescription` dissemination](https://hackmd.io/rD1OfIbJQieDe3GQdyCRTw)

2. Token gating for communities

# References

- [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/)
- https://github.com/status-im/status-go/blob/6072bd17ab1e5d9fc42cf844fcb8ad18aa07760c/protocol/communities/community.go
- https://github.com/status-im/specs/blob/403b5ce316a270565023fc6a1f8dec138819f4b0/docs/raw/organisation-channels.md
