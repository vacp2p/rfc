---
slug: 35
title: 35/WAKU2-NOISE
name: Noise Protocols for Waku2 Payload Encryption
status: raw
editor: Giuseppe <giuseppe@status.im>
contributors: 
---

This specification describes how Waku2 messages can be encrypted
in order to achieve confidentiality, authenticity, and integrity 
as well as some form of identity-hiding on communicating parties.

Specifically, it describes how encryption keys can be exchanged using [Noise protocol](http://www.noiseprotocol.org/noise.html) 
to allow authenticated encryption/decryption in [10/WAKU2](/spec/10) with [14/WAKU-MESSAGE version 2](/spec/14/#version2). 
This ultimately allows Waku2 users to instantiate end-to-end encrypted communication channels.


This specification is an addition to [26/WAKU-PAYLOAD](/spec/26).

## Design requirements

- *Confidentiality*: 
The adversary should not be able to learn what data is being sent from one Waku2 endpoint to one or several other Waku2 endpoints. 
In particular, forward secrecy should be provided on exchanged data. 
- *Authenticity*: The adversary should not be able to cause a Waku2 endpoint to accept messages coming from an endpoint different than their original senders.
- *Integrity*: The adversary should not be able to cause a Waku2 endpoint to accept data that has been tampered with.
- *Identity-hiding*: A passive adversary should not be able to link encrypted messages to their corresponding senders and recipients.


## Supported Cryptographic Protocols

### Noise Protocols

A Noise protocol consists of one or more Noise handshakes. 
During a Noise handshake, two parties exchange multiple **handshake messages**. 
A handshake message contains *ephemeral keys* and/or *static keys* from one of the parties. 
These public keys are used to perform a protocol-dependent sequence of Diffie-Hellman operations, 
whose results are all hashed into a shared secret key. 
After a handshake is complete, each party will then use the derived shared secret key to send and receive authenticated encrypted **transport messages**. 
We refer to [Noise protocol framework specifications](http://www.noiseprotocol.org/noise.html#processing-rules) for the full details on how parties shared secret key is derived from each exchanged message.

Four Noise handshakes are currently supported: `K1K1`, `XK1`, `XX`, `XXpsk0`
(their description can be found [here](https://forum.vac.dev/t/noise-handshakes-as-key-exchange-mechanism-for-waku2/130)). 
These are instantiated combining the following cryptographic primitives:
- [`Curve25519`](http://www.noiseprotocol.org/noise.html#the-25519-dh-functions) for Diffie-Hellman key-exchanges (32 bytes coordinates);
- [`ChaChaPoly`](http://www.noiseprotocol.org/noise.html#the-chachapoly-cipher-functions) for symmetric authenticated encryption (16 bytes block size);
- [`SHA256`](http://www.noiseprotocol.org/noise.html#the-sha256-hash-function) used in [`HMAC`](http://www.noiseprotocol.org/noise.html#hash-functions) and [`HKDF`](http://www.noiseprotocol.org/noise.html#hash-functions) keys derivation chains (32 bytes output size);

We note that all [design requirements](#Design-requirements) on exchanged messages would be satisfied only *after* a supported Noise handshake is completed, 
corresponding to a total of 1 Round Trip Time communication *(1-RTT)*.

In the following, we assume that communicating parties reciprocally know an initial [`contentTopic`](https://rfc.vac.dev/spec/14/#wakumessage) 
where they can send/receive their first handshake messages. 

### Encryption Primitives

Symmetric primitives supported are:
- [`ChaChaPoly`](https://www.ietf.org/rfc/rfc7539.txt) for authenticated encryption (16 bytes block size).

## Specification

When [14/WAKU-MESSAGE](https://rfc.vac.dev/spec/14/#payload-encryption) version is set to 2, 
the corresponding (https://rfc.vac.dev/spec/10) `payload` field used in a `WakuMessage` contains either:
- a Noise handhshake message and/or a Noise transport message;
- a `ChaChaPoly` ciphertext.

The fields that are concatenated to form the `payload` field are:

 - `protocol-id`: identifies the protocol or primitive in use (1 byte). 
 Supported values are:
     - `0`:  protocol specification omitted (set for [after-handshake](#After-handshake) messages);
     - `10`: Noise protocol `Noise_K1K1_25519_ChaChaPoly_SHA256`;
     - `11`: Noise protocol `Noise_XK1_25519_ChaChaPoly_SHA256`;
     - `12`: Noise protocol `Noise_XX_25519_ChaChaPoly_SHA256`;
     - `13`: Noise protocol `Noise_XXpsk0_25519_ChaChaPoly_SHA256`;
     - `30`: `ChaChaPoly` symmetric encryption.
 - `handshake-message-len`: the length in bytes of the Noise handshake message  (1 byte). 
 If `protocol-id` is not equal to `0`, `10`, `11`, `12`, `13`, this field MUST be set to `0`;
 - `handshake-message`: the Noise handshake message (`handshake-message-len` bytes). 
If `handshake-message-len` is not `0`, 
it contains the concatenation of one or more Noise Diffie-Hellman ephemeral or static keys 
encoded as in [Public Keys Encoding](#Public-Keys-Encoding);
 - `transport-message-len-len`: the length in bytes of `transport-message-len` (1 byte);
 - `transport-message-len`: the length in bytes of `transport-message` (`transport-message-len-len` bytes);
 - `transport-message`: the transport message (`transport-message-len` bytes);
 - `transport-message-ad`: the symmetric encryption authentication data for `transport-message` (16 bytes).


### ABNF

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; protocol ID
protocol-id                 = 1OCTET

; contains the size of handshake-message
handshake-message-len       = 1OCTET

; containes one or more Diffie-Hellman public keys
handshake-message           = *OCTET

; contains the size of message-len
transport-message-len-len   = 1OCTET

; contains the size of transport-message
transport-message-len       = *OCTET

; contains the transport message, eventually encrypted
transport-message           = *OCTET

; contains authentication data for transport-message, if encrypted
transport-message-ad        = 16OCTET

; the Waku2 WakuMessage payload field
payload  = protocol-id handshake-message-len handshake-message transport-message-len-len transport-message-len transport-message transport-message-ad 
```

### Protocol Payload Format

Based on the specified `protocol-id`, 
the Waku2 message payload will encode different types of protocol-dependent messages. 

In particular, if `protocol-id` is
- `0`:  payload encodes an [after-handshake](#After-handshake) message.  
    - `handshake-message-len` MAY be 0;
    -  `transport-message` contains the Noise transport message;
- `10`,`11`,`12`,`13`: payload encodes a supported Noise handshake message.  
    -  `transport-message` contains the Noise transport message;
- `30`: payload encapsulate a `ChaChaPoly` ciphertext `ct`.  
    - `handshake-message-len` is set to `0`;
    - `transport-message` contains the concatenation of the encryption 12 bytes nonce `n` followed by the ciphertext `ct`;
    - `transport-message-len-len` and `transport-message-len` are set accordingly to `transport-message` length;
    - `transport-message-ad` contains the authentication data for `ct`.


### Public Keys Serialization

Diffie-Hellman public keys can be trasmitted in clear 
or in encrypted form with authentication data attached. 
To distinguish between these two cases, 
public keys are serialized as the concatenation of the following three fields:

- `flag`: 
is equal to `1` if the public key is encrypted; 
`0` otherwise (1 byte);
- `pk`: 
if `flag = 0`, it contains an encoding of the coordinates of the public key. 
If `flag = 1`, it contains a symmetric encryption of an encoding of the coordinates of the public key;
- `pk_ad`: 
if `flag = 0`, it is empty; 
if `flag = 1`, it contains the authentication data for `pk`;


As regards the underlying supported [cryptographic primitives](#Cryptographic-primitives):
- `Curve25519` public keys coordinates are encoded in little-endian as 32 bytes arrays;
- `ChaChaPoly` authentication data consists of 16 bytes 
(nonces are implicitely defined by Noise [processing rules](http://www.noiseprotocol.org/noise.html#processing-rules)).

In supported Noise protocols, 
parties' static keys are always transmitted encrypted, 
while ephemeral keys MAY be encrypted after an handshake is complete.


### Padding

To prevent some metadata leakage, 
encrypted transport messages SHOULD be padded before encryption 
to a multiple of the underlying symmetric cipher block size 
(16 bytes for `ChaChaPoly`).

It is therefore recommended to pad transport messages to a multiple of 256 bytes.



## After-handshake

We note that 
during the initial 1-RTT communication, 
handshake messages can be linked to the respective parties 
through the `contentTopic` employed for such communication. 

After a handshake is completed, 
parties SHOULD derive from their shared secret key (preferably using `HKDF`) a new random `contentTopic` 
and continue their communication there.

When communicating on the new `contentTopic`, 
parties SHOULD set `protocol-id` to `0` 
to reduce metadata leakages.

It is recommended that each party attaches an (unencrypted) ephemeral key in `handshake-message` to every message sent.
According to [Noise processing rules](http://www.noiseprotocol.org/noise.html#processing-rules), 
this allows updates to the shared secret key 
by hashing the result of an ephemeral-ephemeral Diffie-Hellman exchange every 1-RTT communication.


## Backward Support for Symmetric/Asymmetric Encryption

It is possible to have backward compatibility to symmetric/asymmetric encryption primitives from [26/WAKU-PAYLOAD](/spec/26), 
effectively encapsulating  payload encryption [14/WAKU-MESSAGE version 1](/spec/14/#version1) in current version 2.

It suffices to extend the list of supported `protocol-id` to:
- `254`: AES-256-GCM symmetric encryption;
- `255`: ECIES asymmetric encryption.

and encode/decode the Waku2 message `payload` field accordingly, whenever these values are set. 

Namely, if `protocol-id = 254, 255` then:
- `handshake-message-len`: is set to `0`;
- `handshake-message`: is left empty;
- `transport-message`: contains the [26/WAKU-PAYLOAD](/spec/26) `data` field;
- `transport-message-ad`: is set to `0`.

## References

1. [10/WAKU2](/spec/10)
2. [26/WAKU-PAYLOAD](/spec/26)
3. [14/WAKU-MESSAGE](/spec/14/#version1)
4. [Noise protocol](http://www.noiseprotocol.org/noise.html)
5. [Noise handshakes as key-exchange mechanism for Waku2](https://forum.vac.dev/t/noise-handshakes-as-key-exchange-mechanism-for-waku2/130)
6. [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234)


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
