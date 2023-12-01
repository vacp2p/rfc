---
slug: 65
title: 65/STATUS-ACCOUNTS
name: Status Accounts
status: draft
category: Standards Track
description: Details of what a Status account is, how accounts are created, and how accounts are used.
editor: Jimmy Debe <jimmy@status.im>
contributors:
- Corey Petty <corey@status.im>
- Oskar Thorén <oskar@status.im>
- Samuel Hawksby-Robinson <samuel@status.im>
- Aaryamann Challani <aaryamann@status.im>
---

# Abstract

This specification explains what a Status account is, and how it is created and used.

# Background

The core concept of an account in Status is a set of cryptographic keypairs. Namely, the combination of the following:
1. a Waku chat identity keypair
1. a set of cryptocurrency wallet keypairs

The Status node verifies or derives everything else associated with the contact from the above items, including:
- Ethereum address (future verification, currently the same base keypair)
- identicon
- message signatures

# Initial Key Generation
## Public/Private Keypairs 
- An ECDSA (secp256k1 curve) public/private keypair MUST be generated via a [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) derived path from a [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) mnemonic seed phrase.
- The default paths are defined as such:
    - Waku Chat Key (`IK`): `m/43'/60'/1581'/0'/0`  (post Multiaccount integration)
        - following [EIP1581](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1581.md)
    - Status Wallet paths: `m/44'/60'/0'/0/i` starting at `i=0`
        - following [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
        - NOTE: this (`i=0`) is also the current (and only) path for Waku key before Multiaccount integration

# Account Broadcasting
- A user is responsible for broadcasting certain information publicly so that others may contact them.

## X3DH Prekey bundles
- Refer to [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/) for details on the X3DH prekey bundle broadcasting, as well as regeneration.

# Optional Account additions

## ENS Username
- A user MAY register a public username on the Ethereum Name System (ENS).  This username is a user-chosen subdomain of the `stateofus.eth` ENS registration that maps to their Waku identity key (`IK`). 

## User Profile Picture
- An account MAY edit the `IK` generated identicon with a chosen picture.  This picture will become part of the publicly broadcasted profile of the account.

<!-- TODO: Elaborate on wallet account and multiaccount -->

# Wire Format

Below is the wire format for the account information that is broadcasted publicly.
An Account is referred to as a Multiaccount in the wire format.

```proto
message MultiAccount {
  string name = 1; // name of the account
  int64 timestamp = 2; // timestamp of the message
  string identicon = 3; // base64 encoded identicon
  repeated ColorHash color_hash = 4; // color hash of the identicon
  int64 color_id = 5; // color id of the identicon
  string keycard_pairing = 6; // keycard pairing code
  string key_uid = 7; // unique identifier of the account
  repeated IdentityImage images = 8; // images associated with the account
  string customization_color = 9; // color of the identicon
  uint64 customization_color_clock = 10; // clock of the identicon color, to track updates

  message ColorHash {
    repeated int64 index = 1;
  }

  message IdentityImage {
    string key_uid = 1; // unique identifier of the image
    string name = 2; // name of the image
    bytes payload = 3; // payload of the image
    int64 width = 4; // width of the image
    int64 height = 5; // height of the image
    int64 filesize = 6; // filesize of the image
    int64 resize_target = 7; // resize target of the image
    uint64 clock = 8; // clock of the image, to track updates
  }
}
```

The above payload is broadcasted when 2 devices that belong to a user need to be paired.

# Security Considerations

- This specification inherits security considerations of [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/) and [54/WAKU2-X3DH-SESSIONS](https://rfc.vac.dev/spec/54/).

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## normative

- [53/WAKU2-X3DH](https://rfc.vac.dev/spec/53/)
- [54/WAKU2-X3DH-SESSIONS](https://rfc.vac.dev/spec/54/)
- [55/STATUS-1TO1-CHAT](https://rfc.vac.dev/spec/55/)

## informative

- [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki)
- [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
- [EIP1581](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1581.md)
- [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
- [Ethereum Name System](https://ens.domains/)
- [Status Multiaccount](/spec/63)
