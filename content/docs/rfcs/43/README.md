---
slug: 43
title: 43/WAKU2-DEVICE-PAIRING
name: Device pairing and secure transfers with Noise
status: raw
category: Standards Track
tags: waku/core-protocol
editor: Giuseppe <giuseppe@status.im>
contributors:
---

# Abstract

In this document we describe a compound protocol 
for enabling two devices to mutually authenticate 
and securely exchange (arbitrary) information over the Waku network. 

# Background / Rationale / Motivation

In order to implement multi-device communications using one of the Noise session management mechanisms proposed in [37/WAKU2-NOISE-SESSIONS](https://rfc.vac.dev/spec/37/), 
we require a protocol to securely exchange (cryptographic) information between 2 or more devices possessed by a user.

Since, in this scenario, the devices would be close to each other, 
authentication can be initialized by exchanging a QR code out-of-band 
and then securely completed over the Waku network.

The protocol we propose consists of two main subprotocols or *phases*:

- [Device Pairing](#Device-Pairing): two phisically close devices initialize the *pairing* by exchanging a QR code out-of-band. The devices then exchange and authenticate their respective long-term device ID static key by exchanging handshake messages over the Waku network;
- [Secure Transfer](#Secure-Transfer): the devices securely exchange information in encrypted form using key material obtained during a successful pairing phase. The communication will happen over the Waku network, hence the devices do not need to be phisically close in this phase.

# Theory / Semantics

## Device Pairing

In the pairing phase, device `B` requests to be paired to a device `A`. 
Once the two devices are paired, the devices will be mutually authenticated 
and will share a Noise session within which they can securely exchange information.

The request is made by exposing a QR code that, by default, has to be scanned by device `A`. 
If device `A` doesn't have a camera while device `B` does, 
[it is possible](#Rationale) to execute a slightly different pairing (with same security guarantees), 
where `A` is exposing a QR code instead.

This protocol is designed in order to achieve two main security objectives:

- resistance to Man-in-the-Middle attacks;
- provide network anonymity on devices' static keys, i.e. only paired devices will learn each other static key.

### Employed Cryptographic Primitives

- `H`: the underlying cryptographically-secure hash function, e.g. SHA-256;
- `HKDF`: the key derivation function (based on `H`);
- `Curve25519`: the underlying elliptic curve for Diffie-Hellman (DH) operations.

### The `WakuPairing` Noise Handshake

The devices execute a custom handshake derived from `XX`, 
where they mutually exchange and authenticate their respective device static key 
by exchanging messages over the content topic with the following [format](https://rfc.vac.dev/spec/23/#content-topic-format)

```
contentTopic = /{application-name}/{application-version}/wakunoise/1/sessions_shard-{shard-id}/proto
```

The handshake, detailed in next section, can be summarized as:

```
WakuPairing:
a.   <- eB              {H(sB||r), contentTopicParams, messageNametag}
     ...
b.   -> eA, eAeB        {H(sA||s)}   [authcode]
c.   <- sB, eAsB        {r}
d.   -> sA, sAeB, sAsB  {s}

{}: payload,    []: user interaction
```

### Protocol Flow

1. The device `B` exposes through a QR code a Base64 serialization of:
    - An ephemeral public key `eB`;
    - The content topic parameters `contentTopicParams = {application-name}, {application-version}, {shard-id}`.
    - A (randomly generated) 16-bytes long `messageNametag`.
    - A commitment `H(sB||r)` for its static key `sB` where `r` is a random fixed-lenght value.

2. The device `A`:
    - scans the QR code;
    - obtains `eB`, `contentTopicParams`, `messageNametag`, `Hash(sB||r)`;
    - checks if `{application-name}` and `{application-version}` from `contentTopicParams` match the local application name and version: if not, aborts the pairing. Sets `contentTopic = /{application-name}/{application-version}/wakunoise/1/sessions_shard-{shard-id}/proto`;
    - initializes the Noise handshake by passing `contentTopicParams`, `messageNametag` and `Hash(sB||r)` to the handshake prologue;
    - executes the pre-handshake message, i.e. processes the key `eB`;
    - executes the first handshake message over `contentTopic`, i.e. 
        - processes and sends a Waku message containing an ephemeral key `eA`; 
        - performs `DH(eA,eB)` (which computes a symmetric encryption key);
        - attaches as payload to the handshake message the (encrypted) commitment `H(sA||s)` for `A`'s static key `sA`, where `s` is a random fixed-length value;
    - an 8-digits authorization code `authcode` obtained as `HKDF(h) mod 10^8` is displayed on the device, where `h` is the [handshake hash value](https://noiseprotocol.org/noise.html#overview-of-handshake-state-machine) obtained once the first handshake message is processed.

3. The device `B`:
    - sets `contentTopic = /{application-name}/{application-version}/wakunoise/1/sessions_shard-{shard-id}/proto`;
    - listens to messages sent to `contentTopic` and locally filters only those with [Waku payload](https://rfc.vac.dev/spec/35/#abnf) starting with `messageNametag`. If any, continues.
    - initializes the Noise handshake by passing `contentTopicParams`, `messageNametag` and `Hash(sB||r)` to the handshake prologue;
    - executes the pre-handshake message, i.e. processes its ephemeral key `eB`;
    - executes the first handshake message, i.e.
        - obtains from the received message a public key `eA`. If `eA` is not a valid public key, the protocol is aborted.
        - performs `DH(eA,eB)` (which computes a symmetric encryption key);
        - decrypts the commitment `H(sA||s)` for `A`'s static key `sA`.
    - an 8 decimal digits authorization code `authcode` obtained as `HKDF(h) mod 10^8` is displayed on the device, where `h`is the [handshake hash value](https://noiseprotocol.org/noise.html#overview-of-handshake-state-machine) obtained once the first handshake message is processed.

4. Device `A` and `B` wait for the user to confirm with an interaction (button press) 
that the authorization code displayed on both devices are the same. 
If not, the protocol is aborted.
    
5. The device `B`:
    - executes the second handshake message, i.e.
        - processes and sends his (encrypted) device static key `sB` over `contentTopic`;
        - performs `DH(eA,sB)` (which updates the symmetric encryption key);
        - attaches as payload the (encrypted) commitment randomness `r` used to compute `H(sB||r)`.

6. The device `A`:
    - listens to messages sent to `contentTopic` and locally filters only those with Waku payload starting with `messageNametag`. If any, continues.
    - decrypts the received message and obtains the public key `sB`. If `sB` is not a valid public key, the protocol is aborted.
    - performs `DH(eA,sB)` (which updates a symmetric encryption key);
    - decrypts the payload to obtain the randomness `r`. 
    - computes `H(sB||r)` and checks if this value corresponds to the commitment obtained in step 2. If not, the protocol is aborted.
    - executes the third handshake message, i.e.
        - processes and sends his (encrypted) device static key `sA` over `contentTopic`;
        - performs `DH(sA,eB)` (which updates the symmetric encryption key);
        - performs `DH(sA,sB)` (which updates the symmetric encryption key);
        - attaches as payload the (encrypted) commitment randomness `s` used to compute `H(sA||s)`.
    - calls [Split()](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object) and obtains two cipher states to encrypt inbound and outbound messages.
    
7. The device `B`:

    - listens to messages sent to `contentTopic` and locally filters only those with Waku payload starting with `messageNametag`. If any, continues.
    - obtains from decrypting the received message a public key `sA`. If `sA` is not a valid public key, the protocol is aborted.
    - performs `DH(sA,eB)` (which updates a symmetric encryption key);
    - performs `DH(sA,sB)` (which updates a symmetric encryption key);
    - decrypts the payload to obtain the randomness `s`. 
    - Computes `H(sA||s)` and checks if this value corresponds to the commitment obtained in step 3. If not, the protocol is aborted.
    - Calls [Split()](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object) and obtains two cipher states to encrypt inbound and outbound messages.

### The `WakuPairing` for Devices without a Camera

In the above pairing handshake, the QR is by default exposed by device `B` and not by `A` 
because in most use-cases we foresee, the secure transfer phase would consist in 
exchanging a single message (e.g., Noise sessions, cryptographic keys, signatures, etc.) from device `A` to `B`.

However, since the user(s) confirm(s) at the end of message `b.` that the authorization code is the same on both devices, 
the role of the handhsake initiator and responder can be safely swapped in message `a.` and `b.`.

Indeed, if the pairing phase successfully completes on both devices, 
the authentication code, the committed static keys and the Noise processing rules will ensure that no Man-in-the-Middle attack took place
and that messages can be securely exchanged bi-directionally in the transfer phase.

This allows pairing in case device `A` does not have a camera to scan a QR (e.g. a desktop client) while device `B` has.

The resulting handshake would then be:
```
WakuPairing2:
a.   -> eA              {H(sB||r), contentTopicParams, messageNametag}
     ...
b.   <- eB, eAeB        {H(sB||r)}   [authcode]
c.   <- sB, eAsB        {r}
d.   -> sA, sAeB, sAsB  {s}

{}: payload,    []: user interaction
```

# Secure Transfer

The pairing phase is designed to be application-agnostic 
and should be flexible enough to mutually authenticate 
and allow exchange of cryptographic key material 
between two devices over a distributed network of Waku2 nodes.

Once the handshake is concluded, 
(privacy-sensitive) information can be exchanged using the encryption keys agreed upon the pairing phase. 
If stronger security guarantees are required, 
some [additional tweaks](#Implementation-Suggestions) are possible.

# Implementation Suggestions

## Timebox QR exposure

We suggest to timebox the exposure of each pairing QR code to few seconds, e.g. 30.
After this time limit, a QR code containing a new ephemeral key, random static key commitment and message nametag (content topic parameters could remain the same) 
should replace the previously exposed QR, which can then be discarded.

The reason for such suggestion is due to the fact that if an attacker is able to compromise one of the ephemeral keys, 
he might successfully realize an undetected MitM attack up to the `authcode` confirmation 
(we note that compromising ephemeral keys is outside our and Noise security assumptions).

The attacker could indeed proceed as follows:
- intercepts the QR;
- blocks/delays the delivery of the pairing message `b.`;
- compromises `A` or `B` ephemeral key;
- recovers the genuine `authcode` that would have been generated by `A` and `B`;
- generates ~`10^8` random `t` values until the Noise processing of the message `b'. -> eC, eCeB  {H(sC||t)} `, where `eC` and `sC` are the attacker ephemeral and static key, respectively, results in computing the same `authcode` as the one between `A` and `B`; 
- delivers the message `b'. -> eC, eCeB  {H(sC||t)}` to `B` (before `A` is able to deliver its message `b.`).

At this point `A` and `B` will observe the same `authcode` (and would then confirm it),
but `B` will process the attacker's ephemeral key `eC` instead of `eA`. 

However, the attacker would not be able to open to device `A` the static key commitment `H(sB||s)` sent by device `B` out-of-band, 
and the pairing will abort on `A` side before it reveals its static key. 
Device `B`, instead, will successfully complete the pairing with the attacker.

Hence, timeboxing the QR exposure,
also in combination with increasing the number of decimal digits of the `authcode`,
will strongly limit the probability that an attacker can successfully impersonate device `A` to `B`.

We stress once more, that such attack requires the compromise of an ephemeral key (outside our security model)
and that device `A` will in any case detect a mismatch and abort the pairing, 
regardless of the fact that the QR timeboxing mitigation is implemented or not.

## Randomized Rekey

The Noise Protocol framework supports [`Rekey()`](http://www.noiseprotocol.org/noise.html#rekey) 
in order to update encryption keys *"so that a compromise of cipherstate keys will not decrypt older* \[exchanged\] *messages"*. 
However, if a certain cipherstate key is compromised, 
it will be possible for the attacker not only to decrypt messages encrypted under that key, 
but also all those messages encrypted under any successive new key obtained through a call to `Rekey()`.

This could be mitigated by attaching an ephemeral key to messages sent after a [Split()](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object)
so that a new random symmetric key can be derived, 
in a similar fashion to [Double-Ratchet](https://signal.org/docs/specifications/doubleratchet/).

This can be practically achieved by:
- keeping the full Handhshake State even after the handshake is complete (*by Noise specification a call to [`Split()`](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object) should delete the Handshake State*)
- continuing updating the Handshake State by processing every after-handshake exchanged message (i.e. the `payload`) according to the Noise [processing rules](http://www.noiseprotocol.org/noise.html#processing-rules) (i.e. by calling `EncryptAndHash(payload)` and `DecryptAndHash(payload)`);
- adding to each (or every few) message exchanged in the transfer phase a random ephemeral key `e` and perform Diffie-Hellman operations with the other party's ephemeral/static keys in order to update the underlying CipherState and recover new random inbound/outbound encryption keys by calling [`Split()`](http://www.noiseprotocol.org/noise.html#the-symmetricstate-object).

In short, the transfer phase would look like (but not necessarily the same as):

```
TransferPhase:
   -> eA, eAeB, eAsB  {payload}
   <- eB, eAeB, sAeB  {payload}
   ...
   
{}: payload
```

## Messages Nametag Derivation

To reduce metadata leakages and increase devices's anonymity over the p2p network, 
[35/WAKU2-NOISE](https://rfc.vac.dev/spec/37/#session-states) suggests to use some common secrets `mntsInbound, mntsOutbound` (e.g. `mntsInbound, mntsOutbound = HKDF(h)` 
where `h` is the [handshake hash value](https://noiseprotocol.org/noise.html#overview-of-handshake-state-machine) of the Handshake State at some point of the pairing phase) 
in order to frequently and deterministically change the `messageNametag` of messages exchanged during the pairing and transfer phase - 
ideally, at each message exchanged. 

Given the proposed construction,
the `mntsInbound` and `mntsOutbound` secrets can be used to iteratively generate the `messageNametag` field of Waku payloads 
for inbound and outbound messages, respectively. 

The derivation of `messageNametag` should be deterministic only for communicating devices 
and independent from message content, 
otherwise lost messages will prevent computing the next message nametag. 
A possible approach consists in computing the `n`-th `messageNametag` as `H( mntsInbound || n)`, 
where `n` is serialized as `uint64`.

In this way, sender's and recipient's devices 
can keep updated a buffer of `messageNametag` to sieve 
while listening to messages sent over `/{application-name}/{application-version}/wakunoise/1/sessions-{shard-id}/` (i.e., the next 50 not yet seen). 
They will then be able to further identify if one or more messages were eventually lost 
or not-yet-delivered during the communication.
This approach brings also the advantage that 
communicating devices can efficiently identify encrypted messages addressed to them.

We note that since the `ChaChaPoly` cipher used to encrypt messages supports *additional data*, 
an encrypted payload can be further authenticated by passing the `messageNametag` as additional data to the encryption/decryption routine. 
In this way, an attacker would be unable to craft an authenticated Waku message 
even in case the currently used symmetric encryption key is compromised, 
unless `mntsInbound`, `mntsOutbound` or the `messageNametag` buffer lists were compromised too.

# Security/Privacy Considerations

### Assumptions

- The attacker is active, i.e. can interact with both devices `A` and `B` by sending messages over `contentTopic`.

- The attacker has access to the QR code, that is knows the ephemeral key `eB`, the commitment `H(sB||r)` and the `contentTopic` exposed by the device `B`.

- Devices `A` and `B` are considered trusted (otherwise the attacker will simply exfiltrate the relevant information from the attacked device).

- As common for Noise, we assume that ephemeral keys cannot be compromised, while static keys might be later compromised. However, we enforce in the pairing phase extra security mechanisms (i.e. use of commitments for static keys) that will prevent some attacks possible when ephemeral keys are weak or get compromised.
 
### Rationale

- The device `B` exposes a commitment to its static key `sB` because:
    - it can commit to its static key before the authentication code is confirmed without revealing it.
    - If the private key of `eB` is weak or gets compromised, an attacker can impersonate `B` by sending in message `c.` to device `A` his own static key and successfully complete the pairing phase. Note that being able to compromise `eB` is not contemplated by our security assumptions.
    - `B` cannot adaptively choose a static key based on the state of the Noise handshake at the end of message `b.`, i.e. after the authentication code is confirmed. Note that device `B` is trusted in our security assumptions.
    - Confirming the authentication code after processing message `b.` will ensure that no Man-in-the-Middle (MitM) can later send a static key different than `sB`.

- The device `A` sends a commitment to its static key `sA` because:
    - it can commit to its static key before the authentication code is confirmed without revealing it.
    - `A` cannot adaptively choose a static key based on the state of the Noise handshake at the end of message `b.`, i.e. after the authentication code is confirmed. Note that device `A` is trusted in our security assumptions.
    - Confirming the authentication code after processing message `b.` will ensure that no MitM can later send a static key different than `sA`.

- The authorization code is shown and has to be confirmed at the end of message `b.` because:
    - an attacker that frontruns device `A` by sending faster his own ephemeral key would be detected before  he's able to know device `B` static key `sB`;
    - it ensures that no MitM attacks will happen during *the whole* pairing handshake, since commitments to the (later exchanged) device static keys will be implicitly acknowledged by the authorization code confirmation;
    - it enables to safely swap the role of handshake initiator and responder (see above);

- Device `B` sends his static key first because:
    - by being the pairing requester, it cannot probe device `A` identity without revealing its own (static key) first. Note that device `B` static key and its commitment can be bound to other cryptographic material (e.g., seed phrase).

- Device `B` opens a commitment to its static key at message `c.` because:
    - if device `A` replies concluding the handshake according to the protocol, device `B` acknowledges that device `A` correctly received his static key `sB`, since `r` was encrypted under an encryption key derived from the static key `sB` and the genuine (due to the previous `authcode` verification) ephemeral keys `eA` and `eB`.

- Device `A` opens a commitment to its static key at message `d.` because:
    - if device `B` doesn't abort the pairing, device `A` acknowledges that device `B` correctly received his static key `sA`, since `s` was encrypted under an encryption key derived from the static keys `sA` and `sB` and the genuine (due to the previous `authcode` verification) ephemeral keys `eA` and `eB`.

# Application to Noise Sessions

## The N11M session management mechanism

In the [`N11M` session management mechanism](https://rfc.vac.dev/spec/37/#the-n11m-session-management-mechanism), 
one of Alice's devices is already communicating with one of Bob's devices within an active Noise session, 
e.g. after a successful execution of a Noise handshake.

Alice and Bob would then share some cryptographic key material, 
used to encrypt their communications. 
According to [37/WAKU2-NOISE-SESSIONS](https://rfc.vac.dev/spec/37/) this information consists of:
- A `session-id` (32 bytes)
- Two cipher state `CSOutbound`, `CSInbound`, where each of them contains:
    - an encryption key `k` (2x32bytes)
    - a nonce `n` (2x8bytes)
    - (optionally) an internal state hash `h` (2x32bytes)
        
for a total of **176 bytes** of information.

In a [`N11M`](https://rfc.vac.dev/spec/37/#the-n11m-session-management-mechanism) session mechanism scenario, 
all (synced) Alice's devices that are communicating with Bob 
share the same Noise session cryptographic material.
Hence, if Alice wishes to add a new device, 
she must securely transfer a copy of such data from one of her device `A` to a new device `B` in her possession.

In order to do so she can:
- pair device `A` with `B` in order to have a Noise session between them; 
- securely transfer within such session the 176 bytes serializing the active session with Bob;
- manually instantiate in `B` a Noise session with Bob from the received session serialization.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## Normative
- [35/WAKU2-NOISE](https://rfc.vac.dev/spec/37/#session-states)
- [37/WAKU2-NOISE-SESSIONS](https://rfc.vac.dev/spec/37/)

## Informative
- [26/WAKU2-PAYLOAD](https://rfc.vac.dev/spec/35/#abnf)
- [The Double-Ratchet Algorithm](https://signal.org/docs/specifications/doubleratchet/)
- [The Noise Protocol Framework specifications](http://www.noiseprotocol.org/noise.html)