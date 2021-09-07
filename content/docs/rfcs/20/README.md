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

This protocol does not guarantee Perfect Forward Secrecy nor Future Secrecy:
If Bob's private key is compromised, past and future messages could be decrypted.
A solution combining regular [X3DH](https://www.signal.org/docs/specifications/x3dh/)
bundle broadcast with [Double Ratchet](https://signal.org/docs/specifications/doubleratchet/) encryption would remove these limitations;
See the [Status secure transport spec](https://specs.status.im/spec/5) for an example of a protocol that achieves this in a peer-to-peer setting.

Bob MUST decide to participate in the protocol before Alice can send him a message.
This is discussed in more in details in [Consideration for a non-interactive/uncoordinated protocol](#consideration-for-a-non-interactiveuncoordinated-protocol)

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

## Consideration for a non-interactive/uncoordinated protocol

Alice has to get Bob's public Key to send a message to Bob.
Because an Ethereum Address is part of the hash of the public key's account,
it is not enough in itself to deduce Bob's Public Key.

This is why the protocol dictates that Bob MUST send his Public Key first,
and Alice MUST receive it before she can send him a message.

Moreover, nim-waku, the reference implementation of [13/WAKU2-STORE](/spec/13/),
stores messages for a maximum period of 30 days.
This means that Bob would need to broadcast his public key at least every 30 days to be reachable.

Below we are reviewing possible solutions to mitigate this "sign up" step.

### Retrieve the public key from the blockchain

If Bob has signed at least one transaction with his account then his Public Key can be extracted from the transaction's ECDSA signature.
The challenge with this method is that standard Web3 Wallet API does not allow Alice to specifically retrieve all/any transaction sent by Bob.

Alice would instead need to use the `eth.getBlock` API to retrieve Ethereum blocks one by one.
For each block, she would need to check the `from` value of each transaction until she finds a transaction sent by Bob.

This process is resource intensive and can be slow when using services such as Infura due to rate limits in place,
which makes it inappropriate for a browser or mobile phone environment.

An alternative would be to either run a backend that can connect directly to an Ethereum node,
use a centralized blockchain explorer
or use a decentralized indexing service such as [The Graph](https://thegraph.com/).

Note that these would resolve a UX issue only if a sender wants to proceed with _air drops_.

Indeed, if Bob does not publish his Public Key in the first place
then it can be an indication that he simply does not participate in this protocol and hence will not receive messages.

However, these solutions would be helpful if the sender wants to proceed with an _air drop_ of messages:
Send messages over Waku for users to retrieve later, once they decide to participate in this protocol.
Bob may not want to participate first but may decide to participate at a later stage
and would like to access previous messages.
This could make sense in an NFT offer scenario:
Users send offers to any NFT owner,
NFT owner may decide at some point to participate in the protocol and retrieve previous offers.

### Publishing the public in long term storage

Another improvement would be for Bob not having to re-publish his public key every 30 days or less.
Similarly to above, if Bob stops publishing his public key then it may be an indication that he does not participate in the protocol anymore.

In any case, the protocol could be modified to store the Public Key in a more permanent storage, such as a dedicated smart contract on the blockchain.

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
