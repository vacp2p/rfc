---
slug: 36
title: 36/WAKU2-BINDINGS-API
name: Waku v2 C Bindings API
status: draft
tags: waku-core
editor: Richard Ramos <richard@status.im>
contributors:
- Franck Royer <franck@status.im>
---

# Introduction

Native applications that wish to integrate Waku may not be able to use nim-waku and its JSON RPC API due to constraints
on packaging, performance or executables.

An alternative is to link existing Waku implementation as a static or dynamic library in their application.

This specification describes the C API that SHOULD be implemented by native Waku library and that SHOULD be used to
consume them.

# Design requirements

The API should be generic enough, so:

- it can be implemented by both nim-waku and go-waku C-Bindings,
- it can be consumed from a variety of languages such as C#, Kotlin, Swift, Rust, C++, etc.

The selected format to pass data to and from the API is `JSON`.

It has been selected due to its widespread usage and easiness of use. Other alternatives MAY replace it in the future (C
structure, protobuf) if it brings limitations that need to be lifted.

# The API

## General

### `JsonResponse` type

All the API functions return a `JsonResponse` unless specified otherwise.
`JsonResponse` is a `char *` whose format depends on whether the function was executed successfully or not.

On failure:

```ts
{
    error: string;
}
```

For example: 

```json
{
  "error": "the error message"
}
```

On success:

```ts
{
    result: any
}
```

The type of the `result` object depends on the function it was returned by. 

### `JsonMessage` type

A Waku Message in JSON Format:

```ts
{
    payload: string;
    contentTopic: string;
    version: number;
    timestamp: number;
}
```

Fields:

- `payload`: base64 encoded payload, [`waku_utils_base64_encode`](#extern-char-waku_utils_base64_encodechar-data) can be used for this.
- `contentTopic`: The content topic to be set on the message.
- `version`: The Waku Message version number.
- `timestamp`: Unix timestamp in nanoseconds.

## Events

Asynchronous events require a callback to be registered.
An example of an asynchronous event that might be emitted is receiving a message.
When an event is emitted, this callback will be triggered receiving a JSON string of type `JsonSignal`.

### `JsonSignal` type

```ts
{
    nodeId: number;
    type: string;
    event: any;
}
```

Fields:

- `nodeId`: Integer, go-waku node that emitted the signal.
- `type`: Type of signal being emitted. Currently, only `message` is available.
- `event`: Format depends on the type of signal.

For example:

```json
{
  "nodeId": 0,
  "type": "message",
  "event": {
    "subscriptionId": 1,
    "pubsubTopic": "/waku/2/default-waku/proto",
    "messageId": "0x6496491e40dbe0b6c3a2198c2426b16301688a2daebc4f57ad7706115eac3ad1",
    "wakuMessage": {
      "payload": "TODO",
      "contentTopic": "/my-app/1/notification/proto",
      "version": 1,
      "timestamp": 1647826358000000000
    }
  }
}
```

| `type`    | `event` Type       |
|:----------|--------------------|
| `message` | `JsonMessageEvent` | 

### `JsonMessageEvent` type

Type of `event` field for a `message` event:

```ts
{
    subscriptionId: number;
    pubsubTopic: string;
    messageId: string;
    wakuMessage: JsonMessage;
}
```

- `subscriptionId`: The id of the subscription from which the message was received, returned by [`waku_relay_subscribe`](#extern-char-waku_relay_subscribeint-nodeid-char-pubsubtopic).
- `pubsubTopic`: The pubsub topic on which the message was received.
- `messageId`: The message id.
- `wakuMessage`: The message in [`JsonMessage`](#jsonmessage-type) format.

### `extern void waku_set_event_callback(void* cb)`

Register callback to act as event handler and receive application signals,
which are used to react to asynchronous events in Waku.

**Parameters**

1. `void* cb`: callback that will be executed when an async event is emitted.
  The function signature for the callback should be `void myCallback(char* jsonSignal)`

## Node management

### `JsonConfig` type

Type holding a node configuration:

```ts
interface JsonSignal {
    host?: string;
    port?: number;
    advertiseAddr?: string;
    nodeKey?: string;
    keepAliveInterval?: number;
    relay?: boolean;
    minPeersToPublish?: number
}
```

Fields: 

All fields are optional.
If a key is `undefined`, or `null`, a default value will be set.

- `host`: Listening IP address.
  Default `0.0.0.0`.
- `port`: Libp2p TCP listening port.
  Default `60000`. 
  Use `0` for random.
- `advertiseAddr`: External address to advertise to other nodes.
  Can be ip4, ip6 or dns4, dns6.
  Default: ?
- `nodeKey`: Secp256k1 private key in Hex format (`0x123...abc`).
  Default random.
- `keepAliveInterval`: Interval in seconds for pinging peers to keep the connection alive.
  Default `20`.
- `relay`: Enable relay protocol.
  Default `true`.
- `minPeersToPublish`: The minimum number of peers required on a topic to allow broadcasting a message.
  Default `0`.

For example:
```json
{
  "host": "0.0.0.0",
  "port": 60000,
  "advertiseAddr": "1.2.3.4",
  "nodeKey": "0x123...567",
  "keepAliveInterval": 20,
  "relay": true,
  "minPeersToPublish": 0
}
```

### `extern char* waku_new(char* jsonConfig)`

Instantiates a Waku node.

**Parameters**

1. `char* jsonConfig`: JSON string containing the options used to initialize a go-waku node.
   Type [`JsonConfig`](#jsonconfig-type).
   It can be `NULL` to use defaults.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains an `integer` which represents the `nodeId`.
This node id MUST be used in following API calls to interact with the instantiated node.

For example:

```json
{
  "result": 0
}
```

### `extern char* waku_start(int nodeId)`

Start a Waku node mounting all the protocols that were enabled during the Waku node instantiation.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
  "result": null
}
```

### `extern char* waku_stop(int nodeID)`

Stops a Waku node.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
  "result": null
}
```

### `extern char* waku_peerid(int nodeId)`

Get the peer ID of the waku node.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains the peer ID as a `string` (base58 encoded).

For example:

```json
{
  "result": "QmWjHKUrXDHPCwoWXpUZ77E8o6UbAoTTZwf1AD1tDC4KNP"
}
```

### `extern char* waku_listen_addresses(int nodeId)`

Get the multiaddresses the Waku node is listening to.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains an array of multiaddresses.
The multiaddresses are `string`s.

For example:

```json
{
  "result": [
    "/ip4/127.0.0.1/tcp/30303",
    "/ip4/1.2.3.4/tcp/30303",
    "/dns4/waku.node.example/tcp/8000/wss"
  ]
}
```

## Connecting to peers

### `extern char* waku_add_peer(int nodeId, char* address, char* protocolId)`

Add a node multiaddress and protocol to the waku node's peerstore.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* address`: A multiaddress (with peer id) to reach the peer being added.
3. `char* protocolId`: A protocol we expect the peer to support.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains the peer ID as a base58 `string` of the peer that was added.

For example:

```json
{
  "result": "QmWjHKUrXDHPCwoWXpUZ77E8o6UbAoTTZwf1AD1tDC4KNP"
}
```

### `extern char* waku_connect_peer(int nodeId, char* address, int timeoutMs)`

Dial peer using a multiaddress.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* address`: A multiaddress to reach the peer being dialed.
3. `int timeoutMs`: Timeout value in milliseconds to execute the call.
   If the function execution takes longer than this value,
   the execution will be canceled and an error returned.
   Use `0` for no timeout.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
   "result": null
}
```

### `extern char* waku_connect_peerid(int nodeId, char* peerId, int timeoutMs)`

Dial peer using its peer ID.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* peerID`: Peer ID to dial.
   The peer must be already known.
   It must have been added before with [`waku_add_peer`](#extern-char-waku_add_peerint-nodeid-char-address-char-protocolid)
   or previously dialed with [`waku_connect_peer`](#extern-char-waku_connect_peerint-nodeid-char-address-int-timeoutms).
3. `int timeoutMs`: Timeout value in milliseconds to execute the call.
   If the function execution takes longer than this value,
   the execution will be canceled and an error returned.
   Use `0` for no timeout.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
   "result": null
}
```

### `extern char* waku_disconnect_peer(int nodeId, char* peerId)`

Disconnect a peer using its peerID
**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* peerID`: Peer ID to disconnect.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
   "result": null
}
```

### `extern char* waku_peer_count(int nodeID)`

Get number of connected peers.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains an `integer` which represents the number of connected peers.

For example:

```json
{
  "result": 0
}
```

### `extern char* waku_peers(int nodeId)`

Retrieve the list of peers known by the Waku node.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).

**Returns**

A [`JsonResponse`](#jsonresponse-type) containing a list of peers.
The list of peers has this format:

```json
{
  "result": [
    {
      "peerID": "16Uiu2HAmJb2e28qLXxT5kZxVUUoJt72EMzNGXB47RedcBafeDCBA",
      "protocols": [
        "/ipfs/id/1.0.0",
        "/vac/waku/relay/2.0.0",
        "/ipfs/ping/1.0.0"
      ],
      "addrs": [
        "/ip4/1.2.3.4/tcp/30303"
      ],
      "connected": true
    }
  ]
}
```

## Waku Relay

### `extern char* waku_content_topic(char* applicationName, unsigned int applicationVersion, char* contentTopicName, char* encoding)`

Create a content topic string according to [RFC 23](https://rfc.vac.dev/spec/23/).

**Parameters**

1. `char* applicationName`
2. `unsigned int applicationVersion`
3. `char* contentTopicName`
4. `char* encoding`: depending on the payload, use `proto`, `rlp` or `rfc26`

**Returns**

`char *` containing a content topic formatted according to [RFC 23](https://rfc.vac.dev/spec/23/).

```
/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}
```

### `extern char* waku_pubsub_topic(char* name, char* encoding)`

Create a pubsub topic string according to [RFC 23](https://rfc.vac.dev/spec/23/).

**Parameters**

1. `char* name`
2. `char* encoding`: depending on the payload, use `proto`, `rlp` or `rfc26`

**Returns**

`char *` containing a content topic formatted according to [RFC 23](https://rfc.vac.dev/spec/23/).

```
/waku/2/{topic-name}/{encoding}
```

### `extern char* waku_default_pubsub_topic()`

Returns the default pubsub topic used for exchanging waku messages defined in [RFC 10](https://rfc.vac.dev/spec/10/).

**Returns**

`char *` containing the default pubsub topic:

```
/waku/2/default-waku/proto
```

### `extern char* waku_relay_publish(int nodeId, char* messageJson, char* pubsubTopic, int timeoutMs)`

Publish a message using waku relay.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* messageJson`: JSON string containing the [Waku Message](https://rfc.vac.dev/spec/14/) as [`JsonMessage`](#jsonmessage-type).
3. `char* pubsubTopic`: pubsub topic on which to publish the message.
   If `NULL`, it uses the default pubsub topic.
4. `int timeoutMs`: Timeout value in milliseconds to execute the call.
   If the function execution takes longer than this value,
   the execution will be canceled and an error returned.
   Use `0` for no timeout.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains the message ID.

### `extern char* waku_relay_enough_peers(int nodeId, char* pubsubTopic)`

Determine if there are enough peers to publish a message on a given pubsub topic.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* pubsubTopic`: Pubsub topic to verify.
   If `NULL`, it verifies the number of peers in the default pubsub topic.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains a `boolean` indicating whether there are enough peers.

For example:

```json
{
  "result": true
}
```

### `extern char* waku_relay_subscribe(int nodeId, char* pubsubTopic)`

Subscribe to a Waku Relay pubsub topic to receive messages.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* pubsubTopic`: Pubsub topic to subscribe to. 
   If `NULL`, it subscribes to the default pubsub topic.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains an `integer` that is the subscription id.

For example:

```json
{
  "result": 0
}
```

**Events**

When a message is received, a ``"message"` event` is emitted containing the message, pubsub topic, and node ID in which
the message was received.

The `event` type is [`JsonMessageEvent`](#jsonmessageevent-type).

For Example:

```json
{
  "nodeId": 1,
  "type": "message",
  "event": {
    "subscriptionID": 1,
    "pubsubTopic": "/waku/2/default-waku/proto",
    "messageID": "0x6496491e40dbe0b6c3a2198c2426b16301688a2daebc4f57ad7706115eac3ad1",
    "wakuMessage": {
      "payload": "TODO",
      "contentTopic": "/my-app/1/notification/proto",
      "version": 1,
      "timestamp": 1647826358000000000
    }
  }
}
```

### `extern char* waku_relay_close_subscription(int nodeId, char* subscriptionId)`

Closes a Waku Relay subscription.
No more messages will be received from this subscription.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* subscriptionId`: Subscription ID to close.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
   "result": null
}
```

### `extern char* waku_relay_unsubscribe_from_topic(int nodeId, char* topic)`

Closes the pubsub subscription to a pubsub topic.
Existing subscriptions will not be closed, but they will stop receiving messages.

**Parameters**

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-jsonconfig).
2. `char* pusubTopic`: Pubsub topic to unsubscribe from.
  If `NULL`, unsubscribes from the default pubsub topic.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field is set to `null`.

For example:

```json
{
   "result": null
}
```

## Utils

### `extern char* waku_encode_payload(char* data, char* keyType, char* key, char* signingKey, int version)`

Encode a byte array according to [RFC 26](https://rfc.vac.dev/spec/26/).
This function can be used to encrypt the payload of a Waku Message.

**Parameters**

1. `char* data`: Byte array to encode in base64 format.
   ([`waku_utils_base64_encode`](#extern-char-waku_utils_base64_encodechar-data) can be used to encode the data).
2. `char* keyType`: defines the type of key to use:
    - `NONE`: No encryption will be applied,
    - `ASYMMETRIC`: Encrypt the payload using a secp256k1 public key,
    - `SYMMETRIC`: Encrypt the payload using a 32-bit secret key.
3. `char* key`: Key to be used for encrypting the `data`, in hex format (`0x123...abc`).
    - When `version` is 0: No encryption is used,
    - When `version` is 1
        - If using `ASYMMETRIC` encoding, `key` must contain a secp256k1 public key to encrypt the data with,
        - If using `SYMMETRIC` encoding, `key` must contain a 32 bytes symmetric key.
4. `char* signingKey`: Hex string containing a secp256k1 private key to sign the encoded message.
   If `NULL`, the message is not signed.
   Only available if the message is encrypted.
5. `int version`: Is used to define the type of payload encryption.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains payload as a base 64 encoded `string`.

For example:

```json
{
   "result": "TODO"
}
```


### `extern char* waku_decode_payload(char* payload, char* keyType, char* key, int version)`

Decode a byte array according to [RFC 26](https://rfc.vac.dev/spec/26/).
This function can be used to decrypt the payload of a Waku Message.

**Parameters**

1. `char* data`: Byte array to decode, in base64.
2. `char* keyType`: defines the type of key to use:
    - `NONE`: no encryption was used in the payload,
    - `ASYMMETRIC`: decrypt the payload using a secp256k1 public key,
    - `SYMMETRIC`: decrypt the payload using a 32 bit key.
3. `char* key`: Key to be used for decrypting the `data`, in hex format (`0x123...abc`).
    - When `version` is 0: No encryption is used,
    - When `version` is 1
        - If using `ASYMMETRIC` encoding, `key` must contain a secp256k1 private key to decrypt the data,
        - If using `SYMMETRIC` encoding, `key` must contain a 32 bytes symmetric key.
4. `int version`: is used to define the type of payload encryption

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains the decoded payload in the following format:

```ts
{
    pubkey: string; // secp256k1 public key
    signature: string; // secp256k1 signature
    data: string; // base64 encoded
    padding: string;
}
```

For example:

```json
{
  "result": {
    "pubkey": "0x04123...abc",
    "signature": "0x123...abc",
    "data": "...",
    "padding": "..."
  }
}
```

### `extern char* waku_utils_base64_encode(char* data)`

Encode a byte array to base64.
Useful for creating the payload of a Waku Message in the format understood by [`waku_relay_publish`](#extern-char-waku_relay_publishint-nodeid-char-messagejson-char-pubsubtopic-int-timeoutms)

**Parameters**

1. `char* data`: Byte array to encode

**Returns**

A `char *` containing the base64 encoded byte array.

### `extern char* waku_utils_base64_decode(char* data)`

Decode a base64 string (useful for reading the payload from Waku Messages).

**Parameters**

1. `char* data`: base64 encoded byte array to decode.

**Returns**

A [`JsonResponse`](#jsonresponse-type).
If the execution is successful, the `result` field contains the decoded payload.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
