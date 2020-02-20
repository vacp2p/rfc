# Envelope data field

> Version: 0.1.0 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

## Table of Contents

 1. [Abstract](#abstract)
 2. [Specification](#specification)
    1. [ABNF](#abnf)
    2. [Signature](#signature)
    3. [Padding](#padding)
 3. [Copyright](#copyright)

## Abstract

In this specification, we describe the [data field used in Waku](waku.md#abnf-specification) for the encryption, decryption and signing of messages.

## Specification

The `data` field is used within the `waku envelope`, the field MUST contain the encrypted message of the envelope.

The fields that are concatenated and encrypted as part of the `data` field are:
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

envelope        = flags auxiliary-field payload padding [signature] [salt]
```

### Signature

Those unable to decrypt the message data are also unable to access the signature. The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity. The signature is serialised as the concatenation of the `R`, `S` and `V` parameters of the SECP-256k1 ECDSA signature, in that order. `R` and `S` MUST be big-endian encoded, fixed-width 256-bit unsigned. `V` MUST be an 8-bit big-endian encoded, non-normalised and should be either 27 or 28.

### Padding

The padding field is used to align message size, since message size alone might reveal important metainformation. Padding can be arbitrary size. However, it is recommended that the size of Data Field (excluding the Salt) before encryption (i.e. plain text) SHOULD be factor of 256 bytes.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
