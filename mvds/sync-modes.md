#  MVDS Sync Modes

> Version: 0.0.1 (Draft)
> 
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

##  Table of Contents

1. [Abstract](#abstract)
2. [Format](#format)

## Abstract

In this specification, we describe a method to provide consistency through various means as well as modifying synchronization of [MVDS](./README.md).

## Format

### Fields

| Name      |  Description                                         |
| --------- | -----------------------------------------------------|
| `parents` |  contains a list of parent [`message identifiers`](./README.md#payloads) |

