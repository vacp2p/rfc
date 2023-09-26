---
slug: 66
title: 66/WAKU2-METADATA
name: Waku Metadata Protocol
status: raw
editor: Alvaro Revuelta <alrevuelta@status.im>
contributors:
---

# Metadata Protocol

Waku specifies a req/resp protocol that provides information about the node's medatadata.


## Protocol id

`/vac/waku/metadata/1.0.0`

## Request

```proto
message WakuMetadataRequest {
}
```

## Response

```proto
message WakuMessageResponse {
  uint64 network_id = 1;
}
```
