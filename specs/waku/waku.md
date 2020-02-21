---
title: Waku
version: 0.4.0
status: Draft
authors: Adam Babik <adam@status.im>, Andrea Maria Piana <andreap@status.im>, Dean Eigenmann <dean@status.im>, Kim De Mey <kimdemey@status.im>, Oskar Thorén <oskar@status.im>
---

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Definitions](#definitions)
- [Underlying Transports and Prerequisites](#underlying-transports-and-prerequisites)
    - [Use of DevP2P](#use-of-devp2p)
    - [Gossip based routing](#gossip-based-routing)
- [Wire Specification](#wire-specification)
    - [Use of RLPx transport protocol](#use-of-rlpx-transport-protocol)
    - [ABNF specification](#abnf-specification)
    - [Packet Codes](#packet-codes)
    - [Packet usage](#packet-usage)
    - [Payload Encryption](#payload-encryption)
    - [Packet code Rationale](#packet-code-rationale)
- [Additional capabilities](#additional-capabilities)
    - [Light node](#light-node)
    - [Accounting for resources (experimental)](#accounting-for-resources-experimental)
- [Backwards Compatibility](#backwards-compatibility)
    - [Waku-Whisper bridging](#waku-whisper-bridging)
- [Forwards Compatibility](#forwards-compatibility)
- [Appendix A: Security considerations](#appendix-a-security-considerations)
    - [Scalability and UX](#scalability-and-ux)
    - [Privacy](#privacy)
    - [Spam resistance](#spam-resistance)
    - [Censorship resistance](#censorship-resistance)
- [Appendix B: Implementation Notes](#appendix-b-implementation-notes)
    - [Implementation Matrix](#implementation-matrix)
    - [Recommendations for clients](#recommendations-for-clients)
    - [Node discovery](#node-discovery)
- [Footnotes](#footnotes)
- [Changelog](#changelog)
    - [Version 0.4](#version-04)
    - [Version 0.3](#version-03)
    - [Version 0.2](#version-02)
    - [Version 0.1](#version-01)
    - [Differences between shh/6 waku/0](#differences-between-shh6-waku0)
- [Acknowledgements](#acknowledgements)
- [Copyright](#copyright)

## Abstract

This specification describes the format of Waku messages within the ÐΞVp2p Wire Protocol. This spec substitutes [EIP-627](https://eips.ethereum.org/EIPS/eip-627). Waku is a fork of the original Whisper protocol that enables better usability for resource restricted devices, such as mostly-offline bandwidth-constrained smartphones. It does this through (a) light node support, (b) historic messages (with a mailserver) (c) expressing topic interest for better bandwidth usage and (d) basic rate limiting.

## Motivation

Waku was created to incrementally improve in areas that Whisper is lacking in, with special attention to resource restricted devices. We specify the standard for Waku messages in order to ensure forward compatibility of different Waku clients, backwards compatibility with Whisper clients, as well as to allow multiple implementations of Waku and its capabilities. We also modify the language to be more unambiguous, concise and consistent.

## Definitions

| Term            | Definition                                            |
| --------------- | ----------------------------------------------------- |
| **Light node**  | A Waku node that does not forward any messages.       |
| **Envelope**    | Messages sent and received by Waku nodes.             |
| **Node**        | Some process that is able to communicate for Waku.    |

## Underlying Transports and Prerequisites

### Use of DevP2P

For nodes to communicate, they MUST implement devp2p and run RLPx. They MUST have some way of connecting to other nodes. Node discovery is largely out of scope for this spec, but see the appendix for some suggestions on how to do this.

### Gossip based routing

In Whisper, messages are gossiped between peers. Whisper is a form of rumor-mongering protocol that works by flooding to its connected peers based on some factors. Messages are eligible for retransmission until their TTL expires. A node SHOULD relay messages to all connected nodes if an envelope matches their PoW and bloom filter settings. If a node works in light mode, it MAY choose not to forward envelopes. A node MUST NOT send expired envelopes, unless the envelopes are sent as a [mailserver](./mailserver.md) response. A node SHOULD NOT send a message to a peer that it has already sent before.

## Wire Specification

### Use of RLPx transport protocol

All Waku messages are sent as devp2p RLPx transport protocol, version 5<sup>[1](https://github.com/ethereum/devp2p/blob/master/rlpx.md)</sup> packets. These packets MUST be RLP-encoded arrays of data containing two objects: packet code followed by another object (whose type depends on the packet code).  See [informal RLP spec](https://github.com/ethereum/wiki/wiki/RLP) and the [Ethereum Yellow Paper, appendix B](https://ethereum.github.io/yellowpaper/paper.pdf) for more details on RLP.

Waku is a RLPx subprotocol called `waku` with version `0`. The version number corresponds to the major version in the header spec. Minor versions should not break compatiblity of `waku`, this would result in a new major. (Some expections to this apply in the Draft stage of where client implementation is rapidly change).

### ABNF specification

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; Packet codes 0 - 127 are reserved for Waku protocol
packet-code = 1*3DIGIT

; rate limits
limit-ip     = 1*DIGIT
limit-peerid = 1*DIGIT
limit-topic  = 1*DIGIT

rate-limits = "[" limit-ip limit-peerid limit-topic "]"

pow-requirement-key = 0
bloom-filter-key = 1
light-node-key = 2
confirmations-enabled-key = 3
rate-limits-key = 4
topic-interest-key = 5

status-options = "["
  [ pow-requirement-key pow-requirement ]
  [ bloom-filter-key bloom-filter ]
  [ light-node-key light-node ]
  [ confirmations-enabled-key confirmations-enabled ]
  [ rate-limits-key rate-limits ]
  [ topic-interest-key topic-interest ]
"]"

status = "[" version status-options "]"

status-update = status-options

; version is "an integer (as specified in RLP)"
version = DIGIT

confirmations-enabled = BIT

light-node = BIT

; pow is "a single floating point value of PoW.
; This value is the IEEE 754 binary representation
; of a 64-bit floating point number.
; Values of qNAN, sNAN, INF and -INF are not allowed.
; Negative values are also not allowed."
pow             = 1*DIGIT "." 1*DIGIT
pow-requirement = pow

; bloom filter is "a byte array"
bloom-filter = *OCTET

waku-envelope = "[" expiry ttl topic data nonce "]"

; List of topics interested in
topic-interest = "[" *10000topic "]"

; 4 bytes (UNIX time in seconds)
expiry = 4OCTET

; 4 bytes (time-to-live in seconds)
ttl = 4OCTET

; 4 bytes of arbitrary data
topic = 4OCTET

; byte array of arbitrary size
; (contains encrypted message)
data = OCTET

; 8 bytes of arbitrary data
; (used for PoW calculation)
nonce = 8OCTET

messages = 1*waku-envelope

; mail server / client specific
p2p-request = waku-envelope
p2p-message = 1*waku-envelope

; packet-format needs to be paired with its
; corresponding packet-format
packet-format = "[" packet-code packet-format "]"

required-packet = 0 status /
                  1 messages /
		  22 status-update /

optional-packet = 126 p2p-request / 127 p2p-message

packet = "[" required-packet [ optional-packet ] "]"
```

All primitive types are RLP encoded. Note that, per RLP specification, integers are encoded starting from `0x00`.

### Packet Codes

The message codes reserved for Waku protocol: 0 - 127.

Messages with unknown codes MUST be ignored without generating any error, for forward compatibility of future versions.

The Waku sub-protocol MUST support the following packet codes:

| Name                       |     Int Value |
| -------------------------- | ------------- |
| Status                     |     0         |
| Messages                   |     1         |
| Status Update              |     22        |

The following message codes are optional, but they are reserved for specific purpose.

| Name                       | Int Value | Comment |
|----------------------------|-----------|---------|
| Batch Ack                  |     11    |         |
| Message Response           |     12    |         |
| P2P Request                |    126    | |
| P2P Message                |    127    | |

### Packet usage

#### Status

The Status message serves as a Waku handshake and peers MUST exchange this
message upon connection. It MUST be sent after the RLPx handshake and prior to
any other Waku messages.

A Waku node MUST await the Status message from a peer before engaging in other Waku protocol activity with that peer.
When a node does not receive the Status message from a peer, before a configurable timeout, it SHOULD disconnect from that peer.

Upon retrieval of the Status message, the node SHOULD validate the message
received and validated the Status message. Note that its peer might not be in
the same state.

When a node is receiving other Waku messages from a peer before a Status
message is received, the node MUST ignore these messages and SHOULD disconnect from that peer. Status messages received after the handshake is completed MUST also be ignored.

The status message MUST contain an association list containing various options. All options within this association list are OPTIONAL, ordering of the key-value pairs is not guaranteed and therefore MUST NOT be relied on.

#### Messages

This packet is used for sending the standard Waku envelopes.

#### Status Update

The Status Update message is used to communicate an update of the settings of the node.
The format is the same as the Status message, all fields are optional.
If none of the options are specified the message MUST be ignored and considered a noop.
Fields that are omitted are considered unchanged, fields that haven't changed SHOULD not 
be transmitted.

**PoW Requirement update**

When PoW is updated, peers MUST NOT deliver the sender envelopes with PoW lower than specified in this message.

PoW is defined as average number of iterations, required to find the current BestBit (the number of leading zero bits in the hash), divided by message size and TTL:

	PoW = (2**BestBit) / (size * TTL)

PoW calculation:

	fn short_rlp(envelope) = rlp of envelope, excluding env_nonce field.
	fn pow_hash(envelope, env_nonce) = sha3(short_rlp(envelope) ++ env_nonce)
	fn pow(pow_hash, size, ttl) = 2**leading_zeros(pow_hash) / (size * ttl)

where size is the size of the RLP-encoded envelope, excluding `env_nonce` field (size of `short_rlp(envelope)`).

**Bloom filter update**

The bloom filter is used to identify a number of topics to a peer without compromising (too much) privacy over precisely what topics are of interest. Precise control over the information content (and thus efficiency of the filter) may be maintained through the addition of bits.

Blooms are formed by the bitwise OR operation on a number of bloomed topics. The bloom function takes the topic and projects them onto a 512-bit slice. At most, three bits are marked for each bloomed topic.

The projection function is defined as a mapping from a 4-byte slice S to a 512-bit slice D; for ease of explanation, S will dereference to bytes, whereas D will dereference to bits.

	LET D[*] = 0
	FOREACH i IN { 0, 1, 2 } DO
	LET n = S[i]
	IF S[3] & (2 ** i) THEN n += 256
	D[n] = 1
	END FOR

A full bloom filter (all the bits set to 1) means that the node is to be considered a `Full Node` and it will accept any topic.

If both Topic Interest and bloom filter are specified, Topic Interest always takes precedence and bloom filter MUST be ignored.

If only bloom filter is specified, the current Topic Interest MUST be discarded and only the updated bloom filter MUST be used when forwarding or posting envelopes.

A bloom filter with all bits set to 0 signals that the node is not currently interested in receiving any envelope.

**Topic Interest update**

This packet is used by Waku nodes for sharing their interest in messages with specific topics. It does this in a more bandwidth considerate way, at the expense of some metadata protection. Peers MUST only send envelopes with specified topics.


It is currently bounded to a maximum of 10000 topics. If you are interested in more topics than that, this is currently underspecified and likely requires updating it. The constant is subject to change.

If only Topic Interest is specified, the current bloom filter MUST be discarded and only the updated Topic Interest MUST be used when forwarding or posting envelopes.

An empty array signals that the node is not currently interested in receiving any envelope.

**Rate Limits update**

This packet is used for informing other nodes of their self defined rate limits.

In order to provide basic Denial-of-Service attack protection, each node SHOULD define its own rate limits. The rate limits SHOULD be applied on IPs, peer IDs, and envelope topics.

Each node MAY decide to whitelist, i.e. do not rate limit, selected IPs or peer IDs.

If a peer exceeds node's rate limits, the connection between them MAY be dropped.

Each node SHOULD broadcast its rate limits to its peers using the rate limits packet. The rate limits MAY also be sent as an optional parameter in the handshake.

Each node SHOULD respect rate limits advertised by its peers. The number of packets SHOULD be throttled in order not to exceed peer's rate limits. If the limit gets exceeded, the connection MAY be dropped by the peer.

**Message Confirmations update**

Message confirmations tell a node that a message originating from it has been received by its peers, allowing a node to know whether a message has or has not been received.

A node MAY send a message confirmation for any batch of messages received with a packet Messages Code.

A message confirmation is sent using Batch Acknowledge packet or Message Response packet. The Batch Acknowledge packet is followed by a keccak256 hash of the envelopes batch data.

The current `version` of the message response is `1`.

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; a version of the Message Response
version = 1*DIGIT

; keccak256 hash of the envelopes batch data (raw bytes) for which the confirmation is sent
hash = *OCTET

hasherror = *OCTET

; error code
code = 1*DIGIT

; a descriptive error message
description = *ALPHA

error  = "[" hasherror code description "]"
errors = *error

response = "[" hash errors "]"

confirmation = "[" version response "]"
```

The supported codes:
`1`: means time sync error which happens when an envelope is too old or created in the future (the root cause is no time sync between nodes).

The drawback of sending message confirmations is that it increases the noise in the network because for each sent message, a corresponding confirmation is broadcasted by one or more peers.


#### P2P Request

This packet is used for sending Dapp-level peer-to-peer requests, e.g. Waku Mail Client requesting old messages from the [Waku Mail Server](./mailserver.md).

#### P2P Message

This packet is used for sending the peer-to-peer messages, which are not supposed to be forwarded any further. E.g. it might be used by the Waku Mail Server for delivery of old (expired) messages, which is otherwise not allowed.


### Payload Encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme with SECP-256k1 public key.

Symmetric encryption uses AES GCM algorithm with random 96-bit nonce.

### Packet code Rationale

Packet codes `0x00` and `0x01` are already used in all Waku / Whisper versions. Packet code `0x02` and `0x03` were previously used in Whisper but are deprecated as of Waku v0.4

Packet code `0x22` is used to dynamically change the settings of a node.

Packet codes `0x7E` and `0x7F` may be used to implement Waku Mail Server and Client. Without P2P messages it would be impossible to deliver the old messages, since they will be recognized as expired, and the peer will be disconnected for violating the Whisper protocol. They might be useful for other purposes when it is not possible to spend time on PoW, e.g. if a stock exchange will want to provide live feed about the latest trades.

## Additional capabilities

Waku supports multiple capabilities. These include light node, rate limiting and bridging of traffic. Here we list these capabilities, how they are identified, what properties they have and what invariants they must maintain.

Additionally there is the capability of a mailserver which is documented in its on [specification](mailserver.md).

### Light node

The rationale for light nodes is to allow for interaction with waku on resource restricted devices as bandwidth can often be an issue.

Light nodes MUST NOT forward any incoming messages, they MUST only send their own messages. When light nodes happen to connect to each other, they SHOULD disconnect. As this would result in messages being dropped between the two.

Light nodes are identified by the `light_node` value in the status message.

### Accounting for resources (experimental)

Nodes MAY implement accounting, keeping track of resource usage. It is heavily inspired by Swarm's [SWAP protocol](https://www.bokconsulting.com.au/wp-content/uploads/2016/09/tron-fischer-sw3.pdf), and works by doing pairwise accounting for resources.

Each node keeps track of resource usage with all other nodes. Whenever an envelope is received from a node that is expected (fits bloom filter or topic interest, is legal, etc) this is tracked.

Every epoch (say, every minute or every time an event happens) statistics SHOULD be aggregated and saved by the client:

| peer  | sent | received |
|-------|------|----------|
| peer1 | 0    | 123 |
| peer2 | 10   | 40  |

In later versions this will be amended by nodes communication threshholds, settlements and disconnect logic.

## Upgradability and Compatibility

### General principles and policy

These are policies that guide how we make decisions when it comes to upgradability, compatibility, and extensibility:

1. Waku aims to be compatible with previous and future versions.

2. In cases where we want to break this compatibility, we do so gracefully and as a single decision point.

3. To achieve this, we employ the following two general strategies:
- a) Accretion (including protocol negotiation) over changing data
- b) When we want to change things, we give it a new name (for example, a version number).

Examples:

- We enable bridging between `shh/6` and `waku/0` until such a time as when we are ready to gracefully drop support for `shh/6` (1, 2, 3).
- When we add parameter fields, we (currently) do so by accreting them in a list, so old clients can ignore new fields (dynamic list) and new clients can use new capabilities (1, 3).
- To better support (2) and (3) in the future, we will likely release a new version that gives better support for open, growable maps (association lists or native map type) (3)
- When we we want to provide a new set of messages that have different requirements, we do so under a new protocol version and employ protocol versioning. This is a form of accretion at a level above - it ensures a client can support both protocols at once and drop support for legacy versions gracefully. (1,2,3)

### Backwards Compatibility

Waku is a different subprotocol from Whisper so it isn't directly compatible. However, the data format is the same, so compatibility can be achieved by the use of a bridging mode as described below. Any client which does not implement certain packet codes should gracefully ignore the packets with those codes. This will ensure the forward compatibility.

### Waku-Whisper bridging

`waku/0` and `shh/6` are different DevP2P subprotocols, however they share the same data format making their envelopes compatible. This means we can bridge the protocols naively, this works as follows.

**Roles:**
- Waku client A, only Waku capability
- Whisper client B, only Whisper capability
- WakuWhisper bridge C, both Waku and Whisper capability

**Flow:**
1. A posts message; B posts message.
2. C picks up message from A and B and relays them both to Waku and Whisper.
3. A receives message on Waku; B on Whisper.

**Note**: This flow means if another bridge C1 is active, we might get duplicate relaying for a message between C1 and C2. I.e. Whisper(<>Waku<>Whisper)<>Waku, A-C1-C2-B. Theoretically this bridging chain can get as long as TTL permits.

### Forward Compatibility

It is desirable to have a strategy for maintaining forward compatibility between `waku/0` and future version of waku. Here we outline some concerns and strategy for this.

- **Connecting to nodes with multiple versions:** The way this SHOULD be accomplished in the future is by negotiating the versions of subprotocols, within the `hello` message nodes transmit their capabilities along with a version. As suggested in [EIP-8](https://eips.ethereum.org/EIPS/eip-8), if a node connects that has a higher version number for a specific capability, the node with a lower number SHOULD assume backwards compatiblity. The node with the higher version will decide if compatibility can be assured between versions, if this is not the case it MUST disconnect.
- **Adding new packet codes:** New packet codes can be added easily due to the available packet codes, upgrades that add new packet codes should implement some fallback mechanism if no response was received for nodes that do not yet understand this packet.

## Appendix A: Security considerations

There are several security considerations to take into account when running Waku. Chief among them are: scalability, DDoS-resistance and privacy. These also vary depending on what capabilities are used. The security considerations for extra capabilities such as [mailservers](./wms.md#security-considerations) can be found in their respective specifications.

### Scalability and UX

**Bandwidth usage:**

In version 0 of Waku, bandwidth usage is likely to be an issue. For more investigation into this, see the theoretical scaling model described [here](https://github.com/vacp2p/research/tree/dcc71f4779be832d3b5ece9c4e11f1f7ec24aac2/whisper_scalability).

**Gossip-based routing:**

Use of gossip-based routing doesn't necessarily scale. It means each node can see a message multiple times, and having too many light nodes can cause propagation probability that is too low. See [Whisper vs PSS](https://our.status.im/whisper-pss-comparison/) for more and a possible Kademlia based alternative.

**Lack of incentives:**

Waku currently lacks incentives to run nodes, which means node operators are more likely to create centralized choke points.

### Privacy

**Light node privacy:**

The main privacy concern with light nodes is that directly connected peers will know that a message originates from them (as it are the only ones it sends). This means nodes can make assumptions about what messages (topics) their peers are interested in.

**Bloom filter privacy:**

By having a bloom filter where only the topics you are interested in are set, you reveal which messages you are interested in. This is a fundamental tradeoff between bandwidth usage and privacy, though the tradeoff space is likely suboptimal in terms of the [Anonymity](https://eprint.iacr.org/2017/954.pdf) [trilemma](https://petsymposium.org/2019/files/hotpets/slides/coordination-helps-anonymity-slides.pdf).

**Privacy guarantees not rigorous:**

Privacy for Whisper / Waku haven't been studied rigorously for various threat models like global passive adversary, local active attacker, etc. This is unlike e.g. Tor and mixnets.

**Topic hygiene:**

Similar to bloom filter privacy, if you use a very specific topic you reveal more information. See scalability model linked above.

### Spam resistance

**PoW bad for heterogenerous devices:**

Proof of work is a poor spam prevention mechanism. A mobile device can only have a very low PoW in order not to use too much CPU / burn up its phone battery. This means someone can spin up a powerful node and overwhelm the network.

### Censorship resistance

**Devp2p TCP port blockable:**

By default Devp2p runs on port `30303`, which is not commonly used for any other service. This means it is easy to censor, e.g. airport WiFi. This can be mitigated somewhat by running on e.g. port `80` or `443`, but there are still outstanding issues. See libp2p and Tor's Pluggable Transport for how this can be improved.

## Appendix B: Implementation Notes

### Implementation Matrix

| Client | Version |
| ------ | ------- |
| go-ethereum (geth) | [v1.9.7](https://github.com/ethereum/go-ethereum/tree/v1.9.7) |
| status whisper | [25321](https://github.com/status-im/whisper/tree/25321b2c035b6e03dbae85a2f54cf89f9f873dd9) |
| nimbus | [9c19f](https://github.com/status-im/nim-eth/tree/9c19f1e5b17b36ebcf1c7513428818f585a3cb16) |
| status-go | [ed5a5](https://github.com/status-im/status-go/commit/ed5a5c154daf5362cdf0c35fd1bc204e6a6d49ae) |

|    | Light mode    | Mail Client | Mail Server | shh/6 | waku/0 |
| -: | :-----------: | :---------: | :---------: |  :-:  | :-:   |
| **geth**           | x           | x           | x     | x | - |
| **status whisper** | x           | x           | -     | x | - |
| **nimbus**         | x           | -           | -     | x | - |
| **status-go**      | x           | x           | x     |x | - |


### Recommendations for clients

Notes useful for implementing Waku mode.

 1. Avoid duplicate envelopes

	To avoid duplicate envelopes, only connect to one Waku node. Benign duplicate envelopes is an intrinsic property of Whisper which often leads to a N factor increase in traffic, where N is the number of peers you are connected to.

 2. Topic specific recommendations

	Consider partition topics based on some usage, to avoid too much traffic on a single topic.

### Node discovery

Resource restricted devices SHOULD use [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459) to discover nodes.

Known static nodes MAY also be used.

## Footnotes

1. <https://github.com/ethereum/devp2p/blob/master/rlpx.md>

## Changelog

### Version 0.4

Released [February 20, 2020](https://github.com/vacp2p/specs/commit/ce12c499e0ac484dee2b05a307c86a164ae3f178).

- Introduces a new required packet code Status Code (`0x22`) for communicating option changes
- Deprecates the following packet codes: PoW Requirement (`0x02`), Bloom Filter (`0x03`), Rate limits (`0x20`), Topic interest (`0x21`) - all superseded by the new Status Code (`0x22`)
- Increased `topic-interest` capacity from 1000 to 10000

### Version 0.3

Released [February 13, 2020](https://github.com/vacp2p/specs/commit/73138d6ba954ab4c315e1b8d210ac7631b6d1428).

- Recommend DNS based node discovery over other Discovery methods.
- Mark spec as Draft mode in terms of its lifecycle.
- Simplify Changelog and misc formatting.
- Handshake/Status message not compatible with shh/6 nodes; specifying options as association list.
- Include topic-interest in Status handshake.
- Upgradability policy.
- `topic-interest` packet code.

### Version 0.2

Released [December 10, 2019](https://github.com/vacp2p/specs/blob/waku-0.2.0/waku.md).

- General style improvements.
- Fix ABNF grammar.
- Mailserver requesting/receiving.
- New packet codes: topic-interest (experimental), rate limits (experimental).
- More details on handshake modifications.
- Accounting for resources mode (experimental)
- Appendix with security considerations: scalablity and UX, privacy, and spam resistance.
- Appendix with implementation notes and implementation matrix across various clients with breakdown per capability.
- More details on handshake and parameters.
- Describe rate limits in more detail.
- More details on mailserver and mail client API.
- Accounting for resources mode (very experimental).
- Clarify differences with Whisper.

### Version 0.1

Initial version. Released [November 21, 2019](https://github.com/vacp2p/specs/blob/b59b9247f2ac1bf45c75bd3227a2e5dd87b6d7b0/waku.md).

### Differences between shh/6 and waku/0

Summary of main differences between this spec and Whisper v6, as described in [EIP-627](https://eips.ethereum.org/EIPS/eip-627):

- RLPx subprotocol is changed from `shh/6` to `waku/0`.
- Light node capability is added.
- Optional rate limiting is added.
- Status packet has following additional parameters: light-node,
confirmations-enabled and rate-limits
- Mail Server and Mail Client functionality is now part of the specification.
- P2P Message packet contains a list of envelopes instead of a single envelope.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
