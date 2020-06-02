---
title: Waku RPC API
version: 1.0.0
status: Draft
authors: Dean Eigenmann <dean@status.im>
---

## Table of Contents

1. [Abstract](#abstract)
2. [Wire Protocol](#wire-protocol)
    1. [Transport](#transport)
    1. [Objects](#objects)
    1. [Methods](#methods)
3. [Copyright](#copyright)

## Abstract

In this specification we describe the RPC API that Waku nodes MAY adhere to. The unified API allows clients to easily
be able to connect to any node implementation. The API described is privileged as a node stores the keys of clients. 

## Introduction

This API is based off the [Whisper V6 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API).

## Wire Protocol

### Transport

Nodes SHOULD expose a [JSON RPC](https://www.jsonrpc.org/specification) API that can be accessed. The JSON RPC version SHOULD be `2.0`. Below is an example request:

```json
{
  "jsonrpc":"2.0",
  "method":"shh_version",
  "params":[],
  "id":1
}
```

#### Fields

| Field     | Description                                         |
| --------- | --------------------------------------------------- |
| `jsonrpc` | Contains the used JSON RPC version (`Default: 2.0`) |
| `method`  | Contains the JSON RPC method that is being called   |
| `params`  | An array of parameters for the request              |
| `id`      | The request ID                                      |

### Objects

In this section you will find objects used throughout the JSON RPC API.

#### Message

The message object represents a Waku message. Below you will find the description of the attributes contained in the message object. A message is the decrypted payload and padding of an [envelope](./envelope-data-format.md) along with all of its metadata and other extra information such as the hash.

| Field | Type | Description |
| ----: | :--: | ----------- |
| `sig` | string | Public Key that signed this message |
| `recipientPublicKey` | string | The recipients public key |
| `ttl` | number | Time-to-live in seconds |
| `timestamp` | number | Unix timestamp of the message generation |
| `topic` | string | 4 bytes, the message topic |
| `payload` | string | Decrypted payload |
| `padding` | string | Optional padding, byte array of arbitrary length |
| `pow` | number | The proof of work value |
| `hash` | string | Hash of the enveloped message |

#### Filter

The filter object represents filters that can be applied to retrieve messages. Below you will find the description of the attributes contained in the filter object.

| Field | Type | Description |
| ----: | :--: | ----------- |
| `symKeyID` | string | ID of the symmetric key for message decryption |
| `privateKeyID` | string | ID of private (asymmetric) key for message decryption |
| `sig` | string | Public key of the signature |
| `minPow` | number | Minimal PoW requirement for incoming messages |
| `topics` | array | Array of possible topics, this can also contain partial topics |
| `allowP2P` | boolean | Indicates if this filter allows processing of direct peer-to-peer messages |

All fields are optional, however `symKeyID` or `privateKeyID` must be present, it cannot be both. Additionally, the `topics` field is only optional when an asymmetric key is used.

### Methods

#### `waku_version`

The `waku_version` method returns the current version number.

##### Parameters

none

##### Response

- **string** - The version number.

#### `waku_info`

The `waku_info` method returns information about a Waku node.

##### Parameters

none

##### Response

The response is an `Object` containing the following fields:

- **`minPow` [number]** - The current PoW requirement.
- **`maxEnvelopeSize` [float]** - The current maximum envelope size in bytes.
- **`memory` [number]** - The memory size of the floating messages in bytes.
- **`envelopes` [number]** - The number of floating envelopes.

#### `waku_setMaxEnvelopeSize`

Sets the maximum envelope size allowed by this node. Any envelopes larger than this size both incoming and outgoing will be rejected. The envelope size can never exceed the underlying envelope size of `10mb` <!-- is this still accurate -->

##### Parameters

 - **number** - The message size in bytes.

##### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_setMinPoW`

Sets the minimal PoW required by this node.

##### Parameters

 - **number** - The new PoW requirement.

##### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_markTrustedPeer`

Marks a specific peer as trusted allowing it to send expired messages.

##### Parameters

- **string** - `enode` of the peer.

##### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_newKeyPair`

Generates a keypair used for message encryption and decryption.

##### Parameters

none

##### Response

 - **string** - Key ID on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_addPrivateKey`

Stores a key and returns its ID.

##### Parameters

- **string** - Private key as hex bytes.

##### Response

 - **string** - Key ID on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_deleteKeyPair`

Deletes a specific key if it exists.

##### Parameters

- **string** - ID of the Key pair.

##### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_hasKeyPair`

Checks if the node has a private key of a key pair matching the given ID.

##### Parameters

- **string** - ID of the Key pair.

##### Response

- **bool** - `true` or `false` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_getPublicKey`

Returns the public key for an ID.

##### Parameters

- **string** - ID of the Key.

##### Response

 - **string** - The public key or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_getPrivateKey`

Returns the private key for an ID.

##### Parameters

- **string** - ID of the Key.

##### Response

 - **string** - The private key or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_newSymKey`

Generates a random symmetric key and stores it under an ID. This key can be used to encrypt and decrypt messages where the key is known to both parties.

##### Parameters

none

##### Response

 - **string** - The key ID or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_addSymKey`

Stores the key and returns its ID.

##### Parameters

- **string** - The raw key for symmetric encryption hex encoded.

##### Response

 - **string** - The key ID or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_generateSymKeyFromPassword`

Generates the key from a password and stores it.

##### Parameters

 - **string** - The password.

##### Response

 - **string** - The key ID or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_hasSymKey`

Returns whether there is a key associated with the ID.

##### Parameters

- **string** - ID of the Key.

##### Response

- **bool** - `true` or `false` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_getSymKey`

Returns the symmetric key associated with an ID.

##### Parameters

- **string** - ID of the Key.

##### Response

- **string** - Raw key on success or an [error](https://www.jsonrpc.org/specification#error_object) of failure.

#### `waku_deleteSymKey`

Deletes the key associated with an ID.

##### Parameters

- **string** - ID of the Key.

##### Response

- **bool** - `true` or `false` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_subscribe`

Creates and registers a new subscription to receive notifications for inbound Waku messages.

##### Parameters

The parameters for this request is an array containing the following fields:

 1. **string** - The ID of the function call, in case of Waku this must contain the value "messages".
 2. **object** - The [message filter](#filter).

##### Response

- **string** - ID of the subscription or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

###### Notifications

Notifications received by the client contain a [message](#message) matching the filter. Below is an example notification:

```json
{
  "jsonrpc": "2.0",
  "method": "shh_subscription",
  "params": {
    "subscription": "02c1f5c953804acee3b68eda6c0afe3f1b4e0bec73c7445e10d45da333616412",
    "result": {
      "sig": "0x0498ac1951b9078a0549c93c3f6088ec7c790032b17580dc3c0c9e900899a48d89eaa27471e3071d2de6a1f48716ecad8b88ee022f4321a7c29b6ffcbee65624ff",
      "recipientPublicKey": null,
      "ttl": 10,
      "timestamp": 1498577270,
      "topic": "0xffaadd11",
      "payload": "0xffffffdddddd1122",
      "padding": "0x35d017b66b124dd4c67623ed0e3f23ba68e3428aa500f77aceb0dbf4b63f69ccfc7ae11e39671d7c94f1ed170193aa3e327158dffdd7abb888b3a3cc48f718773dc0a9dcf1a3680d00fe17ecd4e8d5db31eb9a3c8e6e329d181ecb6ab29eb7a2d9889b49201d9923e6fd99f03807b730780a58924870f541a8a97c87533b1362646e5f573dc48382ef1e70fa19788613c9d2334df3b613f6e024cd7aadc67f681fda6b7a84efdea40cb907371cd3735f9326d02854",
      "pow": 0.6714754098360656,
      "hash": "0x17f8066f0540210cf67ef400a8a55bcb32a494a47f91a0d26611c5c1d66f8c57"
    }
  }
}
```

#### `waku_unsubscribe`

Cancels and removes an existing subscription. The node MUST stop sending the client notifications.

##### Parameters

 - **string** - The subscription ID.

##### Response

- **bool** - `true` or `false`

#### `waku_newMessageFilter`

Creates a new message filter within the node. This filter can be used to poll for new messages that match the criteria.

##### Parameters

The request must contain a [message filter](#filter) as its parameter. 

##### Response

 - **string** - The ID of the filter.

#### `waku_deleteMessageFilter`

Removes a message filter from the node.

##### Parameters

- **string** - ID of the filter created with [`waku_newMessageFilter`](#waku_newMessageFilter).

##### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_getFilterMessages`

Retrieves messages that match a filter criteria and were received after the last time this function was called.

##### Parameters

- **string** - ID of the filter created with [`waku_newMessageFilter`](#waku_newMessageFilter).

##### Response

The response contains an array of [messages](#messages) or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `waku_post`

The `waku_post` method creates a waku envelope and disseminates it to the network.

##### Parameters

The parameters is an `Object` containing the following fields:
 - **`symKeyID` [string]** `optional` - The ID of the symmetric key used for encryption
 - **`pubKey` [string]** `optional` - The public key for message encryption.
 - **`sig` [string]** `optional` - The ID of the signing key.
 - **`ttl` [number]** - The time-to-live in seconds.
 - **`topic` [string]** - 4 bytes message topic.
 - **`payload` [string]** - The payload to be encrypted.
 - **`padding` [string]** `optional` - The padding, a byte array of arbitrary length.
 - **`powTime` [number]** - Maximum time in seconds to be spent on the proof of work.
 - **`powTarget` [number]** - Minimal PoW target required for this message.
 - **`targetPeer` [string]** `optional` - The optional peer ID for peer-to-peer messages.
 
*Either the **`symKeyID`** or the **`pubKey`** need to be present. It can not be both.*

#### Response

- **bool** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
