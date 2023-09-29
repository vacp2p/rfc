---
slug: 66
title: 66/WAKU2-METADATA
name: Waku Metadata Protocol
status: raw
editor: Alvaro Revuelta <alrevuelta@status.im>
contributors:
---

# Metadata Protocol

Waku specifies a req/resp protocol that provides information about the node's medatadata. Such metadata is meant to be used
by the node to decide if a peer is worth connecting or not. The node that makes the request, includes its metadata
so that the receiver is aware of it, without requiring an extra interaction. The parameters are the following:
* `networkId`: Unique identifier of the network that the node is running in.
* `shards`: Shard indexes that the node is subscribed to.


## Protocol id

`/vac/waku/metadata/1.0.0`

## Request

```proto
message WakuMetadataRequest {
  uint32 network_id = 1;
  repeated unt32 shards = 2;
}
```

## Response

```proto
message WakuMetadataResponse {
  uint32 network_id = 1;
  repeated unt32 shards = 2;
}
```
