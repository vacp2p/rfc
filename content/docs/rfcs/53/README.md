---
slug: 55
title: 55/WAKU2-ORGANIZATION-CHATS
name: Waku v2 Organization Chats
status: raw
category: Standards Track
tags: waku-application
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
---

# Abstract

This document describes the design of organization chats for Waku v2, allowing for multiple users to communicate in a group chat. 
This is a key feature for the Status messaging app. 

# Background and Motivation

Large group chats enable organizations to communicate.
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

Due to the nature of organizations, the following requirements are necessary for the design of organization chats -

1. The Organization owner is trusted.
2. The Organization owner can add or remove peers from the Organization.
3. The Organization owner can add or remove channels.
4. The peers in the Organization can send/receive messages to the channels which they have access to.
5. Organizations may be encrypted or unencrypted (public).
6. An Organization is uniquely identified by a public key.
7. The public key of the Organization is shared out of band.
8. The metadata of the Organization can be found by listening on a content topic derived from the public key of the Organization.
9. Peers part of an Organization run their own infrastructure.

# Design

## Cryptographic Primitives

The following cryptographic primitives are used in the design -

- X3DH
- Single Ratchet 
    - The single ratchet is used to encrypt the messages sent to the Organization.
    - The single ratchet is re-keyed when a peer is added/removed from the Organization.

The following cryptographic functions are used in the design -

1. `ECDH-ES` - Elliptic Curve Diffie Hellman key exchange with Elliptic Curve Integrated Encryption Scheme.
2. `AES-GCM` - AES Galois Counter Mode.


## Wire format

<!--   
The wire format is described first to give an overview of the protocol.
It is referenced in the flow of organization creation and organization management.
More or less an intersection of https://github.com/status-im/specs/blob/403b5ce316a270565023fc6a1f8dec138819f4b0/docs/raw/organisation-channels.md and https://github.com/status-im/status-go/blob/6072bd17ab1e5d9fc42cf844fcb8ad18aa07760c/protocol/protobuf/communities.proto,
but with changes to make it more generic.
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

// ChatIdentity represents identity of an organization/chat
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
}

message Grant {
  // Organization ID (The public key of the organization)
  bytes organization_id = 1;
  // The member ID (The public key of the member)
  bytes member_id = 2;
  // The chat for which the grant is given
  string chat_id = 3;
  // The Lamport timestamp of the grant
  uint64 clock = 4;
}

message OrganizationMember {
  // The roles an organization member MAY have
  enum Roles {
    UNKNOWN_ROLE = 0;
    ROLE_ALL = 1;
    ROLE_MANAGE_USERS = 2;
    ROLE_MODERATE_CONTENT = 3;
  }
  repeated Roles roles = 1;
}

message OrganizationPermissions {
  // The type of access an organization MAY have
  enum Access {
    UNKNOWN_ACCESS = 0;
    NO_MEMBERSHIP = 1;
    INVITATION_ONLY = 2;
    ON_REQUEST = 3;
  }

  // If the organization should be available only to ens users
  bool ens_only = 1;
  // If the organization is private
  bool private = 2;
  Access access = 3;
}

message OrganizationAdminSettings {
  // If the Organization admin may pin messages
  bool pin_message_all_members_enabled = 1;
}

message OrganizationChat {
  // A map of members in the organization to their roles in a chat
  map<string,OrganizationMember> members = 1;
  // The permissions of the chat
  OrganizationPermissions permissions = 2;
  // The metadata of the chat
  ChatIdentity identity = 3;
  // The category of the chat
  string category_id = 4;
}

message OrganizationCategory {
  // The category id 
  string category_id = 1;
  // The name of the category
  string name = 2;
}

message OrganizationInvitation {
  // Encrypted/unencrypted organization description
  bytes organization_description = 1;
  // The grant offered by the organization
  bytes grant = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the organization
  bytes public_key = 4;
}

message OrganizationRequestToJoin {
  // The Lamport timestamp of the request  
  uint64 clock = 1;
  // The ENS name of the requester
  string ens_name = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the organization
  bytes organization_id = 4;
  // The display name of the requester
  string display_name = 5;
}

message OrganizationCancelRequestToJoin {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The ENS name of the requester
  string ens_name = 2;
  // The chat id requested to join
  string chat_id = 3;
  // The public key of the organization
  bytes organization_id = 4;
  // The display name of the requester
  string display_name = 5;
}

message OrganizationRequestToJoinResponse {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The organization description
  OrganizationDescription organization = 2;
  // If the request was accepted
  bool accepted = 3;
  // The grant offered by the organization
  bytes grant = 4;
  // The organization public key
  bytes organization_id = 5;
}

message OrganizationRequestToLeave {
  // The Lamport timestamp of the request
  uint64 clock = 1;
  // The organization public key
  bytes organization_id = 2;
}

message OrganizationDescription {
  // The Lamport timestamp of the message
  uint64 clock = 1;
  // A mapping of members in the organization to their roles
  map<string,OrganizationMember> members = 2;
  // The permissions of the Organization
  OrganizationPermissions permissions = 3;
  // The metadata of the Organization
  ChatIdentity identity = 5;
  // A mapping of chats to their details
  map<string,OrganizationChat> chats = 6;
  // A list of banned members
  repeated string ban_list = 7;
  // A mapping of categories to their details
  map<string,OrganizationCategory> categories = 8;
  // The admin settings of the Organization
  OrganizationAdminSettings admin_settings = 10;
  // If the organization is encrypted
  bool encrypted = 13;
  // The list of tags
  repeated string tags = 14;
}
```

## Organization Management

The flows for Organization management are as described below.

### Organization Creation Flow

1. The Organization owner generates a public/private key pair.
2. The Organization owner configures the Organization metadata, according to the wire format "OrganizationDescription".
3. The Organization owner publishes the Organization metadata on a content topic derived from the public key of the Organization. 
he Organization metadata SHOULD be encrypted with the public key of the Organization. <!-- TODO: Verify this-->
The Organization metadata MAY be sent during fixed intervals, to ensure that the Organization metadata is available to peers.
The Organization metadata SHOULD be sent every time the Organization metadata is updated.
4. The Organization owner MAY advertise the Organization out of band, by sharing the public key of the Organization on other mediums of communication.

### Organization Join Flow (peer requests to join an Organization)

1. A peer generates a public/private key pair.
2. The peer requests to join an Organization by sending a "OrganizationRequestToJoin" message to the Organization.
At this point, the peer MAY send a "OrganizationCancelRequestToJoin" message to cancel the request.
3. The Organization owner MAY accept or reject the request.
4. If the request is accepted, the Organization owner sends a "OrganizationRequestToJoinResponse" message to the peer.
5. The Organization owner then adds the member to the Organization metadata, and publishes the updated Organization metadata.

### Organization Join Flow (peer is invited to join an Organization)

1. The peer is invited to join an Organization by the Organization owner, by sending a "OrganizationInvitation" message.
2. A peer generates a public/private key pair.
3. The peer decrypts the "OrganizationInvitation" message, and verifies the signature.
4. The peer requests to join an Organization by sending a "OrganizationRequestToJoin" message to the Organization.
5. The Organization owner MAY accept or reject the request.
6. If the request is accepted, the Organization owner sends a "OrganizationRequestToJoinResponse" message to the peer.
7. The Organization owner then adds the member to the Organization metadata, and publishes the updated Organization metadata.

### Organization Leave Flow

1. A peer requests to leave an Organization by sending a "OrganizationRequestToLeave" message to the Organization.
2. The Organization owner MAY accept or reject the request.
3. If the request is accepted, the Organization owner removes the member from the Organization metadata, and publishes the updated Organization metadata.

### Organization Ban Flow

1. The Organization owner adds a member to the ban list, revokes their grants, and publishes the updated Organization metadata.


## Security Considerations

1. The Organization owner is a single point of failure. If the Organization owner is compromised, the Organization is compromised.

2. Follows the same security considerations as the [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/) protocol.

## Future work

1. To scale and optimize the Organization management, the Organization metadata should be stored on a decentralized storage system, and only the references to the Organization metadata should be broadcasted. The following document describes this method in more detail - [Optimizing the `OrganizationDescription` dissemination](https://hackmd.io/rD1OfIbJQieDe3GQdyCRTw)

## References

- [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/)
- https://github.com/status-im/status-go/blob/6072bd17ab1e5d9fc42cf844fcb8ad18aa07760c/protocol/communities/community.go
- https://github.com/status-im/specs/blob/403b5ce316a270565023fc6a1f8dec138819f4b0/docs/raw/organisation-channels.md
