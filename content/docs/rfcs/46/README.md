---
slug: 46
title: 46/GOSSIPSUB-TOR-PUSH
name: Gossipsub Tor Push
status: raw
category: Standards Track
tags:
editor: Daniel Kaiser <danielkaiser@status.im>
contributors:
---

# Abstract

This document extends the [libp2p gossipsub specification](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
specifying gossipsub Tor Push,
a gossipsub-internal way of pushing messages into a gossipsub network via Tor.
Tor Push adds sender identity protection to gossipsub.

**Protocol identifier**: /meshsub/1.1.0

Note: Gossipsub Tor Push does not have a dedicated protocol identifier.
It uses the same identifier as gossipsub and works with all [pubsub](https://github.com/libp2p/specs/tree/master/pubsub) based protocols.
This allows nodes that are oblivious to Tor Push to process messages received via Tor Push.

# Background

Without extensions, [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
does not protect sender identities.

A possible design of an anonymity extension to gossipsub is pushing messages through an anonymization network before they enter the gossipsub network.
[Tor](https://www.torproject.org/) is currently the largest anonymization network.
It is well researched and works reliably.
Basing our solution on Tor both inherits existing security research, as well as allows for a quick deployment.

Using the anonymization network approach, even the first gossipsub node that relays a given message cannot link the message to its sender (within a relatively strong adversarial model).
Taking the low bandwidth overhead and the low latency overhead into consideration, Tor offers very good anonymity properties.

# Functional Operation

Tor Push allows nodes to push messages over Tor into the gossipsub network.
The approach specified in this document is fully backwards compatible.
Gossipsub nodes that do not support Tor Push can receive and relay Tor Push messages,
because Tor Push uses the same Protocol ID as gossipsub.

Messages are sent over Tor via [SOCKS5](https://www.rfc-editor.org/rfc/rfc1928).
Tor Push uses a dedicated libp2p context to prevent information leakage.
To significantly increase resilience and mitigate circuit failures,
Tor Push establishes several connections, each to a different randomly selected gossipsub node.

# Specification

This section specifies the format of Tor Push messages, as well as how Tor Push messages are received and sent, respectively.

## Wire Format

The wire format of a Tor Push message corresponds verbatim to a typical [libp2p pubsub message](https://github.com/libp2p/specs/tree/master/pubsub#the-message).

```
message Message {
  optional string from = 1;
  optional bytes data = 2;
  optional bytes seqno = 3;
  required string topic = 4;
  optional bytes signature = 5;
  optional bytes key = 6;
}
```

## Receiving Tor Push Messages

Any node supporting a protocol with ID `/meshsub/1.1.0` (e.g. gossipsub), can receive Tor Push messages.
Receiving nodes are oblivious to Tor Push and will process incoming messages according to the respective `meshsub/1.1.0` specification.

## Sending Tor Push Messages

In the following, we refer to nodes sending Tor Push messages as Tp-nodes (Tor Push nodes).

Tp-nodes MUST setup a separate libp2p context, i.e. [libp2p switch](https://docs.libp2p.io/concepts/multiplex/switch/),
which MUST NOT be used for any purpose other than Tor Push.
We refer to this context as Tp-context.
The Tp-context MUST NOT share any data, e.g. peer lists, with the default context.

Tp-peers are peers a Tp-node plans to send Tp-messages to.
Tp-peers MUST support `/meshsub/1.1.0`.
For retrieving Tp-peers, Tp-nodes SHOULD use an ambient peer discovery method that retrieves a random peer sample (from the set of all peers), e.g. [33/WAKU2-DISCV5](/spec/33/).

Tp-nodes MUST establish a connection as described in sub-section [Tor Push Connection Establishment](#connection-establishment) to at least one Tp-peer.
To significantly increase resilience, Tp-nodes SHOULD establish Tp-connections to `D` peers,
where `D` is the [desired gossipsub out-degree](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md#parameters),
with a default value of `8`.

Each Tp-message MUST be sent via the Tp-context over at least one Tp-connection.
To increase resilience, Tp-messages SHOULD be sent via the Tp-context over all available Tp-connections.

Control messages of any kind, e.g. gossipsub graft, MUST NOT be sent via Tor Push.

### Connection Establishment

Tp-nodes establish a `/meshsub/1.1.0` connection to tp-peers via [SOCKS5](https://www.rfc-editor.org/rfc/rfc1928) over [Tor](https://www.torproject.org/).

Establishing connections, which in turn establishes the respective Tor circuits, can be done ahead of time.

### Epochs

Tor Push introduces epochs.
The default epoch duration is 10 minutes.
(We might adjust this default value based on experiments and evaluation in future versions of this document.
It seems a good trad-off between traceablity and circuit building overhead.)

For each epoch, the Tp-context SHOULD be refreshed, which includes

* libp2p peer-ID
* Tp-peer list
* connections to Tp-peers

Both Tp-peer selection for the next epoch and establishing connections to the newly selected peers SHOULD be done during the current epoch
and be completed before the new epoch starts.
This avoids adding latency to message transmission.

# Security/Privacy Considerations

## Fingerprinting Attacks

Protocols that feature distinct patterns are prone to fingerprinting attacks when using them over Tor Push.
Both malicious guards and exit nodes could detect these patterns
and link the sender and receiver, respectively, to transmitted traffic.
As a mitigation, such protocols can introduce dummy messages and/or padding to hide patterns.

## DoS

### General DoS against Tor

Using untargeted DoS to prevent Tor Push messages from entering the gossipsub network would cost vast resources,
because Tor Push transmits messages over several circuits and the Tor network is well established.

### Targeting the Guard

Denying the service of a specific guard node blocks Tp-nodes using the respective guard.
Tor guard selection will replace this guard [TODO elaborate].
Still, messages might be delayed during this window which might be critical to certain applications.

### Targeting the Gossipsub Network

Without sophisticated rate limiting (for example using [17/WAKU2-RLN-RELAY](/spec/17)),
attackers can spam the gossipsub network.
It is not enough to just block peers that send too many messages,
because these messages might actually come from a Tor exit node that many honest Tp-nodes use.
Without Tor Push, protocols on top of gossipsub could block peers if they exceed a certain message rate.
With Tor Push, this would allow the reputation-based DoS attack described in
[Bitcoin over Tor isn't a Good Idea](https://ieeexplore.ieee.org/abstract/document/7163022).

### Peer Discovery

The discovery mechanism could be abused to link requesting nodes to their Tor connections to discovered nodes.
An attacker that controls both the node that responds to a discovery query,
and the node who’s ENR the response contains,
can link the requester to a Tor connection that is expected to be opened to the node represented by the returned ENR soon after.

Further, the discovery mechanism (e.g. discv5) could be abused to distribute disproportionately many malicious nodes.
For instance if p% of the nodes in the network are malicious,
an attacker could manipulate the discovery to return malicious nodes with 2p% probability.
The discovery mechanism needs to be resilient against this attack.

## Roll-out Phase

During the roll-out phase of Tor Push, during which only a few nodes use Tor Push,
attackers can narrow down the senders of Tor messages to the set of gossipsub nodes that do not originate messages.
Nodes who want anonymity guarantees even during the roll-out phase can use separate network interfaces for their default context and Tp-context, respectively.
For the best protection, these contexts should run on separate physical machines.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

* [libp2p gossipsub](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/README.md)
* [libp2p pubsub](https://github.com/libp2p/specs/tree/master/pubsub)
* [libp2p pubsub message](https://github.com/libp2p/specs/tree/master/pubsub#the-message)
* [libp2p switch](https://docs.libp2p.io/concepts/multiplex/switch)
* [SOCKS5](https://www.rfc-editor.org/rfc/rfc1928)
* [Tor](https://www.torproject.org/)
* [33/WAKU2-DISCV5](/spec/33/)
* [Bitcoin over Tor isn't a Good Idea](https://ieeexplore.ieee.org/abstract/document/7163022)
* [17/WAKU2-RLN-RELAY](/spec/17)



