---
slug: 20
title: 20/ETH-DM
name: Ethereum Direct Message
status: raw
editor: Franck Royer <franck@status.im>
contributors:
---

This specification explains the Ethereum Direct Message protocol
which enables a peer to send a direct message to another peer
using the Waku v2 network, and the peer's Ethereum address.

# Goal

Alice wants to send an encrypted message to Bob, where only Bob can decrypt the message.

# Variables

Here are the variables used in the protocol and their definition:

- `A` is Alice's Ethereum root HD public key (also named account, address),
- `B` is Bob's Ethereum root HD public key (also named account, address),
- `a` is the private key of `A`, and is only known by Alice,
- `b` is the private key of `B`, and is only known by Bob.

# Design Requirements

The proposed protocol MUST adhere to the following design requirements:

1. Alice knows Bob's Ethereum root HD public key `B`,
2. Alice wants to send message `M` to Bob,
3. Bob SHOULD be able to get `M` using [13/WAKU2-STORE](/spec/13), when querying a store node that hosts `M`,
4. Bob MUST recognize he is `M`'s recipient when relaying it via [11/WAKU2-RELAY](/spec/11),
5. Carole MUST NOT be able to read `M`'s content even if she is storing it or relaying it.

## Out of scope

At this stage, we acknowledge that:

1. Because `Bw` is part of the message in clear,
Alice can know how many messages from other parties Bob receive,
and Carole can see that how many messages a recipient `Bw` is receiving (unlinkability is broken).

# Steps

1. Alice MUST derive Bob's waku public Key `Bw` from `B`,
2. Alice SHOULD derive her own waku public key `Aw` from `A`,
3. Alice creates `M'` which MUST contain `M` and `Aw`, it MAY contain `A`,
4. Alice encrypts `M'` using `Bw`, resulting in `m'`,
5. Alice creates waku message `Mw` with
   `payload` `m'` and
   `contentTopic` `/waku/2/direct-message/eth-pubkey/Bw/proto`,
   with `Bw` in hex format (`0xAb1..`),
6. Alice publishes `Mw` via [11/WAKU2-RELAY](/spec/11),
7. Bob captures received messages via [11/WAKU2-RELAY](/spec/11) that have `contentTopic` `/waku/2/direct-message/eth-pubkey/Bw/proto`,
8. Bob queries [13/WAKU2-STORE](/spec/13) with `contentTopics` set to `["/waku/2/direct-message/eth-pubkey/Bw/proto"]`,
9. When retrieving `Mw` Bob derives `bw` from `b`,
10. Bob uses `bw` to decrypt message `Mw`, he learns `m` and `Aw`,
11. Bob replies to Alice in the same manner, setting the `contentTopic` to `/waku/2/direct-message/eth-pubkey/Aw/proto`.

## Derivation

Public parent key (`B`) to public child key (`Bw`) derivation is only possible with non-hardened paths [\[1\]](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

TODO: Investigate commonly used derivation path to decide on one.

## Encryption

TODO: Define encryption method to use.

## Reply

To ascertain the fact that Alice receives Bob's reply, she could include connection details such as her peer id and multiaddress in the message.
However, this leads to privacy concerns if she does not use an anonymizing network such as tor.

Because of that, Alice only includes `Aw` in `M'`.

## Message retrieval

To satisfy (c) and (d), we are using the `contentTopic` as a low-computation way (for Bob) to retrieve messages.

Using a prefix such as `direct-message/eth-pubkey` reduces possible conflicts with other use cases that would also use a key or 32 byte array.

We could also consider adding a version to allow an evolution of the field and its usage, e.g. `/waku/2/direct-message/eth-pubkey/1/Aw/proto`

TODO: Point to spec recommending formatting of `contentTopic`, currently tracked in issue [#364](https://github.com/vacp2p/rfc/issues/364) [2].

# Payloads

```protobuf
syntax = "proto3";

message DirectMessage {
  DirectMessageContent message = 1; // `M`
  bytes sender_waku_public_key = 2; // `Aw`
  bytes sender_root_public_key = 3; // `A`
}

message DirectMessageContent {
  bytes message = 1;
  string encoding = 2; // Encoding of the message, utf-8 if not present.
}
```

# References

- [\[1\] https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
- [\[2\] https://github.com/vacp2p/rfc/issues/364](https://github.com/vacp2p/rfc/issues/364)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
