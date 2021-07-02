---
slug: 26
title: 26/WAKU-PAYLOAD
name: Waku Message Payload Encryption
status: draft
editor: Oskar Thoren <oskar@status.im>
contributors:
---

This specification describes how Waku provides provides confidentiality, authenticity, and integrity.
Specifically, it describes how encryption, decryption and signing works in [6/WAKU1](/spec/6) and in [10/WAKU2](/spec/10) with [14/WAKU-MESSAGE version 1](/spec/14/#version1).

This specification effectively replaces [7/WAKU-DATA](/spec/7) as well as [6/WAKU1 Payload encryption](/spec/6/#payload-encryption) but written in a way that is agnostic and self-contained for Waku v1 and Waku v2.

Large sections of the specification originate from [EIP-627: Whisper spec](https://eips.ethereum.org/EIPS/eip-627) as well from [RLPx Transport Protocol spec (ECIES encryption)](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ecies-encryption) with some modifications.

## Design requirements

- *Confidentiality*: The adversary should not be able to learn what data is being exchanged between two Waku nodes.
- *Authenticity*: The adversary should not be able to cause either of two Waku endpoint to accept data from any third party as though it came from the other endpoint.
- *Integrity*: The adversary should not be able to cause either of two Waku endpoints to accept data that has been tampered with.

Notable, *forward secrecy* is not provided for at this layer.
If this property is desired, a more fully featured secure communication protocol can be used on top, such as [Status 5/SECURE-TRANSPORT](https://specs.status.im/spec/5).

## Cryptographic primitives

- AES-256-GCM
- ECIES
- ECSDA
- KECCAK-256

ECIES is using the following cryptosystem:
- Curve: secp256k1
- KDF: NIST SP 800-56 Concatenation Key Derivation Function
- MAC: HMAC with SHA-256
- AES: AES-128-CTR

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


### Payload encryption

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme (ECIES) with SECP-256k1 public key.
For more details, see the section below.
In case of a signature being provided, the public key is recoverable by utilizing the `v` parameter.

Symmetric encryption uses AES-256-GCM for [authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption), with a 16 byte authentication tag 16 and a 12 byte IV (nonce).

### ECIES Encryption

This section originates from the [RLPx Transport Protocol spec](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ecies-encryption) spec with minor modifications.

The cryptosystem used is:

- The elliptic curve secp256k1 with generator `G`.
- `KDF(k, len)`: the NIST SP 800-56 Concatenation Key Derivation Function.
- `MAC(k, m)`: HMAC using the SHA-256 hash function.
- `AES(k, iv, m)`: the AES-128 encryption function in CTR mode.

Special notation used: `X || Y` denotes concatenation of X and Y.

Alice wants to send an encrypted message that can be decrypted by Bobs static private key `kB`. Alice knows about Bobs static public key `KB`.

To encrypt the message `m`, Alice generates a random number `r` and corresponding elliptic curve public key `R = r * G` and computes the shared secret `S = Px` where `(Px, Py) = r * KB`.
She derives key material for encryption and authentication as `kE || kM = KDF(S, 32)` as well as a random initialization vector `iv`.
Alice sends the encrypted message `R || iv || c || d` where `c = AES(kE, iv , m)` and `d = MAC(sha256(kM), iv || c)` to Bob.

For Bob to decrypt the message `R || iv || c || d`, he derives the shared secret `S = Px` where `(Px, Py) = kB * R` as well as the encryption and authentication keys `kE || kM = KDF(S, 32)`.
Bob verifies the authenticity of the message by checking whether `d == MAC(sha256(kM), iv || c)` then obtains the plaintext as `m = AES(kE, iv || c)`.

## References

- Authenticated encryption: https://en.wikipedia.org/wiki/Authenticated_encryption

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
