# Waku Whisper Specification

> Version 0.0.5 (Draft)
>
> Authors: Oskar Thorén oskar@status.im, Dean Eigenmann dean@status.im

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Specification](#specification)
    - [Use of RLPx transport protocol](#use-of-rlpx-transport-protocol)
    - [Packet Codes](#packet-codes)
    - [Packet Format and Usage](#packet-format-and-usage)
    - [Whisper Envelope](#whisper-envelope)
    - [Contents of Data Field of the Message (Optional)](#contents-of-data-field-of-the-message-optional)
    - [Payload Encryption](#payload-encryption)
- [Light node](#light-node)
- [Rationale](#rationale)
- [Backwards Compatibility](#backwards-compatibility)
    - [WakuWhisper bridging](#wakuwhisper-bridging)
- [Forwards Compatibility](#forwards-compatibility)
- [Security Considerations](#security-considerations)
- [Implementation](#implementation)
- [Footnotes](#footnotes)
- [Changelog](#changelog)
    - [Differences between shh/6 waku/0](#differences-between-shh6-waku0)
- [Copyright](#copyright)

## Abstract

This specification describes the format of Waku messages within the ÐΞVp2p Wire Protocol. This spec substitutes [EIP- 627](https://eips.ethereum.org/EIPS/eip-627). Waku is a fork of the original Whisper protocol that enables better usability for resource restricted devices, such as mostly-offline bandwidth-constrained smartphones. It does this primarily through (a) light node support and (b) historic messages (with a mailserver). Additionally, other experimental features for better scalability and DDoS resistance are under development and will be part of future versions.

<!-- TODO: Add waku mode in v1, it isn't in the spec yet:

> Waku is a fork of the original Whisper protocol that enables better scalability and offline messaging, at the cost of some metadata protection guarantees. It does this through (a) light node support (b) historic messages (through a mailserver) (d) basic rate limiting.
-->

## Motivation

Waku was created to incrementally improve in areas that Whisper is lacking in, with special attention to resource restricted devices. We specify the standard for Waku messages in order to ensure forward compatibility of different Waku clients, backwards compatibility with Whisper clients, as well as to allow multiple implementations of Waku and its capabilities. We also modify the language to be more unambiguous, concise and consistent.

## Specification

### Use of RLPx transport protocol

All Waku messages are sent as devp2p RLPx transport protocol, version 5 <sup>[1](https://github.com/ethereum/devp2p/blob/master/rlpx.md)</sup> packets. These packets MUST be RLP-encoded arrays of data containing two objects: packet code followed by another object (whose type depends on the packet code).  See [informal RLP spec](https://github.com/ethereum/wiki/wiki/RLP) and the [Ethereum Yellow Paper, appendix B](https://ethereum.github.io/yellowpaper/paper.pdf) for more details on RLP.

Waku nodes that do not support a particular packet code MUST ignore the packet without generating any error.

Waku is a RLPx subprotocol called `waku` with version `0`. The version number corresponds to the major version in the header spec.

### ABNF specification

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

<!-- TODO: This needs to be run through ABNF validator -->

<!-- TODO: packet-format / packet rules, compact into one somehow? -->

<!-- TOOD: ABNF floating point rep for pow? https://www.cs.cmu.edu/Groups/AI/html/cltl/clm/node19.html -->

```
packet-format ::= "[" packet-code packet-format "]"

; Packet codes 0 - 127 are reserved for Waku protocol
packet-code ::= 0 / 1 / 2 / 3 / [ x4-127 ]

; packet-format needs to be paired with its corresponding packet-format

packet ::= "[" required-packet / [ optional-packet ] "]"

required-packet ::= 0 status / 1 messages / 2 pow-requirement / 3 bloom-filter

optional-packet ::= 126 p2p-request / 127 p2p-message

packet-format ::= "[" packet-code packet-format "]"

status ::= "[" version pow-requirement [ bloom-filter ] [ light-node ] "]"

; version is "an integer (as specified in RLP)"
version ::= integer

messages ::= *whisper-envelope

; pow is "a single floating point value of PoW. This value is the IEEE 754 binary representation of 64-bit floating point number. Values of qNAN, sNAN, INF and -INF are not allowed. Negative values are also not allowed."
pow-requirement ::= pow

; bloom filter is "a byte array"
bloom-filter ::= bytes

light-node ::= bool

p2p-request ::= whisper-envelope

p2p-message ::= whisper-envelope

whisper-envelope ::= "[" expiry ttl topic data nonce "]"

; 4 bytes (UNIX time in seconds)
expiry ::= bytes

; 4 bytes (time-to-live in seconds)
ttl ::= bytes

; 4 bytes of arbitrary data
topic ::= bytes

; byte array of arbitrary size (contains encrypted message)
data ::= bytes

; 8 bytes of arbitrary data (used for PoW calculation)
nonce ::= bytes

```

All primitive types are RLP encoded. Note that, per RLP specification, integers are encoded starting from `0x00`.

### Packet Codes

The message codes reserved for Waku protocol: 0 - 127.

Messages with unknown codes MUST be ignored, for forward compatibility of future versions.

The Waku sub-protocol MUST support the following packet codes:

| Name                       | Int Value |
|----------------------------|-----------|
| Status                     |     0     |
| Messages                   |     1     |
| PoW Requirement            |     2     |
| Bloom Filter               |     3     |

The following message codes are optional, but they are reserved for specific purpose.

| Name                       | Int Value |
|----------------------------|-----------|
| P2P Request                |    126    |
| P2P Message                |    127    |

<!-- TODO: There are more packet codes we use in practice - see Adam's list -->

### Packet usage

**Status**

The bloom filter paramenter is optional; if it is missing or nil, the node is considered to be full node (i.e. accepts all messages).

Status message should be sent after the RLPx handshake and prior to any other messages.

When a node does not receive the status message from a peer, before a configurable timeout, it MUST disconnect from that peer.

**Messages**

This packet is used for sending the standard Whisper envelopes.

**PoW Requirement**

This packet is used by Whisper nodes for dynamic adjustment of their individual PoW requirements. Recipient of this message should no longer deliver the sender messages with PoW lower than specified in this message.

PoW is defined as average number of iterations, required to find the current BestBit (the number of leading zero bits in the hash), divided by message size and TTL:

	PoW = (2**BestBit) / (size * TTL)

PoW calculation:

	fn short_rlp(envelope) = rlp of envelope, excluding env_nonce field.
	fn pow_hash(envelope, env_nonce) = sha3(short_rlp(envelope) ++ env_nonce)
	fn pow(pow_hash, size, ttl) = 2**leading_zeros(pow_hash) / (size * ttl)

where size is the size of the RLP-encoded envelope, excluding env_nonce field (size of `short_rlp(envelope)`).

**Bloom Filter**

This packet is used by Whisper nodes for sharing their interest in messages with specific topics.

The Bloom filter is used to identify a number of topics to a peer without compromising (too much) privacy over precisely what topics are of interest. Precise control over the information content (and thus efficiency of the filter) may be maintained through the addition of bits.

Blooms are formed by the bitwise OR operation on a number of bloomed topics. The bloom function takes the topic and projects them onto a 512-bit slice. At most, three bits are marked for each bloomed topic.

The projection function is defined as a mapping from a 4-byte slice S to a 512-bit slice D; for ease of explanation, S will dereference to bytes, whereas D will dereference to bits.

	LET D[*] = 0
	FOREACH i IN { 0, 1, 2 } DO
	LET n = S[i]
	IF S[3] & (2 ** i) THEN n += 256
	D[n] = 1
	END FOR

OPTIONAL

**P2P Request**

This packet is used for sending Dapp-level peer-to-peer requests, e.g. Whisper Mail Client requesting old messages from the Whisper Mail Server.

**P2P Message**

This packet is used for sending the peer-to-peer messages, which are not supposed to be forwarded any further. E.g. it might be used by the Whisper Mail Server for delivery of old (expired) messages, which is otherwise not allowed.

### Whisper Envelope data field (Optional)

<!-- TODO: ABNF this and make language more strong -->

This section outlines the optional description of Data Field to set up an example. Later it may be moved to a separate spec.

It is only relevant if you want to decrypt the incoming message, but if you only want to send a message, any other format would be perfectly valid and must be forwarded to the peers.

Data field contains encrypted message of the Envelope. In case of symmetric encryption, it also contains appended Salt (a.k.a. AES Nonce, 12 bytes). Plaintext (unencrypted) payload consists of the following concatenated fields: flags, auxiliary field, payload, padding and signature (in this sequence).

	flags: 1 byte; first two bits contain the size of auxiliary field, third bit indicates whether the signature is present.
	
	auxiliary field: up to 4 bytes; contains the size of payload.

	payload: byte array of arbitrary size (may be zero).

	padding: byte array of arbitrary size (may be zero).

	signature: 65 bytes, if present.

	salt: 12 bytes, if present (in case of symmetric encryption).

Those unable to decrypt the message data are also unable to access the signature. The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity. The signature is serialised as the concatenation of the `R`, `S` and `V` parameters of the SECP-256k1 ECDSA signature, in that order. `R` and `S` are both big-endian encoded, fixed-width 256-bit unsigned. `V` is an 8-bit big-endian encoded, non-normalised and should be either 27 or 28.

The padding field was introduced in order to align the message size, since message size alone might reveal important metainformation. Padding can be arbitrary size. However, it is recommended that the size of Data Field (excuding the Salt) before encryption (i.e. plain text) should be factor of 256 bytes.

### Payload Encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme with SECP-256k1 public key.

Symmetric encryption uses AES GCM algorithm with random 96-bit nonce.

### Packet code Rationale

Packet codes 0x00 and 0x01 are already used in all Whisper versions.

Packet code 0x02 will be necessary for the future development of Whisper. It will provide possiblitity to adjust the PoW requirement in real time. It is better to allow the network to govern itself, rather than hardcode any specific value for minimal PoW requirement.

Packet code 0x03 will be necessary for scalability of the network. In case of too much traffic, the nodes will be able to request and receive only the messages they are interested in.

Packet codes 0x7E and 0x7F may be used to implement Whisper Mail Server and Client. Without P2P messages it would be impossible to deliver the old messages, since they will be recognized as expired, and the peer will be disconnected for violating the Whisper protocol. They might be useful for other purposes when it is not possible to spend time on PoW, e.g. if a stock exchange will want to provide live feed about the latest trades.


## Additional capabilities

Waku supports multiple capabilities. These include light node, rate limting, mailserver (client and server) and bridging of traffic. Here we list these capabilities, how they are identified, what properties they have and what invariants they must maintain.

### Light node

The rationale for light nodes is to allow for interaction with waku on resource restricted devices as bandwidth can often be an issue.

Light nodes MUST NOT forward any incoming messages, they MUST only send their own messages. When light nodes happen to connect to each other, they SHOULD disconnect. As this would result in messages being dropped between the two.

Light nodes are identified by the `light_node` value in the status message.

<!-- TODO: Add details on handshake -- >

### Mailserver and client

Mailservers are waku nodes that can archive messages and delivering them to its peers on-demand.

Messages MAY be requested from mailservers by sending a packet with the `p2pRequestCode` code. Once received a mailserver SHOULD respond at some point in the future with packets containing the `p2pMessageCode` code.

<!-- TODO: Clean up vocabulary, is it mailserver and mailserver client? mailclient? mailserver node? historynode? etc -->

## Backwards Compatibility

Waku is a different subprotocol from Whisper so it isn't directly compatible. However, the data format is the same, so compatibility can be achieved by the use of a bridging mode as described below. Any client which does not implement certain packet codes should gracefully ignore the packets with those codes. This will ensure the forward compatibility. 

<!-- TODO: Elaborate on waku/1 would be directly compatible with waku/0 if version don't match -->

### Waku-Whisper bridging

`waku/0` and `shh/6` are different DevP2P subprotocols. In order to achieve backwards compatibility, bridging is required. It works as follows.

**Roles:**
- Waku client A, only Waku capability
- Whisper client B, only Whisper capability
- WakuWhisper bridge C, both Waku and Whisper capability

**Flow:**
1. A posts message; B posts message.
2. C picks up message from A and B and relays them both to Waku and Whisper.
3. A receives message on Waku; B on Whisper.

**Note**: This flow means if another bridge C1 is active, we might get duplicate relaying for a message between C1 and C2. I.e. Whisper(<>Waku<>Whisper)<>Waku, A-C1-C2-B. Theoretically this bridging chain can get as long as TTL permits.

## Forwards Compatibility

It is desirable to have a strategy for maintaining forward compatibility between `waku/0` and future version of waku. Here we outline some concerns and strategy for this.

<!-- TODO: Outline difference between _bridging_ and data format -->

<!-- TODO: Think about how to maintain forwards capability for waku/v0 -> v1 -> v2, etc. -->
<!-- Example user story: changing version number to 1; moving to libp2p; changing routing to PSS style; remove PoW; replacing PoW with zkSNARKs; adding packet codes for rate limit / accounting for resources feedback; additional disconnect features -->

<!-- TODO: Right now we have

    if m.protocolVersion == wakuVersion:
      debug "Waku peer", peer, wakuVersion
    else:
      raise newException(UselessPeerError, "Incompatible Waku version")

Is this what we want? Decide!
-->

<!-- TODO: Leave room for RECOMMENDATIONS FOR CLIENTS and DEPRECATION NOTICE when relevant -->


## Security considerations

There are several security considerations to take into account when running Waku. Chief among them are: scalability, DDoS-resistance and privacy. These also varies depending on what capabilities are used, such as mailserver, light node, and so on.

<!-- TODO: elaborate on security considerations -->

<!-- TODO: Light node security considerations

> Running a node as a light node mode impacts privacy due to the fact that it becomes identifiable what nodes care about if they aren't relaying traffic.

Replace with:

> I think the main privacy concern with light nodes is that the directly connected peers will know that a message originates from them (as it are the only ones it sends). And yes, based on that they can make some assumptions on which messages (topics) they are interested in also.

-->


## Implementation Notes

Notes useful for implementing Waku mode.


<!-- TODO: Implementation notes, possibly link to matrix and have something similar but lightweight to https://tools.ietf.org/html/rfc8446#appendix-C

Alt, break this out into issue for enhancement -->

<!-- TODO(Dean): Break out into Status spec and remove this section

The golang implementation of Whisper (v.6) already uses packet codes 0x00 - 0x03. Parity's implementation of v.6 will also use codes 0x00 - 0x03. Codes 0x7E and 0x7F are reserved, but still unused and left for custom implementation of Whisper Mail Server.

-->

## Footnotes

1. https://github.com/ethereum/devp2p/blob/master/rlpx.md


## Changelog

| Version | Comment |
| :-----: | ------- |
| 0.0.5 (current) | [WIP] Initial Release |

<!-- TODO: Start counting at 0.1, which is when this PR is merged -->

### Differences between shh/6 waku/0

Summary of main differences between this spec and Whisper v6, as described in [EIP-627](https://eips.ethereum.org/EIPS/eip-627):

- RLPx subprotocol is changed from `shh/6` to `waku/0`
- Light node capability
- Whisper Mail Server and Whisper Mail Client implemented

<!-- TODO: Document further differences with Whisper v6 -->

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- TODO: Document recommendations for mobile nodes node --->
<!-- TODO: Document spam resistance in practice, rate limiting -->
<!-- TODO: Document accounting for resources, with mention of later settlement -->
<!-- TODO: Consider adding roadmap, or link to -->
