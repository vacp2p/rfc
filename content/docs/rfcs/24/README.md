---
slug: 24
title: 24/WAKU2-DNS-DISCOVERY
name: Waku v2 Peer Discovery via DNS
status: raw
editor: Hanno Cornelius <hanno@status.im>
contributors:
---

`24/WAKU2-DNS-DISCOVERY` specifies a scheme to implement [`libp2p`](https://libp2p.io/) peer discovery via DNS for Waku v2.
The generalised purpose is to retrieve an arbitrarily long, authenticated, updateable list of [`libp2p` peers](https://docs.libp2p.io/concepts/peer-id/) to bootstrap connection to a `libp2p` network.
Since [`10/WAKU2`](https://rfc.vac.dev/spec/10/) currently specifies use of [`libp2p` peer identities](https://docs.libp2p.io/concepts/peer-id/),
this method is suitable for a new Waku v2 node to discover other Waku v2 nodes to connect to.

This specification is largely based on [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459),
with the only deviation being the type of address being encoded (`multiaddr` vs `enr`).
Also see [this earlier explainer](https://vac.dev/dns-based-discovery) for more background on the suitability of DNS based discovery for Waku v2.

# List encoding

The peer list MUST be encoded as a [Merkle tree](https://www.wikiwand.com/en/Merkle_tree).
EIP-1459 specifies [the URL scheme](https://eips.ethereum.org/EIPS/eip-1459#specification) to refer to such a DNS node list.
This specification uses the same approach, but with a `matree` scheme:

```
matree://<key>@<fqdn>
```

where
- `matree` is the selected `multiaddr` Merkle tree scheme
- `<fqdn>` is the fully qualified domain name on which the list can be found
- `<key>` is the base32 encoding of the compressed 32-byte binary public key that signed the list.

The example URL from EIP-1459, adapted to the above scheme becomes:

```
matree://AM5FCQLWIZX2QFPNJAP7VUERCCRNGRHWZG3YYHIUV7BVDQ5FDPRT2@peers.example.org
```

Each entry within the Merkle tree MUST be contained within a [DNS TXT record](https://www.rfc-editor.org/rfc/rfc1035.txt)
and stored in a subdomain (except for the base URL `matree` entry).
The content of any TXT record MUST be small enough to fit into the 512-byte limit imposed on UDP DNS packets,
which limits the number of hashes that can be contained within a branch entry.
The subdomain name for each entry is the base32 encoding of the abbreviated keccak256 hash of its text content.
See [this example](https://eips.ethereum.org/EIPS/eip-1459#dns-record-structure) of a fully populated tree for more information.

# Entry types

The following entry types are derived from [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459)
and adapted for use with `multiaddrs`:

## Root entry

The tree root entry MUST use the following format:

```
matree-root:v1 m=<ma-root> l=<link-root> seq=<sequence number> sig=<signature>
```

where
- `ma-root` and `link-root` refer to the root hashes of subtrees
containing `multiaddrs` and links to other subtrees, respectively
- `sequence-number` is the tree's update sequence number.
This number SHOULD increase with each update to the tree.
- `signature` is a 65-byte secp256k1 EC signature
over the keccak256 hash of the root record content,
excluding the `sig=` part,
encoded as URL-safe base64

## Branch entry

Branch entries MUST take the format:

```
matree-branch:<h₁>,<h₂>,...,<hₙ>
```

where
- `<h₁>,<h₂>,...,<hₙ>` are the hashes of other subtree entries

## Leaf entries

There are two types of leaf entries:

### Link entries

For the subtree pointed to by `link-root`,
leaf entries MUST take the format:

```
matree://<key>@<fqdn>
```

which links to a different list located in another domain.

### `multiaddr` entries

For the subtree pointed to by `ma-root`,
leaf entries MUST take the format:

```
ma:<multiaddr>
```

which contains the `multiaddr` of a `libp2p` peer.

# Client protocol

A client MUST adhere to the [client protocol](https://eips.ethereum.org/EIPS/eip-1459#client-protocol) as specified in EIP-1459,
and adapted for usage with `multiaddr` entry types below:

To find nodes at a given DNS name a client MUST perform the following steps:
1. Resolve the TXT record of the DNS name and check whether it contains a valid `matree-root:v1` entry.
2. Verify the signature on the root against the known public key
and check whether the sequence number is larger than or equal to any previous number seen for that name.
3. Resolve the TXT record of a hash subdomain indicated in the record
and verify that the content matches the hash.
4. If the resolved entry is of type:
	- `matree-branch`: parse the list of hashes and continue resolving them (step 3).
	- `ma`: import the `multiaddr` and add it to a local list of discovered nodes.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [`10/WAKU2`](https://rfc.vac.dev/spec/10/)
1. [EIP-1459: Client Protocol](https://eips.ethereum.org/EIPS/eip-1459#client-protocol)
1. [EIP-1459: Node Discovery via DNS ](https://eips.ethereum.org/EIPS/eip-1459)
1. [`libp2p`](https://libp2p.io/)
1. [`libp2p` peer identity](https://docs.libp2p.io/concepts/peer-id/)
1. [Merkle trees](https://www.wikiwand.com/en/Merkle_tree)
