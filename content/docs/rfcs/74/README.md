---
slug: 74
title: 74/RLN-STEALTH-COMMITMENTS
name: RLN Stealth Commitment Usage
status: raw
category: Standards Track
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
- Jimmy Debe <jimmy@status.im>
---

## Abstract

This specification describes the usage of stealth commitments to add prospective users to a network-governed [32/RLN-V1](/spec/32) membership set.

## Motivation

When [32/RLN-V1](/spec/32) is enforced in [10/Waku2](/spec/10), 
all users are required to register to a membership set.
The membership set will store user identities allowing the secure interaction within an application.
Forcing a user to do an on-chain transaction to join a membership set is an onboarding friction, 
and some projects may be opposed to this method.
To improve the user experience,
stealth commitments can be used by a counterparty to register identities on the user's behalf,
while maintaining the user's anonymity.

This document specifies a privacy-preserving mechanism,
allowing a counterparty to utilize [32/RLN-V1](/spec/32) to register an `identityCommitment` on-chain.
Counterparties will be able to register members to a RLN membership set without exposing the user's private keys.

## Background
The [32/RLN-V1](/spec/32) protocol, 
consists of a smart contract that stores a `idenitityCommitment` in a membership set.
In order for a user to join the membership set,
the user is required to make a transaction on the blockchain.
A set of public keys is used to compute a stealth address for a user,
as described in [ERC-5564](https://eips.ethereum.org/EIPS/eip-5564).
This specification is an implementation of the [ERC-5564](https://eips.ethereum.org/EIPS/eip-5564) scheme, 
tailored to the curve that is used in the [32/RLN-V1](/spec/32) protocol.

This can be used in a couple of ways in applications:
1. Applications can add users to the [32/RLN-V1](/spec/32) membership set in a batch.
2. Users of the application can register other users to the [32/RLN-V1](/spec/32) membership set.

This is useful when the prospective user does not have access to funds on the network that [32/RLN-V1](/spec/32) is deployed on.

## Wire Format Specification

The two parties, the requester and the receiver, MUST exchange the following information:

```protobuf

message Request {
  // The spending public key of the requester
  bytes spending_public_key = 1;

  // The viewing public key of the requester
  bytes viewing_public_key = 2;
}

```
### Generate Stealth Commitment

The application or user SHOULD generate a `stealth_commitment` after a request to do so is received.
This commitment MAY be inserted into the corresponding application membership set.

Once the membership set is updated, the receiver SHOULD exchange the following as a response to the request:

```protobuf

message Response {
  // If the request was accepted
  bool accepted = 1;

  // The private key used to see if transaction was made
  bytes view_tag = 2;

  // The stealth commitment for the requester
  bytes stealth_commitment = 3;

  // The ephemeral public key used to generate the commitment
  bytes ephemeral_public_key = 4;

}

```

- The receiver MUST generate an `ephemeral_public_key`, `view_tag` and `stealth_commitment`.
This will be used to check the stealth commitment used to register to the membership set,
and the user MUST be able to check ownership with their `viewing_public_key`.

## Implementation Suggestions

An implementation of the Stealth Address scheme is available in the [erc-5564-bn254](https://github.com/rymnc/erc-5564-bn254) repository,
which also includes a test to generate a stealth commitment for a given user.

## Security/Privacy Considerations

This specification inherits the security and privacy considerations of the [Stealth Address](https://eips.ethereum.org/EIPS/eip-5564) scheme.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## References

- [10/Waku2](/spec/10)
- [32/RLN-V1](/spec/32)
- [ERC-5564](https://eips.ethereum.org/EIPS/eip-5564)
