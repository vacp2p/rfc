---
slug: 67
title: 67/WAKU2-MESSAGE-HASH-BASED-QUERY
name: Waku Message Hash Based Query
status: raw
editor: Kaichao Sun <kaichao@status.im>
contributors:
---

# Message Hash Based Query

Waku nodes can query messages by given an array of message hash. This is useful for checking if some messages are stored in the response node, and retrieve missing messages by only knowing its hash.

## Protocol id

`/vac/waku/messages/1.0.0-beta1`

## Request

The parameters are the following:
* `digests`: array of unique identifiers of the messages that request node is interested in, refer to [Deterministic Message Hashing](/spec/14/#deterministic-message-hashing). The max length of `digests` should be 20.

```proto
message QueryWakuMessagesRequest {
  repeated bytes digests = 1;
}
```

## Response

The response are the following:
* `messages`: array of messages that the response node has, refer to [Waku Message](/spec/14). The order of the response messages is not ensured to be the same as the order of the request digests. And the response messages could be less than `digests`, which means the response node don't have the messages corresponding to specific digests. The request node needs to recalculate the message hash to match digest with a WakuMessage in `messages`.

```proto
message QueryWakuMessagesResponse {
  repeated WakuMessage messages = 1;
}
```

## Security Consideration

The main security consideration is that the response node could connect participants by looking into who are interested in same messages. This could be mitigated by setting preferred response node or only request to nodes that share the same interest, such as members in a community.

## Recommend Implementations

[Store service](/spec/13) provider is recommend to implement this protocol for fine-grained message retrieval.

## References

- [14/WAKU2-MESSAGE](/spec/14)
- [14/WAKU2-STORE](/spec/13)
