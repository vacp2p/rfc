---
slug: 31
title: 31/WAKU2-ENR
name: Waku v2 usage of ENR
status: raw
editor: Franck Royer <franck@status.im>
contributors:
---

# Abstract

This RFC describes the usage of the ENR (Ethereum Node Records) format for [10/WAKU2](/specs/10) purposes.
The ENR format is defined in [EIP-778](https://eips.ethereum.org/EIPS/eip-778).

This RFC is an extension of EIP-778, ENR used in Waku v2 MUST adhere to both EIP-778 and 31/WAKU2-ENR.

# Motivation

EIP-1459 with the usage of ENR has been implemented [[1]](#references) [[2]](#references) as a discovery protocol for Waku v2.

EIP-778 specifies a number of pre-defined keys.
However, the usage of these keys alone does not allow for certain transport capabilities to be encoded,
such as Websocket.
Currently, Waku v2 nodes running in a Browser only support websocket transport protocol.
Hence, new ENR keys needs to be defined to allow for the encoding of transport protocol other than raw TCP.

## Usage of multiaddr format

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

We propose the definition of the `multiaddrs` key.

- The value MUST be a list of binary encoded multiaddr prefixed by their size.
- The size of the multiaddr MUST be encoded in a Big Endian unsigned 16-bit integer.
- The size of the multiaddr MUST be encoded in 2 bytes.
- The `secp256k1` MUST be present on the record.
- The node's peer id SHOULD be deduced from the `secp256k1` value.
- The multiaddresses SHOULD NOT contain a peer id.
- TCP, UDP, IP (IPv4 and IPv6) connection details SHOULD be encoded using the respecting pre-defined ENR keys `tcp`, `udp`, `ip` (and `tcp6`, `udp6`, `ip6` for IPv6);
  Multiaddr format can be large, using the predefined keys when available allows more connection details to be encoded in one ENR.
- The 300 bytes size limit as defined by [EIP-778](https://eips.ethereum.org/EIPS/eip-778) still applies;
  In practice, an ENR MAY hold up to 3 multiaddresses, depending on the size of each multiaddress.

## Usage

Alice is a node operator, she runs a node that supports inbound connection for the following protocols:
- TCP 10101 on 1.2.3.4
- UDP 20202 on 1.2.3.4
- TCP 30303 on 1234:5600:100:1::142
- UDP 40404 on 1234:5600:100:1::142
- Secure Websocket on wss://example.com:443/
- QUIC on quic://quic.example.com:443/

Alice SHOULD structure the ENR for her node as follows:

| key    | value   |
|---     |---      |
| `tcp`  | `10101` |
| `udp`  | `20202` |
| `tcp6` | `30303` |
| `udp6` | `40404` |
| `ip`   | `1.2.3.4` |
| `ip6`  | `1234:5600:100:1::142` |
| `secp256k1` | Alice's public key |
| `multiaddrs` | <code>len1 &#124; /dns4/example.com/tcp/443/wss &#124; len2 &#124; /dns4/quic.examle.com/tcp/443/quic</cpoode> |

Where:
- `|` is the concatenation operator,
- `len1` is the length of `/dns4/example.com/tcp/443/wss` byte representation,
- `len2` is the length of `/dns4/quic.examle.com/tcp/443/quic` byte representation.

## Limitations

Supported key type is `secp256k1` only.

# References

- [1] https://github.com/status-im/nim-waku/pull/690
- [2] https://github.com/vacp2p/rfc/issues/462#issuecomment-943869940 
