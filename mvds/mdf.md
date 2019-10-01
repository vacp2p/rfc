#  MVDS Meta Data Field

> Version: 0.1.0 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

##  Table of Contents

1. [Abstract](#abstract)
2. [Format](#format)
    1. [Fields](#fields) 
1. [Usage](#usage)
    1. [Information](#information)
    2. [Behavior](#behavior)

## Abstract

In this specification, we describe a method to construct a message DAG (Directed Acyclic Graph) that will aid the consistency of [MVDS](./README.md). Additionally we explain how data sync can be used for more lightweight messages that do not require full synchronization. This specification extends the [MVDS message](./README.md#payloads) to modify the functionality of MVDS.

## Format

The meta data field is used to convey various information on a message and how it SHOULD be handled.

```protobuf
package vac.mvds;

message MetaData {
  repeated bytes parents = 7001;
  optional int64 sequence = 7002;
  optional bool ack = 7003 [default = true];
}
```

We MAY transmit a `MetaData` message by extending the MVDS [message](./README.md#payloads) with a `meta_data` field.

```diff
message Message {
+ MetaData meta_data = 6000;
  bytes group_id = 6001;
  int64 timestamp = 6002;
  bytes body = 6003;
}
```
### Fields

| Name          |  Description                                                              |
| ------------- | ------------------------------------------------------------------------- |
| `parents`     |  contains a list of parent [`message identifiers`](./README.md#payloads). |
| `sequence`    |  sequence number of the message.                                          |
| `ack`         |  contains a flag whether a message needs to be acknowledged or not.       |

## Usage

The flags provided through the `MetaData` message are either informational or behavioral. Informational fields MAY be used by a node, configurational fields SHOULD be recognized and the nodes behavior SHOULD change accordingly.

### Information

Below are the list of informational flags and what they MAY be used for by a node.

#### `parents`

This field contains a list of a messages parents, or messages that have been seen by the node in a given context. This helps establish ordering by creating a Directed Acyclic Graph (DAG)<sup>1</sup>.

#### `sequence`

<!-- This will be related to remote log -->

### Behavior

Below are a list of the behavioral flags and how they modify node behavior.

#### `ack`

When the `ack` flag is set to `true`, a node MUST acknowledge when they have received and processed  a message. If it is set to `false`, it MUST not send any acknowledgement.

**NOTE:** Messages that are not required to be acknowledged can be considered ephemeral, meaning nodes MAY decide to not persist them and they MUST not be shared as part of the message history.

## Footnotes
1. <https://github.com/matrix-org/matrix-doc/blob/master/specification/server_server_api.rst#pdus>
