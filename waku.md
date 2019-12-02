# Waku Whisper Specification

> Version 0.1.0 (Initial release)
>
> Authors: Oskar Thorén oskar@status.im, Dean Eigenmann dean@status.im

## Table of Contents

- [Abstract](#abstract)
- [Motivation](#motivation)
- [Specification](#specification)
    - [Use of RLPx transport protocol](#use-of-rlpx-transport-protocol)
    - [ABNF specification](#abnf-specification)
    - [Packet Codes](#packet-codes)
    - [Packet usage](#packet-usage)
    - [Whisper Envelope data field (Optional)](#whisper-envelope-data-field-optional)
    - [Payload Encryption](#payload-encryption)
    - [Packet code Rationale](#packet-code-rationale)
- [Additional capabilities](#additional-capabilities)
    - [Light node](#light-node)
    - [Mailserver and client](#mailserver-and-client)
- [Backwards Compatibility](#backwards-compatibility)
    - [Waku-Whisper bridging](#waku-whisper-bridging)
- [Forwards Compatibility](#forwards-compatibility)
- [Security considerations](#security-considerations)
- [Implementation Notes](#implementation-notes)
    - [Implementation Matrix](#implementation-matrix)
- [Footnotes](#footnotes)
- [Changelog](#changelog)
    - [Differences between shh/6 waku/0](#differences-between-shh6-waku0)
- [Acknowledgements](#acknowledgements)
- [Copyright](#copyright)

## Abstract

This specification describes the format of Waku messages within the ÐΞVp2p Wire Protocol. This spec substitutes [EIP- 627](https://eips.ethereum.org/EIPS/eip-627). Waku is a fork of the original Whisper protocol that enables better usability for resource restricted devices, such as mostly-offline bandwidth-constrained smartphones. It does this primarily through (a) light node support and (b) historic messages (with a mailserver). Additionally, other experimental features for better scalability and DDoS resistance are under development and will be part of future versions.

## Motivation

Waku was created to incrementally improve in areas that Whisper is lacking in, with special attention to resource restricted devices. We specify the standard for Waku messages in order to ensure forward compatibility of different Waku clients, backwards compatibility with Whisper clients, as well as to allow multiple implementations of Waku and its capabilities. We also modify the language to be more unambiguous, concise and consistent.

## Specification

### Use of RLPx transport protocol

All Waku messages are sent as devp2p RLPx transport protocol, version 5<sup>[1](https://github.com/ethereum/devp2p/blob/master/rlpx.md)</sup> packets. These packets MUST be RLP-encoded arrays of data containing two objects: packet code followed by another object (whose type depends on the packet code).  See [informal RLP spec](https://github.com/ethereum/wiki/wiki/RLP) and the [Ethereum Yellow Paper, appendix B](https://ethereum.github.io/yellowpaper/paper.pdf) for more details on RLP.

Waku is a RLPx subprotocol called `waku` with version `0`. The version number corresponds to the major version in the header spec.

### ABNF specification

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```
; Packet codes 0 - 127 are reserved for Waku protocol
packet-code     = 0 / 1 / 2 / 3 / [ x4-127 ]

status          = "[" version pow-requirement [ bloom-filter ] [ light-node ] "]"

; version is "an integer (as specified in RLP)"
version         = DIGIT

; pow is "a single floating point value of PoW. 
; This value is the IEEE 754 binary representation 
; of a 64-bit floating point number. 
; Values of qNAN, sNAN, INF and -INF are not allowed.
; Negative values are also not allowed."
pow-requirement = pow

; bloom filter is "a byte array"
bloom-filter    = *OCTET

light-node      = BIT

waku-envelope   = "[" expiry ttl topic data nonce "]"

; 4 bytes (UNIX time in seconds)
expiry          = 4*OCTET

; 4 bytes (time-to-live in seconds)
ttl             = 4*OCTET

; 4 bytes of arbitrary data
topic           = 4*OCTET

; byte array of arbitrary size 
; (contains encrypted message)
data            = OCTET

; 8 bytes of arbitrary data 
; (used for PoW calculation)
nonce           = 8*OCTET

messages        = *waku-envelope

p2p-request     = waku-envelope

p2p-message     = waku-envelope

packet-format   = "[" packet-code packet-format "]"

required-packet = 0 status / 1 messages / 2 pow-requirement / 3 bloom-filter

optional-packet = 126 p2p-request / 127 p2p-message

; packet-format needs to be paired with its corresponding packet-format
packet          = "[" required-packet / [ optional-packet ] "]"

packet-format   = "[" packet-code packet-format "]"
```

All primitive types are RLP encoded. Note that, per RLP specification, integers are encoded starting from `0x00`.

### Packet Codes

The message codes reserved for Waku protocol: 0 - 127.

Messages with unknown codes MUST be ignored without generating any error, for forward compatibility of future versions.

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

### Packet usage

**Status**

The bloom filter paramenter is optional; if it is missing or nil, the node is considered to be full node (i.e. accepts all messages).

The Status message serves as a Waku handshake and peers MUST exchange this
message upon connection. It MUST be sent after the RLPx handshake and prior to
any other Waku messages.

A Waku node MUST await the Status message from a peer before engaging in other Waku protocol activity with that peer.
When a node does not receive the Status message from a peer, before a configurable timeout, it SHOULD disconnect from that peer.

Upon retrieval of the Status message, the node SHOULD validate the message
content and decide whether it is compatible with the Waku version and mode
its peer is advertising. The handshake is completed when the node has sent,
received and validated the Status message. Note that its peer might not be in
the same state.

When a node is receiving other Waku messages from a peer before a Status
message is received, the node MUST ignore these messages and SHOULD disconnect
from that peer.
Status messages received after the handshake is completed MUST also be ignored.


**Messages**

This packet is used for sending the standard Waku envelopes.

**PoW Requirement**

This packet is used by Waku nodes for dynamic adjustment of their individual PoW requirements. Recipient of this message should no longer deliver the sender messages with PoW lower than specified in this message.

PoW is defined as average number of iterations, required to find the current BestBit (the number of leading zero bits in the hash), divided by message size and TTL:

	PoW = (2**BestBit) / (size * TTL)

PoW calculation:

	fn short_rlp(envelope) = rlp of envelope, excluding env_nonce field.
	fn pow_hash(envelope, env_nonce) = sha3(short_rlp(envelope) ++ env_nonce)
	fn pow(pow_hash, size, ttl) = 2**leading_zeros(pow_hash) / (size * ttl)

where size is the size of the RLP-encoded envelope, excluding env_nonce field (size of `short_rlp(envelope)`).

**Bloom Filter**

This packet is used by Waku nodes for sharing their interest in messages with specific topics.

The Bloom filter is used to identify a number of topics to a peer without compromising (too much) privacy over precisely what topics are of interest. Precise control over the information content (and thus efficiency of the filter) may be maintained through the addition of bits.

Blooms are formed by the bitwise OR operation on a number of bloomed topics. The bloom function takes the topic and projects them onto a 512-bit slice. At most, three bits are marked for each bloomed topic.

The projection function is defined as a mapping from a 4-byte slice S to a 512-bit slice D; for ease of explanation, S will dereference to bytes, whereas D will dereference to bits.

	LET D[*] = 0
	FOREACH i IN { 0, 1, 2 } DO
	LET n = S[i]
	IF S[3] & (2 ** i) THEN n += 256
	D[n] = 1
	END FOR

**P2P Request**

This packet is used for sending Dapp-level peer-to-peer requests, e.g. Waku Mail Client requesting old messages from the Waku Mail Server.

**P2P Message**

This packet is used for sending the peer-to-peer messages, which are not supposed to be forwarded any further. E.g. it might be used by the Waku Mail Server for delivery of old (expired) messages, which is otherwise not allowed.

### Whisper Envelope data field (Optional)

This section outlines the description of the Data Field.

It is only relevant if you want to decrypt the incoming message, but if you only want to send a message, any other format would be perfectly valid and must be forwarded to the peers.

The Data field contains the encrypted message of the envelope. In case of symmetric encryption, it also contains appended Salt (a.k.a. AES Nonce, 12 bytes). Plaintext (unencrypted) payload consists of the following concatenated fields: flags, auxiliary field, payload, padding and signature (in this sequence).

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```
; 1 byte; first two bits contain the size of auxiliary field, third bit indicates whether the signature is present.
flags           = 1*OCTET

; contains the size of payload.
auxiliary-field = 4*OCTET

; byte array of arbitrary size (may be zero)
payload         = *OCTET

; byte array of arbitrary size (may be zero).
padding         = *OCTET

; 65 bytes, if present.
signature       = 65*OCTET

; 2 bytes, if present (in case of symmetric encryption).
salt            = 2*OCTET

envelope        = flags auxiliary-field payload padding [signature] salt
```

Those unable to decrypt the message data are also unable to access the signature. The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity. The signature is serialised as the concatenation of the `R`, `S` and `V` parameters of the SECP-256k1 ECDSA signature, in that order. `R` and `S` are both big-endian encoded, fixed-width 256-bit unsigned. `V` is an 8-bit big-endian encoded, non-normalised and should be either 27 or 28.

The padding field was introduced in order to align the message size, since message size alone might reveal important metainformation. Padding can be arbitrary size. However, it is recommended that the size of Data Field (excuding the Salt) before encryption (i.e. plain text) should be factor of 256 bytes.

### Payload Encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme with SECP-256k1 public key.

Symmetric encryption uses AES GCM algorithm with random 96-bit nonce.

### Packet code Rationale

Packet codes `0x00` and `0x01` are already used in all Waku / Whisper versions.

Packet code `0x02` will be necessary for the future development of Whisper. It will provide possiblitity to adjust the PoW requirement in real time. It is better to allow the network to govern itself, rather than hardcode any specific value for minimal PoW requirement.

Packet code `0x03` will be necessary for scalability of the network. In case of too much traffic, the nodes will be able to request and receive only the messages they are interested in.

Packet codes `0x7E` and `0x7F` may be used to implement Waku Mail Server and Client. Without P2P messages it would be impossible to deliver the old messages, since they will be recognized as expired, and the peer will be disconnected for violating the Whisper protocol. They might be useful for other purposes when it is not possible to spend time on PoW, e.g. if a stock exchange will want to provide live feed about the latest trades.


## Additional capabilities

Waku supports multiple capabilities. These include light node, rate limting, mailserver (client and server) and bridging of traffic. Here we list these capabilities, how they are identified, what properties they have and what invariants they must maintain.

### Light node

The rationale for light nodes is to allow for interaction with waku on resource restricted devices as bandwidth can often be an issue.

Light nodes MUST NOT forward any incoming messages, they MUST only send their own messages. When light nodes happen to connect to each other, they SHOULD disconnect. As this would result in messages being dropped between the two.

Light nodes are identified by the `light_node` value in the status message.

### Mailserver and client

Mailservers are waku nodes that can archive messages and deliver them to its peers on-demand. A node which wants to provide mailserver functionality MUST store envelopes from incoming message packets (Waku packet-code 0x01). The envelopes can be stored in any format, however they MUST be serialized and deserialized to the Waku envelope format.

A mailserver SHOULD store envelopes for all topics to be generally useful for any peer, however for specific use cases it MAY store envelopes for a subset of topics.

#### Requesting messages

In order to request historic messages, a node MUST send a packet P2P Request (`0x7e`) to a peer providing mailserver functionality. This packet requires one argument which MUST be a Waku envelope.

In the Waku envelope's payload section, there MUST be RLP-encoded information about the details of the request:

```
[ Lower, Upper, Bloom, Limit, Cursor ]
```

`Lower`: 4-byte wide unsigned integer (UNIX time in seconds; oldest requested envelope's creation time)
`Upper`: 4-byte wide unsigned integer (UNIX time in seconds; newest requested envelope's creation time)
`Bloom`: 64-byte wide array of Waku topics encoded in a bloom filter to filter envelopes
`Limit`: 4-byte wide unsigned integer limiting the number of returned envelopes
`Cursor`: 32-byte wide array of a cursor returned from the previous request (optional)

The `Cursor` field SHOULD be filled in if a number of envelopes between `Lower` and `Upper` is greater than `Limit` so that the requester can send another request using the obtained `Cursor` value. What exactly is in the `Cursor` is up to the implementation. The requester SHOULD NOT use a `Cursor` obtained from one mailserver in a request to another mailserver because the format or the result MAY be different.

The envelope MUST be signed with a symmetric key agreed between the requester and Mailserver.

#### Receiving historic messages

Historic messages MUST be sent to a peer as a packet with a P2P Message code (`0x7f`) followed by an array of Waku envelopes. It is incompatible with the original Whisper spec (EIP-627) because it allows only a single envelope, however, an array of envelopes is much more performant. In order to stay compatible with EIP-627, a peer receiving historic message MUST handle both cases.

In order to receive historic messages from a mailserver, a node MUST trust the selected mailserver, that is allow to receive packets with the P2P Message code. By default, such packets are discarded.

Received envelopes MUST be passed through the Waku envelopes pipelines so that they are picked up by registered filters and passed to subscribers.

## Backwards Compatibility

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

## Forwards Compatibility

It is desirable to have a strategy for maintaining forward compatibility between `waku/0` and future version of waku. Here we outline some concerns and strategy for this.


## Security considerations

There are several security considerations to take into account when running Waku. Chief among them are: scalability, DDoS-resistance and privacy. These also vary depending on what capabilities are used, such as mailserver, light node, and so on.

### Light node privacy

The main privacy concern with light nodes is that directly connected peers will know that a message originates from them (as it are the only ones it sends). This means nodes can make assumptions about what messages (topics) their peers are interested in.

## Implementation Notes

### Implementation Matrix

| Client | Version |
| ------ | ------- |
| go-ethereum (geth) | [v1.9.7](https://github.com/ethereum/go-ethereum/tree/v1.9.7) |
| status whisper | [25321](https://github.com/status-im/whisper/tree/25321b2c035b6e03dbae85a2f54cf89f9f873dd9) |
| nimbus | [9c19f](https://github.com/status-im/nim-eth/tree/9c19f1e5b17b36ebcf1c7513428818f585a3cb16) |
| status-go | [ed5a5](https://github.com/status-im/status-go/commit/ed5a5c154daf5362cdf0c35fd1bc204e6a6d49ae) |

| | Light mode | Mailserver client | Mailserver server | shh/6 | waku/0 |
| -: | :--------: | :----------------: | :--------------: |  :-: | :-: |
| **geth** | x | x | x | x | - |
| **status whisper** | x | x | - | x | - |
| **nimbus** | x | - | - | x | - |
| **status-go** | x | x | x |x | - |

Notes useful for implementing Waku mode.

## Footnotes

1. <https://github.com/ethereum/devp2p/blob/master/rlpx.md>

## Changelog

| Version | Comment |
| :-----: | ------- |
| 0.1.0 (current) | Initial Release |

### Differences between shh/6 waku/0

Summary of main differences between this spec and Whisper v6, as described in [EIP-627](https://eips.ethereum.org/EIPS/eip-627):

- RLPx subprotocol is changed from `shh/6` to `waku/0`
- Light node capability
- Whisper Mail Server and Whisper Mail Client implemented

## Acknowledgements
 - Kim De Mey
 - Andrea Maria Piana
 - Adam Babik

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
