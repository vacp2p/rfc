---
slug: 26
title: 26/WAKU-PAYLOAD
name: Waku Message Payload Encryption
status: draft
editor: Oskar Thoren <oskar@status.im>
contributors:
---

This specification describes how Waku provides confidentiality, authenticity, and integrity, as well as some form of unlinkability.
Specifically, it describes how encryption, decryption and signing works in [6/WAKU1](/spec/6) and in [10/WAKU2](/spec/10) with [14/WAKU-MESSAGE version 1](/spec/14/#version1).

This specification effectively replaces [7/WAKU-DATA](/spec/7) as well as [6/WAKU1 Payload encryption](/spec/6/#payload-encryption) but written in a way that is agnostic and self-contained for Waku v1 and Waku v2.

Large sections of the specification originate from [EIP-627: Whisper spec](https://eips.ethereum.org/EIPS/eip-627) as well from [RLPx Transport Protocol spec (ECIES encryption)](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ecies-encryption) with some modifications.

## Design requirements

- *Confidentiality*: The adversary should not be able to learn what data is being sent from one Waku node to one or several other Waku nodes.
- *Authenticity*: The adversary should not be able to cause Waku endpoint to accept data from any third party as though it came from the other endpoint.
- *Integrity*: The adversary should not be able to cause a Waku endpoint to accept data that has been tampered with.

Notable, *forward secrecy* is not provided for at this layer.
If this property is desired, a more fully featured secure communication protocol can be used on top, such as [Status 5/SECURE-TRANSPORT](https://specs.status.im/spec/5).

It also provides some form of *unlinkability* since:
- only participants who are able to decrypt a message can see its signature
- payload are padded to a fixed length

## Cryptographic primitives

- AES-256-GCM (for symmetric encryption)
- ECIES
- ECDSA
- KECCAK-256

ECIES is using the following cryptosystem:
- Curve: secp256k1
- KDF: NIST SP 800-56 Concatenation Key Derivation Function, with SHA-256 option
- MAC: HMAC with SHA-256
- AES: AES-128-CTR

## Specification

For 6/WAKU1, the `data` field is used in the `waku envelope`, and the field MAY contain the encrypted payload.

For 10/WAKU2, the `payload` field is used in `WakuMessage` and MAY contain the encrypted payload.

The fields that are concatenated and encrypted as part of the `data`/`payload` field are:
 - flags
 - payload/data length
 - payload/data
 - padding
 - signature

### ABNF

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; 1 byte; first two bits contain the size of payload-length field,
; third bit indicates whether the signature is present.
flags           = 1OCTET

; contains the size of payload.
payload-length  = 4*OCTET

; byte array of arbitrary size (may be zero).
payload         = *OCTET

; byte array of arbitrary size (may be zero).
padding         = *OCTET

; 65 bytes, if present.
signature       = 65OCTET

data            = flags payload-length payload padding [signature]
```

### Signature

Those unable to decrypt the envelope data are also unable to access the signature.
The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity.
The signature is serialized as the concatenation of the `r`, `s` and `v` parameters of the SECP-256k1 ECDSA signature, in that order.
`r` and `s` MUST be big-endian encoded, fixed-width 256-bit unsigned.
`v` MUST be an 8-bit big-endian encoded, non-normalized and should be either 27 or 28.

### Encryption

#### Symmetric

Symmetric encryption uses AES-256-GCM for [authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption). 
The output of encryption is of the form (`ciphertext`, `tag`, `iv`) where `ciphertext` is the encrypted message, `tag` is a 16 byte message authentication tag and `iv` is a 12 byte initialization vector  (nonce). 
The message authentication `tag` and initialization vector `iv` field MUST be appended to the resulting `ciphertext`, in that order.
Note that previous specifications and some implementations might refer to `iv` as `nonce` or `salt`.

#### Asymmetric

Asymmetric encryption uses the standard Elliptic Curve Integrated Encryption Scheme (ECIES) with SECP-256k1 public key.

#### ECIES

This section originates from the [RLPx Transport Protocol spec](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ecies-encryption) spec with minor modifications.

The cryptosystem used is:

- The elliptic curve secp256k1 with generator `G`.
- `KDF(k, len)`: the NIST SP 800-56 Concatenation Key Derivation Function.
- `MAC(k, m)`: HMAC using the SHA-256 hash function.
- `AES(k, iv, m)`: the AES-128 encryption function in CTR mode.

Special notation used: `X || Y` denotes concatenation of `X` and `Y`.

Alice wants to send an encrypted message that can be decrypted by Bob's static private key `kB`. Alice knows about Bobs static public key `KB`.

To encrypt the message `m`, Alice generates a random number `r` and corresponding elliptic curve public key `R = r * G` and computes the shared secret `S = Px` where `(Px, Py) = r * KB`.
She derives key material for encryption and authentication as `kE || kM = KDF(S, 32)` as well as a random initialization vector `iv`.
Alice sends the encrypted message `R || iv || c || d` where `c = AES(kE, iv , m)` and `d = MAC(sha256(kM), iv || c)` to Bob.

For Bob to decrypt the message `R || iv || c || d`, he derives the shared secret `S = Px` where `(Px, Py) = kB * R` as well as the encryption and authentication keys `kE || kM = KDF(S, 32)`.
Bob verifies the authenticity of the message by checking whether `d == MAC(sha256(kM), iv || c)` then obtains the plaintext as `m = AES(kE, iv || c)`.

### Padding

The padding field is used to align data size, since data size alone might reveal important metainformation.
Padding can be arbitrary size.
However, it is recommended that the size of Data Field (excluding the IV and tag) before encryption (i.e. plain text) SHOULD be a multiple of 256 bytes.

### Decoding a message

In order to decode a message, a node SHOULD try to apply both symmetric and asymmetric decryption operations.
This is because the type of encryption is not included in the message.

## References

1. [6/WAKU1](/spec/6)
2. [10/WAKU2](/spec/10)
3. [14/WAKU-MESSAGE version 1](/spec/14/#version1)
4. [7/WAKU-DATA](/spec/7)
5. [6/WAKU1 Payload encryption](/spec/6/#payload-encryption)
6. [EIP-627: Whisper spec](https://eips.ethereum.org/EIPS/eip-627)
7. [RLPx Transport Protocol spec (ECIES encryption)](https://github.com/ethereum/devp2p/blob/master/rlpx.md#ecies-encryption)
8. [Status 5/SECURE-TRANSPORT](https://specs.status.im/spec/5)
9. [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234)
10. [authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
