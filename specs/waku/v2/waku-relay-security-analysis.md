---
title: Security Analysis of Waku Relay
version: 
status: Draft
authors: Sanaz Taheri <sanaz@status.im>
---

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Abstract](#abstract)
  - [Terminologies](#terminologies)
  - [Anonymity](#anonymity)
      - [Message Author Anonymity](#message-author-anonymity)
      - [Topic Subscriber Anonymity](#topic-subscriber-anonymity)
  - [Authenticity](#authenticity)
  - [Integrity](#integrity)
  - [Confidentiality](#confidentiality)
  - [Changelog](#changelog)
    - [2.0.0-beta2](#200-beta2)
    - [2.0.0-beta1](#200-beta1)
- [Copyright](#copyright)
- [References](#references)

# Abstract

In this spec, we analyze the security of the  `relay` protocol concerning data confidentiality, integrity, authenticity, and anonymity. This is to enable users of this protocol to make informed decision about all the secuirty properties that can or can not achieve by deploying `relay` protocol. 

## Terminologies

Honest but Curious / static adversary: Throught this spec we consider an Honest but Curious adversarial model (static adversary interchangebly. That is, an adversarial entity may attempt to collect infromation from other peers (i.e., being curious) in order to succedd in its attack but it does so without violating protocol definitions and instructions (is honest), namely, it does follow the protocol specifications. 

Personally identifiable information (PII): PII indicates any piece of data that can be used to uniquely identify a Peer. For example, the signature verification key, and the hash of one's IP address are unique for each peer and hence count as PII.

## Anonymity
We classify anonymity into the following categories:

#### Message Author Anonymity
This propery indicates that no adversarial entity is able to link a published `Message` to its origin i.e., the peer. 

#### Topic Subscriber Anonymity
This feature stands for the inability of any adversarial entity from linking between a peer to its subscribed topic IDs. 


## Authenticity




## Integrity


## Confidentiality

In order to accomdoate message anonymity, `relay` protocol follows `NoStrictSign` policy due to which the `from`, `signature` and `key` fields of the `Message` MUST be left empty by the message publisher. The reason is that each of these three fields alone count as PII for the message author. 




## Changelog

### 2.0.0-beta2

Next version. Changes:

- 


### 2.0.0-beta1

Initial draft version.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [PubSub interface for libp2p (r2,
   2019-02-01)](https://github.com/libp2p/specs/blob/master/pubsub/README.md)

2. [GossipSub
   v1.0](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md)

3. [GossipSub
   v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md)

4. [Waku v1 spec](specs.vac.dev/waku/waku.html)

5. [Whisper spec (EIP627)](https://eips.ethereum.org/EIPS/eip-627)


<!--
TODO: Don't quite understand this scenario [key field], to clarify. Wouldn't it always be in `from`?
> The key field contains the signing key when it cannot be inlined in the source peer ID. When present, it must match the peer ID. -->

Re topicid:
NOTE: This doesn't appear to be documented in PubSub spec, upstream?
-->
