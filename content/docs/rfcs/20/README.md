---
slug: 20
title: 20/ETH-DM
name: Ethereum Direct Message
status: raw
editor: Franck Royer <franck@status.im>
contributors:
---

**Content Topics**:

- Public Key Broadcast: `/eth-dm/1/public-key/json`,
- Direct Message: `/eth-dm/1/direct-message/json`.

This specification explains the Ethereum Direct Message protocol
which enables a peer to send a direct message to another peer
using the Waku v2 network, and the peer's Ethereum address.

The main purpose of this specification is to demonstrate how Waku v2 can be used for direct messaging purposes.
In the current state, the protocol has privacy and features [limitations](#limitations),
we hope this can be an inspiration for developers wishing to build on top of Waku v2.

# Goal

Alice wants to send an encrypted message to Bob, where only Bob can decrypt the message.
Alice only knows Bob's Ethereum Address.

# Variables

Here are the variables used in the protocol and their definition:

- `B` is Bob's Ethereum address (or account),
- `b` is the private key of `B`, and is only known by Bob.
- `B'` is Bob's Eth-DM Public Key, which has `b'` as private key.

# Design Requirements

The proposed protocol MUST adhere to the following design requirements:

1. Alice knows Bob's Ethereum address, 
1. Bob is willing to participate to Eth-DM, and publishes `B'`, 
1. Alice wants to send message `M` to Bob,
1. Bob SHOULD be able to get `M` using [10/WAKU2](/spec/13),
1. Participants only have access to their Ethereum Wallet via the Web3 API,
1. Carole MUST NOT be able to read `M`'s content even if she is storing it or relaying it,
1. ECDSA Elliptic curve cryptography is used,
1. [eth-crypto](https://www.npmjs.com/package/eth-crypto),
   which uses [eccrypto](https://www.npmjs.com/package/eccrypto),
   is used for encryption and decryption purposes.

## Limitations

Bob's Ethereum Address is present in clear in Direct Messages,
meaning that anyone who is monitoring the Waku network know how many messages Bob receives.

Alice's details are not included in the message's structure,
meaning that there is no programmatic way for Bob to reply to Alice
or verify her identity.

# The protocol

## Eth-DM Key Generation

First, Bob needs to generate an Eth-DM keypair.
To avoid Bob having to save an additional private key or recovery phrase for Eth-DM purposes,
we generate the Eth-DM keypair using Bob's Ethereum account.
This will allow Bob to recover his Eth-DM private key as long as he has access to his Ethereum private key. 


To generate his Eth-DM keypair, Bob MUST use his Ethereum private key 'b' to sign the Eth-DM salt message:
   `Salt for Eth-Dm, do not share a signature of this message or others could decrypt your messages`.

The resulting signature 's' is then concatenated with itself once and hashed using keccak256.
The resulting hash is Bob's Eth-DM private key `b'`:

```
b' = keccak256(s + s)
```

The signature process is as per the current Ethereum best practice:

1. Convert the salt message to a byte array using utf-8 encoding,
2. Use [`eth_sign`](https://eth.wiki/json-rpc/API#eth_sign) Ethereum JSON-RPC command or equivalent.

# Eth-DM Public Key Broadcast

For Bob to be reachable, he SHOULD broadcast his Eth-DM Public Key `B'`.
To prove that he is indeed the owner of his Ethereum account `B`, he MUST sign his Eth-DM Public Key.

To do so, Bob MUST format his Public Key to lower case hex (no prefix) in a JSON Object on the property `ethDmPublicKey`, e.g.:

```json
{
   "ethDmPublicKey": "abcd...0123"
}
```

Then, Bob MUST sign the stringified JSON using [`eth_sign`](https://eth.wiki/json-rpc/API#eth_sign).
This results in the `sigEthDmPubKey` signature.

Bob then creates an Eth-DM Public Key Message containing:

- Bob's Eth-DM Public Key `B'` on property `ethDmPublicKey`,
- Bob's Ethereum Address `B` on property `ethAddress`,
- The signature of Bob's Eth-DM Public Key `sigEthDmPubKey` on property `sig`.

In JSON format as follows:

```json
{
   "ethDmPublicKey": "abcd...0123",
   "ethAddress": "0x01234...eF",
   "sig": "0x2eb...a1b"
}
```

Finally, Bob SHOULD publish the message on Waku v2 with the Public Key Broadcast content topic. 

# Send Direct Message

Alice MAY monitor the Waku v2 to collect Ethereum Address/Eth-DM Public Key tuples.
Alice SHOULD verify that the `sig` property of said message contains a valid signature as constructed above.
She SHOULD drop any message without a signature or with an invalid signature.

Using Bob's Eth-DM Public Key, retrieved via [10/WAKU2](/spec/13), Alice MAY now send an encrypted message to Bob.

If she wishes to do so, Alice MUST encrypt her message `M` using Bob's Eth-DM Public Key `B'`.

The result of the encryption is as follows
(see [eth-crypto's encryptWithPublicKey](https://www.npmjs.com/package/eth-crypto#encryptwithpublickey)):

```json
{
   "iv": "...",
   "ephemPublicKey": "...",
   "ciphertext": "...",
   "mac": "..."
}
```

Alice MUST then set this result in a Direct Message's property `encMessage`,
with Bob's Ethereum address `B` set to property `toAddress`.

```json
{
   "toAddress": "...",
   "encMessage": {
      "iv": "...",
      "ephemPublicKey": "...",
      "ciphertext": "...",
      "mac": "..."
   }
}
```

Alice SHOULD now publish this message on the Direct Message content topic.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
