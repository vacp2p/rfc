---
slug: 26
title: 26/WAKU-PAYLOAD
name: Waku Message Payload Encryption
status: draft
editor: Oskar Thoren <oskar@status.im>
contributors:
---

This specification describes how encryption, decryption and signing works in
[6/WAKU1](/spec/6) and in [10/WAKU2](/spec/10) with [14/WAKU-MESSAGE version
1](/spec/14/#version1).

It effectively replaces [7/WAKU-DATA](/spec/7) as well as [6/WAKU1 Payload
encryption](/spec/6/#payload-encryption) but written in a way that is agnostic
and self-contained for Waku v1 and Waku v2.

## Cryptographic primitives

- AES-256-GCM
- ECIES
- ECSDA
- KECCAK-256

## Payload encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme (ECIES) with SECP-256k1 public key.
In case of a signature being provided, the public key is recoverable from by utilizing the `v` parameter.

Symmetric encryption uses AES-256-GCM for authenticated encryption [TODO], with a 16 byte authentication tag 16 and a 12 byte IV (nonce).

## Specification

For 6/WAKU1, the `data` field is used within the `waku envelope`, the field MUST contain the encrypted payload of the `waku envelope`.

For 10/WAKU2, the `payload` field is used within the `WakuMessage` and MUST contain the encrypted payload of `Waku Message`.

The fields that are concatenated and encrypted as part of the field are:
 - flags
 - auxiliary field
 - payload
 - padding
 - signature
 
In case of symmetric encryption, a `salt`  (a.k.a. AES Nonce, 12 bytes) field MUST be appended. 

### ABNF

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; 1 byte; first two bits contain the size of auxiliary field, 
; third bit indicates whether the signature is present.
flags           = 1OCTET

; contains the size of payload.
auxiliary-field = 4*OCTET

; byte array of arbitrary size (may be zero)
payload         = *OCTET

; byte array of arbitrary size (may be zero).
padding         = *OCTET

; 65 bytes, if present.
signature       = 65OCTET

; 2 bytes, if present (in case of symmetric encryption).
salt            = 2OCTET

data        = flags auxiliary-field payload padding [signature] [salt]
```

### Signature

Those unable to decrypt the envelope data are also unable to access the signature. The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity. The signature is serialized as the concatenation of the `R`, `S` and `V` parameters of the SECP-256k1 ECDSA signature, in that order. `R` and `S` MUST be big-endian encoded, fixed-width 256-bit unsigned. `V` MUST be an 8-bit big-endian encoded, non-normalized and should be either 27 or 28.

### Padding

The padding field is used to align data size, since data size alone might reveal important metainformation. Padding can be arbitrary size. However, it is recommended that the size of Data Field (excluding the Salt) before encryption (i.e. plain text) SHOULD be factor of 256 bytes.

## References

- Authenticated encryption: https://en.wikipedia.org/wiki/Authenticated_encryption

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
