# Minimum Viable Data Synchronization

> Version: 0.6.0 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

## Table of Contents

1. [Abstract](#abstract)
2. [Definitions](#definitions)
3. [Wire Protocol](#wire-protocol)
    1. [Secure Transport](#secure-transport) 
    2. [Payloads](#payloads)
4. [Synchronization](#synchronization)
    1. [State](#state)
    2. [Flow](#flow)
    3. [Retransmission](#retransmission)
5. [Footnotes](#footnotes)
6. [Acknowledgements](#acknowledgements)

## Abstract

In this specification, we describe a minimum viable protocol for data synchronization inspired by the Bramble Synchronization Protocol<sup>1</sup>. This protocol is designed to ensure reliable messaging between peers across an unreliable peer-to-peer (P2P) network where they may be unreachable or unresponsive.

We present a reference implementation<sup>2</sup> including a simulation to demonstrate its performance.

## Definitions

| Term       | Description                                                                         |
|------------|-------------------------------------------------------------------------------------|
| **Peer**   | The other nodes that a node is connected to.                                        |
| **Record** | Defines a payload element of either the type `OFFER`, `REQUEST`, `MESSAGE` or `ACK` |
| **Node**   | Some process that is able to store data, do processing and communicate for MVDS.    |

## Wire Protocol

### Secure Transport

This specification does not define anything related to the transport of packets. It is assumed that this is abstracted in such a way that any secure transport protocol could be easily implemented. Likewise, properties such as confidentiality, integrity, authenticity and forward secrecy are assumed to be provided by a layer below.

### Payloads

Payloads are implemented using [protocol buffers v3](https://developers.google.com/protocol-buffers/).

```protobuf
syntax = "proto3";

package vac.mvds;

message Payload {
  repeated bytes acks = 1;
  repeated bytes offers = 2;
  repeated bytes requests = 3;
  repeated Message messages = 4;
}

message Message {
  bytes group_id = 1;
  int64 timestamp = 2;
  bytes body = 3;
}
```

*The payload field numbers are kept more "unique" to ensure no overlap with other protocol buffers.*

Each payload contains the following fields:

- **Acks:** This field contains a list (can be empty) of `message identifiers` informing the recipient that sender holds a specific message.
- **Offers:** This field contains a list (can be empty) of `message identifiers` that the sender would like to give to the recipient.
- **Requests:** This field contains a list (can be empty) of `message identifiers` that the sender would like to receive from the recipient.
- **Messages:** This field contains a list of messages (can be empty).

**Message Identifiers:** Each `message` has a message identifier calculated by hashing the `group_id`, `timestamp` and `body` fields as follows:

```
HASH("MESSAGE_ID", group_id, timestamp, body);
```

The current `HASH` function used is `sha256`.

## Synchronization

### State

We refer to `state` as a collection of data each node SHOULD hold for records of the types `OFFER`, `REQUEST` and `MESSAGE` per peer. We MUST NOT keep states for `ACK` records as we do not retransmit those periodically. The following information is stored for records:

 - **Type** - Either `OFFER`, `REQUEST` or `MESSAGE`
 - **Send Count** - The amount of times a record has been sent to a peer.
 - **Send Epoch** - The next epoch at which a record can be sent to a peer.

### Flow

A maximum of one payload SHOULD be sent to peers per epoch, this payload contains all `ACK`, `OFFER`, `REQUEST` and `MESSAGE` records for the specific peer. Payloads are created every epoch, containing reactions to previously received records by peers or new records being sent out by nodes. 

Nodes MAY have two modes with which they can send records: `BATCH` and `INTERACTIVE` mode. The following rules dictate how nodes construct payloads every epoch for any given peer for both modes.

> ***NOTE:** A node may send messages both in interactive and in batch mode.*

#### Interactive Mode

 - A node initially offers a `MESSAGE` when attempting to send it to a peer. This means an `OFFER` is added to the next payload and state for the given peer.
 - When a node receives an `OFFER`, a `REQUEST` is added to the next payload and state for the given peer. 
 - When a node receives a `REQUEST` for a previously sent `OFFER`, the `OFFER` is removed from the state and the corresponding `MESSAGE` is added to the next payload and state for the given peer.
 - When a node receives a `MESSAGE`, the `REQUEST` is removed from the state and an `ACK` is added to the next payload for the given peer.
 - When a node receives an `ACK`, the `MESSAGE` is removed from the state for the given peer.
 - All records that require retransmission are added to the payload, given `Send Epoch` has been reached.

<p align="center">
    <img src="./assets/mvds/interactive.png" />
    <br />
    Figure 1: Delivery without retransmissions in interactive mode.
</p>

#### Batch Mode

 1. When a node sends a `MESSAGE`, it is added to the next payload and the state for the given peer.
 2. When a node receives a `MESSAGE`, an `ACK` is added to the next payload for the corresponding peer.
 3. When a node receives an `ACK`, the `MESSAGE` is removed from the state for the given peer.
 4. All records that require retransmission are added to the payload, given `Send Epoch` has been reached.

<!-- diagram -->

<p align="center">
    <img src="./assets/mvds/batch.png" />
    <br />
    Figure 2: Delivery without retransmissions in batch mode.
</p>


> ***NOTE:** Batch mode is higher bandwidth whereas interactive mode is higer latency.*

<!-- Interactions with state, flow chart with retransmissions? -->

### Retransmission

The record of the type `Type` SHOULD be retransmitted every time `Send Epoch` is smaller than or equal to the current epoch.

`Send Epoch` and `Send Count` MUST be increased every time a record is retransmitted. Although no function is defined on how to increase `Send Epoch`, it SHOULD be exponentially increased until reaching an upper bound where it then goes back to a lower epoch in order to prevent a record's `Send Epoch`'s from becoming too large.

> ***NOTE:** We do not retransmission `ACK`s as we do not know when they have arrived, therefore we simply resend them every time we receive a `MESSAGE`.*

## Footnotes

1. <https://code.briarproject.org/briar/briar-spec/blob/master/protocols/BSP.md>
2. <https://github.com/vacp2p/mvds>

## Changelog

| Version | Comment |
| :-----: | ------- |
| [0.6.0](https://github.com/vacp2p/specs/commit/b6335947e09502ec399954002d983a709065a342#diff-a6dbd535bfcb545a64b1c95f6dc8f9f1) | Update protocol buffer field numbers and other minor changes |
| [0.5.2](https://github.com/vacp2p/specs/blob/0adddcdb8d7e71240c2df30d4ba40e49b4275011/mvds.md) | Update protocol buffer field numbers |
| [0.5.1](https://github.com/status-im/bigbrother-specs/blob/20e2659447dcb5123a105526eb26816307c3780c/data_sync/mvds.md) | Clarifications on spec, RFC-2119 and message identifiers. |
| [0.5.0](https://github.com/status-im/bigbrother-specs/blob/6d3d0771b46941791eabe9263f80cbaeaf3d1148/data_sync/mvds.md) | Initial Release |

## Acknowledgements
 - Preston van Loon
 - Greg Markou
 - Rene Nayman
 - Jacek Sieka
