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
for enabling two devices belonging to the same user 
to mutually authenticate and securely exchange (arbitrary) information over Waku. 

# Background / Rationale / Motivation

In order to implement multi-devices communications using one of the Noise session management mechanisms proposed in [37/WAKU2-NOISE-SESSIONS](https://rfc.vac.dev/spec/37/), 
we require a protocol to securely exchange (cryptographic) information between 2 or more devices possessed by a user.

Since, in this scenario, the devices would be close to each other 
and under direct control of the user, 
authentication can be initialized by exchanging out-of-band a QR code 
and securely completed over the network.

The protocol we propose consists of two main subprotocols or *phases*:

- [Device Pairing](#Device-Pairing): the devices exchange and authenticate their long term device ID static keys;
- [Secure Transfer](#Secure-Transfer): the devices securely exchange in encrypted form information using key material obtained during a successful pairing phase.
 
# Theory / Semantics

## Device Pairing

In the pairing phase, a device `B` requests to be paired to a device `A`. 
Once the two devices are paired, the devices will be mutually authenticated 
and will share a Noise session within which they can securely exchange information.

The request is made by exposing a QR code that, by default, has to be scanned by device `A`. 
If device `A` doesn't have a camera while device `B` does, 
[it is possible](#Rationale) to execute a slightly different pairing (with same security guarantees), 
where `A` is exposing a QR code instead.

### Employed Cryptographic Primitives

- `H`: the underlying hash function, i.e. SHA-256;
- `HKDF`: a key derivation function (based on SHA-256);
- `Curve25519`: the underlying elliptic curve for Diffie-Hellman (DH) operations.

### The `WakuPairing` Noise Handshake

The devices execute a custom handshake derived from `X1X1`, 
where they mutually exchange and authenticate their device static keys 
by exchanging messages over the content topic

```
contentTopic = /{application-name}/{application-version}/wakunoise/1/sessions-{shard-id}/proto
```

The handshake, detailed in next section, can be summarized as:

```
WakuPairing:
0.   <- eB              {H(sB||r), contentTopicParams, messageNametag}
     ...
1.   -> eA, eAeB        {H(sA||s)}   [authcode]
2.   <- sB, eAsB        {r}
3.   -> sA, sAeB, sAsB  {s}

{}: payload,    []: user interaction
```

### Protocol Flow

1. The device `B` exposes through a QR code a Base64 serialization of:
    - An ephemeral public key `eB`;
    - The content topic parameters `contentTopicParams = {application-name}, {application-version}, {shard-id}`.
    - A randomly generated 8-bytes long `messageNametag`. 
    - A commitment `H(sB||r)` for its static key `sB` where `r` is a random fixed-lenght value.

2. The device `A`:
    - scans the QR code;
    - obtains `eB`, `contentTopicParams`, `messageNametag`, `Hash(sB|r)`;
    - checks if `{application-name}` and `{application-version}` from `contentTopicParams` match the local application name and version: if not, aborts the pairing. 
    - initializes the Noise handshake by passing `contentTopicParams`, `messageNametag` and `Hash(sB||r)` to the handshake prologue;
    - executes the pre-handshake message, i.e. processes the key `eB`;
    - executes the first handshake message over `contentTopic`, i.e. 
        - processes and sends a Waku message containing an ephemeral key `eA`; 
        - performs `DH(eA,eB)` (which computes a symmetric encryption key);
        - attach as payload to the handshake message a commitment `H(sA|s)` for `A`'s static key `sA`, where `s` is a random fixed-lenght value;
    - an 8-digits authorization code `authcode` obtained as `HKDF(h) mod 10^8` is displayed on the device, where `h`is the handshake value obtained once the first handshake message is processed.

3. The device `B`:
    - listens to messages sent to `/{application-name}/{application-version}/wakunoise/1/sessions-{shard-id}/proto` and locally filters only those with [Waku payload](https://rfc.vac.dev/spec/35/#abnf) starting with `messageNametag`. If any, continues.
    - initializes the Noise handshake by passing `contentTopicParams`, `messageNametag` and `Hash(sB||r)` to the handshake prologue;
    - executes the pre-handshake message, i.e. processes its static key `eB`;
    - executes the first handshake message, i.e.
        - obtains from the received message a public key `eA`. If `eA` is not a valid public key, the protocol is aborted.
        - performs `DH(eA,eB)` (which computes a symmetric encryption key);
        - decrypts the commitment `H(sA||s)` for `A`'s static key `sA`.
    - an 8-digits authorization code `authcode` obtained as `HKDF(h) mod 10^8` is displayed on the device, where `h`is the handshake value obtained once the first handshake message is processed.

4. Device `A` and `B` wait the user to confirm with an interaction (button press) 
that the authorization code displayed on both devices are the same. 
If not, the protocol is aborted.
    
5. The device `B`:
    - executes the second handshake message, i.e.
        - processes and sends his (encrypted) device static key `sB` over `contentTopic`;
        - performs `DH(eA,sB)` (which updates the symmetric encryption key);
        - attaches as payload the (encrypted) commitment randomness `r` used to compute `H(sB||r)`.

6. The device `A`:
    - listens to messages sent to `/{application-name}/{application-version}/wakunoise/1/sessions-{shard-id}/proto` and locally filters only those with Waku payload starting with `messageNametag`. If any, continues.
    - obtains from decrypting the received message a public key `sB`. If `sB` is not a valid public key, the protocol is aborted.
    - performs `DH(eA,sB)` (which updates a symmetric encryption key);
    - decrypts the payload to obtain the randomness `r`. 
    - Computes `H(sB||r)` and checks if this value corresponds to the commitment obtained in step 2. If not, the protocol is aborted.
    - executes the third handshake message, i.e.
        - processes and sends his (encrypted) device static key `sA` over `contentTopic`;
        - performs `DH(sA,eB)` (which updates the symmetric encryption key);
        - performs `DH(sA,sB)` (which updates the symmetric encryption key);
        - attaches as payload the (encrypted) commitment randomness `s` used to compute `H(sA||s)`.
    - Calls Split() and obtains two cipher states to encrypt inbound and outbound messages.
    
7. The device `B`:

    - listens to messages sent to `/{application-name}/{application-version}/wakunoise/1/sessions-{shard-id}/proto` and locally filters only those with Waku payload starting with `messageNametag`. If any, continues.
    - obtains from decrypting the received message a public key `sA`. If `sA` is not a valid public key, the protocol is aborted.
    - performs `DH(sA,eB)` (which updates a symmetric encryption key);
    - performs `DH(sA,sB)` (which updates a symmetric encryption key);
    - decrypts the payload to obtain the randomness `s`. 
    - Computes `H(sA||s)` and checks if this value corresponds to the commitment obtained in step 6. If not, the protocol is aborted.
    - Calls Split() and obtains two cipher states to encrypt inbound and outbound messages.

### The `WakuPairing` for Devices without a Camera

In the above pairing handshake, the QR is by default exposed by device `B` and not by `A` because device `B` locally stores no relevant cryptographic material, 
so an active local attacker that scans the QR code first 
would only be able to transfer *his own* session information 
and get nothing from `A`. 

However, since the user confirms at the end of message `1` that the authorization code is the same on both devices, 
the role of handhsake initiator and responder can be safely swapped in message `0` and `1`. 

This allows pairing in case device `A` does not have a camera to scan a QR (e.g. a desktop client) while device `B` has.

The resulting handshake would then be:
```
WakuPairing2:
0.   -> eA              {H(sB||r), contentTopicParams, messageNametag}
     ...
1.   <- eB, eAeB        {H(sB||r)}   [authcode]
2.   <- sB, eAsB        {r}
3.   -> sA, sAeB, sAsB  {s}

{}: payload,    []: user interaction
```

# Secure Transfer

The pairing phase is designed to be application-agnostic 
and should be flexible enough to mutually authenticate 
and allow exchange of cryptographic key material 
between two devices over a distributed network of Waku2 nodes.

Once the handshake is concluded, 
sensitive information can be exchanged using the encryption keys agreed during the pairing phase. 
If stronger security guarantees are required, 
some [additional tweaks](#Implementation-Suggestions) are possible.

# Implementation Suggestions

## Randomized Rekey

The Noise Protocol framework supports [`Rekey()`](http://www.noiseprotocol.org/noise.html#rekey) 
in order to update encryption keys *"so that a compromise of cipherstate keys will not decrypt older* \[exchanged\] *messages"*. 
However, if a certain cipherstate key is compromised, 
it will be possible for the attacker not only to decrypt messages encrypted under that key, 
but also all those messages encrypted under any successive new key obtained through a call to `Rekey()`.

This can be mitigated by:
- keeping the full Handhshake State even after the handshake is complete (*by Noise specification a call to `Split()` should delete the Handshake State*)
- continuing updating the Handshake State by processing every after-handshake exchanged message (i.e. the `payload`) according to the Noise [processing rules](http://www.noiseprotocol.org/noise.html#processing-rules) (i.e. by calling `EncryptAndHash(payload)` and `DecryptAndHash(payload)`);
- adding to each (or every few) message exchanged in the transfer phase a random ephemeral key `e` and perform Diffie-Hellman operations with the other party's ephemeral/static keys in order to update the underlying CipherState and recover new random inbound/outbound encryption keys by calling `Split()`.

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
[35/WAKU2-NOISE](https://rfc.vac.dev/spec/37/#session-states) suggests to use some common secrets `ctsInbound, ctsOutbound` (e.g. `ctsInbound, ctsOutbound = HKDF(h)` 
where `h` is the handshake hash value of the Handshake State at some point of the pairing phase) 
in order to frequently and deterministically change the `messageNametag` of messages exchanged during the pairing and transfer phase - 
ideally, at each message exchanged. 

Given the proposed construction,
the `ctsInbound` and `ctsOutbound` secrets can be used to iteratively generate the `messageNametag` field of Waku payloads 
for inbound and outbound messages, respectively. 

The derivation of `messageNametag` should be deterministic only for communicating devices 
and independent from message content, 
otherwise lost messages will prevent computing the next message nametag. 
A possible approach consists in computing the `n`-th `messageNametag` as `H( ctsInbound || n)`, 
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
unless `ctsInbound`, `ctsOutbound` or the `messageNametag` buffer lists were compromised too.

# Security/Privacy Considerations

### Assumptions

- The attacker is active, i.e. can interact with both devices `A` and `B` by sending messages over `contentTopic`.

- The attacker has access to the QR code, that is knows the ephemeral key `eB`, the commitment `H(sB||r)` and the `contentTopic` exposed by the device `B`.

- Devices `A` and `B` are considered trusted (otherwise the attacker will simply exfiltrate the relevant information from the attacked device).

- As common for Noise, we assume that ephemeral keys cannot be compromised, while static keys might be later compromised. However, we enforce in the pairing phase extra security mechanisms (i.e. use of commitments for static keys) that will prevent some attacks possible when ephemeral keys are weak or get compromised.
 
### Rationale

- The device `B` exposes a commitment to its static key `sB` because:
    - if the private key of `eB` is weak or gets compromised, an attacker can impersonate `B` by sending in message `2` to device `A` his own static key and successfully complete the pairing phase. Note that being able to compromise `eB` is not contemplated by our security assumptions.
    - `B` cannot adaptively choose a static key based on the state of the Noise handshake at the end of message `1`, i.e. after the authentication code is confirmed. Note that device `B` is trusted in our security assumptions.
    - Confirming the authentication code after processing message `1` will ensure that no Man-in-the-Middle (MitM) can send a static key different than `sB`.

- The device `A` sends a commitment to its static key `sA` because:
    - `A` cannot adaptively choose a static key based on the state of the Noise handshake at the end of message `1`, i.e. after the authentication code is confirmed. Note that device `A` is trusted in our security assumptions.
    - Confirming the authentication code after processing message `1` will ensure that no MitM can send a static key different than `sA`.

- The authorization code is shown and has to be confirmed at the end of message `1` because:
    - an attacker that frontruns device `A` by sending faster his own ephemeral key would be detected before  he's able to know device `B` static key `sB`;
    - it ensures that no MitM attacks will happen during *the whole* pairing handshake, since commitments to the (later exchanged) device static keys will be implicitly acknowledged by the authorization code confirmation;
    - it enables to safely swap the role of handshake initiator and responder (see above);

- Device `B` sends his static key first because:
    - by being the pairing requester, it cannot probe device `A` identity without revealing its own (static key) first. Note that device `B` static key and its commitment can be binded to other cryptographic material (e.g., seed phrase).

- Device `B` opens a commitment to its static key at message `2.` because:
    - if device `A` replies concluding the handshake according to the protocol, device `B` acknowledges that device `A` correctly received his static key `sB`, since `r` was encrypted under an encryption key derived from the static key `sB` and the genuine (due to the previous `authcode` verification) ephemeral keys `eA` and `eB`.

- Device `A` opens a commitment to its static key at message `3.` because:
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
- [The Noise Protocol Framework specifications](http://www.noiseprotocol.org/noise.html)