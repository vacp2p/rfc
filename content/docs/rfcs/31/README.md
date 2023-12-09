---
slug: 31
title: 31/WAKU2-ENR
name: Waku v2 usage of ENR
status: raw
tags: waku-core-protocol
editor: Franck Royer <franck@status.im>
contributors:
---

# Abstract

This RFC describes the usage of the ENR (Ethereum Node Records) format for [10/WAKU2](/specs/10) purposes.
The ENR format is defined in [EIP-778](https://eips.ethereum.org/EIPS/eip-778) [[3]](#references).

This RFC is an extension of EIP-778, ENR used in Waku v2 MUST adhere to both EIP-778 and 31/WAKU2-ENR.

# Motivation

EIP-1459 with the usage of ENR has been implemented [[1]](#references) [[2]](#references) as a discovery protocol for Waku v2.

EIP-778 specifies a number of pre-defined keys.
However, the usage of these keys alone does not allow for certain transport capabilities to be encoded,
such as Websocket.
Currently, Waku v2 nodes running in a Browser only support websocket transport protocol.
Hence, new ENR keys need to be defined to allow for the encoding of transport protocol other than raw TCP.

## Usage of Multiaddr Format Rationale

One solution would be to define new keys such as `ws` to encode the websocket port of a node.
However, we expect new transport protocols to be added overtime such as quic.
Hence, this would only provide a short term solution until another RFC would need to be added.

Moreover, secure websocket involves SSL certificates.
SSL certificates are only valid for a given domain and ip, so an ENR containing the following information:
- secure websocket port
- ipv4 fqdn
- ipv4 address
- ipv6 address

Would carry some ambiguity: Is the certificate securing the websocket port valid for the ipv4 fqdn?
the ipv4 address?
the ipv6 address?

The [10/WAKU2](/specs/10) protocol family is built on the [libp2p](https://github.com/libp2p/specs) protocol stack.
Hence, it uses [multiaddr](https://github.com/multiformats/multiaddr) to format network addresses.

Directly storing one or several multiaddresses in the ENR would fix the issues listed above:
- multiaddr is self-describing and support addresses for any network protocol:
  No new RFC would be needed to support encoding other transport protocols in an ENR.
- multiaddr contains both the host and port information, allowing the ambiguity previously described to be resolved.

# `multiaddrs` ENR key

We define a  `multiaddrs` key.

- The value MUST be a list of binary encoded multiaddr prefixed by their size.
- The size of the multiaddr MUST be encoded in a Big Endian unsigned 16-bit integer.
- The size of the multiaddr MUST be encoded in 2 bytes.
- The `secp256k1` value MUST be present on the record;
  `secp256k1` is defined in [EIP-778](https://eips.ethereum.org/EIPS/eip-778) and contains the compressed secp256k1 public key.
- The node's peer id SHOULD be deduced from the `secp256k1` value.
- The multiaddresses SHOULD NOT contain a peer id.
- For raw TCP & UDP connections details, [EIP-778](https://eips.ethereum.org/EIPS/eip-778) pre-defined keys SHOULD be used;
  The keys `tcp`, `udp`, `ip` (and `tcp6`, `udp6`, `ip6` for IPv6) are enough to convey all necessary information;
- To save space, `multiaddrs` key SHOULD only be used for connection details that cannot be represented using the [EIP-778](https://eips.ethereum.org/EIPS/eip-778) pre-defined keys.
- The 300 bytes size limit as defined by [EIP-778](https://eips.ethereum.org/EIPS/eip-778) still applies;
  In practice, it is possible to encode 3 multiaddresses in ENR, more or less could be encoded depending on the size of each multiaddress.

## Usage

### Many connection types

Alice is a node operator, she runs a node that supports inbound connection for the following protocols:
- TCP 10101 on `1.2.3.4`
- UDP 20202 on `1.2.3.4`
- TCP 30303 on `1234:5600:101:1::142`
- UDP 40404 on `1234:5600:101:1::142`
- Secure Websocket on `wss://example.com:443/`
- QUIC on `quic://quic.example.com:443/`

Alice SHOULD structure the ENR for her node as follows:

| key    | value   |
|---     |---      |
| `tcp`  | `10101` |
| `udp`  | `20202` |
| `tcp6` | `30303` |
| `udp6` | `40404` |
| `ip`   | `1.2.3.4` |
| `ip6`  | `1234:5600:101:1::142` |
| `secp256k1` | Alice's compressed secp256k1 public key, 33 bytes |
| `multiaddrs` | <code>len1 &#124; /dns4/example.com/tcp/443/wss &#124; len2 &#124; /dns4/quic.examle.com/tcp/443/quic</cpoode> |

Where:
- `|` is the concatenation operator,
- `len1` is the length of `/dns4/example.com/tcp/443/wss` byte representation,
- `len2` is the length of `/dns4/quic.examle.com/tcp/443/quic` byte representation.

### Raw TCP only

Bob is a node operator, he runs a node that supports inbound connection for the following protocols:
- TCP 10101 on `1.2.3.4`

Bob SHOULD structure the ENR for her node as follows:

| key    | value   |
|---     |---      |
| `tcp`  | `10101` |
| `ip`   | `1.2.3.4` |
| `secp256k1` | Bob's compressed secp256k1 public key, 33 bytes |

Indeed, as Bob's node's connection details can be represented with EIP-778's pre-defined keys only
then it is not needed to use the `multiaddrs` key.

## Limitations

Supported key type is `secp256k1` only.

In the future, an extension of this RFC could be made to support other elliptic curve cryptography such as `ed25519`.

# `waku2` ENR key

We define a `waku2` field key:

- The value MUST be an 8-bit flag field,
where bits set to `1` indicate `true` and bits set to `0` indicate `false` for the relevant flags.
- The flag values already defined are set out below,
with `bit 7` the most significant bit and `bit 0` the least significant bit.

| bit 7 | bit 6 | bit 5 | bit 4 | bit 3 | bit 2 | bit 1 | bit 0 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `undef` | `undef` | `undef` | `undef` | `lightpush` | `filter` | `store` | `relay` |

- In the scheme above, the flags `lightpush`, `filter`, `store` and `relay` correlates with support for protocols with the same name.
If a protocol is not supported, the corresponding field MUST be set to `false`.
Indicating positive support for any specific protocol is OPTIONAL,
though it MAY be required by the relevant application or discovery process.
- Flags marked as `undef` is not yet defined.
These SHOULD be set to `false` by default.

## Usage

- A Waku v2 node MAY choose to populate the `waku2` field for enhanced discovery capabilities,
such as indicating supported protocols.
Such a node MAY indicate support for any specific protocol by setting the corresponding flag to `true`.
- Waku v2 nodes that want to participate in [Node Discovery Protocol v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) [[4]](#references), however,
MUST implement the `waku2` key with at least one flag set to `true`.
- Waku v2 nodes that discovered other participants using Discovery v5,
MUST filter out participant records that do not implement this field or do not have at least one flag set to `true`.
- In addition, such nodes MAY choose to filter participants on specific flags (such as supported protocols),
or further interpret the `waku2` field as required by the application.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- [1] https://github.com/status-im/nim-waku/pull/690
- [2] https://github.com/vacp2p/rfc/issues/462#issuecomment-943869940 
- [3] https://eips.ethereum.org/EIPS/eip-778
- [4] https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md
