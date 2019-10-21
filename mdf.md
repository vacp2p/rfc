#  MVDS Metadata Field

> Version: 0.1.0 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

##  Table of Contents

1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Format](#format)
    1. [Fields](#fields)
4. [Usage](#usage)
    1. [`parents`](#parents)
    2. [`ack_required`](#ack_required)
5. [Footnotes](#footnotes)
6. [Acknowledgements](#acknowledgements)

## Abstract

In this specification, we describe a method to construct message history that will aid the consistency guarantees of [MVDS](./mvds.md). Additionally we explain how data sync can be used for more lightweight messages that do not require full synchronization.

## Motivation

In order for more efficient synchronization of conversational messages, information should be provided allowing a node to more effectively synchronize the dependencies for any given message.

## Format

We introduce the metadata message which is used to convey information about a message and how it SHOULD be handled.

```protobuf
package vac.mvds;

message Metadata {
  bytes parents = 1;
  bool ack_required = 2 [default = true];
}
```

Nodes MAY transmit a `Metadata` message by extending the MVDS [message](./mvds.md#payloads) with a `metadata` field.

```diff
message Message {
  bytes group_id = 1;
  int64 timestamp = 2;
  bytes body = 3;
+ Metadata metadata = 4;
}
```

### Fields

| Name                   |   Description                                                                                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `parents`               |   list of parent [`message identifier`s](./mvds.md#payloads) for the specific message. |            
| `ack_required`         |   indicates whether a message needs to be acknowledged or not.                                                             |

## Usage

### `parents`

This field contains a list of parent [`message identifier`s](./mvds.md#payloads) for the specific message. It MUST NOT contain any messages as parent whose `ack` flag was set to `false`. This creates a directed acyclic graph (DAG)<sup>1</sup> of persistent messages.

Nodes MAY buffer messages until dependencies are satisfied for causal consistency<sup>2</sup>, they MAY also pass the messages straight away for eventual consistency<sup>3</sup>.

A parent is any message before a new message that a node is aware of that has no children.

The number of parents for a given message is bound by [0, N], where N is the number of nodes participating in the conversation, therefore the space requirements for the `parents` field is O(N).

If a message has no parents it is considered a root. There can be multiple roots, which might be disconnected, giving rise to multiple DAGs.

### `ack_required`

When the `ack_required` flag is set to `true`, a node MUST acknowledge when they have received and processed  a message. If it is set to `false`, it SHOULD NOT send any acknowledgement.

Messages that are not required to be acknowledged can be considered **ephemeral**, meaning nodes MAY decide to not persist them and they MUST NOT be shared as part of the message history.

Nodes SHOULD send ephemeral messages in batch mode. As their delivery is not needed to be guaranteed.

## Footnotes
1. <https://en.wikipedia.org/wiki/Directed_acyclic_graph>
2. <https://jepsen.io/consistency/models/causal>
3. <https://en.wikipedia.org/wiki/Eventual_consistency>

## Acknowledgements
 - Andrea Maria Piana
