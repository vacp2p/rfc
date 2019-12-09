# Envelope data field

> Version: 0.1.0 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

## Table of Contents

 - [Specification](#specification)

## Specification
This section outlines the description of the Data Field.

It is only relevant if you want to decrypt the incoming message, any other format would be perfectly valid for sending messages and must be forwarded to the peers.

The Data field MUST contain the encrypted message of the envelope. In case of symmetric encryption, it also contains appended Salt (a.k.a. AES Nonce, 12 bytes). Plaintext (unencrypted) payload consists of the following concatenated fields: flags, auxiliary field, payload, padding and signature (in this sequence).

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```
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

Those unable to decrypt the message data are also unable to access the signature. The signature, if provided, is the ECDSA signature of the Keccak-256 hash of the unencrypted data using the secret key of the originator identity. The signature is serialised as the concatenation of the `R`, `S` and `V` parameters of the SECP-256k1 ECDSA signature, in that order. `R` and `S` are MUST be big-endian encoded, fixed-width 256-bit unsigned. `V` is MUST be an 8-bit big-endian encoded, non-normalised and should be either 27 or 28.

The padding field is used to align message size, since message size alone might reveal important metainformation. Padding can be arbitrary size. However, it is recommended that the size of Data Field (excluding the Salt) before encryption (i.e. plain text) SHOULD be factor of 256 bytes.
