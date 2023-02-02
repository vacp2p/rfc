---
slug: 54
title: 54/WAKU2-X3DH-SESSIONS
name: Session management for Waku X3DH 
status: stable
category: Standards Track
tags: waku-application
editor: Aaryamann Challani <aaryamann@status.im>
contributors:
- Andrea Piana <andreap@status.im>
- Pedro Pombeiro <pedro@status.im>
- Corey Petty <corey@status.im>
- Oskar Thorén <oskar@status.im>
- Dean Eigenmann <dean@status.im>
---

# Abstract

In [53/WAKU2-X3DH](/spec/53) we specified the `X3DH` protocol for end-to-end encryption. 

Once two peers complete an X3DH handshake, an X3DH session would be established.

This RFC describes how to manage these sessions, including how to establish new ones, how to re-establish them, how to maintain them, and how to close them.

# Session Establishment

A node identifies a peer by their `installation-id` which MAY be interpreted as a device identifier.

## Initialization

A node initializes a new session once a successful X3DH exchange has taken place. 
Subsequent messages will use the established session until re-keying is necessary.

## Concurrent sessions

If a node creates two sessions concurrently between two peers, the one with the symmetric key first in byte order SHOULD be used, this marks that the other has expired.

## Re-keying

On receiving a bundle from a given peer with a higher version, the old bundle SHOULD be marked as expired and a new session SHOULD be established on the next message sent.

## Multi-device support

Multi-device support is quite challenging as there is not a central place where information on which and how many devices (identified by their respective `installation-id`) a peer has, is stored.

Furthermore, account recovery always needs to be taken into consideration, where a user wipes clean the whole device and the nodes loses all the information about any previous sessions.
Taking these considerations into account, the way the network propagates multi-device information using X3DH bundles, which will contain information about paired devices as well as information about the sending device.
This means that every time a new device is paired, the bundle needs to be updated and propagated with the new information, the user has the responsibility to make sure the pairing is successful.

The method is loosely based on [Signal's Sesame Algorithm](https://signal.org/docs/specifications/sesame/).

## Pairing

A new `installation-id` MUST be generated on a per-device basis. 
The device should be paired as soon as possible if other devices are present. 

If a bundle is received, which has the same `IK` as the keypair present on the device, the devices MAY be paired.
Once a user enables a new device, a new bundle MUST be generated which includes pairing information.

The bundle MUST be propagated to contacts through the usual channels.

Removal of paired devices is a manual step that needs to be applied on each device, and consist simply in disabling the device, at which point pairing information will not be propagated anymore.

### Sending messages to a paired group

When sending a message, the peer SHOULD send a message to other `installation-id` that they have seen. 
The node caps the number of devices to `n`, ordered by last activity. 
The node sends messages using pairwise encryption, including their own devices.

Where `n` is the maximum number of devices that can be paired.

## Account recovery

Account recovery is the same as adding a new device, and it MUST be handled the same way.

## Partitioned devices

In some cases (i.e. account recovery when no other pairing device is available, device not paired), it is possible that a device will receive a message that is not targeted to its own `installation-id`.
In this case an empty message containing bundle information MUST be sent back, which will notify the receiving end not to include the device in any further communication.

# Security Considerations

1. Inherits all security considerations from [53/WAKU2-X3DH](/spec/53).

# Recommendations

1. The value of `n` SHOULD be configured by the app-protocol.
    - The default value SHOULD be 3, since a larger number of devices will result in a larger bundle size, which may not be desirable in a peer-to-peer network.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [53/WAKU2-X3DH](/spec/53)
2. [Signal's Sesame Algorithm](https://signal.org/docs/specifications/sesame/)
3. [5/SECURE-TRANSPORT](https://specs.status.im/spec/5)