---
slug: 35
title: 35/WAKU2-NOISE
name: Noise Protocols for Waku Payload Encryption
status: raw
tags: waku-core-protocol
editor: Giuseppe <giuseppe@status.im>
contributors: 
---

This specification describes how payloads of [Waku messages](spec/14/) with [version 2](/spec/14/#version2) can be encrypted
in order to achieve confidentiality, authenticity, and integrity 
as well as some form of identity-hiding on communicating parties.


This specification extends the functionalities provided by [26/WAKU-PAYLOAD](/spec/26), 
adding support to modern symmetric encryption primitives 
and asymmetric key-exchange protocols.


Specifically, it adds support to the [`ChaChaPoly`](https://www.ietf.org/rfc/rfc7539.txt) cipher for symmetric authenticated encryption. 
It further describes how the [Noise Protocol Framework](http://www.noiseprotocol.org/noise.html) can be used to exchange cryptographic keys and encrypt/decrypt messages 
in a way that the latter are authenticated and protected by *strong forward secrecy*.


This ultimately allows Waku applications to instantiate end-to-end encrypted communication channels with strong conversational security guarantees,
as similarly done by [5/SECURE-TRANSPORT](https://specs.status.im/spec/5) but in a more modular way, 
adapting key-exchange protocols to the knowledge communicating parties have of each other.


## Design requirements

- *Confidentiality*: the adversary should not be able to learn what data is being sent from one Waku endpoint to one or several other Waku endpoints. 
    - *Strong forward secrecy*: an active adversary cannot decrypt messages nor infer any information on the employed encryption key, 
even in the case he has access to communicating parties' long-term private keys (during or after their communication).
- *Authenticity*: the adversary should not be able to cause a Waku endpoint to accept messages coming from an endpoint different than their original senders.
- *Integrity*: the adversary should not be able to cause a Waku endpoint to accept data that has been tampered with.
- *Identity-hiding*: once a secure communication channel is established, 
a passive adversary should not be able to link exchanged encrypted messages to their corresponding sender and recipient.


## Supported Cryptographic Protocols

### Noise Protocols

Two parties executing a Noise protocol exchange one or more [*handshake messages*](http://www.noiseprotocol.org/noise.html#message-format) and/or [*transport messages*](http://www.noiseprotocol.org/noise.html#message-format).
A Noise protocol consists of one or more Noise handshakes. 
During a Noise handshake, two parties exchange multiple handshake messages. 
A handshake message contains *ephemeral keys* and/or *static keys* from one of the parties
and an encrypted or unencrypted payload that can be used to transmit optional data. 
These public keys are used to perform a protocol-dependent sequence of Diffie-Hellman operations, 
whose results are all hashed into a shared secret key. 
After a handshake is complete, each party will then use the derived shared secret key to send and receive authenticated encrypted transport messages. 
We refer to [Noise protocol framework specifications](http://www.noiseprotocol.org/noise.html#processing-rules) for the full details on how parties shared secret key is derived from each exchanged message.

Four Noise handshakes are currently supported: `K1K1`, `XK1`, `XX`, `XXpsk0`. Their description can be found in [Appendix: Supported Handshakes Description](#Appendix-Supported-Handshake-Description). 
These are instantiated combining the following cryptographic primitives:
- [`Curve25519`](http://www.noiseprotocol.org/noise.html#the-25519-dh-functions) for Diffie-Hellman key-exchanges (32 bytes curve coordinates);
- [`ChaChaPoly`](http://www.noiseprotocol.org/noise.html#the-chachapoly-cipher-functions) for symmetric authenticated encryption (16 bytes block size);
- [`SHA256`](http://www.noiseprotocol.org/noise.html#the-sha256-hash-function) hash function used in [`HMAC`](http://www.noiseprotocol.org/noise.html#hash-functions) and [`HKDF`](http://www.noiseprotocol.org/noise.html#hash-functions) keys derivation chains (32 bytes output size);

#### Content Topics of Noise Handshake Messages

We note that all [design requirements](#Design-requirements) on exchanged messages would be satisfied only *after* a supported Noise handshake is completed, 
corresponding to a total of 1 Round Trip Time communication *(1-RTT)*.  
In particular, identity-hiding properties can be guaranteed only if the recommendation described in [After-handshake](#After-handshake) are implemented.

In the following, we assume that communicating parties reciprocally know an initial [`contentTopic`](/spec/14/#wakumessage) 
where they can send/receive the first handshake message.
We further assume that multiple initiators can start an handshake with the same recipient on an initial `contentTopic`,
which can be then directly linked to recipient's identity (no identity-hiding guarantee for the recipient).

The second handshake message SHOULD be sent/received on a `contentTopic` deterministically derived from the first handshake message (using, for example, `HKDF`).
This allows
- the recipient to efficiently continue the handshakes started by each initiator;
- the initiators to efficiently associate the recipient's second handshake message to their first handshake message,
However, this does not provide any identity-hiding guarantee to the recipient.

After the second handshake message is correctly received by initiators, the recommendation described in [After-handshake](#After-handshake) SHOULD be implemented to provide full identity-hiding guarantees for both initiator and recipient against passive attackers.

### Encryption Primitives

The symmetric primitives supported are:
- [`ChaChaPoly`](https://www.ietf.org/rfc/rfc7539.txt) for authenticated encryption (16 bytes block size).

## Specification

When [14/WAKU-MESSAGE version](/spec/14/#payload-encryption) is set to 2, 
the corresponding `WakuMessage`'s `payload` will encapsulate the two fields `handshake-message` and `transport-message`.

The `handshake-message` field MAY contain 
- a Noise handhshake message (only encrypted/unencrypted public keys).

The `transport-message` field MAY contain
- a Noise handshake message payload (encrypted/unencrypted);
- a Noise transport message;
- a `ChaChaPoly` ciphertext.

When a `transport-message` encodes a `ChaChaPoly` ciphertext, the corresponding `handshake-message` field MUST be empty.

The following fields are concatenated to form the `payload` field:

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
 Only during a Noise handshake, this field would contain the Noise handshake message payload.
 - `transport-message-auth`: the symmetric encryption authentication data for `transport-message` (16 bytes).


### ABNF

Using [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234) we have the following format:

```abnf
; protocol ID
protocol-id                 = 1OCTET

; contains the size of handshake-message
handshake-message-len       = 1OCTET

; contains one or more Diffie-Hellman public keys
handshake-message           = *OCTET

; contains the size of message-len
transport-message-len-len   = 1OCTET

; contains the size of transport-message
transport-message-len       = *OCTET

; contains the transport message, eventually encrypted
transport-message           = *OCTET

; contains authentication data for transport-message, if encrypted
transport-message-auth        = 16OCTET

; the Waku WakuMessage payload field
payload  = protocol-id handshake-message-len handshake-message transport-message-len-len transport-message-len transport-message transport-message-auth 
```

### Protocol Payload Format

Based on the specified `protocol-id`, 
the Waku message `payload` field will encode different types of protocol-dependent messages. 

In particular, if `protocol-id` is
- `0`:  payload encodes an [after-handshake](#After-handshake) message.  
    - `handshake-message-len` MAY be 0;
    -  `transport-message` contains the Noise transport message;
- `10`,`11`,`12`,`13`: payload encodes a supported Noise handshake message.  
    -  `transport-message` contains the Noise transport message;
- `30`: payload encapsulate a `ChaChaPoly` ciphertext `ct`.  
    - `handshake-message-len` is set to `0`;
    - `transport-message` contains the concatenation of the encryption nonce (12 bytes) followed by the ciphertext `ct`;
    - `transport-message-len-len` and `transport-message-len` are set accordingly to `transport-message` length;
    - `transport-message-auth` contains the authentication data for `ct`.


### Public Keys Serialization

Diffie-Hellman public keys can be trasmitted in clear 
or in encrypted form (cf. [`WriteMessage`](http://www.noiseprotocol.org/noise.html#the-handshakestate-object)) with authentication data attached. 
To distinguish between these two cases, public keys are serialized as the concatenation of the following three fields:

- `flag`: 
is equal to `1` if the public key is encrypted; 
`0` otherwise (1 byte);
- `pk`: 
if `flag = 0`, it contains an encoding of the X coordinate of the public key. 
If `flag = 1`, it contains a symmetric encryption of an encoding of the X coordinate of the public key;
- `pk-auth`: 
if `flag = 0`, it is empty; 
if `flag = 1`, it contains the authentication data for `pk`;

The corresponding serialization is obtained as `flag pk pk-auth`.

As regards the underlying supported [cryptographic primitives](#Cryptographic-primitives):
- `Curve25519` public keys X coordinates are encoded in little-endian as 32 bytes arrays;
- `ChaChaPoly` authentication data consists of 16 bytes 
(nonces are implicitely defined by Noise [processing rules](http://www.noiseprotocol.org/noise.html#processing-rules)).

In all supported Noise protocols, 
parties' static public keys are transmitted encrypted (cf. [`EncryptAndHash`](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object)), 
while ephemeral keys MAY be encrypted after a handshake is complete.


### Padding

To prevent some metadata leakage, 
encrypted transport messages SHOULD be padded before encryption 
to a multiple of the underlying symmetric cipher block size 
(16 bytes for `ChaChaPoly`).

It is therefore recommended to right pad transport messages to a multiple of 256 bytes.


## After-handshake

During the initial 1-RTT communication, 
handshake messages [can be linked](#Content-Topics-of-Noise-Handshake-Messages) to the respective parties 
through the `contentTopic` employed for such communication

After a handshake is completed, 
parties SHOULD derive from their shared secret key (preferably using `HKDF`) a new random `contentTopic` 
and continue their communication there.

When communicating on the new `contentTopic`, 
parties SHOULD set `protocol-id` to `0` 
to reduce metadata leakages and indicate that the message is an *after-handshake* message.

Each party SHOULD attach an (unencrypted) ephemeral key in `handshake-message` to every message sent.
According to [Noise processing rules](http://www.noiseprotocol.org/noise.html#processing-rules), 
this allows updates to the shared secret key 
by hashing the result of an ephemeral-ephemeral Diffie-Hellman exchange every 1-RTT communication.


## Backward Support for Symmetric/Asymmetric Encryption

It is possible to have backward compatibility to symmetric/asymmetric encryption primitives from [26/WAKU-PAYLOAD](/spec/26), 
effectively encapsulating  payload encryption [14/WAKU-MESSAGE version 1](/spec/14/#version1) in [version 2](/spec/14/#version2).

It suffices to extend the list of supported `protocol-id` to:
- `254`: AES-256-GCM symmetric encryption;
- `255`: ECIES asymmetric encryption.

and set the `transport-message` field to the [26/WAKU-PAYLOAD](/spec/26) `data` field, whenever these `protocol-id` values are set. 

Namely, if `protocol-id = 254, 255` then:
- `handshake-message-len`: is set to `0`;
- `handshake-message`: is empty;
- `transport-message`: contains the [26/WAKU-PAYLOAD](/spec/26) `data` field (AES-256-GCM or ECIES, depending on `protocol-id`);
- `transport-message-len-len` and `transport-message-len` are set accordingly to `transport-message` length;
- `transport-message-auth`: is set to `0`.

When a `transport-message` corresponding to `protocol-id = 254, 255` is retrieved, 
it SHOULD be decoded as the `data` field in [26/WAKU-PAYLOAD](/spec/26) specification.

## Appendix: Supported Handshakes Description


Supported Noise handshakes address four typical scenarios occurring when an encrypted communication channel between Alice and Bob is going to be created: 

- Alice and Bob know each others' static key.
- Alice knows Bob's static key;
- Alice and Bob share no key material and they don't know each others' static key.
- Alice and Bob share some key material, but they don't know each others' static key.


**Adversarial Model**: an active attacker who compromised one party's static key may lower the identity-hiding security guarantees provided by some handshakes. In our security model we exclude such adversary, but for completeness we report a summary of possible de-anonymization attacks that can be performed by an active attacker.

### The `K1K1` Handshake

If Alice and Bob know each others' static key (e.g., these are public or were already exchanged in a previous handshake) , they MAY execute a `K1K1` handshake.  Using [Noise notation](https://noiseprotocol.org/noise.html#overview-of-handshake-state-machine) *(Alice is on the left)* this can be sketched as:

```
 K1K1:
    ->  s
    <-  s
       ...
    ->  e
    <-  e, ee, es
    ->  se
```

We note that here only ephemeral keys are exchanged. This handshake is useful in case Alice needs to instantiate a new separate encrypted communication channel with Bob, e.g. opening multiple parallel connections, file transfers, etc.

**Security considerations on identity-hiding (active attacker)**: no static key is transmitted, but an active attacker impersonating Alice can check candidates for Bob's static key.

### The `XK1` Handshake

Here, Alice knows how to initiate a communication with Bob and she knows his public static key: such discovery can be achieved, for example, through a publicly accessible register of users' static keys, smart contracts, or through a previous public/private advertisement of Bob's static key.

A Noise handshake pattern that suits this scenario is `XK1`:

```
 XK1:
    <-  s
       ...
    ->  e
    <-  e, ee, es
    ->  s, se
```

Within this handshake, Alice and Bob reciprocally authenticate their static keys `s` using ephemeral keys `e`. We note that while Bob's static key is assumed to be known to Alice (and hence is not transmitted), Alice's static key is sent to Bob encrypted with a key derived from both parties ephemeral keys and Bob's static key.

**Security considerations on identity-hiding (active attacker)**: Alice's static key is encrypted with forward secrecy to an authenticated party. An active attacker initiating the handshake can check candidates for Bob's static key against recorded/accepted exchanged handshake messages.


### The `XX` and `XXpsk0` Handshakes

If Alice is not aware of any static key belonging to Bob (and neither Bob knows anything about Alice), she can execute an `XX` handshake, where each party tran**X**mits to the other its own static key. 

The handshake goes as follows:

```
 XX:
    ->  e
    <-  e, ee, s, es
    ->  s, se
```

We note that the main difference with `XK1` is that in second step Bob sends to Alice his own static key encrypted with a key obtained from an ephemeral-ephemeral Diffie-Hellman exchange.


This handshake can be slightly changed in case both Alice and Bob pre-shares some secret `psk` which can be used to strengthen their mutual authentication during the handshake execution. One of the resulting protocol, called `XXpsk0`, goes as follow:

```
 XXpsk0:
    ->  psk, e
    <-  e, ee, s, es
    ->  s, se
```
The main difference with `XX` is that Alice's and Bob's static keys, when transmitted, would be encrypted with a key derived from `psk` as well.


**Security considerations on identity-hiding (active attacker)**: Alice's static key is encrypted with forward secrecy to an authenticated party for both `XX` and `XXpsk0` handshakes. In `XX`, Bob's static key is encrypted with forward secrecy but is transmitted to a non-authenticated user which can then be an active attacker. In `XXpsk0`, instead, Bob's secret key is protected by forward secrecy to a partially authenticated party (through the pre-shared secret `psk` but not through any static key), provided that `psk` was not previously compromised (in such case identity-hiding properties provided by the `XX` handshake applies).


## References

1. [5/SECURE-TRANSPORT](https://specs.status.im/spec/5)
2. [10/WAKU2](/spec/10)
3. [26/WAKU-PAYLOAD](/spec/26)
4. [14/WAKU-MESSAGE](/spec/14/#version1)
5. [Noise protocol](http://www.noiseprotocol.org/noise.html)
6. [Noise handshakes as key-exchange mechanism for Waku2](https://forum.vac.dev/t/noise-handshakes-as-key-exchange-mechanism-for-waku2/130)
7. [Augmented Backus-Naur form (ABNF)](https://tools.ietf.org/html/rfc5234)


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
