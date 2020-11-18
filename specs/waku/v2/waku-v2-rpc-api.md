---
title: Waku v2 JSON-RPC REST API
version: 0.0.1
status: Raw
authors: Hanno Cornelius <hanno@status.im>
---

## Table of Contents

1. [Introduction](#introduction)
2. [Wire Protocol](#wire-protocol)
    1. [Transport](#transport)
    1. [Types](#types)
    1. [Debug API](#debug-api)
    1. [Relay API](#relay-api)
    1. [Store API](#store-api)
    1. [Filter API](#filter-api)
3. [Copyright](#copyright)

## Introduction

This specification describes the JSON-RPC API that Waku v2 nodes MAY adhere to. Refer to the [Waku v2 specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-v2.md) for more information on Waku v2.

## Wire Protocol

### Transport

Nodes SHOULD expose an accessible [JSON-RPC](https://www.jsonrpc.org/specification) API. The JSON-RPC version SHOULD be `2.0`. Below is an example request:

```json
{
  "jsonrpc":"2.0",
  "method":"get_waku_v2_debug_info",
  "params":[],
  "id":1
}
```

#### Fields

| Field     | Description                                         |
| --------- | --------------------------------------------------- |
| `jsonrpc` | Contains the used JSON-RPC version (`Default: 2.0`) |
| `method`  | Contains the JSON-RPC method that is being called   |
| `params`  | An array of parameters for the request              |
| `id`      | The request ID                                      |

### Types

In this specification, the primitive types `Boolean`, `String`, `Number` and `Null`, as well as the structured types `Array` and `Object`, are to be interpreted according to the [JSON-RPC specification](https://www.jsonrpc.org/specification#conventions). It also adopts the same capitalisation conventions.

The following structured types are defined for use throughout the document:

#### WakuMessage

`WakuMessage` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `payload` | `String` | mandatory | The payload being sent |
| `contentTopic` | `Number` | optional | Message content topic for optional content-based filtering |

#### WakuInfo

`WakuInfo` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `listenStr` | `String` | mandatory | Address that the node is listening for |

#### HistoryQuery

`HistoryQuery` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `topics` | `Array`[`Number`] | mandatory | Array of content topics to query for historical messages |
| `pagingInfo` | [`PagingInfo`](#PagingInfo) | optional | Pagination information |

#### HistoryResponse

`HistoryResponse` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `messages` | `Array`[[`WakuMessage`](#WakuMessage)] | mandatory | Array of retrieved historical messages |
| `pagingInfo` | [`PagingInfo`](#PagingInfo) | [conditional](#store-api) | Paging information from which to resume further historical queries |

#### PagingInfo

`PagingInfo` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `pageSize` | `Number` | mandatory | Number of messages to retrieve per page |
| `cursor` | [`Index`](#Index) | optional | Message [`Index`](#Index) from which to perform pagination. If not included and `forward` is set to `true`, paging will be performed from the beginning of the list. If not included and `forward` is set to `false`, paging will be performed from the end of the list.|
| `forward` | `Bool` | mandatory | `true` if paging forward, `false` if paging backward |

#### Index

`Index` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `digest` | `String` | mandatory | A hash for the message at this [`Index`](#Index) |
| `receivedTime` | `Number` | mandatory | UNIX timestamp at which the message at this [`Index`](#Index) was received |

#### FilterRequest

`FilterRequest` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `contentFilters` | `Array`[[`ContentFilter`](#ContentFilter)] | mandatory | Array of content filters applying to the request |
| `topic` | `String` | optional | Message topic |

#### ContentFilter

`ContentFilter` is an `Object` containing the following fields:
| Field | Type | Inclusion | Description |
| ----: | :--: | :--: |----------- |
| `topics` | `Array`[`Number`] | mandatory | Array of message content topics |

### Debug API

#### `get_waku_v2_debug_info`

The `get_waku_v2_debug_info` method retrieves information about a Waku v2 node

##### Parameters

none

##### Response
- [**`WakuInfo`**](#WakuInfo) - information about a Waku v2 node

### Relay API

Refer to the [Waku Relay specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-relay.md) for more information on the relaying of messages.

#### `post_waku_v2_relay_publish`

The `post_waku_v2_relay_publish` method publishes a message on a [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor)

##### Parameters

- **`String`** - the [PubSub `topic`](https://github.com/libp2p/specs/blob/master/pubsub/README.md#the-topic-descriptor) being published on
- [**`WakuMessage`**](#WakuMessage) - the `message` to publish

##### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

### Store API

Refer to the [Waku Store specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-store.md) for more information on message history retrieval.

#### `get_waku_v2_store_query`

The `get_waku_v2_store_query` method retrieves historical messages on specific content topics.

##### Parameters

- [**`HistoryQuery`**](#HistoryQuery) - the history `query` to perform. This query MAY be called with [`PagingInfo`](#PagingInfo), to retrieve historical messages on a per-page basis.

##### Response

- [**`HistoryResponse`**](#HistoryResponse) - the response to a history `query`. If the `query` included [`PagingInfo`](#PagingInfo), the node MUST return messages on a per-page basis and include [`PagingInfo`](#PagingInfo) in the response. This [`PagingInfo`](#PagingInfo) MUST contain a `cursor` pointing to the [`Index`](#Index) from which a new page can be requested.

### Filter API

Refer to the [Waku Filter specification](https://github.com/vacp2p/specs/blob/master/specs/waku/v2/waku-filter.md) for more information on content filtering.

#### `post_waku_v2_filter_subscribe`

The `post_waku_v2_filter_subscribe` method subscribes to receive notifications for any inbound [`WakuMessage`](#WakuMessage) that matches a content filter.

##### Parameters

- [**`FilterRequest`**](#FilterRequest) - filter criteria being subscribed to.

##### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

#### `post_waku_v2_filter_unsubscribe`

The `post_waku_v2_filter_unsubscribe` method unsubscribes from notifications for inbound [`WakuMessages`](#WakuMessage) matching a content filter.

##### Parameters

- [**`FilterRequest`**](#FilterRequest) - filter criteria being unsubscribed from.

##### Response

- **`Bool`** - `true` on success or an [error](https://www.jsonrpc.org/specification#error_object) on failure.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
