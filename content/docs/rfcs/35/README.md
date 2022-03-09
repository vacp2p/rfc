---
slug: 35
title: 35/WAKU2-NOISE
name: Noise Protocols for Waku2 Payload Encryption
status: raw
editor: Giuseppe <giuseppe@status.im>
contributors: 
---

This specification describes how Waku2 messages can encrypted in order to achieve confidentiality, authenticity, and integrity on exchanged messages as well as some form of identity-hiding on communicating parties.

Specifically, it describes how encryption keys can be exchanged using [Noise protocol](http://www.noiseprotocol.org/noise.html) to allow authenticated encryption/decryption in [10/WAKU2](/spec/10) with [14/WAKU-MESSAGE version 2](/spec/14/#version2). This ultimately allows Waku2 users to instantiate end-to-end encrypted communication channels.


This specification effectively replaces [26/WAKU-PAYLOAD](/spec/26).

## Design requirements

- *Confidentiality*: The adversary should not be able to learn what data is being sent from one Waku2 endpoint to one or several other Waku2 endpoints. In particular, forward secrecy should be provided on exchanged data. 
- *Authenticity*: The adversary should not be able to cause a Waku2 endpoint to accept messages coming from an endpoint different than their original senders.
- *Integrity*: The adversary should not be able to cause a Waku2 endpoint to accept data that has been tampered with.
- *Identity-hiding*: A passive adversary should not be able to link encrypted messages to their corresponding senders and recipients.


## Supported Noise Protocols

During a Noise handshake, two parties exchange multiple **handshake messages**. An handshake message contains *ephemeral keys* and/or *static keys* from one of the parties. These public keys are used to perform an handshake-dependent sequence of Diffie-Hellman operations, whose results are hashed into a shared secret key. After an handshake is complete, each party will then use the derived shared secret key to send and receive authenticated encrypted **transport messages**. We refer to [Noise specifications](http://www.noiseprotocol.org/noise.html#processing-rules) for the full details on how parties shared secret key is derived from each exchanged message.

Four Noise handshake patterns are currently supported: `K1K1`, `XK1`, `XX`, `XXpsk0`. Their descriptions can be found [here](https://forum.vac.dev/t/noise-handshakes-as-key-exchange-mechanism-for-waku2/130). 

We note that all design requirements on exchanged messages would be satisfied only *after* a supported handshake is completed, all corresponding to 1 Round Trip Time communication *(1-RTT)*.

In the following, we assume that communicating parties reciprocally know an initial [`contentTopic`](https://rfc.vac.dev/spec/14/#wakumessage) where they can send/receive first handshake messages. 

## Cryptographic primitives

Noise protocols are instantiated with the following underlying cryptographic primitives:
- `Curve25519` for Diffie-Hellman key-exchanges;
- `ChaChaPoly` for symmetric authenticated encryption;
- `SHA256` used in `HMAC` and `HKDF` keys derivation chains.


## Specification

For 10/WAKU2, the `payload` field is used in `WakuMessage` and MAY contain a Noise handhshake message and a Noise transport message.

The fields that are concatenated as part of the `payload` Waku2 field are:

 - `protocol-id`: identifies the protocol in use (1 byte). 
 Supported values are:
     - `0`: none or omitted;
     - `10`: Noise `K1K1` handshake;
     - `11`: Noise `XK1` handshake;
     - `12`: Noise `XX` handshake;
     - `13`: Noise `XXpsk0` handshake;
 - `handshake-length`: the length in bytes of the Noise handshake message (1 byte);
 - `handshake-message`: the Noise handshake message. It contains the concatenation of one or more Diffie-Hellman public keys `pk` (ephemeral or static).
A public key `pk` is encoded as `(flag || pk_serialized || pk_ad)`, where:
     - `flag`: is equal to `1` if the `pk_serialized` field is encrypted; is equal to `0` if `pk_serialized` is not encrypted (1 byte);
     - `pk_encoding`: a `Curve25519` serialization of the public key. If `flag = 1`, this field is encrypted (56 bytes);
     - `pk_ad`: if `flag = 1`, it contains the authentication data for the encrypted field `pk_encoding` (16 bytes). It is empty otherwise (0 bytes).
 - `message-len-size`: the number of bytes required to represent the Noise transport message length in bytes (1 byte);
 - `message-length`: the length in bytes of the Noise transport message (`message-len-size` bytes);
 - `transport-message`: the Noise transport message;
 - `message-ad`: the authentication data for the Noise transport message. If `protocol-id = 0`, this field is set to `0` (16 bytes).


### ABNF

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; protocol ID
protocol-id           = 1OCTET

; contains the size of handshake-message
handshake-length      = 1OCTET

; containes one or more Diffie-Hellman public keys
handshake-message     = *OCTET

; contains the size of message-length
message-len-length   = 1OCTET

; contains the size of transport-message
message-length        = *OCTET

; contains the transport message, eventually encrypted
transport-message     = *OCTET

; contains authentication data for transport-message, if encrypted
message-ad           = 16OCTET


payload               = protocol-id handshake-length handshake-message message-len-lenght message-length transport-message message-ad 
```

### Padding

To prevent some metadata leakage, encrypted transport messages SHOULD be padded before encryption to a multiple of the underlying symmetric cipher block size (16 bytes for `ChaChaPoly`).  

It is therefore recommended to pad transport messages to a multiple of 256 bytes.

## After-handshake

We note that during the initial 1-RTT, handshake messages can be linked to the respective parties through the employed `contentTopic`. 

After an handshake is completed, parties SHOULD derive from their shared secret key a new random `contentTopic` and continue their communication there.

When communicating on the new `contentTopic`, parties SHOULD set `protocol-id` to `0` to reduce metadata leakages.

It is recommended that each party attaches an unencrypted new ephemeral key in `handshake-message` in every sent message.  According to [Noise processing rules](http://www.noiseprotocol.org/noise.html#processing-rules), this allows every 1 round trip time random updates to the shared secret key by hashing the result of an ephemeral-ephemeral Diffie-Hellman exchange.


## Backward Support for Symmetric/Asymmetric Encryption

It is possible to have backward compatibility to symmetric/asymmetric encryption primitives from [26/WAKU-PAYLOAD](/spec/26), effectively encapsulating  payload encryption [14/WAKU-MESSAGE version 1](/spec/14/#version1) in current version 2.

It suffices to extend the list of supported `protocol-id` to:
- `254`: AES-256-GCM symmetric encryption;
- `255`: ECIES asymmetric encryption.

and encode/decode the Waku2 payload accordingly, whenever these values are set. 

Namely, if `protocol-id = 254, 255` then:
- `handshake-length`: is set to `0`;
- `handshake-message`: is left empty;
- `transport-message`: contains [26/WAKU-PAYLOAD](/spec/26) `data` field;
- `message-ad`: is set to `0`.

## References

1. [10/WAKU2](/spec/10)
2. [26/WAKU-PAYLOAD](/spec/26)
3. [14/WAKU-MESSAGE](/spec/14/#version1)
4. [Noise protocol](http://www.noiseprotocol.org/noise.html)
5. [Noise handshakes as key-exchange mechanism for Waku2](https://forum.vac.dev/t/noise-handshakes-as-key-exchange-mechanism-for-waku2/130)
6. [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234)


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
