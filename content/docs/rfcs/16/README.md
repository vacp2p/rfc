---
slug: 16
title: 16/WAKU2-RPC
name: Waku v2 RPC API
status: draft
editor: Hanno Cornelius <hanno@status.im>
---

# Introduction

This specification describes the JSON-RPC API that Waku v2 nodes MAY adhere to. Refer to the [Waku v2 specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-v2.md) for more information on Waku v2.

# Wire Protocol

## Transport

Nodes SHOULD expose an accessible [JSON-RPC](https://www.jsonrpc.org/specification) API. The JSON-RPC version SHOULD be `2.0`. Below is an example request:

```json
{
  "jsonrpc":"2.0",
  "method":"get_waku_v2_debug_info",
  "params":[],
  "id":1
}
```

### Fields

| Field     | Description                                         |
| --------- | --------------------------------------------------- |
| `jsonrpc` | Contains the used JSON-RPC version (`Default: 2.0`) |
| `method`  | Contains the JSON-RPC method that is being called   |
| `params`  | An array of parameters for the request              |
| `id`      | The request ID                                      |

## Types

In this specification, the primitive types `Boolean`, `String`, `Number` and `Null`, as well as the structured types `Array` and `Object`, are to be interpreted according to the [JSON-RPC specification](https://www.jsonrpc.org/specification#conventions). It also adopts the same capitalisation conventions.

The following structured types are defined for use throughout the document:

### WakuMessage

Refer to [`Waku Message` specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-message.md) for more information.

`WakuMessage` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ---: | :---: | :---: | --- |
| `payload` | `String` | mandatory | The message payload as a hex encoded data string |
| `contentTopic` | `String` | optional | Message content topic for optional content-based filtering |
| `version` | `Number` | optional | Message version. Used to indicate type of payload encryption. Default version is 0 (no payload encryption). |

## Method naming

The JSON-RPC methods in this document are designed to be mappable to HTTP REST endpoints. Method names follow the pattern `<method_type>_waku_<protocol_version>_<api>_<api_version>_<resource>`

- `<method_type>`: prefix of the HTTP method type that most closely matches the JSON-RPC function. Supported `method_type` values are `get`, `post`, `put`, `delete` or `patch`.
- `<protocol_version>`: Waku version. Currently **v2**.
- `<api>`: one of the listed APIs below, e.g. `store`, `debug`, or `relay`.
- `<api_version>`: API definition version. Currently **v1** for all APIs.
- `<resource>`: the resource or resource path being addressed

The method `post_waku_v2_relay_v1_message`, for example, would map to the HTTP REST endpoint `POST /waku/v2/relay/v1/message`.

## Debug API

### Types

The following structured types are defined for use on the Debug API:

#### WakuInfo

`WakuInfo` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `listenStr` | `String` | mandatory | Address that the node is listening for |

### `get_waku_v2_debug_v1_info`

The `get_waku_v2_debug_v1_info` method retrieves information about a Waku v2 node

#### Parameters

none

#### Response
- [**`WakuInfo`**](#WakuInfo) - information about a Waku v2 node

## Relay API

Refer to the [Waku Relay specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md) for more information on the relaying of messages.

### Types

The following structured types are defined for use on the Relay API:

#### WakuRelayMessage

`WakuRelayMessage` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: | ----------- |
| `payload` | `String` | mandatory | The payload being relayed as a hex encoded data string |
| `contentTopic` | `String` | optional | Message content topic for optional content-based filtering |

 > **_NOTE:_** `WakuRelayMessage` maps directly to a [`WakuMessage`](#WakuMessage), except that the latter contains an explicit message `version`. For `WakuRelay` purposes, the versioning is handled by the API.

### `post_waku_v2_relay_v1_message`

The `post_waku_v2_relay_v1_message` method publishes a message to be relayed on a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor)

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being published on |
| `message` | [`WakuRelayMessage`](#WakuRelayMessage) | mandatory | The `message` being relayed |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `post_waku_v2_relay_v1_subscriptions`

The `post_waku_v2_relay_v1_subscriptions` method subscribes a node to an array of [PubSub `topics`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor).

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topics` | `Array`[`String`] | mandatory | The [PubSub `topics`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being subscribed to |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `delete_waku_v2_relay_v1_subscriptions`

The `delete_waku_v2_relay_v1_subscriptions` method unsubscribes a node from an array of [PubSub `topics`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor).

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topics` | `Array`[`String`] | mandatory | The [PubSub `topics`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being unsubscribed from |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `get_waku_v2_relay_v1_messages`

The `get_waku_v2_relay_v1_messages` method returns a list of messages that were received on a subscribed [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) after the last time this method was called. The server MUST respond with an [error](https://www.jsonrpc.org/specification#error_object) if no subscription exists for the polled `topic`. If no message has yet been received on the polled `topic`, the server SHOULD return an empty list. This method can be used to poll a `topic` for new messages.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) to poll for the latest messages |

#### Response

- **`Array`[[`WakuMessage`](#WakuMessage)]** - the latest `messages` on the polled `topic` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

## Store API

Refer to the [Waku Store specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-store.md) for more information on message history retrieval.

### Types

The following structured types are defined for use on the Store API:

#### StoreResponse

`StoreResponse` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `messages` | `Array`[[`WakuMessage`](#WakuMessage)] | mandatory | Array of retrieved historical messages |
| `pagingOptions` | [`PagingOptions`](#PagingOptions) | [conditional](#get_waku_v2_store_v1_messages) | Paging information from which to resume further historical queries |


#### PagingOptions

`PagingOptions` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `pageSize` | `Number` | mandatory | Number of messages to retrieve per page |
| `cursor` | [`Index`](#Index) | optional | Message [`Index`](#Index) from which to perform pagination. If not included and `forward` is set to `true`, paging will be performed from the beginning of the list. If not included and `forward` is set to `false`, paging will be performed from the end of the list.|
| `forward` | `Bool` | mandatory | `true` if paging forward, `false` if paging backward |

#### Index

`Index` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `digest` | `String` | mandatory | A hash for the message at this [`Index`](#Index) |
| `receivedTime` | `Number` | mandatory | UNIX timestamp at which the message at this [`Index`](#Index) was received |

#### ContentFilter

`ContentFilter` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `contentTopic` | `String` | mandatory | The content topic of a [`WakuMessage`](#WakuMessage) |

### `get_waku_v2_store_v1_messages`

The `get_waku_v2_store_v1_messages` method retrieves historical messages on specific content topics. This method MAY be called with [`PagingOptions`](#PagingOptions), to retrieve historical messages on a per-page basis. If the request included [`PagingOptions`](#PagingOptions), the node MUST return messages on a per-page basis and include [`PagingOptions`](#PagingOptions) in the response. These [`PagingOptions`](#PagingOptions) MUST contain a `cursor` pointing to the [`Index`](#Index) from which a new page can be requested.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `pubsubTopic` | `String` | mandatory | The pubsub topic on which a  [`WakuMessage`](#WakuMessage) is published |
| `contentFilters` | `Array`[[`ContentFilter`](#contentfilter)] | mandatory | Array of content filters to query for historical messages |
| `pagingOptions` | [`PagingOptions`](#PagingOptions) | optional | Pagination information |

#### Response

- [**`StoreResponse`**](#StoreResponse) - the response to a `query` for historical messages.

## Filter API

Refer to the [Waku Filter specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-filter.md) for more information on content filtering.

### Types

The following structured types are defined for use on the Filter API:

#### ContentFilter

`ContentFilter` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topics` | `Array`[`String`] | mandatory | Array of message content topics |

### `post_waku_v2_filter_v1_subscription`

The `post_waku_v2_filter_v1_subscription` method creates a subscription in a [light node](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-filter.md#rationale) for messages that matches a content filter and, optionally, a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor).

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `contentFilters` | `Array`[[`ContentFilter`](#ContentFilter)] | mandatory | Array of content filters being subscribed to |
| `topic` | `String` | optional | Message topic |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `delete_waku_v2_filter_v1_subscription`

The `delete_waku_v2_filter_v1_subscription` method removes subscriptions in a [light node](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-filter.md#rationale) matching a content filter and, optionally, a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor).

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `contentFilters` | `Array`[[`ContentFilter`](#ContentFilter)] | mandatory | Array of content filters being unsubscribed from |
| `topic` | `String` | optional | Message topic |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `get_waku_v2_filter_v1_messages`

The `get_waku_v2_filter_v1_messages` method returns a list of messages that were received on a subscribed content `topic` after the last time this method was called. The server MUST respond with an [error](https://www.jsonrpc.org/specification#error_object) if no subscription exists for the polled content `topic`. If no message has yet been received on the polled content `topic`, the server SHOULD respond with an empty list. This method can be used to poll a content `topic` for new messages.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `contentTopic` | `String` | mandatory | The content topic to poll for the latest messages |

#### Response

- **`Array`[[`WakuMessage`](#WakuMessage)]** - the latest `messages` on the polled content `topic` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

## Admin API

The Admin API provides privileged accesses to the internal operations of a Waku v2 node.

### Types

The following structured types are defined for use on the Admin API:

#### WakuPeer

`WakuPeer` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `multiaddr` | `String` | mandatory | Multiaddress containing this peer's location and identity |
| `protocol` | `String` | mandatory | Protocol that this peer is registered for |
| `connected` | `bool` | mandatory | `true` if peer has active connection for this `protocol`, `false` if not |

### `get_waku_v2_admin_v1_peers`

The `get_waku_v2_admin_v1_peers` method returns an array of peers registered on this node. Since a Waku v2 node may open either continuous or ad hoc connections, depending on the negotiated protocol, these peers may have different connected states. The same peer MAY appear twice in the returned array, if it is registered for more than one protocol.

#### Parameters

none

#### Response
- **`Array`[[`WakuPeer`](#WakuPeer)]** - Array of peers registered on this node

### `post_waku_v2_admin_v1_peers`

The `post_waku_v2_admin_v1_peers` method connects a node to a list of peers.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `peers` | `Array`[`String`] | mandatory | Array of peer `multiaddrs` to connect to. Each `multiaddr` must contain the [location and identity addresses](https://docs.libp2p.io/concepts/addressing/) of a peer. |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.


## Private API

The Private API provides functionality to encrypt/decrypt `WakuMessage` payloads using either symmetric or asymmetric cryptography. This allows backwards compatibility with [Waku v1 nodes](https://github.com/vacp2p/specs/blob/master/specs/waku/v1/waku-1.md).
It is the API client's responsibility to keep track of the keys used for encrypted communication. Since keys must be cached by the client and provided to the node to encrypt/decrypt payloads, a Private API SHOULD NOT be exposed on non-local or untrusted nodes.

### Types

The following structured types are defined for use on the Private API:

#### KeyPair

`KeyPair` is an `Object` containing the following fields:

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `privateKey` | `String` | mandatory | Private key as hex encoded data string |
| `publicKey` | `String` | mandatory | Public key as hex encoded data string |

### `get_waku_v2_private_v1_symmetric_key`

Generates and returns a symmetric key that can be used for message encryption and decryption.

#### Parameters

none

#### Response
- **`String`** - A new symmetric key as hex encoded data string

### `get_waku_v2_private_v1_asymmetric_keypair`

Generates and returns a public/private key pair that can be used for asymmetric message encryption and decryption.

#### Parameters

none

#### Response
- **[`KeyPair`](#KeyPair)** - A new public/private key pair as hex encoded data strings

### `post_waku_v2_private_v1_symmetric_message`

The `post_waku_v2_private_v1_symmetric_message` method publishes a message to be relayed on a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor). 

Before being relayed, the message payload is encrypted using the supplied symmetric key. The client MUST provide a symmetric key.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being published on |
| `message` | [`WakuRelayMessage`](#WakuRelayMessage) | mandatory | The (unencrypted) `message` being relayed |
| `symkey` | `String` | mandatory | The hex encoded symmetric key to use for payload encryption. This field MUST be included if symmetric key cryptography is selected |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `post_waku_v2_private_v1_asymmetric_message`

The `post_waku_v2_private_v1_asymmetric_message` method publishes a message to be relayed on a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor). 

Before being relayed, the message payload is encrypted using the supplied public key. The client MUST provide a public key.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being published on |
| `message` | [`WakuRelayMessage`](#WakuRelayMessage) | mandatory | The (unencrypted) `message` being relayed |
| `publicKey` | `String` | mandatory | The hex encoded public key to use for payload encryption. This field MUST be included if asymmetric key cryptography is selected |

#### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `get_waku_v2_private_v1_symmetric_messages`

The `get_waku_v2_private_v1_symmetric_messages` method decrypts and returns a list of messages that were received on a subscribed [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) after the last time this method was called. The server MUST respond with an [error](https://www.jsonrpc.org/specification#error_object) if no subscription exists for the polled `topic`. If no message has yet been received on the polled `topic`, the server SHOULD return an empty list. This method can be used to poll a `topic` for new messages.

Before returning the messages, the server decrypts the message payloads using the supplied symmetric key. The client MUST provide a symmetric key.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) to poll for the latest messages |
| `symkey` | `String` | mandatory | The hex encoded symmetric key to use for payload decryption. This field MUST be included if symmetric key cryptography is selected |

#### Response

- **`Array`[[`WakuRelayMessage`](#WakuRelayMessage)]** - the latest `messages` on the polled `topic` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### `get_waku_v2_private_v1_asymmetric_messages`

The `get_waku_v2_private_v1_asymmetric_messages` method decrypts and returns a list of messages that were received on a subscribed [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) after the last time this method was called. The server MUST respond with an [error](https://www.jsonrpc.org/specification#error_object) if no subscription exists for the polled `topic`. If no message has yet been received on the polled `topic`, the server SHOULD return an empty list. This method can be used to poll a `topic` for new messages.

Before returning the messages, the server decrypts the message payloads using the supplied private key. The client MUST provide a private key.

#### Parameters

| Field | Type | Inclusion | Description |
| ----: | :---: | :---: |----------- |
| `topic` | `String` | mandatory | The [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) to poll for the latest messages |
| `privateKey` | `String` | mandatory | The hex encoded private key to use for payload decryption. This field MUST be included if asymmetric key cryptography is selected |

#### Response

- **`Array`[[`WakuRelayMessage`](#WakuRelayMessage)]** - the latest `messages` on the polled `topic` or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

# Example usage

## Store API

### `get_waku_v2_store_v1_messages`

This method is part of the `store` API and the specific resources to retrieve are (historical) `messages`. The protocol (`waku`) is on `v2`, whereas the Store API definition is on `v1`.

1. `get` *all* the historical messages for content topic **"/waku/2/default-content/proto"**; no paging required

#### Request

```curl -d '{"jsonrpc":"2.0","id":"id","method":"get_waku_v2_store_v1_messages", "params":["", ["/waku/2/default-content/proto"]]}' --header "Content-Type: application/json" http://localhost:8545```

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "method": "get_waku_v2_store_v1_messages",
  "params": [
    "",
    [
      {"contentTopic": "/waku/2/default-content/proto"}
    ]
  ]
}
```

#### Response

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "result": {
    "messages": [
      {
        "payload": [
          1
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      },
      {
        "payload": [
          2
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      },
      {
        "payload": [
          3
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      }
    ],
    "pagingInfo": null
  },
  "error": null
}
```

---

2. `get` a single page of historical messages for content topic **"/waku/2/default-content/proto"**; 2 messages per page, backward direction. Since this is the initial query, no `cursor` is provided, so paging will be performed from the end of the list.

#### Request

```curl -d '{"jsonrpc":"2.0","id":"id","method":"get_waku_v2_store_v1_messages", "params":[ "", ["/waku/2/default-content/proto"],{"pageSize":2,"forward":false}]}' --header "Content-Type: application/json" http://localhost:8545```

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "method": "get_waku_v2_store_v1_messages",
  "params": [
    "",
    [
      {"contentTopic": "/waku/2/default-content/proto"}
    ],
    {
      "pageSize": 2,
      "forward": false
    }
  ]
}
```

#### Response

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "result": {
    "messages": [
      {
        "payload": [
          2
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      },
      {
        "payload": [
          3
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      }
    ],
    "pagingInfo": {
      "pageSize": 2,
      "cursor": {
        "digest": "abcdef",
        "receivedTime": 1605887187
      },
      "forward": false
    }
  },
  "error": null
}
```

---

3. `get` the next page of historical messages for content topic **"/waku/2/default-content/proto"**, using the cursor received above; 2 messages per page, backward direction.

#### Request

```curl -d '{"jsonrpc":"2.0","id":"id","method":"get_waku_v2_store_v1_messages", "params":[ "", ["/waku/2/default-content/proto"],{"pageSize":2,"cursor":{"digest":"abcdef","receivedTime":1605887187.00},"forward":false}]}' --header "Content-Type: application/json" http://localhost:8545```

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "method": "get_waku_v2_store_v1_messages",
  "params": [
    "",
    [
      {"contentTopic": "/waku/2/default-content/proto"}
    ],
    {
      "pageSize": 2,
      "cursor": {
        "digest": "abcdef",
        "receivedTime": 1605887187
      },
      "forward": false
    }
  ]
}
```

#### Response

```jsonrpc
{
  "jsonrpc": "2.0",
  "id": "id",
  "result": {
    "messages": [
      {
        "payload": [
          1
        ],
        "contentTopic": "/waku/2/default-content/proto",
        "version": 0
      }
    ],
    "pagingInfo": {
      "pageSize": 2,
      "cursor": {
        "digest": "123abc",
        "receivedTime": 1605866187
      },
      "forward": false
    }
  },
  "error": null
}
```

# References

1. [JSON-RPC specification](https://www.jsonrpc.org/specification)
1. [LibP2P Addressing](https://docs.libp2p.io/concepts/addressing/)
1. [LibP2P PubSub specification - topic descriptor](https://github.com/libp2p/specs/tree/master/pubsub#the-topic-descriptor)
1. [Waku v2 specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-v2.md)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
