---
title: Waku SWAP Accounting
version: 2.0.0-alpha1
status: Raw
authors: Oskar Thorén <oskar@status.im>
---

# Table of Contents

TODO.

# Abstract

This specification provides a way to do accounting based on the SWAP model proposed in Swarm. This is used for pairwise accounting to get nodes to cooperate and provide useful service and/or exchange of funds.

TODO: Elaborate more here

TODO: Add link

**Protocol identifier***: `/vac/waku/swap/2.0.0-alpha1`

# Motivation

The Waku network makes up a service network, and some nodes provide a useful service. We want to account for that, laying the foundation for settlement. Furthermore...

TODO: Fill in more

TODO: Game theoretical foundation, Bittorrent, relaxed tit-for-tat

# SWAP Accounting

See [Book of Swarm](TODO) section 3.2. on Peer-to-peer accounting etc., for more context and details.

This approach is based on communicating payment treshholds and sending cheques
as indications of later payments. The first part is done with a handshake, the
second done once payment threshhold is hit.

TODO: Illustrate payment and disconnection treshholds

TODO: Illustrate flow

A cheque is best thought of as a promise to pay at a later date.

TODO Specify chequebook

## Enhancements

1. Settlements. At a later stage, these cheques can be redeemed. For now they
are better thought of as a form of Karma.

2. More information to make a decision. E.g. incorporate QoS metrics such as
   online time to inform peer selection (slightly orthogonal to this).

## Protobuf

```
// TODO What information to include here? For now "payment" threshhold
message Handshake {
    bytes payment_threshhold = 1;
}

// TODO Signature?
message Cheque {
    bytes beneficiary = 1;
    // TODO epoch time or block time?
    uint32 date = 2;
    // TODO ERC20 extension?
    // For now karma counter
    uint32 amount = 3;
}

```

# Changelog

TBD.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
