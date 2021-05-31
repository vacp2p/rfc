---
slug: 23
title: 23/WAKU2-TOPICS
name: Waku v2 Topic Usage Recommendations
status: draft
editor: Oskar Thoren <oskar@status.im>
contributors:
---

This document outlines recommended usage of topics in Waku v2. In [10/WAKU2 spec](/spec/10) there are two types of topics:

- Pubsub topics, used for routing
- Content topics, used for content-based filtering

## Pubsub topics

Pubsub topics are used for routing of mesages, see [11/WAKU2-RELAY](/spec/11) spec for more details of how this routing works.

As written in [10/WAKU2 spec](/spec/10) there is a default Pubsub topic:

`/waku/2/default-waku/proto`

This indicates that

1) It relates to the Waku problem domain
2) version is 2
3) name indicates what is being exchanged, which in this case is WakuMessages over a single topic and
4) that the data in PubSub field is protobuf (unlike Eth2 where it is `ssz_snappy`) as determined by WakuMessage.

### Pubsub topic format

PubSub topics SHOULD follow the following structure:

`/waku/2/{topic-name}/{encoding}`

This namespaced structure makes things like compatibility and discoverability and automatic handling of new topics easier.
For example, if the encoding of the payload is changed, compression is introduced, etc.

For more on this format of PubSub topics, see [Ethereum 2 P2P spec](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/phase0/p2p-interface.md#topics-and-messages) where inspiration for this format was taken from.

### Default PubSub topic

Unless there's a good reason, the default PubSub topic SHOULD be used for all protocols.
However, in certain situations other topics MAY be used.

Using a single PubSub topic ensures a connected network, as well some degree of metadata protection.
See [section on Anonymity/Unlinkability](/spec/10/#anonymity--unlinkability).

If you use another PubSub topic, be aware that metadata protection might be weakened,
and other nodes in the network might not subscribe or store messages for your given Pubsub topic.
This means you are likely to have to run your own full nodes which may make your application less robust.

Below we outline some examples where this might apply.

### Separation of two applications example

Let's say we have two different topics that are both experience heavy traffic but are completely separate in terms of problem domain and infrastructure.
This can be segregated into:

```
/waku/2/waku-status/proto
/waku/2/waku-walletconnect/proto
```

This indicates that they are WakuMessages but for different domains completely.

### Topic sharding example

Topic sharding is currently not supported by default, but is planned for the future in order to deal with increased network traffic.
Here's an example of what this might look like:

```
waku/2/waku-9_shard-0/proto
...
waku/2/waku-9_shard-9/proto
```

This indicates explicitly that the network traffic has been partitioned into 10 buckets.

### Compression example

Not yet implemented, but would be easy to add with:

`/waku/2/default-waku/proto_snappy`

## Content topics

The other type of topic that exists in Waku v2 is content topics.
This is used for content based filtering.
See [14/WAKU2-MESSAGE spec](/spec/14) for where this is specified.
Note that this doesn't impact routing of messages between relaying nodes,
but it does impact how request/reply protocols such as 
[12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/) and [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) are used.

This is especially useful for nodes that have limited bandwidth,
and only want to pull down messages that match this filter.

Since all messages are relayed regardless of content topic, you MAY use any content topic you wish without impacting how messages are relayed.

### Content topic format

The format of content topics is as follows:

`/waku/2/ContentTopic/Encoding`

The name of ContentTopic is application-specific. As an example, here's the content topic used for an upcoming testnet:

`/waku/2/huilong/proto`

## Differences with Waku v1

In [6/WAKU1](/spec/6) there is no actual routing.
All messages are sent to all other nodes.
This means that we are implicitly using the same PubSub topic that would be something like:

```
/waku/1/default-waku/rlp
```

Topics in Waku v1 correspond to Content Topics in Waku v2.

### Bridging Waku v1 and Waku v2

To bridge Waku v1 and Waku v2 we have a [15/WAKU-BRIDGE](/spec/15).
For mapping Waku v1 topics to Waku v2 content topics,
the following structure is used:

```
/waku/v1/<4bytes-waku-v1-topic>/rlp
```

This creates a direct mapping between the two protocols.
For example:

```
/waku/1/0x007f80ff/rlp
```

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [10/WAKU2 spec](/spec/10)

2. [11/WAKU2-RELAY](/spec/11)

3. [Ethereum 2 P2P spec](https://github.com/ethereum/eth2.0-specs/blob/dev/specs/phase0/p2p-interface.md#topics-and-messages)

4. [14/WAKU2-MESSAGE spec](/spec/14)

5. [12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/)

6. [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)

7. [6/WAKU1](/spec/6)

8. [15/WAKU-BRIDGE](/spec/15)
