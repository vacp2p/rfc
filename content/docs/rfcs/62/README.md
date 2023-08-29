---
slug: 62
title: 62/STATUS-Payloads
name: Status Message Payloads
status: raw
editor: r4bbit <r4bbit@status.im>
contributors: 
- Adam Babik <adam@status.im>
- Andrea Maria Piana <andreap@status.im>
- Oskar Thoren <oskar@status.im>
- Samuel Hawksby-Robinson <samuel@status.im>
---

# Abstract

This specification describes how the payload of each message in Status looks
like.
It is primarily centered around chat and chat-related use cases.

The payloads aims to be flexible enough to support messaging but also cases
described in the [Status Whitepaper](https://status.im/whitepaper.pdf) as well
as various clients created using different technologies.

# Wire Format Specification

## Payload wrapper

The node wraps all payloads in a [protobuf record](https://developers.google.com/protocol-buffers/)
record:

```protobuf
message StatusProtocolMessage {
  bytes signature = 4001;
  bytes payload = 4002;
}
```

`signature` is the bytes of the signed `SHA3-256` of the payload, signed with the key of the author of the message.
The node needs the signature to validate authorship of the message, so that the message can be relayed to third parties.
If a signature is not present, but an author is provided by a layer below, the message is not to be relayed to third parties, and it is considered plausibly deniable.

## Encoding

The node encodes the payload using [Protobuf](https://developers.google.com/protocol-buffers)

## Types of messages

### ChatMessage

The type `ChatMessage` represents a chat message exchanged between clients.

#### Payload

The protobuf description is:

```protobuf
message ChatMessage {
  // Lamport timestamp of the chat message
  uint64 clock = 1;
  // Unix timestamps in milliseconds, currently not used as we use whisper as
  // more reliable, but here so that we don't rely on it
  uint64 timestamp = 2;
  // Text of the message
  string text = 3;
  // Id of the message that we are replying to
  string response_to = 4;
  // Ens name of the sender
  string ens_name = 5;
  // Chat id, this field is symmetric for public-chats and private group chats,
  // but asymmetric in case of one-to-ones, as the sender will use the chat-id
  // of the received, while the receiver will use the chat-id of the sender.
  // Probably should be the concatenation of sender-pk & receiver-pk in
  // alphabetical order
  string chat_id = 6;

  // The type of message (public/one-to-one/private-group-chat)
  MessageType message_type = 7;
  // The type of the content of the message
  ContentType content_type = 8;

  oneof payload {
    StickerMessage sticker = 9;
    ImageMessage image = 10;
    AudioMessage audio = 11;
    bytes community = 12;
    DiscordMessage discord_message = 99;
  }

  // Grant for community chat messages
  bytes grant = 13;

  // Message author's display name, introduced in version 1
  string display_name = 14;

  ContactRequestPropagatedState contact_request_propagated_state = 15;

  repeated UnfurledLink unfurled_links = 16;

  enum ContentType {
    UNKNOWN_CONTENT_TYPE = 0;
    TEXT_PLAIN = 1;
    STICKER = 2;
    STATUS = 3;
    EMOJI = 4;
    TRANSACTION_COMMAND = 5;
    // Only local
    SYSTEM_MESSAGE_CONTENT_PRIVATE_GROUP = 6;
    IMAGE = 7;
    AUDIO = 8;
    COMMUNITY = 9;
    // Only local
    SYSTEM_MESSAGE_GAP = 10;
    CONTACT_REQUEST = 11;
    DISCORD_MESSAGE = 12;
    IDENTITY_VERIFICATION = 13;
    // Only local
    SYSTEM_MESSAGE_PINNED_MESSAGE = 14;
    // Only local
    SYSTEM_MESSAGE_MUTUAL_EVENT_SENT = 15;
    // Only local
    SYSTEM_MESSAGE_MUTUAL_EVENT_ACCEPTED = 16;
    // Only local
    SYSTEM_MESSAGE_MUTUAL_EVENT_REMOVED = 17;
  }
}
```

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | The clock of the chat|
| 2 | timestamp | `uint64` | The sender timestamp at message creation |
| 3 | text | `string` | The content of the message |
| 4 | response_to | `string` | The ID of the message replied to |
| 5 | ens_name | `string` | The ENS name of the user sending the message |
| 6 | chat_id | `string` | The local ID of the chat the message is sent to |
| 7 | message_type | `MessageType` | The type of message, different for one-to-one, public or group chats |
| 8 | content_type | `ContentType` | The type of the content of the message | 
| 9 | payload | `Sticker` I `Image` I `Audio` I `DiscordMessage` I `bytes` I nil` | The payload of the message based on the content type |
| 13 | grant | `bytes` | Grant for community chat messages |
| 14 | display_name | `string` | The message author's display name |
| 15 | contact_request_propagated_state | `ContactRequestPropagatedState` | Contact request clock values |
| 16 | unfurled_links | `UnfurledLink[]` | Unfurled links and their metadata that have been attached to the message |

#### Content types

A node requires content types for a proper interpretation of incoming messages. Not each message is plain text but may carry different information.

The following content types MUST be supported:
* `TEXT_PLAIN` identifies a message which content is a plaintext.

There are other content types that MAY be implemented by the client:
* `STICKER`
* `STATUS`
* `EMOJI`
* `TRANSACTION_COMMAND`
* `IMAGE`
* `AUDIO`
* `COMMUNITY`
* `CONTACT_REQUEST`
* `DISCORD_MESSAGE`
* `IDENTITY_VERIFICATION`

##### Mentions 

A mention MUST be represented as a string with the `@0xpk` format, where `pk` is the public key of the [user account](https://specs.status.im/spec/2) to be mentioned, within the `text` field of a message with content_type `TEXT_PLAIN`.
A message MAY contain more than one mention.
This specification RECOMMENDs that the application does not require the user to enter the entire pk. 
This specification RECOMMENDs that the application allows the user to create a mention by typing @ followed by the related ENS or 3-word pseudonym.
This specification RECOMMENDs that the application provides the user auto-completion functionality to create a mention.
For better user experience, the client SHOULD display a known [ens name or the 3-word pseudonym corresponding to the key](https://specs.status.im/spec/2#contact-verification) instead of the `pk`.

##### Sticker content type

A `ChatMessage` with `STICKER` `Content/Type` MUST also specify the ID of the `Pack` and 
the `Hash` of the pack, in the `Sticker` field of `ChatMessage`

```protobuf
message StickerMessage {
  string hash = 1;
  int32 pack = 2;
}
```

##### Image content type

A `ChatMessage` with `IMAGE` `Content/Type` MUST also specify the `payload` of the image
and the `type`.

Clients MUST sanitize the payload before accessing its content, in particular: 
- Clients MUST choose a secure decoder
- Clients SHOULD strip metadata if present without parsing/decoding it
- Clients SHOULD NOT add metadata/exif when sending an image file for privacy and security reasons
- Clients MUST make sure that the transport layer constraints the size of the payload to limit they are able to handle securely


```protobuf
message ImageMessage {
  bytes payload = 1;
  ImageType type = 2;
  enum ImageType {
    UNKNOWN_IMAGE_TYPE = 0;
    PNG = 1;
    JPEG = 2;
    WEBP = 3;
    GIF = 4;
  }
}
```

##### Audio content type

A `ChatMessage` with `AUDIO` `Content/Type` MUST also specify the `payload` of the audio,
the `type` and the duration in milliseconds (`duration_ms`).

Clients MUST sanitize the payload before accessing its content, in particular: 
- Clients MUST choose a secure decoder
- Clients SHOULD strip metadata if present without parsing/decoding it
- Clients SHOULD NOT add metadata/exif when sending an audio file for privacy and security reasons
- Clients MUST make sure that the transport layer constraints the size of the payload to limit they are able to handle securely

```protobuf
message AudioMessage {
  bytes payload = 1;
  AudioType type = 2;
  uint64 duration_ms = 3;
  enum AudioType {
    UNKNOWN_AUDIO_TYPE = 0;
    AAC = 1;
    AMR = 2;
```

##### Community content type

A `ChatMessage` with `COMMUNITY` `Content/Type` MUST also specify the `payload` of the community as bytes from a [CommunityDescription](#communitydescription).

##### DiscordMessage content type

A `ChatMessage` with `DISCORD_MESSAGE` `Content/Type` MUST also specify the `payload` of the `DiscordMessage`.

```protobuf
message DiscordMessage {
  string id = 1;
  string type = 2;
  string timestamp = 3;
  string timestampEdited = 4;
  string content = 5;
  DiscordMessageAuthor author = 6;
  DiscordMessageReference reference = 7;
  repeated DiscordMessageAttachment attachments = 8;
}

message DiscordMessageAuthor {
  string id = 1;
  string name = 2;
  string discriminator = 3;
  string nickname = 4;
  string avatarUrl = 5;
  bytes avatarImagePayload = 6;
  string localUrl = 7;
}

message DiscordMessageReference {
  string messageId = 1;
  string channelId = 2;
  string guildId = 3;
}

message DiscordMessageAttachment {
  string id = 1;
  string messageId = 2;
  string url = 3;
  string fileName = 4;
  uint64 fileSizeBytes = 5;
  string contentType = 6;
  bytes payload = 7;
  string localUrl = 8;
}
```

#### Message types

A node requires message types to decide how to encrypt a particular message and what metadata needs to be attached when passing a message to the transport layer.
For more on this, see [10/WAKU2](/spec/10).

<!-- TODO: This reference is a bit odd, considering the layer payloads should interact with is Secure Transport, and not Whisper/Waku. This requires more detail -->


The following messages types MUST be supported:
* `ONE_TO_ONE` is a message to the public group
* `PUBLIC_GROUP` is a private message
* `PRIVATE_GROUP` is a message to the private group.
* `COMMUNITY_CHAT` is a message to a community channe.

```protobuf
enum MessageType {
  UNKNOWN_MESSAGE_TYPE = 0;
  ONE_TO_ONE = 1;
  PUBLIC_GROUP = 2;
  PRIVATE_GROUP = 3;
  // Only local
  SYSTEM_MESSAGE_PRIVATE_GROUP = 4;
  COMMUNITY_CHAT = 5;
  // Only local
  SYSTEM_MESSAGE_GAP = 6;
}
```

#### Clock vs Timestamp and message ordering

If a user sends a new message before the messages sent while the user was offline are received, the new message is supposed to be displayed last in a chat.
This is where the basic algorithm of Lamport timestamp would fall short as it's only meant to order causally related events.

The status client therefore makes a "bid", speculating that it will beat the current chat-timestamp, s.t. the status client's Lamport timestamp format is: `clock = `max({timestamp}, chat_clock + 1)`

This will satisfy the Lamport requirement, namely: a -> b then T(a) < T(b)

`timestamp` MUST be Unix time calculated, when the node creates the message, in milliseconds. 
This field SHOULD not be relied upon for message ordering.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). 
When there are messages available in a chat, the node calculates `clock`'s value based on the last received message in a particular chat: `max(timeNowInMs, last-message-clock-value + 1)`. 
If there are no messages, `clock` is initialized with `timestamp`'s value.

Messages with a `clock` greater than `120` seconds over the Whisper/Waku timestamp SHOULD be discarded, in order to avoid malicious users to increase the `clock` of a chat arbitrarily.

Messages with a `clock` less than `120` seconds under the Whisper/Waku timestamp might indicate an attempt to insert messages in the chat history which is not distinguishable from a `datasync` layer re-transit event. 
A client MAY mark this messages with a warning to the user, or discard them.

The node uses `clock` value for the message ordering. The algorithm used, and the distributed nature of the system produces casual ordering, which might produce counter-intuitive results in some edge cases. 
For example, when a user joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

#### Chats

Chat is a structure that helps organize messages. 
It's usually desired to display messages only from a single recipient, or a group of recipients at a time and chats help to achieve that.

All incoming messages can be matched against a chat. 
The below table describes how to calculate a chat ID for each message type.

|Message Type|Chat ID Calculation|Direction|Comment|
|------------|-------------------|---------|-------|
|PUBLIC_GROUP|chat ID is equal to a public channel name; it should equal `chatId` from the message|Incoming/Outgoing||
|ONE_TO_ONE|let `P` be a public key of the recipient; `hex-encode(P)` is a chat ID; use it as `chatId` value in the message|Outgoing||
|user-message|let `P` be a public key of message's signature; `hex-encode(P)` is a chat ID; discard `chat-id` from message|Incoming|if there is no matched chat, it might be the first message from public key `P`; the node MAY discard the message or MAY create a new chat; Status official clients create a new chat|
|PRIVATE_GROUP|use `chatId` from the message|Incoming/Outgoing|find an existing chat by `chatId`; if none is found, the user is not a member of that chat or the user hasn't joined that chat, the message MUST be discarded |
|COMMUNITY_CHAT|use `chatId` from the message|Incoming/Outgoing| find existing community chat within a community based on `chatId`; the `chatId` includes a community id prefix |

### ContactUpdate

`ContactUpdate` is a message exchange to notify peers that either the user has been added as a contact, or that information about the sending user have changed.

```protobuf
message ContactUpdate {
  uint64 clock = 1;
  string ens_name = 2;
  string profile_image = 3;
  string display_name = 4;
  uint64 contact_request_clock = 5;
  ContactRequestPropagatedState contact_request_propagated_state = 6;
  string public_key = 7;
}

message ContactRequestPropagatedState {
  uint64 local_clock = 1;
  uint64 local_state = 2;
  uint64 remote_clock = 3;
  uint64 remote_state = 4;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | The clock of the chat with the user |
| 2 | ens_name | `string` | The ENS name if set |
| 3 | profile_image | `string` | The base64 encoded profile picture of the user |
| 4 | display_name | `string` | The display name of the user |
| 5 | contact_request_clock | `uint64` | The clock at which a contact request was sent |
| 6 | contact_request_propagated_state | `ContactRequestPropagatedState` | Additional propagation clock values |
| 7 | public_key | `string` | The public key of the user |

#### Contact update

A client SHOULD send a `ContactUpdate` to all the contacts each time:

- The ens_name has changed
- A user edits the profile image

A client SHOULD also periodically send a `ContactUpdate` to all the contacts, the interval is up to the client, the Status official client sends these updates every 48 hours.

### EmojiReaction

`EmojiReaction`s represents a user's "reaction" to a specific chat message. 
For more information about the concept of emoji reactions see [Facebook Reactions](https://en.wikipedia.org/wiki/Facebook_like_button#Use_on_Facebook).

This specification RECOMMENDS that the UI/UX implementation of sending `EmojiReactions` requires only a single click operation, as users have an expectation that emoji reactions are effortless and simple to perform.  

```protobuf
message EmojiReaction {
  // clock Lamport timestamp of the chat message
  uint64 clock = 1;

  // chat_id the ID of the chat the message belongs to, for query efficiency the chat_id is stored in the db even though the
  // target message also stores the chat_id
  string chat_id = 2;

  // message_id the ID of the target message that the user wishes to react to
  string message_id = 3;

  // message_type is (somewhat confusingly) the ID of the type of chat the message belongs to
  MessageType message_type = 4;

  // type the ID of the emoji the user wishes to react with
  Type type = 5;

  enum Type {
    UNKNOWN_EMOJI_REACTION_TYPE = 0;
    LOVE = 1;
    THUMBS_UP = 2;
    THUMBS_DOWN = 3;
    LAUGH = 4;
    SAD = 5;
    ANGRY = 6;
  }

 // whether this is a retraction of a previously sent emoji
  bool retracted = 6;
}
```

Clients MUST specify `clock`, `chat_id`, `message_id`, `type` and `message_type`.

This specification RECOMMENDS that the UI/UX implementation of retracting an `EmojiReaction`s requires only a single click operation, as users have an expectation that emoji reaction removals are effortless and simple to perform.  

### MembershipUpdateMessage and MembershipUpdateEvent

`MembershipUpdateEvent` is a message used to propagate information about group membership changes in a group chat.
The details are in the [Group chats specs](./7-group-chat.md).


```protobuf
message MembershipUpdateMessage {
  // The chat id of the private group chat
  string chat_id = 1;
  // A list of events for this group chat, first x bytes are the signature, then is a 
  // protobuf encoded MembershipUpdateEvent
  repeated bytes events = 2;

  // An optional chat message
  oneof chat_entity {
    ChatMessage message = 3;
    EmojiReaction emoji_reaction = 4;
  }
}

message MembershipUpdateEvent {
  // Lamport timestamp of the event
  uint64 clock = 1;
  // List of public keys of objects of the action
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
    CHAT_CREATED = 1;
    NAME_CHANGED = 2;
    MEMBERS_ADDED = 3;
    MEMBER_JOINED = 4;
    MEMBER_REMOVED = 5;
    ADMINS_ADDED = 6;
    ADMIN_REMOVED = 7;
    COLOR_CHANGED = 8;
    IMAGE_CHANGED = 9;
  }
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | chat_id | `string` | The chat id of the private group chat |
| 2 | events | `bytes[]` | The list of events for this group chat |

#### Chat entities

A `MembershipUpdateMessage` includes either a `ChatMessage` or `EmojiReaction`.

### SyncInstallationContactV2

The node uses `SyncInstallationContact` messages to synchronize in a best-effort the contacts to other devices.

```protobuf
message SyncInstallationContactV2 {
  uint64 last_updated_locally = 1;
  string id = 2;
  string profile_image = 3;
  string ens_name = 4;
  uint64 last_updated = 5;
  repeated string system_tags = 6;
  string local_nickname = 7;
  bool added = 9;
  bool blocked = 10;
  bool muted = 11;
  bool removed = 12;
  bool has_added_us = 13;
  int64 verification_status = 14;
  int64 trust_status = 15;
  int64 contact_request_local_state = 16;
  int64 contact_request_local_clock = 17;
  int64 contact_request_remote_state = 18;
  int64 contact_request_remote_clock = 19;
  string display_name = 20;
}
```


#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | last_updated_locally | `uint64` | Timestamp of last local update | 
| 2 | id | `string` | id of the contact synced |
| 3 | profile_image | `string` |  `base64` encoded profile picture of the user |
| 4 | ens_name | `string` | ENS name of the contact |
| 5 | `array[string]` | Array of `system_tags` for the user, this can currently be: `":contact/added", ":contact/blocked", ":contact/request-received"`|
| 7 | local_nickname | `string` | Local display name of the contact |
| 9 | added | `bool` | Wether the contact is added |
| 10 | blocked | `bool` | Wether the contact is blocked |
| 11 | muted | `bool` | Wether the contact is muted |
| 12 | removed | `bool` | Wether the contact is removed |
| 13 | has_added_us | `bool` | Wether the contact has added us |
| 14 | verification_status | `int64` | The verification status of the contact |
| 15 | trust_status | `int64` | The trust status of the contact |
| 16 | contact_request_local_state | `int64` | The local contact request state |
| 17 | contact_request_local_clock | `int64` | The local contact request clock |
| 18 | contact_request_remote_state | `int64` | The remote contact request state |
| 19 | contact_request_remote_clock | `int64` | The remote contact request clock |

### SyncInstallationPublicChat

The node uses `SyncInstallationPublicChat` message to synchronize in a best-effort the public chats to other devices.

```protobuf
message SyncInstallationPublicChat {
  uint64 clock = 1;
  string id = 2;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | clock value of the chat | 
| 2 | id | `string` | id of the chat synced |

### SyncPairInstallation

The node uses `PairInstallation` messages to propagate information about a device to its paired devices.

```protobuf
message SyncPairInstallation {
  uint64 clock = 1;
  string installation_id = 2;
  string device_type = 3;
  string name = 4;
  // following fields used for local pairing
  uint32 version = 5;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | clock value of the chat | 
| 2| installation_id | `string` | A randomly generated id that identifies this device |
| 3 | device_type | `string` | The OS of the device `ios`,`android` or `desktop` |
| 4 | name | `string` | The self-assigned name of the device |

### ChatIdentity

`ChatIdentity` represents the user defined identity associated with their public chat key.

```protobuf
message ChatIdentity {
  uint64 clock = 1;
  string ens_name = 2;
  map<string, IdentityImage> images = 3;
  string display_name = 4;
  string description = 5;
  string color = 6;
  string emoji = 7;
  repeated SocialLink social_links = 8;
  // first known message timestamp in seconds (valid only for community chats for now)
  // 0 - unknown
  // 1 - no messages
  uint32 first_message_timestamp = 9;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2| ens_name | `string` | A valid ENS associated with the chat key |
| 3 | images | `map<string, IdentityImage>` | Image data associated with the chat key |
| 4 | display_name | `string` | The self-assigned display_name of the chat key |
| 5 | description | `string` | The description of the chat |
| 6 | color | `string` | The color of the chat |
| 7 | emoji | `string` | The emoji of the chat |
| 8 | social_links | `array<SocialLink>` | A list of links to social platforms |
| 9 | first_message_timestamp | `uint32` | First known message timestamp in seconds |

### CommunityDescription

`CommunityDescription` represents a community metadata that is used to discover communities and share community updates.


```protobuf
message CommunityDescription {
  uint64 clock = 1;
  map<string,CommunityMember> members = 2;
  CommunityPermissions permissions = 3;
  ChatIdentity identity = 5;
  map<string,CommunityChat> chats = 6;
  repeated string ban_list = 7;
  map<string,CommunityCategory> categories = 8;
  uint64 archive_magnetlink_clock = 9;
  CommunityAdminSettings admin_settings = 10;
  string intro_message = 11;
  string outro_message = 12;
  bool encrypted = 13;
  repeated string tags = 14;
  map<string, CommunityTokenPermission> token_permissions = 15;
  repeated CommunityTokenMetadata community_tokens_metadata = 16;
  uint64 active_members_count = 17;
}

message CommunityMember {
  enum Roles {
    reserved 2, 3;
    reserved "ROLE_MANAGE_USERS", "ROLE_MODERATE_CONTENT";
    ROLE_NONE = 0;
    ROLE_OWNER = 1;
    ROLE_ADMIN = 4;
    ROLE_TOKEN_MASTER = 5;
  }
  repeated Roles roles = 1;
  repeated RevealedAccount revealed_accounts = 2 [deprecated = true];
  uint64 last_update_clock = 3;
}

message CommunityPermissions {
  enum Access {
    UNKNOWN_ACCESS = 0;
    NO_MEMBERSHIP = 1;
    INVITATION_ONLY = 2;
    ON_REQUEST = 3;
  }

  bool ens_only = 1;
  // https://gitlab.matrix.org/matrix-org/olm/blob/master/docs/megolm.md is a candidate for the algorithm to be used in case we want to have private communityal chats, lighter than pairwise encryption using the DR, less secure, but more efficient for large number of participants
  bool private = 2;
  Access access = 3;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2| members | `map<string, CommunityMember>` | The members of the community |
| 3 | permissions | `CommunityPermissions` | Image data associated with the chat key |
| 4 | display_name | `string` | The self-assigned display_name of the chat key |
| 5 | description | `string` | The description of the chat |
| 6 | color | `string` | The color of the chat |
| 7 | emoji | `string` | The emoji of the chat |
| 8 | social_links | `array<SocialLink>` | A list of links to social platforms |
| 9 | first_message_timestamp | `uint32` | First known message timestamp in seconds |

### CommunityRequestToJoin

A `CommunityRequestToJoin` represents a request to join a community, sent by a user that is not yet a member of that community. 
A request to join a community includes a list of `RevealedAccount`.
These are wallet addresses that users are willing to reveal with the community's control node and admins.

```protobuf
message CommunityRequestToJoin {
  uint64 clock = 1;
  string ens_name = 2;
  string chat_id = 3;
  bytes community_id = 4;
  string display_name = 5;
  repeated RevealedAccount revealed_accounts = 6;
}

message RevealedAccount {
  string address = 1;
  bytes signature = 2;
  repeated uint64 chain_ids = 3;
  bool isAirdropAddress = 4;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2| ens_name | `string` | The ENS of the user sending the request |
| 3 | chat_id | `string` | The id of the chat to request access to |
| 4 | community_id | `bytes` | The public key of the community |
| 5 | display_name | `string` | The display name of the usre sending the request |
| 6 | revealed_accounts | `array<RevealedAccount>` | A list of accounts to reveal to the control node |

### PinMessage

A `PinMessage` is a signal that tells a client that a specific message has to be marked as pinned.

```protobuf
message PinMessage {
  uint64 clock = 1;
  string message_id = 2;
  string chat_id = 3;
  bool pinned = 4;
  MessageType message_type = 5;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2| message_id | `string` | The id of the message to be pinned |
| 3 | chat_id | `string` | The id of the chat of the message to be pinned |
| 4 | pinned | `bool` | Whether the message should be pinned or unpinned |
| 5 | message_type | `MessageType` | The type of message (public/one-to-one/private-group-chat) |


### EditMessage

A `EditMessage` represents an update to an existing message.


```protobuf
message EditMessage {
  uint64 clock = 1;
  string text = 2;

  string chat_id = 3;
  string message_id = 4;

  bytes grant = 5;

  MessageType message_type = 6;

  ChatMessage.ContentType content_type = 7;
  repeated UnfurledLink unfurled_links = 8;
}

```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2| text | `string` | The updated message text |
| 3 | chat_id | `string` | The id of the chat of the message |
| 4 | message_id | `string` | The id of the message to be edited |
| 5 | grant | `bytes` | A grant for a community edit messages  |
| 6 | message_type | `MessageType` | The type of message  |
| 7 | content_type | `ChatMessage.ContentType` | The updated content type of the message  |
| 8 | unfurled_links | `array<UnfurledLink>` | Updated link metadata  |


### DeleteMessage

A `DeleteMessage` represents a signal to delete a message from the local database of a client.

```protobuf
message DeleteMessage {
  uint64 clock = 1;

  string chat_id = 2;
  string message_id = 3;

  // Grant for community delete messages
  bytes grant = 4;

  // The type of message (public/one-to-one/private-group-chat)
  MessageType message_type = 5;

  string deleted_by = 6;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | chat_id | `string` | The id of the chat of the message |
| 3 | message_id | `string` | The id of the message to delete |
| 4 | grant | `bytes` | A grant for a community edit messages  |
| 5 | message_type | `MessageType` | The type of message  |
| 6 | deleted_by | `string` | The public key of the user who deleted the message |


### CommunityMessageArchiveLink

A `CommunityMessageArchiveLink` contains a magnet uri for a community's message archive, created using [61/STATUS-Community-History-Archives](/spec/61/).

```protobuf
message CommunityMessageArchiveMagnetlink {
  uint64 clock = 1;
  string magnet_uri = 2;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | magnet_uri | `string` | The magnet uri of the community archive torrent |

### AcceptContactRequest

A `AcceptContractRequest` message signals to the requester that the request was accepted.

```protobuf
message AcceptContactRequest {
  string id = 1;
  uint64 clock = 2;
}

```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | id | `string` | The id of the contact request |
| 2 | clock | `uint64` | Clock value of the message | 

### RetractContactRequest

A `RetractContractRequest` message signals to the reiver of a request that the request was retracted.

```protobuf
message RetractContactRequest {
  string id = 1;
  uint64 clock = 2;
}

```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | id | `string` | The id of the contact request |
| 2 | clock | `uint64` | Clock value of the message | 

### CommunityRequestToJoinResponse

A `CommunityRequestToJoinResponse` represents a response to a request to join a community.

```protobuf
message CommunityRequestToJoinResponse {
  uint64 clock = 1;
  CommunityDescription community = 2;
  bool accepted = 3;
  bytes grant = 4;
  bytes community_id = 5;
  string magnet_uri = 6;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | community | `CommunityDescription` | The community metadata |
| 3 | accepted | `bool` | Whether the request was accepted |
| 4 | grant | `bytes` | The grant |
| 5 | community_id | `bytes` | The id of the community |
| 6 | magnet_uri | `string` | The latest magnet uri of the community's archive torrent |

### CommunityRequestToLeave

A `CommunityRequestToLeave` represents a signal to a community that a user wants to be removed from the community's member list.

```protobuf
message CommunityRequestToLeave {
  uint64 clock = 1;
  bytes community_id = 2;
}
```
#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | community_id | `bytes` | The id of the community |


### RequestContactVerification

A `RequestContactVerification` is a request to verify a contact using a "challenge", which can by any string message and typically involves questions that only the contact should know.

```protobuf
message RequestContactVerification {
  uint64 clock = 1;
  string challenge = 3;
}
```
#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | challenge | `string` | The challenge message used for verification |

### AcceptContactVerification

A `AcceptContactVerification` signals that a verification request was accepted and includes a response to the challenge.


```protobuf
message AcceptContactVerification {
  uint64 clock = 1;
  string id = 2;
  string response = 3;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | id | `string` | The verification request id |
| 3 | response | `string` | The response for the challenge |

### DeclineContactVerification

A `DeclineContactVerification` declines a contact verification request.

```protobuf
message DeclineContactVerification {
  uint64 clock = 1;
  string id = 2;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | id | `string` | The verification request id |

### CancelContactVerification

A `CancelContactVerification` cancels a contact verification request.

```protobuf
message CancelContactVerification {
  uint64 clock = 1;
  string id = 2;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | id | `string` | The verification request id |


### CommunityCancelRequestToJoin

A `CommunityCancelRequestToJoin` cancels a pending request to join.

```protobuf
message CommunityCancelRequestToJoin {
  uint64 clock = 1;
  string ens_name = 2;
  string chat_id = 3;
  bytes community_id = 4;
  string display_name = 5;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | ens_name | `string` | The ENS name of the account cancelling the request |
| 3 | chat_id | `string` | The id of the chat |
| 4 | community_id | `bytes` | The id of the community |
| 5 | display_name | `string` | The display name of the account cancelling the request |

### CommunityEditSharedAddresses

A `CommunityEditSharedAddresses` message allows users to edit the shared accounts they've revealed when requesting to join a community.

```protobuf
message CommunityEditSharedAddresses {
  uint64 clock = 1;
  bytes community_id = 2;
  repeated RevealedAccount revealed_accounts = 3;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | Clock value of the message | 
| 2 | community_id | `bytes` | The id of the community |
| 3 | revealed_accounts | `array<RevealedAccount>` | A list of revealed accounts |

## Upgradability

There are two ways to upgrade the protocol without breaking compatibility:

- A node always supports accretion
- A node does not support deletion of existing fields or messages, which might break compatibility

## Security Considerations

-

## Changelog

### Version 0.5

Released [August 25, 2020](https://github.com/status-im/specs/commit/968fafff23cdfc67589b34dd64015de29aaf41f0)

- Added support for emoji reactions

### Version 0.4

Released [July 16, 2020](https://github.com/status-im/specs/commit/ad45cd5fed3c0f79dfa472253a404f670dd47396)

- Added support for images
- Added support for audio

### Version 0.3

Released [May 22, 2020](https://github.com/status-im/specs/commit/664dd1c9df6ad409e4c007fefc8c8945b8d324e8)

- Added language to include Waku in all relevant places

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

