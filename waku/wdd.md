# Waku DNS Discovery

> Version: 0.1.0 (Draft)
>
> Authors: Dean Eigenmann <dean@status.im>

## Table of Contents

1. [Abstract](#abstract)
2. [Specification](#specification)
    1. [Records](#records)
    2. [Traversing](#traversing)
3. [Footnotes](#footnotes)
4. [Copyright](#copyright)

## Abstract

In this specification, we describe the DNS discovery protocol used by [Waku](./waku.md). 

## Specification

This specification is based on the DNS discovery proposal for Ethereum<sup>1</sup>. It traverses a merkle tree that is stored in the DNS records of a chosen domain.

### Records

The node lists are encoded as merkle trees and stored within DNS TXT records. The root of the tree is a `TXT` record:

```
tree-root:v1 r=<root> l=<link-root> seq=<sequence-number> sig=<signature>
```

- `root`: the root hash of the subtree.
- `link-root`:  the root hash of subtrees containing links for different subtrees.
- `sequence-number`: the update sequence number, as an integer.
- `signature`: a 65-byte secp256k1 EV signature of the `sha256` hash of `root`, `link-root`, and `sequence-number` concatenated.

The next `TXT` records on subdomains map hashes to one of three entry types. The subdomain name is the `sha256` hash of its text content abbreviated to 26 characters.

- `tree-branch:<h₁>,<h₂>,...` - A tree entry containing hashes of subtree entries.
- `tree://<key>@<fqdn>` - A leaf pointing to a different list located at another DNS name. Note that this format matches URL encoding. This type of entry MUST only appear in the subtree pointed to by `link-root`.
- `r:<multiaddr>` - A leaf containing a [`multiaddr`](https://github.com/multiformats/multiaddr). This type of entry MUST only appear in the subtree pointed to by `root`.

Whenever the tree is updated, the `sequence number` MUST be increased. The content of any TXT record MUST be small enough to fit into the 512 byte limited imposed on UDP DNS packets.

```
; name                        ttl     class type  content
@                             60      IN    TXT   tree-root:v1 r=JWXYDBPXYWG6FX3GMDIBFA6CJ4 l=C7HRFPF3BLGF3YR4DY5KX3SMBE seq=1 sig=o908WmNp7LibOfPsr4btQwatZJ5URBr2ZAuxvK4UWHlsB9sUOTJQaGAlLPVAhM__XJesCHxLISo94z5Z2a463gA
C7HRFPF3BLGF3YR4DY5KX3SMBE    86900   IN    TXT   tree://<hash>@<addr>
JWXYDBPXYWG6FX3GMDIBFA6CJ4    86900   IN    TXT   tree-branch:2XS2367YHAXJFGLZHVAWLQD4ZY,H4FHT4B454P6UXFD7JCYQ5PWDY,MHTDO6TMUBRIA2XWG5LUDACK24
2XS2367YHAXJFGLZHVAWLQD4ZY    86900   IN    TXT   r:<multiaddr>
H4FHT4B454P6UXFD7JCYQ5PWDY    86900   IN    TXT   r:<multiaddr>
MHTDO6TMUBRIA2XWG5LUDACK24    86900   IN    TXT   r:<multiaddr>

```

### Traversing

To find nodes, the records of a DNS name must be traversed. We will use the example zone file found in this specification for this example.

This is done by initially reading the root `TXT` record of the domain and checking that it contains a valid `tree-root:v1` entry. 

If the root like in the above example contains `JWXYDBPXYWG6FX3GMDIBFA6CJ4` we MUST resolve the `TXT` record for `JWXYDBPXYWG6FX3GMDIBFA6CJ4` of the domain, and then it MUST be verified that the content matches the hash.

If the record contains a `tree-branch` entry, then we MUST parse the list of hashes and continue resolving as seen in the above step.

If the record contains an `r` record, we MUST decode the record, verify the node record and SHOULD import it to our node list.

It SHOULD be avoided to download the entire tree at once during normal operation. The tree entries SHOULD be requested when they are needed instead.

## Footnotes
1. <https://eips.ethereum.org/EIPS/eip-1459>

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).