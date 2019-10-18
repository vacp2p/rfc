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
    1. [Informational Fields](#informational-fields)
    2. [Configurational Fields](#configurational-fields)

## Abstract

In this specification, we describe a method to construct both a linear and DAG (Directed Acyclic Graph) based message history that will aid the consistency of [MVDS](./mvds.md), which currently do not exist, causal consistency is added with the DAG and sequential consistency with the linear history. Additionally we explain how data sync can be used for more lightweight messages that do not require full synchronization.

## Motivation

In order for more efficient synchronization of conversational messages, information should be provided allowing a node to more effectively synchronize the dependencies for any given message. Encoding a DAG within a message allows for an entire portion to be synchronized if messages are missing with a single `REQUEST` message.

## Format

We introduce the metadata message which is used to convey information about a message and how it SHOULD be handled. If an MVDS node implements this, we say that node has MDF capability (This might become part of a future reconciled MVDS spec).


```protobuf
package vac.mvds;

message Metadata {
  bytes parent = 7000;
  bool ack_required = 7001 [default = true];
}
```

We MAY transmit a `Metadata` message by extending the MVDS [message](./mvds.md#payloads) with a `metadata` field.

```diff
message Message {
+ Metadata metadata = 6000;
  bytes group_id = 6001;
  int64 timestamp = 6002;
  bytes body = 6003;
}
```
### Fields

| Name                   |   Description                                                                                                                    |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `parents`               |   contains a list of parent [`message indentifier`](./mvds.md#payloads) for the specific message. |            
| `ack_required`         |   contains a flag whether a message needs to be acknowledged or not.                                                             |

## Usage

The flags provided through the `Metadata` message are either informational or behavioral. Informational fields MAY be used by a node, configurational fields SHOULD be recognized and the nodes behavior SHOULD change accordingly.

### Informational Fields

Below are the list of informational flags and what they MAY be used for by a node.

#### `parent`

This field contains a list of parent [`message indentifier`](./mvds.md#payloads) for the specific message. It MUST NOT contain any messages as parent whose `ack` flag was set to `false`. This creates a Directed Acyclic Graph (DAG)<sup>1</sup> of persistent messages.

This field helps establish consistency that may either be causal or eventual depending on the client.

### Configurational Fields

Below are a list of the configurational flags and how they modify node behavior.

#### `ack_required`

When the `ack_required` flag is set to `true`, a node MUST acknowledge when they have received and processed  a message. If it is set to `false`, it MUST not send any acknowledgement.

> ***NOTE**: Messages that are not required to be acknowledged can be considered **ephemeral**, meaning nodes MAY decide to not persist them and they MUST not be shared as part of the message history.*

Nodes SHOULD send ephemeral messages in batch mode. As their delivery is not needed to be guaranteed.

## Footnotes
1. <https://github.com/matrix-org/matrix-doc/blob/master/specification/server_server_api.rst#pdus>
