---
slug: 64
title: 64/WAKU2-NETWORK
name: Waku v2 Network
status: raw
category: Best Current Practice
editor: Hanno Cornelius <hanno@status.im>
contributors:
---

# Abstract

This RFC specifies an opinionated deployment of [10/WAKU2](https://rfc.vac.dev/spec/10/) protocols
to form a coherent and shared decentralized messaging network
that is open-access,
useful for generalized messaging,
privacy-preserving,
scalable and
accessible even to resource-restricted devices.
We'll refer to this opinionated deployment simply as
_the public Waku Network_, _the Waku Network_ or, if the context is clear, _the network_
in the rest of this document.

# Theory / Semantics

## Routing protocol

The Waku Network is built on the [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/) routing protocol,
which in turn is an extension of [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/) with spam protection measures.

## Network shards

Traffic in the Waku Network is sharded into eight [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/) pubsub topics.
Each pubsub topic is named according to the static shard naming format
defined in [51/WAKU2-RELAY-SHARDING](https://rfc.vac.dev/spec/51/)
with `<shard_cluster_index>` set to `1` and
`<shard_number>` occupying the range `0` to `7`.
In other words, the Waku Network is a [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/) network
routed on the combination of the eight pubsub topics:
```
/waku/2/rs/1/0
/waku/2/rs/1/1
...
/waku/2/rs/1/7
```

## Roles

There are two distinct roles evident in the network, those of:
1) nodes, and
2) applications.

### Nodes

Nodes are the individual software units
using [10/WAKU2](https://rfc.vac.dev/spec/10/) protocols to form a p2p messaging network.
Nodes, in turn, can participate in a shard as full relayers, i.e. _relay nodes_,
or by running a combination of protocols suitable for resource-restricted environments, i.e. _non-relay nodes_.
Nodes can also provide various services to the network,
such as storing historical messages or protecting the network against spam.
See the section on [default services](#default-services) for more.

#### Relay nodes

Relay nodes MUST follow [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/)
to route messages to other nodes in the network
for any of the pubsub topics [defined as the Waku Network shards](#network-shards).
Relay nodes MAY choose to subscribe to any of these shards,
but MUST be subscribed to at least one defined shard.
Each relay node SHOULD be subscribed to as many shards as it has resources to support.
If a relay node supports an encapsulating application,
it SHOULD be subscribed to all the shards servicing that application.
If resource restrictions prevent a relay node from servicing all shards used by the encapsulating application,
it MAY choose to support some shards as a non-relay node.

#### Bootstrapping and discovery

Nodes MAY use any method to bootstrap connection to the network,
but it is RECOMMENDED that each node retrieves a list of bootstrap peers to connect to using [EIP-1459 DNS-based discovery](https://eips.ethereum.org/EIPS/eip-1459).
Relay nodes SHOULD use [33/WAKU2-DISCV5](https://rfc.vac.dev/spec/33/) to continually discover other peers in the network.
Each relay node MUST encode its supported shards into its discoverable ENR
as described in [51/WAKU2-RELAY-SHARDING](https://rfc.vac.dev/spec/51/#discovery).
The ENR MUST be updated if the set of supported shards change.
A node MAY choose to ignore discovered peers that do not support any of the shards in its own subscribed set.

#### Transports

Relay nodes MUST follow [10/WAKU2](https://rfc.vac.dev/spec/10/) specifications with regards to supporting different transports.
If TCP transport is available, each relay node MUST support it as transport for both dialing and listening.
In addition, a relay node SHOULD support secure websockets for bidirectional communication streams,
for example to allow connections from and to web browser-based clients.
A relay node MAY support unsecure websockets if required by the application or running environment.

#### Default services

For each supported shard,
each relay node SHOULD enable and support the following protocols as a service node:
1. [12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/) to allow resource-restricted peers to subscribe to messages matching a specific content filter.
2. [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) to allow other peers to request historical messages from this node.
3. [19/WAKU2-LIGHTPUSH](https://rfc.vac.dev/spec/19/) to allow resource-restricted peers to request publishing a message to the network on their behalf.
4. [34/WAKU2-PEER-EXCHANGE](https://rfc.vac.dev/spec/34/) to allow resource-restricted peers to discover more peers in a resource efficient way.

#### Store service nodes

Each relay node SHOULD support [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) as a store service node,
for each supported shard.
The store SHOULD be configured to retain at least `12` hours of messages per supported shard.
Store service nodes SHOULD only store messages with a valid [`rate_limit_proof`](#message-attributes) attribute.

#### Non-relay nodes

Nodes MAY opt out of relay functionality on any network shard
and instead request services from relay nodes as clients
using any of the defined service protocols:
1. [12/WAKU2-FILTER](https://rfc.vac.dev/spec/12/) to subscribe to messages matching a specific content filter.
2. [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) to request historical messages matching a specific content filter.
3. [19/WAKU2-LIGHTPUSH](https://rfc.vac.dev/spec/19/) to request publishing a message to the network.
4. [34/WAKU2-PEER-EXCHANGE](https://rfc.vac.dev/spec/34/) to discover more peers in a resource efficient way.

#### Store client nodes

Nodes MAY request historical messages from [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) service nodes as store clients.
A store client SHOULD discard any messages retrieved from a store service node that do not contain a valid [`rate_limit_proof`](#message-attributes) attribute.
The client MAY consider service nodes returning messages without a valid [`rate_limit_proof`](#message-attributes) attribute as untrustworthy.
The mechanism by which this may happen is currently underdefined.

### Applications

Applications are the higher-layer projects or platforms that make use of the generalized messaging capability of the network.
In other words, an application defines a payload used in the various [10/WAKU2](https://rfc.vac.dev/spec/10/) protocols.
Any participant in an application SHOULD make use of an underlying node in order to communicate on the network.
Applications SHOULD make use of an [autosharding](#autosharding) API
to allow the underlying node to automatically select the target shard on the Waku Network. See the section on [autosharding](#autosharding) for more.

## RLN rate-limiting

The [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/) network uses [32/RLN-V1](https://rfc.vac.dev/spec/32/) proofs
to ensure that a pre-agreed rate limit is not exceeded by any publisher.
While the network is under capacity,
individual relayers MAY choose to freely route messages without RLN proofs
up to a discretionary bandwidth limit after which messages without proofs MUST be discarded.
This bandwidth limit SHOULD be enforced using [bandwidth validation mechanism](#free-bandwidth-exceeded) separate from RLN rate-limiting.
This implies that quality of service and reliability is significantly lower for messages without proofs
and at times of high network utilization these messages may not be relayed at all.

### RLN Parameters

For the Waku Network,
the `epoch` is set to `1` second
and the maximum number of messages published per `epoch` is limited to `1` per publisher.
The `max_epoch_gap` is set to `20` seconds,
meaning that validators MUST _reject_ messages with an `epoch` more than 20 seconds into the past or future compared to the validator's own clock.
All nodes, validators and publishers,
SHOULD use Network Time Protocol (NTP) to synchronize their own clocks,
thereby ensuring valid timestamps for proof generation and validation.

### Memberships

Each publisher to the Waku Network SHOULD register an RLN membership
with one of the RLN storage contracts
moderated in the Sepolia registry contract with address [0xF1935b338321013f11068abCafC548A7B0db732C](https://sepolia.etherscan.io/address/0xF1935b338321013f11068abCafC548A7B0db732C#code).
Initial memberships are registered in the Sepolia RLN storage contract with address [0x58322513A35a8f747AF5A385bA14C2AbE602AA59](https://sepolia.etherscan.io/address/0x58322513A35a8f747AF5A385bA14C2AbE602AA59#code).
RLN membership setup and registration MUST follow [17/WAKU-RLN-RELAY](https://rfc.vac.dev/spec/17/#setup-and-registration),
with the `staked_fund` set to `0`.
In other words, the Waku Network does not use RLN staking. 

### RLN Proofs

Each RLN member MUST generate and attach an RLN proof to every published message
as described in [17/WAKU-RLN-RELAY](https://rfc.vac.dev/spec/17/#publishing).
Slashing is not implemented for the Waku Network.
Instead, validators will penalise peers forwarding messages exceeding the rate limit
as specified for [the rate-limiting validation mechanism](#rate-limit-exceeded).
This incentivizes all nodes to validate RLN proofs
and reject messages violating rate limits
in order to continue participating in the network.

## Network traffic

All payload on the Waku Network MUST be encapsulated in a [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/)
with rate limit proof extensions defined for [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/#payloads).
Each message on the Waku Network SHOULD be validated by each relayer,
according to the rules discussed under [message validation](#message-validation).

### Message Attributes

- The mandatory `payload` attribute MUST contain the message data payload as crafted by the application.
- The mandatory `content_topic` attribute MUST specify a string identifier that can be used for content-based filtering. This is also crafted by the application. See [Autosharding](#autosharding) for more on the content topic format.
- The optional `meta` attribute MAY be omitted. If present this will form part of the message uniqueness vector described in [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/).
- The optional `version` attribute SHOULD be set to `0`. It MUST be interpreted as `0` if not present.
- The mandatory `timestamp` attribute MUST contain the Unix epoch time at which the message was generated by the application. The value MUST be in nanoseconds. It MAY contain a fudge factor of up to 1 seconds in either direction to improve resistance to timing attacks.
- The optional `ephemeral` attribute MUST be set to `true` if the message should not be persisted by the Waku Network.
- The optional `rate_limit_proof` attribute SHOULD be populated with the RLN proof as set out in [RLN Proofs](#rln-proofs). Messages with this field unpopulated MAY be discarded from the network by relayers. This field MUST be populated if the message should be persisted by the Waku Network.

### Message Size

Any Waku Message published to the network MUST NOT exceed an absolute maximum size of `150` kilobytes.
This limit applies to the entire message after protobuf serialization, including attributes.
It is RECOMMENDED not to exceed an average size of `4` kilobytes for Waku Messages published to the network.

### Message Validation

Relay nodes MUST apply [gossipsub v1.1 validation](https://github.com/libp2p/specs/blob/c96c9ec5909d64fe020d7630f3fd982bc18fd06a/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators) to each relayed message and
SHOULD apply all of the rules set out in the section below to determine the validity of a message.
Validation has one of three outcomes,
repeated here from the [gossipsub specification](https://github.com/libp2p/specs/blob/c96c9ec5909d64fe020d7630f3fd982bc18fd06a/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators) for ease of reference:
1. Accept - the message is considered valid and it MUST be delivered and forwarded to the network.
2. Reject - the message is considered invalid, MUST be rejected and SHOULD trigger a gossipsub scoring penalty against the transmitting peer.
3. Ignore - the message SHOULD NOT be delivered and forwarded to the network, but this MUST NOT trigger a gossipsub scoring penalty against the transmitting peer.

The following validation rules are defined:

#### Decoding failure

If a message fails to decode as a valid [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/).
the relay node MUST _reject_ the message.
This SHOULD trigger a penalty against the transmitting peer.

#### Invalid timestamp

If a message has a timestamp deviating by more than `20` seconds
either into the past or the future
when compared to the relay node's internal clock,
the relay node MUST _reject_ the message.
This allows for some deviation between internal clocks,
network routing latency and
an optional [fudge factor when timestamping new messages](#message-attributes).

#### Free bandwidth exceeded

If a message contains no RLN proof
and the current bandwidth utilization on the shard the message was published to
equals or exceeds `1` Mbps,
the relay node SHOULD _ignore_ the message.

#### Invalid RLN epoch

If a message contains an RLN proof
and the `epoch` attached to the proof deviates by more than `max_epoch_gap` seconds
from the relay node's own `epoch`,
the relay node MUST _reject_ the message.
`max_epoch_gap` is [set to `20` seconds](#rln-parameters) for the Waku Network.

#### Invalid RLN proof

If a message contains an RLN proof
and the zero-knowledge proof is invalid
according to the verification process described in [32/RLN-V1](https://rfc.vac.dev/spec/32/#verification),
the relay node MUST _ignore_ the message.

#### Rate limit exceeded

If a message contains an RLN proof
and the relay node detects double signaling
according to the verification process described in [32/RLN-V1](https://rfc.vac.dev/spec/32/#verification),
the relay node MUST _reject_ the message
for violating the agreed rate limit of `1` message every `1` second.
This SHOULD trigger a penalty against the transmitting peer.

## Autosharding

Nodes in the Waku Network SHOULD allow encapsulating applications to use autosharding,
as defined in [51/WAKU2-RELAY-SHARDING](https://rfc.vac.dev/spec/51/#automatic-sharding)
by automatically determining the appropriate pubsub topic
from the list [of defined Waku Network shards](#network-shards).
This allows the application to omit the target pubsub topic
when invoking any Waku protocol function.
Applications using autosharding MUST use content topics in the format
defined in [51/WAKU2-RELAY-SHARDING](https://rfc.vac.dev/spec/51/#content-topics-format-for-autosharding)
and SHOULD use the short length format:

```
/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}`
```

When an encapsulating application makes use of autosharding
the underlying node MUST determine the target pubsub topic(s)
from the content topics provided by the application
using the hashing mechanism defined in [51/WAKU2-RELAY-SHARDING](https://rfc.vac.dev/spec/51/#automatic-sharding).


# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

(TBD)

## normative
(TBD)
A list of references that MUST be read to fully understand and/or implement this protocol.
See [RFC3967 Section 1.1](https://datatracker.ietf.org/doc/html/rfc3967#section-1.1).

## informative
(TBD)
A list of additional references.

