---
slug: 66
title: 66/WAKU2-METADATA
name: Waku Metadata Protocol
status: raw
editor: Alvaro Revuelta <alrevuelta@status.im>
contributors:
---

# Metadata Protocol

Waku specifies a req/resp protocol that provides information about the node medatadata.


## protocol id

`/vac/waku/metadata/1.0.0`

## request

```proto
message WakuMetadataRequest {
}
```

## response

```proto
message WakuMessageResponse {
  uint64 network_id = 1;
  uint64 rln_tree_block = 2;
  bytes rln_root = 3;
  TODO: capabilities?
}
```
