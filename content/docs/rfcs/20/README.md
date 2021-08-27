---
slug: 20
title: 20/TOY-ETH-PM
name: Toy Ethereum Private Message
status: draft
tags: waku-application
editor: Franck Royer <franck@status.im>
contributors:
---

**Content Topics**:

- Public Key Broadcast: `/eth-pm/1/public-key/proto`,
- Private Message: `/eth-pm/1/private-message/proto`.

This specification explains the Toy Ethereum Private Message protocol
which enables a peer to send an encrypted message to another peer
using the Waku v2 network, and the peer's Ethereum address.

The main purpose of this specification is to demonstrate how Waku v2 can be used for encrypted messaging purposes,
using Ethereum accounts for identity.
This protocol caters for Web3 wallets restrictions, allowing it to be implemented only using standard Web3 API.
In the current state, the protocol has privacy and features [limitations](#limitations), has not been audited
and hence is not fit for production usage.
We hope this can be an inspiration for developers wishing to build on top of Waku v2.

# Goal

Alice wants to send an encrypted message to Bob, where only Bob can decrypt the message.
Alice only knows Bob's Ethereum Address.

# Variables

Here are the variables used in the protocol and their definition:

- `B` is Bob's Ethereum address (or account),
- `b` is the private key of `B`, and is only known by Bob.
- `B'` is Bob's Encryption Public Key, for which `b'` is the private key.
- `M` is the private message that Alice sends to Bob.

# Design Requirements

The proposed protocol MUST adhere to the following design requirements:

1. Alice knows Bob's Ethereum address, 
2. Bob is willing to participate to Eth-PM, and publishes `B'`,
3. Bob's ownership of `B'` MUST be verifiable,
4. Alice wants to send message `M` to Bob,
5. Bob SHOULD be able to get `M` using [10/WAKU2](/spec/13),
6. Participants only have access to their Ethereum Wallet via the Web3 API,
7. Carole MUST NOT be able to read `M`'s content even if she is storing it or relaying it,
8. [Waku Message Version 1](/spec/26/) Asymmetric Encryption is used for encryption purposes.

## Limitations

Alice's details are not included in the message's structure,
meaning that there is no programmatic way for Bob to reply to Alice
or verify her identity.

Private messages are sent on the same content topic for all users.
As the recipient data is encrypted, all participants must decrypt all messages which can lead to scalability issues.

# The protocol

## Generate Encryption KeyPair

First, Bob needs to generate a keypair for Encryption purposes.

Bob SHOULD get 32 bytes from a secure random source as Encryption Private Key, `b'`.
Then Bob can compute the corresponding SECP-256k1 Public Key, `B'`.

# Broadcast Encryption Public Key

For Alice to encrypt messages for Bob,
Bob SHOULD broadcast his Encryption Public Key `B'`.
To prove that the Encryption Public Key `B'` is indeed owned by the owner of Bob's Ethereum Account `B`,
Bob MUST sign `B'` using `B`.

## Sign Encryption Public Key

To prove ownership of the Encryption Public Key,
Bob must sign it using [EIP-712](https://eips.ethereum.org/EIPS/eip-712) v3,
meaning calling `eth_signTypedData_v3` on his Wallet's API.

Note: While v4 also exists,
it is not available on all wallets and the features brought by v4 is not needed for the current use case.

The `TypedData` to be passed to `eth_signTypedData_v3` MUST be as follows, where:

- `encryptionPublicKey` is Bob's Encryption Public Key, `B'`, in hex format, **without** `0x` prefix.
- `bobAddress` is Bob's Ethereum address, corresponding to `B`, in hex format, **with** `0x` prefix.

```js
const typedData = {
    domain: {
      chainId: 1,
      name: 'Ethereum Private Message over Waku',
      version: '1',
    },
    message: {
      encryptionPublicKey: encryptionPublicKey,
      ownerAddress: bobAddress,
    },
    primaryType: 'PublishEncryptionPublicKey',
    types: {
      EIP712Domain: [
        { name: 'name', type: 'string' },
        { name: 'version', type: 'string' },
        { name: 'chainId', type: 'uint256' },
      ],
      PublishEncryptionPublicKey: [
        { name: 'encryptionPublicKey', type: 'string' },
        { name: 'ownerAddress', type: 'string' },
      ],
    },
  }
```

## Public Key Message

The resulting signature is then included in a `PublicKeyMessage`, where

- `encryption_public_key` is Bob's Encryption Public Key `B'`, not compressed,
- `eth_address` is Bob's Ethereum Address `B`,
- `signature` is the EIP-712 as described above.

```protobuf
syntax = "proto3";

message PublicKeyMessage {
   bytes encryption_public_key = 1;
   bytes eth_address = 2;
   bytes signature = 3;
}
```

This MUST be wrapped in a Waku Message version 0, with the Public Key Broadcast content topic.
Finally, Bob SHOULD publish the message on Waku v2. 

# Send Private Message

Alice MAY monitor the Waku v2 to collect Ethereum Address and Encryption Public Key tuples.
Alice SHOULD verify that the `signature`s of `PublicKeyMessage`s she receives are valid as per EIP-712.
She SHOULD drop any message without a signature or with an invalid signature.

Using Bob's Encryption Public Key, retrieved via [10/WAKU2](/spec/13), Alice MAY now send an encrypted message to Bob.

If she wishes to do so, Alice MUST encrypt her message `M` using Bob's Encryption Public Key `B'`,
as per [26/WAKU-PAYLOAD Asymmetric Encryption specs](/spec/26/#asymmetric).

Alice SHOULD now publish this message on the Private Message content topic.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
