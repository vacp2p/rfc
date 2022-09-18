---
slug: 41
title: 41/WAKU2-RLN-CONTRACT
name: Rate Limiting Nullifier Membership Contract
status: raw
category: Standards Track
tags: RLN
editor: Keshav Gupta <keshav@status.im>
contributors:
---

# Abstract
The following standard specifies a standard API for Rate Limiting Nullifier Contract that manages membership of the participants in a peer-to-peer messaging group.
This API is intended to be used in conjunction with [17/WAKU-RLN-RELAY](https://rfc.vac.dev/spec/17/).
Peers that violate the messaging rate in a relay network,
like explained in [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/),
are considered spammers and their messages are considered spam.
Spammers are financially punished and removed from the system.
Rate limiting nullifier (RLN), like explained in [32/RLN](https://rfc.vac.dev/spec/32/),
a construct based on zero-knowledge proofs that provides an anonymous rate-limited signaling/messaging framework suitable for decentralized (and centralized) environments.


# Background / Rationale / Motivation

This contract serves as the basis for the implementation of RLN membership.
New members can be registered by depositing a stake of valuable funds.
They can also be slashed if they misbehave like spamming in a peer-to-peer pseudo-anonymous messaging network.
Peers MUST be registered to the RLN group to be able to publish messages.
Each peer has an RLN key pair denoted by `secret` and `pubkey`.
The state of the membership contract contains the list of registered members' public identity keys i.e., `pubkeys`.
For the registration, a peer creates a transaction that invokes the registration function of the contract via which registers its `pubkey` in the group.
The peer who has the secret key `secret` associated with a registered `pubkey` would be able to withdraw a portion the staked fund by providing valid proof.



# Theory / Semantics

As explained in [17/WAKU-RLN-RELAY](https://rfc.vac.dev/spec/17/),
all peers who want to publish the message in the spam protected peer-to-peer network MUST register for membership to the RLN group.
This is done via the `register` function of the RLN membership contract.
If a registered member of the RLN group spams messages in the network then its secret key get exposed which can be used to slash its membership from the group.
This is done by calling the `withdraw` function of the RLN membership contract, which removes the member from the Merkle tree thereby revoking its rights to send any messages.

# Contract Specification

### Constructor

* `constructor(uint256 membershipDeposit, uint256 depth, address _poseidonHasher) public`
    * `membershipDeposit` sets the amount of ETH that must be deposited to `register` to the RLN group.
    * `depth` is the depth of the Merkle tree that determines the maximum number of members to the RLN group.
    * `_poseidonHasher` is the address of the poseidon hasher contract used in the `hash` function.

### Methods

* register: This function registers the `pubkey` to the list of members.
For successful registration, ETH equal to the amount specified in `membershipDeposit` MUST be paid while calling this function.
This function MUST fire the `MemberRegistered` event.
    * `function register(uint256 pubkey) external payable`



* registerBatch: This function registers multiple `pubkeys` to the list of members.
For succesful registration, ETH equal to the amount specified in `membershipDeposit * pubkeys.length` MUST be paid while calling this function.
This function MUST fire the `MemberRegistered` event for each registered member.
    * `function registerBatch(uint256[] calldata pubkeys) external payable`


* withdraw: This function accepts the `secret` of the public key at index`_pubkeyIndex` to withdraw the ETH equal to the`membershipDeposit` to the `receiver` address.
This function MUST remove the associated member from the list of members.
This function MUST fire the `MemberWithdrawn` event.
    * `function withdraw(uint256 secret, uint256 _pubkeyIndex, address payable receiver) external`


* withdrawBatch: This function accepts multiple `secrets` of the public keys at the indices `pubkeyIndexes` to withdraw the ETH equal to the `membershipDeposit * secrets.length` to the `receiver` address.
This function MUST remove the associated members from the list of members.
This function MUST fire the `MemberWithdrawn` event for each member slashed from the group.
    * `function withdrawBatch(uint256[] calldata secrets, uint256[] calldata pubkeyIndexes, address payable[] calldata receivers) external`


### Events

* MemberRegistered: MUST trigger when a member is registered to the RLN group.
    * `event MemberRegistered(uint256 indexed pubkey, uint256 indexed index)`


* MemberWithdrawn: MUST  trigger when a member is slashed from the RLN group.
    * `event MemberWithdrawn(uint256 indexed pubkey, uint256 indexed index)`


# Example Implementation

Complete implementation of this contract can be found at https://github.com/vacp2p/rln-contract

The contract for Poseidon Hasher can be found at https://github.com/vacp2p/rln-contract/blob/main/contracts/PoseidonHasher.sol

## References

1. [17/WAKU-RLN-RELAY](https://rfc.vac.dev/spec/17/)
2. [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/)
3. [32/RLN](https://rfc.vac.dev/spec/32/)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

