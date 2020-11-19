---
title: Waku Spam Protected Relay
version: 2.0.0-alpha1
status: Raw
authors: Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Abstract](#abstract)
- [Motivation](#motivation)
- [RLN-integrated Relay protocol](#rln-integrated-relay-protocol)
  - [RLN](#rln)
  - [Flow](#flow)
    - [SetUp](#setup)
    - [Registration](#registration)
    - [Publishing](#publishing)
    - [Routing](#routing)
  - [Protobuf](#protobuf)
- [Changelog](#changelog)
- [Copyright](#copyright)

# Abstract

This specification enables spam-protection in the `relay` protocol using Rate Limiting Nullifiers (RLN). The security objective is to control the number of PubSub messages that each peer can publish per epoch, regardless of the published topic.


**Protocol identifier***: `/vac/waku/spam-protected-relay/2.0.0-alpha1`

# Motivation

In open p2p messaging networks, one big problem is spam-resistance. Existing solutions, such as Whisper’s proof of work, are insufficient, especially for heterogeneous nodes. Other reputation-based approaches might not be desirable, due to issues around arbitrary exclusion and privacy.

The current solution shall provide a novel, light, and provable spam prevention mechanism as part of the `relay` protocol which is suitable for resource-constrained nodes.

TODO: Fill in more


# RLN-integrated Relay protocol


## RLN

The idea of Rate Limiting Nullifier is about using nullifiers to limit the rate at which someone can perform an action e.g., sending a message. In the advanced variants of RLNs, effective methods are in place to make sure that those who break the rate will get punished. 

We utilize the following [RLN technique](https://hackmd.io/tMTLMYmTR5eynw2lwK9n1w?both)  which is based on [Semaphore](https://github.com/appliedzkp/semaphore/blob/master/spec/Semaphore%20Spec.pdf) and Shamir Secret Sharing Scheme (SSS). 

The system relies on a smart contract running on the Ethereum blockchain that is used to manage users' registrations and maintain a global state of the current members (through a membership tree realized by a Merkle Tree).

For the registration, the peer shall create a transaction that sends x (TODO to be specified) ETH to the contract which then adds a leaf to the on-chain Merkle tree. Each leaf is a "public key" and the person who has the "private key" (the leaf pre-image) is able to withdraw x ETH by providing a zkSNARK proof of a valid Merkle tree path and a valid Merkle root.

A peer willing to publish a message requires to prove that she is a current member and has not exceeded her messaging quota in the given epoch. The membership proof relies on the state of the contract (the root of the membership tree) and is done anonymously using zkSNARK. The messaging quotas are controlled through the concept of nullifiers. In specific, each message published in the system is associated with a nullifier derived from the peer's private key and an external nullifier which is the epoch. Nullifiers are stored as part of the contract's state and enable monitoring of the number of messages issued by each member per epoch. However, in our current implementation, the set of nullifiers can be locally stored by the routing peers. More details are provided below.
## Flow
### SetUp

There should be a smart contract with a known address.
The state of the smart contract will contain a Merkle Tree `MT` containing the set of registered peers and a `nullifier_map` that stores `share_x`, `share_y`, and `internal_nullifier`. 

TODO: add more details

### Registration

Peers willing to publish in the network shall follow the registration phase of RLN. 
The `a_0` and `auth_path` resultant from the RLN **registration** phase (step 2 of  https://github.com/vacp2p/research/issues/49) must be permanently and locally stored by the peer.

TODO: replicate step 2 of RLN here 

### Publishing

Peer:  
The publisher decides on the `WakuMessage` and calculates the `epoch`.
It proceeds based on the **signaling per epoch** (step 3 of  https://github.com/vacp2p/research/issues/49) of RLN with the following inputs (`signal` = `payload||contentTopic`, `a_0`, `auth_path`, `epoch`) where `payload` and `contentTopic` come from `WakuMessage`.

It is recommended (and necessary for anonymity) that the publisher updates her `auth_path` based on the latest version of the `MT.root` and attempts the proof using her updated `auth_path`.

Contract: 
As a result of this part, the state of the smart contract gets updated and the `internal_nullifier`, `share_x`, `share_y`  gets inserted into the `nullifier_map`.

Peer: 
builds its PubSub message by setting the `proofBundle` part of the `WakuMessage` with the following fields
1. `epoch` 
2. `share_x`
3. `share_y`
4. `internal_nullifier`
5. `zkproof`

### Routing

If the `epoch` attached to the message has a non-reasonable gap (TODO to be defined) with the routing node's current `epoch` then the message must be dropped (this is to prevent a newly registered node spamming the system by messaging for all the past epochs)
For each incoming message, the routing peer checks whether her local `nullifier_map` contains the `internal_nullifier`:
1. if not, then drop the message
2. if yes, compare the recorded `share_x` and `share_y` with the corresponding fields of the incoming message
     1. if different,  verify the `proof`:
        1. if verified then slash the owner
        2. else drop the message
     2. if the same, relay the message
-------

## Protobuf

```protobuf
message WakuMessage {
  optional bytes payload = 1;
  optional fixed32 contentTopic = 2;
  optional uint32 version = 3;
  optional ProofBundle proofBundle = 4;
}

message ProofBundle {
   int64 epoch = 1; //  indicating the intended epoch of the message
   // TODO share_x and share_y
   bytes internal_nullifier;
   // TODO zkproof
}
// TODO ZKproof may be a separate message type
// TODO the protobuf messages for communicating with the contract

```
// TODO to describe protobuf message fields






# Changelog

TBD.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
