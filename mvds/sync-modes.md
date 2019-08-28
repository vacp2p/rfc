#  MVDS Sync Modes

> Version: 0.0.1 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

##  Table of Contents

1. [Abstract](#abstract)
2. [Format](#format)

## Abstract

In this specification, we describe a method to provide consistency through various means as well as modifying synchronization of [MVDS](./README.md). This specification mainly describes a format for the header field of an [MVDS message](./README.md#payloads) that modifies the functionality of MVDS.

## Format

### Fields

| Name       |  Description                                                             |
| ---------- | ------------------------------------------------------------------------ |
| `parents`  |  contains a list of parent [`message identifiers`](./README.md#payloads) |
| `sequence` |  sequence number of the message                                          |
| `ack`      |  contains a flag whether a message needs to be acknowledged or not.      |
