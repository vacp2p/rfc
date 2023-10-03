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
* `clusterId`: Unique identifier of the cluster that the node is running in.
* `shards`: Shard indexes that the node is subscribed to.


## Protocol id

`/vac/waku/metadata/1.0.0`

## Request

```proto
message WakuMetadataRequest {
  optional uint32 cluster_id = 1;
  repeated uint32 shards = 2;
}
```

## Response

```proto
message WakuMetadataResponse {
  optional uint32 cluster_id = 1;
  repeated uint32 shards = 2;
}
```
