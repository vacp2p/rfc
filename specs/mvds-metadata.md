---
title: MVDS Metadata Field
version: 0.1.0
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>
redirect_from:
  - /mvds-metadata.html
---

##  Table of Contents

1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Format](#format)
    1. [Fields](#fields)
4. [Usage](#usage)
    1. [`parents`](#parents)
    2. [`ephemeral`](#ephemeral)
5. [Changelog](#changelog)
6. [Acknowledgements](#acknowledgements)
7. [Copyright](#copyright)
8. [Footnotes](#footnotes)

## Abstract

In this specification, we describe a method to construct message history that will aid the consistency guarantees of [MVDS](mvds.md). Additionally we explain how data sync can be used for more lightweight messages that do not require full synchronization.

## Motivation

In order for more efficient synchronization of conversational messages, information should be provided allowing a node to more effectively synchronize the dependencies for any given message.

## Format

We introduce the metadata message which is used to convey information about a message and how it SHOULD be handled.

```protobuf
package vac.mvds;

message Metadata {
  repeated bytes parents = 1;
  bool ephemeral = 2;
}
```

Nodes MAY transmit a `Metadata` message by extending the MVDS [message](mvds.md#payloads) with a `metadata` field.

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
| `parents`               |   list of parent [`message identifier`s](mvds.md#payloads) for the specific message. |            
| `ephemeral`         |   indicates whether a message is ephemeral or not.                                                             |

## Usage

### `parents`

This field contains a list of parent [`message identifier`s](mvds.md#payloads) for the specific message. It MUST NOT contain any messages as parent whose `ack` flag was set to `false`. This establishes a directed acyclic graph (DAG)[^1] of persistent messages.

Nodes MAY buffer messages until dependencies are satisfied for causal consistency[^2], they MAY also pass the messages straight away for eventual consistency[^3].

A parent is any message before a new message that a node is aware of that has no children.

The number of parents for a given message is bound by [0, N], where N is the number of nodes participating in the conversation, therefore the space requirements for the `parents` field is O(N).

If a message has no parents it is considered a root. There can be multiple roots, which might be disconnected, giving rise to multiple DAGs.

### `ephemeral`

When the `ephemeral` flag is set to `false`, a node MUST send an acknowledgement when they have received and processed a message. If it is set to `true`, it SHOULD NOT send any acknowledgement. The flag is `false` by default.

Nodes MAY decide to not persist ephemeral messages, however they MUST NOT be shared as part of the message history.

Nodes SHOULD send ephemeral messages in batch mode. As their delivery is not needed to be guaranteed.

## Changelog

| Version | Comment |
| :-----: | ------- |
| [0.1.0](https://github.com/vacp2p/specs/blob/53bc8585add58695c28cfaf4382818f4daf8de84/mdf.md)   | Initial Release |

## Acknowledgements
 - Andrea Maria Piana

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Footnotes

[^1]: <https://en.wikipedia.org/wiki/Directed_acyclic_graph>
[^2]: Jepsen. [Causal Consistency](https://jepsen.io/consistency/models/causal). Jepsen, LLC.
[^3]: <https://en.wikipedia.org/wiki/Eventual_consistency>