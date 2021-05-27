---
slug: 18
title: 18/WAKU2-SWAP
name: Waku SWAP Accounting
status: raw
editor: Oskar Thorén <oskar@status.im>
contributor: Ebube Ud <ebube@status.im>
---

This specification outlines how we do accounting and settlement based on the provision and usage of resources, most immediately bandwidth usage and/or storing and retrieving of Waku message. This enables nodes to cooperate and efficiently share resources, and in the case of unequal nodes to settle the difference through a relaxed payment mechanism in the form of sending cheques.

**Protocol identifier***: `/vac/waku/swap/2.0.0-beta1`

# Motivation

The Waku network makes up a service network, and some nodes provide a useful service to other nodes. We want to account for that, and when imbalances arise, settle this. The core of this approach has some theoretical backing in game theory, and variants of it have practically been proven to work in systems such as Bittorrent. The specific model use was developed by the Swarm project (previously part of Ethereum), and we re-use contracts that were written for this purpose.

By using a delayed payment mechanism in the form of cheques, a barter-like mechanism can arise, and nodes can decide on their own policy as opposed to be strictly tied to a specific payment scheme. Additionally, this delayed settlement eases requirements on the underlying network in terms of transaction speed or costs.

Theoretically, nodes providing and using resources over a long, indefinite, period of time can be seen as an iterated form of [Prisoner's Dilemma (PD)](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma). Specifically, and more intuitively, since we have a cost and benefit profile for each provision/usage (of Waku Message's, e.g.), and the pricing can be set such that mutual cooperation is incentivized, this can be analyzed as a form of donations game.

# Game Theory - Iterated prisoner's dilemma / donation game

What follows is a sketch of what the game looks like between two nodes. We can look at it as a special case of iterated prisoner's dilemma called a [Donation game](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma#Special_case:_donation_game) where each node can cooperate with some benefit `b` at a personal cost `c`, where `b>c`.

From A's point of view:

A/B | Cooperate | Defect
-----|----------|-------
Cooperate | b-c | -c
Defect | b | 0

What this means is that if A and B cooperates, A gets some benefit `b` minus a cost `c`. If A cooperates and B defects she only gets the cost, and if she defects and B cooperates A only gets the benefit. If both defect they get neither benefit nor cost.

The generalized form of PD is:

A/B | Cooperate | Defect
-----|----------|-------
Cooperate | R | S
Defect | T | P

With R=reward, S=Sucker's payoff, T=temptation, P=punishment

And the following holds:

- `T>R>P>S`
- `2R>T+S`

In our case, this means `b>b-c>0>-c` and `2(b-c)> b-c` which is trivially true.

As this is an iterated game with no clear finishing point in most circumstances, a tit-for-tat strategy is simple, elegant and functional. To be more theoretically precise, this also requires reasonable assumptions on error rate and discount parameter. This captures notions such as "does the perceived action reflect the intended action" and "how much do you value future (uncertain) actions compared to previous actions". See [Axelrod - Evolution of Cooperation (book)](https://en.wikipedia.org/wiki/The_Evolution_of_Cooperation) for more details. In specific circumstances, nodes can choose slightly different policies if there's a strong need for it. A policy is simply how a node chooses to act given a set of circumstances.

A tit-for-tat strategy basically means:
- cooperate first (perform service/beneficial action to other node)
- defect when node stops cooperating (disconnect and similar actions), i.e. when it stops performing according to set parameters re settlement
- resume cooperation if other node does so

This can be complemented with node selection mechanisms.

# SWAP protocol overview

We use SWAP for accounting and settlement in conjunction with other request/reply protocols in Waku v2,
where accounting is done in a pairwise manner.
It is an acronym with several possible meanings (as defined in the Book
of Swarm), for example:

- service wanted and provided
- settle with automated payments
- send waiver as payment
- start without a penny

This approach is based on communicating payment thresholds and sending cheques as indications of later payments.
Communicating payment thresholds MAY be done out-of-band or as part of the handshake.
Sending cheques is done once payment threshold is hit.

See [Book of Swarm](https://web.archive.org/web/20210126130038/https://gateway.ethswarm.org/bzz/latest.bookofswarm.eth) section 3.2. on Peer-to-peer accounting etc., for more context and details.

## Accounting

Nodes perform their own accounting for each relevant peer based on some "volume"/bandwidth metric. For now we take this to mean the number of `WakuMessage`s exchanged.

Additionally, a price is attached to each unit. Currently, this is simply a "karma counter" and equal to 1 per message.

Each accounting balance SHOULD be w.r.t. to a given protocol it is accounting for.

NOTE: This may later be complemented with other metrics, either as part of SWAP or more likely outside of it. For example, online time can be communicated and attested to as a form of enhanced quality of service to inform peer selection.

## Flow

Assuming we have two store nodes, one operating mostly as a client (A) and another as server (B).

1. Node A performs a handshake with B node. B node responds and both nodes communicate their payment threshold.
2. Node A and B creates an accounting entry for the other peer, keep track of peer and current balance.
3. Node A issues a `HistoryRequest`, and B responds with a `HistoryResponse`. Based on the number of WakuMessages in the response, both nodes update their accounting records.
4. When payment threshold is reached, Node A sends over a cheque to reach a neutral balance. Settlement of this is currently out of scope, but would occur through a SWAP contract (to be specified). (mock and hard phase).
5. If disconnect threshold is reached, Node B disconnects Node A (mock and hard phase).

Note that not all of these steps are mandatory in initial stages, see below for more details. For example, the payment threshold MAY initially be set out of bounds, and policy is only activated in the mock and hard phase.

## Protobufs

We use protobuf to specify the handshake and signature. This current protobuf is a work in progress. This is needed for mock and hard phase.

A handshake gives initial information about payment thresholds and possibly other information. A cheque is best thought of as a promise to pay at a later date.

```
message Handshake {
    bytes payment_threshold = 1;
}

// TODO Signature?
// Should probably be over the whole Cheque type
message Cheque {
    bytes beneficiary = 1;
    // TODO epoch time or block time?
    uint32 date = 2;
    // TODO ERC20 extension?
    // For now karma counter
    uint32 amount = 3;
}
```

# Incremental integration and roll-out

To incrementally integrate this into Waku v2, we have divided up the roll-out into three phases:

- Soft - accounting only
- Mock - send mock cheques and take word for it
- Hard Test - blockchain integration and deployed to public testnet (Goerli, Optimism testnet or similar)
- Hard Main - deployed to a public mainnet

An implementation MAY support any of these phases.

## Soft phase

In the soft phase only accounting is performed, without consequence for the
peers. No disconnect or sending of cheques is performed at this tage.

SWAP protocol is performed in conjunction with another request-reply protocol to account for its usage.
It SHOULD be done for [13/WAKU2-STORE](/spec/13/)
and it MAY be done for other request/reply protocols.

A client SHOULD log accounting state per peer
and SHOULD indicate when a peer is out of bounds (either of its thresholds met).

## Mock phase

In the mock phase, we send mock cheques and send cheques/disconnect peers as appropriate.

- If a node reaches a disconnect threshold, which MUST be outside the payment threshold, it SHOULD disconnect the other peer.
- If a node is within payment balance, the other node SHOULD stay connected to it.
- If a node receives a valid Cheque it SHOULD update its internal accounting records.
- If any node behaves badly, the other node is free to disconnect and pick another node.
  - Peer rating is out of scope of this specification.

## Hard phase

In the hard phase, in addition to sending cheques and activating policy, this is
done with blockchain integration on a public testnet. More details TBD.

This also includes settlements where cheques can be redeemed.

## References

1. [Prisoner's Dilemma](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma)

2. [Donation game](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma#Special_case:_donation_game)

3. [Axelrod - Evolution of Cooperation (book)](https://en.wikipedia.org/wiki/The_Evolution_of_Cooperation)

4. [Book of Swarm](https://web.archive.org/web/20210126130038/https://gateway.ethswarm.org/bzz/latest.bookofswarm.eth/)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).


<!--

General TODOs:

- Find new link for book of swarm
- Illustrate payment and disconnection thresholds (mscgen not great for this?)
- Elaborate on how accounting works with amount in the context of e.g. store
- Illustrate flow
- Specify chequeboo

-->
