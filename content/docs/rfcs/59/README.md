---
slug: 59
title: 59/PUSH-NOTIFICATION-SERVER
name: Push notification server RFC Template
status: raw
editor: Andrea Maria Piana <andreap@status.im>
contributors:
---


# 16/PUSH-NOTIFICATION-SERVER

- [Push Notification Server](#16push-notification-server)
  - [Reason](#reason)
  - [Requirements](#requirements)
  - [Components](#components)
  - [Registering with the push notification service](#registering-with-the-push-notification-service)
  - [Re-registering with the push notification service](#re-registering-with-the-push-notification-server)
  - [Changing options](#changing-options)
  - [Unregistering from push notifications](#unregistering-from-push-notifications)
  - [Advertising a push notification server](#advertising-a-push-notification-server)
  - [Discovering a push notification server](#discovering-a-push-notification-server)
  - [Querying the push notification service](#querying-the-push-notification-server)
  - [Sending a push notification](#sending-a-push-notification)
  - [Flow](#flow)
    - [Registration process](#registration-process)
    - [Sending a notification](#sending-a-notification)
    - [Receiving a push notification](#receiving-a-push-notification)
  - [Protobuf description](#protobuf-description)
    - [PushNotificationRegistration](#pushnotificationregistration)
    - [PushNotificationRegistrationResponse](#pushnotificationregistrationresponse)
    - [ContactCodeAdvertisement](#contactcodeadvertisement)
    - [PushNotificationQuery](#pushnotificationquery)
    - [PushNotificationQueryInfo](#pushnotificationqueryinfo)
    - [PushNotificationQueryResponse](#pushnotificationqueryresponse)
    - [PushNotification](#pushnotification)
    - [PushNotificationRequest](#pushnotificationrequest)
    - [PushNotificationResponse](#pushnotificationacknowledgement)
    - [PushNotificationReport](#pushnotificationreport)
  - [Anonymous mode of operations](#anonymous-mode-of-operations)
  - [Security considerations](#security-considerations)
  - [FAQ](#faq)
  - [Changelog](#changelog)
    - [Version 0.1](#version-01)
  - [Copyright](#copyright)

## Reason

Push notifications for iOS devices and some Android devices can only be
implemented by relying on [APN service](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1) for iOS or [Firebase](https://firebase.google.com/).

This is useful for Android devices that do not support foreground services or that often kill the
foreground service.

iOS only allows certain kind of applications to keep a connection open when in the
background, VoIP for example, which current status client does not qualify for.

Applications on iOS can also request execution time when they are in the [background](https://developer.apple.com/documentation/uikit/app_and_environment/scenes/preparing_your_ui_to_run_in_the_background/updating_your_app_with_background_app_refresh)
but it has a limited set of use cases, for example it won't schedule any time
if the application was force quit, and generally is not responsive enough to implement
a push notification system.

Therefore Status provides a set of Push notification services that can be used
to achieve this functionality.

Because this can't be safely implemented in a privacy preserving manner, clients
MUST be given an option to opt-in to receiving and sending push notifications. They are disabled by default.

## Requirements

The party releasing the app MUST possess a certificate for the Apple Push Notification service and its has to run a [gorush](https://github.com/appleboy/gorush) publicly accessible server for sending the actual notification. 
The party releasing the app, Status in this case, needs to run its own [gorush](https://github.com/appleboy/gorush)

## Components

### Gorush instance

A [gorush](https://github.com/appleboy/gorush) instance MUST be publicly
available, this will be used only by push notification servers.

### Push notification server

A push notification server used by clients to register for receiving and sending push notifications.

### Registering client

A Status client that wants to receive push notifications

### Sending client

A Status client that wants to send push notifications

## Registering with the push notification service

A client MAY register with one or more Push Notification services of their choice.

A `PNR message` (Push Notification Registration) MUST be sent to the [partitioned topic](../stable/10-waku-usage.md#partitioned-topic) for the
public key of the node, encrypted with this key.

The message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_REGISTRATION`. 

The marshaled protobuf payload MUST also be encrypted with AES-GCM using the Diffie–Hellman key
generated from the client and server identity.

This is done in order to ensure that the extracted key from the signature will be
considered invalid if it can't decrypt the payload.

The content of the message MUST contain the following [protobuf record](https://developers.google.com/protocol-buffers/):

```protobuf
message PushNotificationRegistration {
  enum TokenType {
    UNKNOWN_TOKEN_TYPE = 0;
    APN_TOKEN = 1;
    FIREBASE_TOKEN = 2;
  }
  TokenType token_type = 1;
  string device_token = 2;
  string installation_id = 3;
  string access_token = 4;
  bool enabled = 5;
  uint64 version = 6;
  repeated bytes allowed_key_list = 7;
  repeated bytes blocked_chat_list = 8;
  bool unregister = 9;
  bytes grant = 10;
  bool allow_from_contacts_only = 11;
  string apn_topic = 12;
  bool block_mentions = 13;
  repeated bytes allowed_mentions_chat_list = 14;
}
```

A push notification server will handle the message according to the following rules:

- it MUST extract the public key of the sender from the signature and verify that 
  the payload can be decrypted successfully.
- it MUST verify that `token_type` is supported
- it MUST verify that `device_token` is non empty
- it MUST verify that `installation_id` is non empty
- it MUST verify that `version` is non-zero and greater than the currently stored version for the public key and installation id of the sender, if any
- it MUST verify that `grant` is non empty and according to the [specs](#server-grant)
- it MUST verify that `access_token` is a valid [`uuid`](https://tools.ietf.org/html/rfc4122)
- it MUST verify that `apn_topic` is set if `token_type` is `APN_TOKEN`

If the message can't be decrypted, the message MUST be discarded.

If `token_type` is not supported, a response MUST be sent with `error` set to
`UNSUPPORTED_TOKEN_TYPE`.

If `token`,`installation_id`,`device_tokens`,`version` are empty, a response MUST
be sent with `error` set to `MALFORMED_MESSAGE`.

If the `version` is equal or less than the currently stored version, a response MUST
be sent with `error` set to `VERSION_MISMATCH`.

If any other error occurs the `error` should be set to `INTERNAL_ERROR`.

If the response is successful `success` MUST be set to `true` otherwise a response MUST be sent with `success` set to `false`.


`request_id` should be set to the `SHAKE-256` of the encrypted payload.

The response MUST be sent on the [partitioned topic][./10-waku-usage.md#partitioned-topic] of the sender
and MUST not be encrypted using the [secure transport](../docs/stable/5-secure-transport.md) to facilitate
the usage of ephemeral keys.

The payload of the response is:

```protobuf
message PushNotificationRegistrationResponse {
  bool success = 1;
  ErrorType error = 2;
  bytes request_id = 3;

  enum ErrorType {
    UNKNOWN_ERROR_TYPE = 0;
    MALFORMED_MESSAGE = 1;
    VERSION_MISMATCH = 2;
    UNSUPPORTED_TOKEN_TYPE = 3;
    INTERNAL_ERROR = 4;
  }
}
```

The message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_REGISTRATION_RESPONSE`. 

A client SHOULD listen for a response sent on the [partitioned topic][./10-waku-usage.md#partitioned-topic]
that the key used to register.

If `success` is `true` the client has registered successfully.

If `success` is `false`:

- If `MALFORMED_MESSAGE` is returned, the request SHOULD NOT be retried without ensuring
  that it is correctly formed.
- If `INTERNAL_ERROR` is returned, the request MAY be retried, but the client MUST 
  backoff exponentially

A client MAY register with multiple Push Notification Servers in order to increase availability.

A client SHOULD make sure that all the notification services they registered with have the same information about their tokens.

If no response is returned the request SHOULD be considered failed and MAY be retried with the same server or a different one, but clients MUST exponentially backoff after each trial.

If the request is successful the token SHOULD be [advertised](#advertising-a-push-notification-server) as described below

### Query topic

On successful registration the server MUST be listening to the topic derived from:

```
   0XHexEncode(Shake256(CompressedClientPublicKey))
```

Using the topic derivation algorithm described [here](../stable/10-waku-usage.md#public-chats)
and listen for client queries.

### Server grant

A push notification server needs to demonstrate to a client that it was authorized
by the client to send them push notifications. This is done by building 
a grant which is specific to a given client-server pair.
The grant is built as follow:

```
   Signature(Keccak256(CompressedPublicKeyOfClient . CompressedPublicKeyOfServer . AccessToken), PrivateKeyOfClient)
```

When receiving a grant the server MUST be validate that the signature matches the registering client.

## Re-registering with the push notification server

A client SHOULD re-register with the node if the APN or FIREBASE token changes.

When re-registering a client SHOULD ensure that it has the most up-to-date
`PushNotificationRegistration` and increment `version` if necessary.

Once re-registered, a client SHOULD advertise the changes.


## Changing options

This is handled in exactly the same way as re-registering above.

## Unregistering from push notifications

To unregister a client MUST send a `PushNotificationRegistration` request as described
above with `unregister` set to `true`, or removing
their device information.

The server MUST remove all data about this user if `unregistering` is `true`,
apart from the `hash` of the public key and the `version` of the last options,
in order to make sure that old messages are not processed.

A client MAY unregister from a server on explicit logout if multiple chat keys
are used on a single device.

## Advertising a push notification server

Each user registered with one or more push notification servers SHOULD
advertise periodically the push notification services that they have registered with for each device they own.

```protobuf
message PushNotificationQueryInfo {
  string access_token = 1;
  string installation_id = 2;
  bytes public_key = 3;
  repeated bytes allowed_user_list = 4;
  bytes grant = 5;
  uint64 version = 6;
  bytes server_public_key = 7;
}

message ContactCodeAdvertisement {
  repeated PushNotificationQueryInfo push_notification_info = 1;
}
```

The message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_QUERY_INFO`. 

If no filtering is done based on public keys,
the access token SHOULD be included in the advertisement.
Otherwise it SHOULD be left empty.

This SHOULD be advertised on the [contact code topic](./10-waku-usage.md#contact-code-topic)
and SHOULD be coupled with normal contact-code advertisement.

Every time a user register or re-register with a push notification service, their
contact-code SHOULD be re-advertised.

Multiple servers MAY be advertised for the same `installation_id` for redundancy reasons.

## Discovering a push notification server

To discover a push notification service for a given user, their [contact code topic](./10-waku-usage.md#contact-code-topic)
SHOULD be listened to.
A mailserver can be queried for the specific topic to retrieve the most up-to-date
contact code.

## Querying the push notification server

If a token is not present in the latest advertisement for a user, the server
SHOULD be queried directly.

To query a server a message:

```protobuf
message PushNotificationQuery {
  repeated bytes public_keys = 1;
}
```

The message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_QUERY`. 

MUST be sent to the server on the topic derived from the hashed public key of the
key we are querying, as [described above](#query-topic).

An ephemeral key SHOULD be used and SHOULD NOT be encrypted using the [secure transport](../docs/stable/5-secure-transport.md).

If the server has information about the client a response MUST be sent:

```protobuf
message PushNotificationQueryInfo {
  string access_token = 1;
  string installation_id = 2;
  bytes public_key = 3;
  repeated bytes allowed_user_list = 4;
  bytes grant = 5;
  uint64 version = 6;
  bytes server_public_key = 7;
}

message PushNotificationQueryResponse {
  repeated PushNotificationQueryInfo info = 1;
  bytes message_id = 2;
  bool success = 3;
}
```

A `PushNotificationQueryResponse` message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_QUERY_RESPONSE`. 

Otherwise a response MUST NOT be sent.

If `allowed_key_list` is not set `access_token` MUST be set and `allowed_key_list` MUST NOT
be set.

If `allowed_key_list` is set `allowed_key_list` MUST be set and `access_token` MUST NOT be set.

If `access_token` is returned, the `access_token` SHOULD be used to send push notifications.


If `allowed_key_list` are returned, the client SHOULD decrypt each
token by generating an `AES-GCM` symmetric key from the Diffie–Hellman between the
target client and itself
If AES decryption succeeds it will return a valid [`uuid`](https://tools.ietf.org/html/rfc4122) which is what is used for access_token.
The token SHOULD be used to send push notifications.

The response MUST be sent on the [partitioned topic][./10-waku-usage.md#partitioned-topic] of the sender
and MUST not be encrypted using the [secure transport](../docs/stable/5-secure-transport.md) to facilitate
the usage of ephemeral keys.

On receiving a response a client MUST verify `grant` to ensure that the server
has been authorized to send push notification to a given client.

## Sending a push notification

When sending a push notification, only the `installation_id` for the devices targeted
by the message SHOULD be used.

If a message is for all the user devices, all the `installation_id` known to the client MAY be used.

The number of devices MAY be capped in order to reduce resource consumption.

At least 3 devices SHOULD be targeted, ordered by last activity.

For any device that a token is available, or that a token is successfully queried,
a push notification message SHOULD be sent to the corresponding push notification server.

```protobuf
message PushNotification {
  string access_token = 1;
  string chat_id = 2;
  bytes public_key = 3;
  string installation_id = 4;
  bytes message = 5;
  PushNotificationType type = 6;
  enum PushNotificationType {
    UNKNOWN_PUSH_NOTIFICATION_TYPE = 0;
    MESSAGE = 1;
    MENTION = 2;
  }
  bytes author = 7;
}

message PushNotificationRequest {
  repeated PushNotification requests = 1;
  bytes message_id = 2;
}
```

A `PushNotificationRequest` message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_REQUEST`. 

Where `message` is the encrypted payload of the message and `chat_id` is the
`SHAKE-256` of the `chat_id`.
`message_id` is the id of the message
`author` is the `SHAKE-256` of the public key of the sender.

If multiple server are available for a given push notification, only one notification
MUST be sent.

If no response is received
a client SHOULD wait at least 3 seconds, after which the request MAY be retried against a different server


This message SHOULD be sent using an ephemeral key.

On receiving the message, the push notification server MUST validate the access token.
If the access token is valid, a notification MUST be sent to the gorush instance with the
following data:

```
{
  "notifications": [
    {
      "tokens": ["token_a", "token_b"],
      "platform": 1,
      "message": "You have a new message",
      "data": {
        "chat_id": chat_id,
        "message": message,
        "installation_ids": [installation_id_1, installation_id_2]
      }
    }
  ]
}
```

Where platform is `1` for IOS and `2` for Firebase, according to the [gorush
documentation](https://github.com/appleboy/gorush)

A server MUST return a response message:

```protobuf
message PushNotificationReport {
  bool success = 1;
  ErrorType error = 2;
  enum ErrorType {
    UNKNOWN_ERROR_TYPE = 0;
    WRONG_TOKEN = 1;
    INTERNAL_ERROR = 2;
    NOT_REGISTERED = 3;
  }
  bytes public_key = 3;
  string installation_id = 4;
}

message PushNotificationResponse {
  bytes message_id = 1;
  repeated PushNotificationReport reports = 2;
}
```


A `PushNotificationResponse` message MUST be wrapped in a [`ApplicationMetadataMessage`](../stable/6-payloads.6#payload-wrapper) with type set to `PUSH_NOTIFICATION_RESPONSE`. 

Where `message_id` is the `message_id` sent by the client.

The response MUST be sent on the [partitioned topic][./10-waku-usage.md#partitioned-topic] of the sender
and MUST not be encrypted using the [secure transport](../docs/stable/5-secure-transport.md) to facilitate
the usage of ephemeral keys.

If the request is accepted `success` MUST be set to `true`.
Otherwise `success` MUST be set to `false`.

If `error` is `BAD_TOKEN` the client MAY query again the server for the token and
retry the request.

If `error` is `INTERNAL_ERROR` the client MAY retry the request.

## Flow

### Registration process

- A client will generate a notification token through `APN` or `Firebase`.
- The client will [register](#registering-with-the-push-notification-service) with one or more push notification server of their choosing.
- The server should process the response and respond according to the success of the operation
- If the request is not successful it might be retried, and adjusted according to the response. A different server can be also used.
- Once the request is successful the client should [advertise](#advertising-a-push-notification-server) the new coordinates

### Sending a notification

- A client should prepare a message and extract the targeted installation-ids
- It should retrieve the most up to date information for a given user, either by
  querying a push notification server, a mailserver if not listening already to the given topic, or checking
  the database locally
- It should then [send](#sending-a-push-notification) a push notification according
  to the rules described
- The server should then send a request to the gorush server including all the required
  information

### Receiving a push notification

- On receiving the notification, a client can open the right account by checking the
  `installation_id` included. The `chat_id` MAY be used to open the chat if present.
- `message` can be decrypted and presented to the user. Otherwise messages can be pulled from the mailserver if the `message_id` is no already present.

## Protobuf description

### PushNotificationRegistration

`token_type`: the type of token. Currently supported is `APN_TOKEN` for Apple Push
`device_token`: the actual push notification token sent by `Firebase` or `APN`
and `FIREBASE_TOKEN` for firebase.
`installation_id`: the [`installation_id`](./2-account.md) of the device
`access_token`: the access token that will be given to clients to send push notifications
`enabled`: whether the device wants to be sent push notifications
`version`: a monotonically increasing number identifying the current `PushNotificationRegistration`. Any time anything is changed in the record it MUST be increased by the client, otherwise the request will not be accepted.
`allowed_key_list`: a list of `access_token` encrypted with the AES key generated
 by Diffie–Hellman between the publisher and the allowed
contact.
`blocked_chat_list`: a list of `SHA2-256` hashes of chat ids.
Any chat id in this list will not trigger a notification.
`unregister`: whether the account should be unregistered
`grant`: the grant for this specific server
`allow_from_contacts_only`: whether the client only wants push notifications from contacts
`apn_topic`: the APN topic for the push notification
`block_mentions`: whether the client does not want to be notified on mentions
`allowed_mentions_chat_list`:  a list of `SHA2-256` hashes of chat ids where we want to receive mentions

#### Data disclosed

- Type of device owned by a given user
- The `FIREBASE` or `APN` push notification token
- Hash of the chat_id a user is not interested in for notifications
- The times a push notification record has been modified by the user
- The number of contacts a client has, in case `allowed_key_list` is set

### PushNotificationRegistrationResponse

`success`: whether the registration was successful
`error`: the error type, if any
`request_id`: the `SHAKE-256` hash of the `signature` of the request
`preferences`: the server stored preferences in case of an error

### ContactCodeAdvertisement

`push_notification_info`: the information for each device advertised

#### Data disclosed

- The chat key of the sender


### PushNotificationQuery

`public_keys`: the `SHAKE-256` of the public keys the client is interested in

#### Data disclosed

- The hash of the public keys the client is interested in

### PushNotificationQueryInfo

`access_token`: the access token used to send a push notification
`installation_id`: the `installation_id` of the device associated with the `access_token`
`public_key`: the `SHAKE-256` of the public key associated with this `access_token` and `installation_id`
`allowed_key_list`: a list of encrypted access tokens to be returned
to the client in case there's any filtering on public keys in place.
`grant`: the grant used to register with this server.
`version`: the version of the registration on the server.
`server_public_key`: the compressed public key of the server.

### PushNotificationQueryResponse

`info`:  a list of `PushNotificationQueryInfo`.
`message_id`: the message id of the `PushNotificationQueryInfo` the server is replying to.
`success`: whether the query was successful.

### PushNotification

`access_token`: the access token used to send a push notification.
`chat_id`: the `SHAKE-256` of the `chat_id`.
`public_key`: the `SHAKE-256` of the compressed public key of the receiving client.
`installation_id`: the installation id of the receiving client.
`message`: the encrypted message that is being notified on.
`type`: the type of the push notification, either `MESSAGE` or `MENTION`
`author`: the `SHAKE-256` of the public key of the sender

### Data disclosed

- The `SHAKE-256` of the `chat_id` the notification is to be sent for
- The cypher text of the message
- The `SHAKE-256` of the public key of the sender
- The type of notification

### PushNotificationRequest

`requests`: a list of `PushNotification`
`message_id`: the [status message id](./6-payloads.md)

### Data disclosed

- The status message id for which the notification is for

### PushNotificationResponse

`message_id`: the `message_id` being notified on.
`reports`: a list of `PushNotificationReport`

### PushNotificationReport

`success`: whether the push notification was successful.
`error`: the type of the error in case of failure.
`public_key`: the public key of the user being notified.
`installation_id`: the installation id of the user being notified.

## Anonymous mode of operations

An anonymous mode of operations MAY be provided by the client, where the
responsibility of propagating information about the user is left to the client,
in order to preserve privacy.

A client in anonymous mode can register with the server using a key different
from their chat key.
This will hide their real chat key.

This public key is effectively a secret and SHOULD only be disclosed to clients that you the user wants to be notified by.

A client MAY advertise the access token on the contact-code topic of the key generated.
A client MAY share their public key through [contact updates](./6-payloads.md#contact-update)

A client receiving a push notification public key SHOULD listen to the contact code
topic of the push notification public key for updates.

The method described above effectively does not share the identity of the sender
nor the receiver to the server, but MAY result in missing push notifications as
the propagation of the secret is left to the client.

This can be mitigated by [device syncing](./6-payloads.md), but not completely
addressed.

## Security considerations

If no anonymous mode is used, when registering with a push notification service a client discloses:

- The chat key
- The devices that will receive notifications

A client MAY disclose:

- The hash of the chat_ids they want to filter out

When running in anonymous mode, the client's chat key is not disclosed.

When querying a push notification server a client will disclose:

- That it is interested in sending push notification to another client,
  but the querying client's chat key is not disclosed

When sending a push notification a client discloses:
- The `SHAKE-256` of the chat id

[//]: This section can be removed, for now leaving it here in order to help with the
review process. Point can be integrated, suggestion welcome.

## FAQ

### Why having ACL done at the server side and not the client?

We looked into silent notification for
[IOS](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/pushing_background_updates_to_your_app) (android has no equivalent)
but can't be used as it's expected to receive maximum 2/3 per hour, so not our use case. There
are also issue when the user force quit the app.

### Why using an access token?

The access token is used to decouple the requesting information from the user from
actually sending the push notification.

Some ACL is necessary otherwise it would be too easy to spam users (it's still fairly
trivial, but with this method you could allow only contacts to send you push notifications).

Therefore your identity must be revealed to the server either when sending or querying.

By using an access token we increase deniability, as the server would know
who requested the token but not necessarily who sent a push notification.
Correlation between the two can be trivial in some cases.

This also allows a mode of use as we had before, where the server does not propagate
info at all, and it's left to the user to propagate the token, through contact requests
for example.

### Why advertise with the bundle?

Advertising with the bundle allows us to piggy-back on an already implemented behavior
and save some bandwidth in cases where is not filtering by public keys

### What's the  bandwidth impact for this?

Generally speaking, for each 1-to-1 message and group chat message you will sending
1 and `number of participants` push notifications. This can be optimized if
multiple users are using the same push notification server. Queries have also
a bandwidth impact but they are made only when actually needed

### What's the information disclosed?

The data disclosed with each message sent by the client is above, but for a summary:

When you register with a push notification service you may disclose:

1) Your chat key
2) Which devices you have
3) The hash of the chat_ids you want to filter out
4) The hash of the public keys you are interested/not interested in

When you query a notification service you may disclose:

1) Your chat key
2) The fact that you are interested in sending push notification to a given user

Effectively this is fairly revealing if the user has a whitelist implemented.
Therefore sending notification should be optional.

### What prevents a user from generating a random key and getting an access token and spamming?

Nothing really, that's the same as the status app as a whole. the only mechanism that prevents
this is using a white-list as described above, but that implies disclosing your true identity to the push notification server.

### Why not 0-knowledge proofs/quantum computing

We start simple, we can iterate

### How to handle backward/forward compatibility

Most of the request have a target, so protocol negotiation can happen. We cannot negotiated
the advertisement as that's effectively a broadcast, but those info should not change and we can
always accrete the message.

### Why ack_key?

That's necessary to avoid duplicated push notifications and allow for the retry
in case the notification is not successful.

Deduplication of the push notification is done on the client side, to reduce a bit
of centralization and also in order not to have to modify gorush.

### Can I run my own node?

Sure, the methods allow that

### Can I register with multiple nodes for redundancy

Yep

### What does my node disclose?

Your node will disclose the IP address is running from, as it makes an HTTP post to
gorush. A waku adapter could be used, but please not now.

### Does this have high-reliability requirements?

The gorush server yes, no way around it.

The rest, kind of, at least one node having your token needs to be up for you to receive notifications.
But you can register with multiple servers (desktop, status, etc) if that's a concern.

### Can someone else (i.e not status) run this?

Push notification servers can be run by anyone. Gorush can be run by anyone I take,
but we are in charge of the certificate, so they would not be able to notify status-clients.

## Changelog

### Version 0.1

Released [](https://github.com/status-im/specs/commit/)

- Initial version

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
