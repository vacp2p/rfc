---
slug: 15
title: 15/WAKU-BRIDGE
name: Waku Bridge
status: draft
category: core
editor: Hanno Cornelius <hanno@status.im>
---

A bridge between Waku v1 and Waku v2.

# Bridge

A bridge requires supporting both Waku versions:

* Waku v1 - using devp2p RLPx protocol
* Waku v2 - using libp2p protocols

Packets received on the Waku v1 network SHOULD be published just once on the
Waku v2 network. More specifically, the bridge SHOULD publish
this through the Waku Relay (PubSub domain).

Publishing such packet will require the creation of a new `Message` with a
new `WakuMessage` as data field. The `data` and `topic` field from the Waku v1
`Envelope` MUST be copied to the `payload` and `contentTopic` fields of the
`WakuMessage`. Other fields such as nonce, expiry and ttl will be dropped as
they become obsolete in Waku v2.

Before this is done, the usual envelope verification still applies:

* Expiry & future time verification
* PoW verification
* Size verification

Bridging SHOULD occur through the `WakuRelay`, but it MAY also be done on other Waku
v2 protocols (e.g. `WakuFilter`). The latter is however not advised as it will
increase the complexity of the bridge and because of the
[Security Considerations](#security-considerations) explained further below.

Packets received on the Waku v2 network SHOULD be posted just once on the Waku
v1 network. The Waku v2 `WakuMessage` contains only the `payload` and
`contentTopic` fields. The bridge MUST create a new Waku v1 `Envelope` and
copy over the `payload` and `contentFilter` fields to the `data` and `topic`
fields. Next, before posting on the network, the bridge MUST set a new expiry
and ttl and do the PoW nonce calculation.

### Security Considerations
As mentioned above, a bridge will be posting new Waku v1 envelopes, which
requires doing the PoW nonce calculation.

This could be a DoS attack vector, as the PoW calculation will make it more
expensive to post the message compared to the original publishing on the Waku v2
network. Low PoW setting will lower this problem, but it is likely that it is
still more expensive.

For this reason, bridges SHOULD probably be run independently of other nodes, so
that a bridge that gets overwhelmed does not disrupt regular Waku v2 to v2
traffic.

Bridging functionality SHOULD also be carefully implemented so that messages do
not bounce back and forth between the two networks. The bridge SHOULD properly
track messages with a seen filter so that no amplification can be achieved here.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
