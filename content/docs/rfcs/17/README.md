---
slug: 17
title: 17/WAKU-RLN-RELAY
name: Waku v2 RLN Relay
status: raw
tags: waku-core
editor: Sanaz Taheri <sanaz@status.im>
---

The current specification embodies the details of the spam-protected version of `relay` protocol empowered by Rate Limiting Nullifiers (RLN). 
<!-- More details on RLN can be found in [this spec]() (TODO: to link the spec). -->

The security objective is to control the number of PubSub messages that each peer can publish per epoch where epoch is a system design parameter, regardless of the published topic.


**Protocol identifier***: `/vac/waku/waku-rln-relay/2.0.0-alpha1`

# Motivation

In open p2p messaging networks, one big problem is spam-resistance. 
Existing solutions, such as Whisper’s proof of work, are insufficient, especially for heterogeneous nodes. 
Other reputation-based approaches might not be desirable, due to issues around arbitrary exclusion and privacy.

We augment the `relay` protocol with a novel, light, and effective spam prevention mechanism which also suits the resource-constrained nodes.

<!-- TODO: Fill in more -->


# Flow
## SetUp and Registration
A peer willing to publish a message is required to register. 
Registration is moderated through a smart contract deployed on the Ethereum blockchain. 
The state of the contract contains the list of registered members public keys `pk`. 
An overview of registration is illustrated in Figure 1.

For the registration, a peer creates a transaction that registers its `pk` into the group.
The transaction also transfers x (TODO to be specified) amount of ETH to the contract to be staked. 
<!-- some portion of fund might be burnt at the registration -->
The peer can later remove itself by sending a valid `sk` whose `pk` belongs to the group.
The corresponding deposit would be sent x to the peer who performs deletion. 
Each `sk` is initially only known by the owning peer however it may get exposed to other peers in case the owner attempts spamming the system i.e., sending more than one message per epoch.
More details on that are given in the slashing section.
`sk` is a private information and the peer MUST persist it safely. 
Losing `sk` means losing the associated fund and inability to participate in the waku-rln-relay network.


![Figure 1: Registration.](../../../../rfcs/17/rln-relay.png)

<!-- TODO: the function calls in this figure as well as messages are subject to change -->

## Publishing

In order to publish at a given `epoch`, the publishing peer proceeds based on the regular `Waku2-Relay` protocol.  
However, in order to protect against spamming, each `WakuMessage` SHOULD carry a `RateLimitPRoof`  which encapsulates the following data items:

- `epoch`: the epoch at which the message is published.
- `proof`: is a zero-knowledge proof signifying that the publishing peer is a  registered member, and  has not exceeded the messaging rate at the given `epoch`.
  <!-- TODO: to clarify what a zero-knowledge proof means  -->

The proof generation relies on the knowledge of two pieces of private information i.e., `sk` and `authPath`.
`authPath` is  the information by which one can prove its membership in the group.
 <!-- TODO explain what is atuh path -->

To construct `authPath`, peers need to locally store a Merkle tree out of the group members public keys. 
Peers need to keep the tree updated with the recent state of the group.  

Public inputs to the proof generation are tree's `merkle_root`, `epoch` and `payload||contentTopic`  where `payload` and `contentTopic` come from the `WakuMessage`. 


- `merkle_root`: The root of membership tree at the time of publishing the message.
The `merkle_root` can be obtained from the locally maintained Merkle tree.
Peers are recommended to always stay updated with the state of the group and use the latest merkle tree root. 
Using older roots would allow inference about the node's `pk` index in the tree hence harming user anonymity.

- `share_x` and `share_y` which can be seen as partial disclosure of peer's `sk` for the intended `epoch`. 
Having two distinct pairs of `share_x`, `share_y` allows full disclosure of peer's `sk` and hence burning the associated deposit.
- `nullifier` is peer's finger print for the current epoch and calculated as `H(H(sk, epoch))` where `H` indicates Poseidon hash. 
As the `nullifier` is a deterministic value derived from peer's `sk` and `epoch`.
Any two distinct messages issued by the same peer (i.e., sing the same `sk`) for the same `epoch` are guaranteed to have identical `nullifier`s.


## Routing

Upon the receipt of a PubSub message, the routing peer needs to extract and parse the `proof` from the `data` field.  
If the `epoch` attached to the message has a non-reasonable gap (TODO: the gap should be defined) with the routing peer's current `epoch` then the message must be dropped (this is to prevent a newly registered peer spamming the system by messaging for all the past epochs). 
Furthermore, the routing peers MUST check whether the `proof` is valid and the message is not spam. 
If both checks are passed successfully, then the message is relayed. 
If `proof` is invalid then the message is dropped. 
If spamming is detected, the publishing peer gets slashed. 
An overview of routing procedure is depicted in Figure 2.

### Spam Detection and Slashing
In order to enable local spam detection and slashing, routing peers MUST record the `nullifier`, `share_x`, and `share_y` of any incoming message conditioned that it is not spam and has valid proof. 
To do so, the peer should follow the following steps. 
1. The routing peer first verifies the `zkSNARKs` and drops the message if not verified. 
2. Otherwise, it checks whether a message with an identical `nullifier` has already been relayed. 
   1. If such message exists and its `share_x` and `share_y` components are different from the incoming message, then slashing takes place (if the `share_x` and `share_y` fields of the previously relayed message is identical to the incoming message, then the message is a duplicate and shall be dropped).
   2. If none found, then the message gets relayed.

An overview of slashing procedure is provided in Figure 2.

<!-- TODO: may shorten or delete the Spam detection and slashing process -->

<!-- TODO: may consider [validator functions](https://github.com/libp2p/specs/tree/master/pubsub#topic-validation) or [extended validators](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md#extended-validators) for the spam detection -->

![Figure 2: Publishing, Routing and Slashing workflow.](../../../../rfcs/17/rln-message-verification.png)

<!-- TODO: the function calls in this figure as well as messages are subject to change -->

# Security Considerations

<!-- TODO: add discussion about the anonymity (e.g., the `StrictNoSign` policy) -->

<!-- TODO: discuss about the economic spam guarantees -->

-------

# Payloads

Payloads are protobuf messages implemented using [protocol buffers v3](https://developers.google.com/protocol-buffers/).
Nodes MAY extend the  [14/WAKU2-MESSAGE](/spec/14) with a `proof` field to indicate that their message is not a spam.

```diff 

syntax = "proto3";

message RateLimitProof {
  bytes proof = 1;
  bytes merkle_root = 2;
  bytes epoch = 3;
  bytes share_x = 4;
  bytes share_y = 5;
  bytes nullifier = 6;
}

message WakuMessage {
  bytes payload = 1;
  string contentTopic = 2;
  uint32 version = 3;
  double timestamp = 4;
+ RateLimitProof rate_limit_proof = 21;
}

```
## WakuMessage

`rate_limit_proof` holds the information required to prove that the message owner has not exceeded the message rate limit.
 
## RateLimitProof

The `proof` field is an array of 256 bytes and carries the zkSNARK proof as explained in the [Publishing process](##Publishing).
The proof asserts that:
1. The message publisher is the current member of the group i.e., her/his identity commitment key is part of the membership group Merkle tree with the root `merkle_root`.
2. `share_x` and `share_y`  are correctly computed.
3. The `nullifier` is constructed correctly.

Other fields of the `RateLimitProof` message are the public inputs to the rln circuit and used for the generation of the `proof`.

The `merkle_root` is an array of 32 bytes which holds the root of membership group Merkle tree at the time of publishing the message.

The `epoch` is an array of 32 bytes that represents the epoch in which the message is published.
<!-- TODO epoch is going to change to a different type -->

`share_x` and `share_y` are shares of the user's identity key.
These shares are created using [Shamir secret sharing scheme](##Publishing). 
`share_x` is an array of 32 bytes and contains the hash of the `WakuMessage`'s `payload` concatenated with its `contentTopic`. 
<!-- TODO hash other fields if necessary-->
`share_y` is also an array of 32 bytes which is calculated using [Shamir secret sharing scheme](##Publishing).

The `nullifier` is an internal nullifier which allows specifying whether two messages are published by the same publisher during the same `epoch`.
It is an array of 32 bytes.

<!-- TODO to reflect this change on WakuMessage spec once the PR gets mature -->

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [RLN documentation](https://hackmd.io/tMTLMYmTR5eynw2lwK9n1w?view)
2. [Public inputs to the rln circuit](https://hackmd.io/tMTLMYmTR5eynw2lwK9n1w?view#Public-Inputs)
3. [Shamir secret sharing scheme used in RLN](https://hackmd.io/tMTLMYmTR5eynw2lwK9n1w?view#Linear-Equation-amp-SSS)
4. [RLN internal nullifier](https://hackmd.io/tMTLMYmTR5eynw2lwK9n1w?view#Nullifiers)

