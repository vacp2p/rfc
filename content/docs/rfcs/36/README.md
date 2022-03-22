---
slug: 35 title: 35/WAKU2-BINDINGS-API
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

### JsonResponse

All the API functions return a `JsonResponse` unless specified otherwise.
`JsonResponse` is a `char *` whose format depends on whether the function was executed successfully or not.

On failure:

```json
{
  "error": "the error message"
}
```

On success:

```json
{
  "result": {}
}
```

The format of the `result` object depends on the function it was returned by. 

## Events

Asynchronous events require a callback to be registered.
An example of an asynchronous event that might be emitted is receiving a message.
When an event is emitted, this callback will be triggered receiving a json string with the following format:

```json
{
  "nodeId": 0, // go-waku node that emitted the signal
  "type": "message", // type of signal being emitted. Currently only "message" is available
  "event": ... // format depends on the type of signal. In the case of "message", a waku message can be expected.
}
```

| `type` | `event` Type  |
|:------|---------------|
| `message` | TODO      | 


### `extern void waku_set_event_callback(void* cb)`

Register callback to act as event handler and receive application signals,
which are used to react to asynchronous events in Waku.

**Parameters**

1. `void* cb`: callback that will be executed when an async event is emitted.
  The function signature for the callback should be `void myCallback(char* signalJSON)`

## Node management

### `extern char* waku_new(char* configJson)`

Instantiates a Waku node.

**Parameters**

1. `char* configJson`: JSON string containing the options used to initialize a go-waku node.
   It can be `null` to use defaults.
   All keys are optional.
   If a key is `undefined`, or `null`, a default value will be set. Example:
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
    - `host` - `string` (optional): Listening IP address.
      Default `0.0.0.0`.
    - `port` - `number` (optional): Libp2p TCP listening port.
      Default `60000`.
      Use `0` for random.
    - `advertiseAddr` - `string` (optional): External address to advertise to other nodes.
    - `nodeKey` - `string` (optional): secp256k1 private key in Hex format (`0x123...abc`).
      Default random.
    - `keepAliveInterval` - `number` (optional): Interval in seconds for pinging peers to keep the connection alive.
      Default `20`.
    - `relay` - `boolean` (optional): Enable relay protocol.
      Default `true`.
    - `minPeersToPublish` - `number` (optional).
      The minimum number of peers required on a topic to allow broadcasting a message.
      Default `0`.

**Returns**

A [`JsonResponse`](#jsonresponse).
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

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-configjson).

**Returns**

A [`JsonResponse`](#jsonresponse).
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

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-configjson).

**Returns**

A [`JsonResponse`](#jsonresponse).
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

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-configjson).

**Returns**

A [`JsonResponse`](#jsonresponse).
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

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-configjson).

**Returns**

A [`JsonResponse`](#jsonresponse).
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

1. `int nodeId`: The node identifier obtained from a successful execution of [`waku_new`](#extern-char-waku_newchar-configjson).
2. `char* address`: A multiaddress to reach the peer being added.
3. `char* protocolId`: A protocol we expect the peer to support.

**Returns**

A [`JsonResponse`](#jsonresponse).
If the execution is successful, the `result` field contains the peer ID as a base58 `string` of the peer that was added.

For example:

```json
{
  "result": "QmWjHKUrXDHPCwoWXpUZ77E8o6UbAoTTZwf1AD1tDC4KNP"
}
```

### `extern char* waku_dial_peer(int nodeId, char* address, int ms)`

Dial peer at multiaddress.

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* address`: multiaddress of the peer being dialed
3. `int ms`: max duration in milliseconds this function might take to execute. If the function execution takes longer
   than this value, the execution will be canceled and an error returned. Use `0` for unlimited duration

**Returns**
`JSONResponse` with a null `result` if the function executes successfully. An `error` message otherwise

---

### `extern char* gowaku_dial_peerid(int nodeID, char* id, int ms)`

Dial peer using peerID.

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* peerID`: peerID to dial. The peer must be already known. It must have been added before with `gowaku_add_peer`
   or previously dialed with `gowaku_dial_peer`
3. `int ms`: max duration in milliseconds this function might take to execute. If the function execution takes longer
   than this value, the execution will be canceled and an error returned. Use `0` for unlimited duration

**Returns**
`JSONResponse` with a null `result` if the function executes successfully. An `error` message otherwise

---

### `extern char* gowaku_close_peer(int nodeID, char* address)`

Disconnect a peer using its multiaddress

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* address`: multiaddress of the peer being disconnected.

**Returns**
`JSONResponse` with a null `result` if the function executes successfully. An `error` message otherwise

---

### `extern char* gowaku_close_peerid(int nodeID, char* id)`

Disconnect a peer using its peerID
**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* peerID`: peerID to disconnect.

**Returns**
`JSONResponse` with a null `result` if the function executes successfully. An `error` message otherwise

---

### `extern char* gowaku_peer_cnt(int nodeID)`

Obtain number of connected peers

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`

**Returns**
`JSONResponse` containing an `int` with the number of connected peers. An `error` message otherwise

---

### `extern char* gowaku_peers(int nodeID)`

Retrieve the list of peers known by the go-waku node

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`

**Returns**
`JSONResponse` containing a list of peers. An `error` message otherwise. The list of peers has this format:

```json
{
  "result": [
    ...
    {
      "peerID": "16Uiu2HAmJb2e28qLXxT5kZxVUUoJt72EMzNGXB47RedcBafeDCBA",
      "protocols": [
        "/ipfs/id/1.0.0",
        "/vac/waku/relay/2.0.0",
        "/ipfs/ping/1.0.0",
        ...
      ],
      "addrs": [
        "/ip4/1.2.3.4/tcp/30303",
        ...
      ],
      "connected": true
    }
  ]
}
```

## Waku Relay

### `extern char* gowaku_content_topic(char* applicationName, unsigned int applicationVersion, char* contentTopicName, char* encoding)`

Create a content topic string according to [RFC 23](https://rfc.vac.dev/spec/23/)

**Parameters**

1. `char* applicationName`
2. `unsigned int applicationVersion`
3. `char* contentTopicName`
4. `char* encoding`: depending on the payload, use `proto`, `rlp` or `rfc26`

**Returns**
`char *` containing a content topic formatted according to [RFC 23](https://rfc.vac.dev/spec/23/)

```
/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}
```

--

### `extern char* gowaku_pubsub_topic(char* name, char* encoding)`

Create a pubsub topic string according to [RFC 23](https://rfc.vac.dev/spec/23/)

**Parameters**

1. `char* name`
2. `char* encoding`: depending on the payload, use `proto`, `rlp` or `rfc26`

**Returns**
`char *` containing a content topic formatted according to [RFC 23](https://rfc.vac.dev/spec/23/)

```
/waku/2/{topic-name}/{encoding}
```

---

### `extern char* gowaku_default_pubsub_topic()`

Returns the default pubsub topic used for exchanging waku messages defined in [RFC 10](https://rfc.vac.dev/spec/10/)

**Returns**
`char *` containing the default pubsub topic:

```
/waku/2/default-waku/proto
```

---

### `extern char* gowaku_relay_publish(int nodeID, char* messageJSON, char* topic, int ms)`

Publish a message using waku relay.

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* messageJSON`: json string containing the [Waku Message](https://rfc.vac.dev/spec/14/)
    ```json
    {
        "payload":"", // base64 encoded payload. gowaku_utils_base64_encode can be used for this
        "contentTopic: "...",
        "version": 1,
        "timestamp": 1647963508000000000 // Unix timestamp in nanoseconds
    }
    ```
3. `char* topic`: pubsub topic. Set to `NULL` to use the default pubsub topic
4. `int ms`: max duration in milliseconds this function might take to execute. If the function execution takes longer
   than this value, the execution will be canceled and an error returned. Use `0` for unlimited duration

**Returns**
`JSONResponse` containing the message ID. An `error` message otherwise

---

### `extern char* gowaku_enough_peers(int nodeID, char* topic)`

Determine if there are enough peers to publish a message on a topic.

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* topic`: pubsub topic to verify. Use `NULL` to verify the number of peers in the default pubsub topic

**Returns**
`JSONResponse` with a boolean indicating if there are enough peers or not. An `error` message otherwise

---

### `extern char* gowaku_relay_subscribe(int nodeID, char* topic)`

Subscribe to a WakuRelay topic to receive messages.

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* topic`: pubsub topic to subscribe to. Use `NULL` for subscribing to the default pubsub topic

**Returns**
`JSONResponse` with a subscription ID. An `error` message otherwise

**Events**
When a message is received, a ``"message"` event` is emitted containing the message, pubsub topic, and nodeID in which
the message was received. Here's an example event that could be received:

```json
{
  "nodeId": 1,
  "type": "message",
  "event": {
    "subscriptionID": "...",
    "pubsubTopic": "/waku/2/default-waku/proto",
    "messageID": "0x6496491e40dbe0b6c3a2198c2426b16301688a2daebc4f57ad7706115eac3ad1",
    "wakuMessage": {
      "payload": "...",
      // base64 encoded message. Use gowaku_decode_data to decode
      "contentTopic": "ABC",
      "version": 1,
      "timestamp": 1647826358000000000
      // in nanoseconds
    }
  }
}
```

### `extern char* gowaku_relay_close_subscription(int nodeID, char* subsID)`

Closes a waku relay subscription. No more messages will be received from this subscription

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* subsID`: subscription ID to close

**Returns**
`JSONResponse` with null `response` if successful. An `error` message otherwise

---

### `extern char* gowaku_relay_unsubscribe_from_topic(int nodeID, char* topic)`

Closes the pubsub subscription to a pubsub topic. Existing subscriptions will not be closed, but they will stop
receiving messages

**Parameters**

1. `int nodeID`: the node identifier obtained from a succesful execution of `gowaku_new`
2. `char* topic`: pubsub topic to unsubscribe. Use `NULL` for unsubscribe from the default pubsub topic

**Returns**
`JSONResponse` with null `response` if successful. An `error` message otherwise

## Waku Message Utils

### `extern char* gowaku_encode_data(char* data, char* keyType, char* key, char* signingKey, int version)`

Encode a byte array according to [RFC 26](https://rfc.vac.dev/spec/26/). This function can be used to encrypt the
payload of a waku message

**Parameters**

1. `char* data`: byte array to encode in base64 format. (gowaku_utils_base64_encode can be used to encode the message)
2. `char* keyType`: defines the type of key to use:
    - `NONE`: no encryption will be applied
    - `ASYMMETRIC`: encrypt the payload using a secp256k1 public key
    - `SYMMETRIC`: encrypt the payload using a 32 bit key.
3. `char* key`: hex key (`0x123...abc`) used for encrypting the `data`.
    - When `version` is 0: No encryption is used
    - When `version` is 1
        - If using `ASYMMETRIC` encoding, `key` must contain a secp256k1 public key to encrypt the data with
        - If using `SYMMETRIC` encoding, `key` must contain a 32 bytes symmetric key.
4. `char* signingKey`: Hex string containing a secp256k1 private key to sign the encoded message, It's optional. To not
   sign the message use `NULL` instead.
5. `int version`: is used to define the type of payload encryption

**Returns**
`JSONResponse` with the base64 encoded payload. An `error` message otherwise.

---

### `extern char* gowaku_decode_data(char* data, char* keyType, char* key, int version)`

Decode a byte array according to [RFC 26](https://rfc.vac.dev/spec/26/). This function can be used to decrypt the
payload of a waku message

**Parameters**

1. `char* data`: byte array to decode, in base64.
2. `char* keyType`: defines the type of key to use:
    - `NONE`: no encryption was used in the payload
    - `ASYMMETRIC`: decrypt the payload using a secp256k1 public key
    - `SYMMETRIC`: decrypt the payload using a 32 bit key.
3. `char* key`: hex key (`0x123...abc`) used for decrypting the `data`.
    - When `version` is 0: No encryption is used
    - When `version` is 1
        - If using `ASYMMETRIC` encoding, `key` must contain a secp256k1 private key to decrypt the data
        - If using `SYMMETRIC` encoding, `key` must contain a 32 bytes symmetric key.
4. `int version`: is used to define the type of payload encryption

**Returns**
`JSONResponse` with the decoded payload. An `error` message otherwise. The decoded payload has this format:

```json
{
  "result": {
    "pubkey": "0x04123...abc",
    // secp256k1 public key
    "signature": "0x123...abc",
    // secp256k1 signature
    "data": "...",
    // base64 encoded
    "padding": "..."
    // base64 encoded
  }
}
```

---

### `extern char* gowaku_utils_base64_encode(char* data)`

Encode a byte array to base64 useful for creating the payload of a waku message in the format understood
by `gowaku_relay_publish`

**Parameters**

1. `char* data`: byte array to encode

**Returns**
A `char *` containing the base64 encoded byte array

---

### `extern char* gowaku_utils_base64_decode(char* data)`

Decode a base64 string (useful for reading the payload from waku messages)

**Parameters**

1. `char* data`: base64 encoded byte array to decode

**Returns**
`JSONResponse` with the decoded payload. An `error` message otherwise.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
