---
slug: 23
title: 23/WAKU2-TOPICS
name: Waku v2 Topic Usage Recommendations
status: draft
editor: Oskar Thoren <oskar@status.im>
contributors:
---

This document outlines recommended usage of topics in Waku v2.
In [10/WAKU2 spec](/spec/10) there are two types of topics:

- PubSub topics, used for routing
- Content topics, used for content-based filtering

## PubSub topics

PubSub topics are used for routing of messages, see [11/WAKU2-RELAY](/spec/11) spec for more details of how this routing works.

As written in [10/WAKU2 spec](/spec/10) there is a default PubSub topic:

`/waku/2/default-waku/proto`

This indicates that

1) It relates to the Waku problem domain
2) version is 2
3) `default-waku` indicates that it is the default topic for exchanging WakuMessages
4) that the [data field](/spec/11/#protobuf-definition) in PubSub is serialized/encoded as protobuf as determined by WakuMessage

### PubSub topic format

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

If you use another PubSub topic, be aware that:

- Metadata protection might be weakened
- Nodes subscribing to other topics might not be connected to the rest of the network
- Other nodes in the network might not subscribe and relay on your given PubSub topic
- Store nodes might not store messages for your given PubSub topic

This means you are likely to have to run your own full nodes which may make your application less robust.

Below we outline some scenarios where this might apply.

### Separation of two applications example

Let's say we have two different topics that are both experience heavy traffic but are completely separate in terms of problem domain and infrastructure.
This can be segregated into:

```
/waku/2/status/proto
/waku/2/walletconnect/proto
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

The other type of topic that exists in Waku v2 is a content topic.
This is used for content based filtering.
See [14/WAKU2-MESSAGE spec](/spec/14) for where this is specified.
Note that this doesn't impact routing of messages between relaying nodes,
but it does impact how request/reply protocols such as 
[12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/) and [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) are used.

This is especially useful for nodes that have limited bandwidth,
and only want to pull down messages that match this given content topic.

Since all messages are relayed using the relay protocol regardless of content topic,
you MAY use any content topic you wish without impacting how messages are relayed.

### Content topic format

The format for content topics is as follows:

`/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`

The name of a content topic is application-specific.
As an example, here's the content topic used for an upcoming testnet:

`/toychat/2/huilong/proto`

## Using content topics for your application

Make sure you have a unique application-name to avoid conflicting issues with other protocols.

If you have different versions of your protocol, this can be specified in the version field.

The name of the content topic is up to your application and depends on the problem domain, how you want to separate content, what the bandwidth and privacy guarantees are, etc.

The encoding field indicates the serialization/encoding scheme for the [WakuMessage payload](/spec/14/#payloads) field.

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
the following structure for the content topic SHOULD be used:

```
/waku/1/<4bytes-waku-v1-topic>/rlp
```

The `<4bytes-waku-v1-topic>` SHOULD be the lowercase hex representation of the 4-byte Waku v1 topic.
A `0x` prefix SHOULD be added to the hex string.

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
