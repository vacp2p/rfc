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
    1. [Methods](#methods)
3. [Copyright](#copyright)

## Abstract

In this specification we describe the RPC API that Waku nodes SHOULD adhere to. The unified API allows clients to easily
be able to connect to any node implementation.

## Introduction

This API is based off the [Whisper V5 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v5-RPC-API). It focuses on the end-user API semantics for communicating with a Node. However, it does not document APIs used to maintain a node.

## Wire Protocol

### Transport

Nodes SHOULD expose a [JSON RPC](https://www.jsonrpc.org/specification) API that can be publicly accessed. The JSON RPC version SHOULD be `2.0`. Below is an example request:

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
- **`maxMessageSize` [float]** - The current maximum message size in bytes.
- **`memory` [number]** - The memory size of the floating messages in bytes.
- **`messages` [number]** - The number of floating messages.

#### `waku_post`

The `waku_post` method creates a [waku message](./waku-1.md#messages) and disseminates it to the network.

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
 
*Either the **symKeyID** or the **pubKey** need to be present. It can not be both.*

#### Response

- **bool** - `true` on success or an error on failure.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
