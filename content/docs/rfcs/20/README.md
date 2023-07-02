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

- Public Key Broadcast: `/eth-pm/2/public-key/proto`,

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

- `0xA` is Alice's Ethereum address (or account),
- `0xB` is Bob's Ethereum address (or account),
- `sA` is Bob's public static key,
- `sB` is Bob's public static key,
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
8. Noise Protocols for Waku Payload Encryption](/spec/35/) is used for key exchange and encryption purposes.

## Limitations

(TODO: Check if still true) Private messages are sent on the same content topic for all users.
As the recipient data is encrypted, all participants must decrypt all messages which can lead to scalability issues.

TODO: Specify what is prrovided re forwarded seccrecy thanks to the noise protocol framework.

TODO: highlight that contact attempt can be revealed if static key is compromised.

Bob MUST decide to participate in the protocol before Alice can send him a message.
This is discussed in more in details in [Consideration for a non-interactive/uncoordinated protocol](#consideration-for-a-non-interactiveuncoordinated-protocol)

# The protocol

## 1. Generate Static Key Pair

All participants to the protocol need to generate a static key pair.

To reduce the likelihood of a user loosing their static key pair, the static key pair SHOULD be generated from the user's Web3 wallet.

For Bob to generate his static key pair, he SHOULD sign the data below using [EIP-712](https://eips.ethereum.org/EIPS/eip-712) v3,
meaning calling `eth_signTypedData_v3` on his Wallet's API.

Note: While v4 also exists,
it is not available on all wallets and the features brought by v4 is not needed for the current use case.

The `TypedData` to be passed to `eth_signTypedData_v3` MUST be as follows, where:

- `dAppFqdn`: The FQDN of the dApp requesting the signature. 

```js
const typedData = {
    domain: {
        chainId: 1,
        name: dAppFqdn,
        version: '1',
    },
    message: {
        disclaimer: "The signature will be used to generate your static encryption keys, do ensure the domain name in the signature matches the URL of this dApp",
    },
    primaryType: 'GenerateStaticKeyPair',
    types: {
        EIP712Domain: [
            { name: 'name', type: 'string' },
            { name: 'version', type: 'string' },
            { name: 'chainId', type: 'uint256' },
        ],
        GenerateStaticKeyPair: [
            { name: 'disclaimer', type: 'string' },
        ],
    },
}
```

The resulting signature `sig` MUST then be hashed using `SHA-256` to generate Bob's static private key: `secretB = sha256(sig)`.
Finally, Bob's static public key `sB` can be calculated from `secretB` using Curve25519 algorithm.

## 2. Broadcast Static Public Key

Participants MAY broadcast their static public key associated to their Ethereum address to be reachable.
In the scenario covered by this RFC, Bob MUST broadcast their static public key `SB` to be reachable by Alice.

### Sign Encryption Public Key

To prove ownership of `sB`,
Bob MUST sign it using [EIP-712](https://eips.ethereum.org/EIPS/eip-712) v3,
meaning calling `eth_signTypedData_v3` on his Wallet's API.

Note: While v4 also exists,
it is not available on all wallets and the features brought by v4 is not needed for the current use case.

The `TypedData` to be passed to `eth_signTypedData_v3` MUST be as follows, where:

- `sB` is Bob's static public key, in hex format, **without** `0x` prefix.
- `bobAddress` is Bob's Ethereum address, `0xB`, in hex format, **with** `0x` prefix.
- `dAppFqdn`: The FQDN of the dApp requesting the signature.

```js
const typedData = {
    domain: {
      chainId: 1,
      name: dAppFqdn,
      version: '1',
    },
    message: {
      staticPublicKey: sB,
      ownerAddress: bobAddress,
    },
    primaryType: 'PublishStaticPublicKey',
    types: {
      EIP712Domain: [
        { name: 'name', type: 'string' },
        { name: 'version', type: 'string' },
        { name: 'chainId', type: 'uint256' },
      ],
        PublishStaticPublicKey: [
        { name: 'staticPublicKey', type: 'string' },
        { name: 'ownerAddress', type: 'string' },
      ],
    },
  }
```

### Static Public Key Message

The resulting signature is then included in a `PublicKeyMessage`, where

- `static_public_key` is Bob's static public key `sB`, not compressed,
- `eth_address` is Bob's Ethereum Address `0xB`,
- `signature` is the EIP-712 as described above.

```protobuf
syntax = "proto3";

message PublicKeyMessage {
   bytes static_public_key = 1;
   bytes eth_address = 2;
   bytes signature = 3;
}
```

This MUST be wrapped in a Waku Message version 0, with the Public Key Broadcast content topic.

### Consideration for a non-interactive/uncoordinated protocol

Alice has to get Bob's public Key to send a message to Bob.
Because an Ethereum Address is part of the hash of the public key's account,
it is not enough in itself to deduce Bob's Public Key.

This is why the protocol dictates that Bob MUST send his Public Key first,
and Alice MUST receive it before she can send him a message.

Moreover, WAKU STORE only stores messages for a limited period of time (as setup by the node's operator).
This means that Bob would need to broadcast his public key on a regular basis to ensure Alice can access it by querying
a store node.

Below we are reviewing possible solutions to mitigate this "sign up" step.

#### Retrieve the Ethereum Public Key from The Blockchain

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

#### Publishing the Public Key in Long Term Storage

Another improvement would be for Bob not having to re-publish his public key every 30 days or less.
Similarly to above, if Bob stops publishing his public key then it may be an indication that he does not participate in the protocol anymore.

In any case, the protocol could be modified to store the Public Key in a more permanent storage, such as a dedicated smart contract on the blockchain.

## 3. `XK1` Handshake

Alice initiates a `XK1` handshake.

```
 XK1:
    <-  s (Bob Broadcast Static Public Key)
       ...
    ->  e
    <-  e, ee, es
    ->  s, se (Alice to include Public Key Message with signature)
    
(): comment
```

TODO: describe the protocol

TODO: define content topic

Once Bob has received Alice static key `sA` and her `PublicKeyMessage`,
he MUST verify that `sA` received via the handshake is the same as `static_public_key` from `PublicKeyMessage`.

Once verified, the application can show this information to the user as a contact request feature coming from Alice's
Ethereum address `0xA`.

Note: Both parties having to exchange their Ethereum addresses to communicate is a privacy compromise.
55/STATUS-1TO1-CHAT SHOULD be preferred if the application does not want to force users to reveal their address.

Note: The handshake must be completed before Bob can _block_ Alice based on her Ethereum address.
To reduce potential unnecessary handshake runs, then Alice could share her `PublicKeyMessage` as a first step.
However, to stop any other party to use `Alice`'s `PublicKeyMessage` to initiate a handshake with Bob,
this message should include recipient's details (Bob's address) in the signed data.
This would mean that Alice would need to sign a message with her wallet before a contact request, slightly worsening the UX.

##  4. Send Private Message

Using Bob's _ephemeral key_ received during the `XK1` handshake, Alice can now send an encrypted message to Bob  's Encryption Public Key, retrieved via [10/WAKU2](/spec/13), Alice MAY now send an encrypted message to Bob as defined in 35/WAKU2-NOISE.

TODO: Define content topic

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
