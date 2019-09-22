#  MVDS Meta Data Field

> Version: 0.0.1 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

##  Table of Contents

1. [Abstract](#abstract)
2. [Format](#format)
    1. [Fields](#fields) 
4. [Behavior](#behavior)

## Abstract

@TODO

<!-- In this specification, we describe a method to provide consistency through various means as well as modifying synchronization of [MVDS](./README.md). This specification mainly describes a format for the header field of an [MVDS message](./README.md#payloads) that modifies the functionality of MVDS. -->

## Format

The meta data field is used to convey various information on a message and how it MUST be handled.

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

| Name       |  Description                                                             |
| ---------- | ------------------------------------------------------------------------ |
| `parents`  |  contains a list of parent [`message identifiers`](./README.md#payloads) |
| `sequence` |  sequence number of the message                                          |
| `ack`      |  contains a flag whether a message needs to be acknowledged or not.      |

## Behavior

The flags provided through the `MetaData` message are either informational or configurational. Informational fields MAY be used by a node, configurational fields MUST be recognized and the nodes behavior MUST change accordingly.