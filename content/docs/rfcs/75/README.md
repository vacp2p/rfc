---
slug: "75"
title: 75/WAKU2-SYNC
name: Synchronization protocol for Waku messages
status: raw
category: Standards Track
tags:
  - sync-protocol
editor: Abhimanyu Rawat <abhi@status.im>
contributors:
---

# Abstract

The Sync protocol allows operators within peer-to-peer Waku network to efficiently synchronize Waku messages by exchanging their [message hashes](https://rfc.vac.dev/spec/14/#deterministic-message-hashing) using their respective Prolly trees (Spec of Prolly tree to be published).
This process ensures that all operators stay updated with minimal information exchange, accommodating situations where operators may have missed some messages due to any reason.

This document describes the Sync store protocol and how it is used to synchronize the operators in the network.
It also explains the various components of the protocol and how they interact with each other.
In a Sync request-response protocol, a client receives the list of missing Waku message hashes through Sync mechanism using their peer nodes only.

# Background / Rationale / Motivation

Operators who want to synchronize with each other should be able to do so with minimal amount of communication and bandwidth usage.
Existing methods present in Waku Store protocol are based on full Waku message delivery using time range query where there is no way to know apart from a given timestamp range if other Waku messages are also missing, making Store only method unreliable and bandwidth consuming.
Operator that wants to synchronize sends a request to its peer, and in return peer operator send the response.
Root of the Prolly tree is used in this Sync mechanism, which is a Merkle tree with a probabilistic data structure.
As per the Prolly tree design, there is a fake root present at every level/height of the Prolly tree, to Sync properly both roots must be from the same height/level inside the Prolly tree.
After a brief low-bandwidth consuming back and flow exchange, resulting will be a list message hashes that the requesting operator needs to sync with the peer operator.
These messages hashes can be used by other parts of the Store protocol to make sure the related full messages are wired to the requesting operator and stored in its accessible (local for now) Store/DB.
Making the requesting operator fully synchronized with its peer operator.

# Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Consider a request-response protocol with two roles: a client and a server.
Client and Server both MUST be peers supporting the Waku Sync protocol.
Each operator (client/server) MUST contain empty or populated Prolly tree of Waku message hashes.
There is no other eligibility for the operators at this point in time to be a client or a server who can synchronize with each other.

# Wire Format Specification / Syntax

```
    syntax = "proto3";

    package prollytree;

    message Node {
        bytes data = 1;              // The data stored in the node
        int64 timestamp = 2;          // Timestamp or other unique identifier
        bytes node_hash = 3;         // Hash of the node
        int32 level = 4;              // Level of the node in the tree
        bytes merkle_hash = 5;       // Merkle hash of the node
        bool is_tail = 6;             // Flag indicating if it's a tail node

        // Relationships (IDs or indices of related nodes)
        // Note: Relationships like left, right, up, and down are typically represented using references
        // (like indices or IDs) rather than directly embedding the objects to avoid deep nesting.
        // In our case we can use the key of the node itself and figure out the level based on the current node level.
        int64 left = 7;               // Reference to the left node
        int64 right = 8;              // Reference to the right node
        int64 up = 9;                 // Reference to the parent node
        int64 down = 10;              // Reference to the child node
    }

    // Request message for ExchangeRootRequest
    message ExchangeRootRequest {
        Node root = 1;    // The root node of the Prolly tree
    }

    // Response message for ExchangeRootRequest
    message ExchangeRootResponse {
        Node root = 1;  // The root node of the Prolly tree, same height as requested root
        // Add additional fields if necessary, e.g., a status message or an error code.
    }

    // Request message for GetIntermediateNode to resolve the reference indices as mentioned above
    message GetIntermediateNodeRequest {
        int64 node_id = 1;  // ID/timestamp of the node to fetch
        int32 level = 2;    // Level of the node, if necessary for retrieval
    }

    // Response message for GetIntermediateNode
    message GetIntermediateNodeResponse {
        Node node = 1;  // The requested node
    }

    // Request message for non boundary nodes
    message GetNonBoundaryNodesRequest {
        repeated Node start_nodes = 1;  // Nodes from which to start searching
    }

    // Response message for GetNonBoundaryNodes
    message GetNonBoundaryNodesResponse {
        repeated Node non_boundary_nodes = 1;  // Non-boundary nodes found
    }
```

A brief description of each message request-response is described as follows:

### ExchangeRootRequest

This message is sent by a client that wants to sync with its peer/server.
It contains local Prolly Tree root as input.
The server will respond with the GetRootResponse message.

### ExchangeRootResponse

This message is sent by the peer/server in response to the GetRootRequest message.
It contains the root Node of the tree present at a same or lower (maximum at the server) height as the client.
Client can then use this root Node to traverse the tree and iteratively find the intermediate/leaf Nodes that it needs to Sync with peer.

### GetIntermediateNodeRequest

This message is sent by a client that wants to Sync with its peer.
This request contains a Node's id/key and level inside Prolly tree, using this information server searches and in response send back the subjected Node.
The server will respond with the GetIntermediateNodeResponse message.

### GetIntermediateNodeResponse

This message is sent by the peer/server in response to the GetIntermediateNodeRequest message.
It contains an intermediate Node of the tree corresponding to the requesting key-level inside the Prolly tree.
Client can then use this response Node to traverse the tree further and find the missing Nodes.

### GetNonBoundaryNodesRequest

This message is sent by a client that wants to sync with its peer.
It is a list of Nodes of which are either missing or have a merkle mismatch inside the local Prolly tree when compared with corresponding Nodes present in the peer's Prolly tree.
Peer will respond with GetNonBoundaryNodesResponse message.

### GetNonBoundaryNodesResponse

This message is sent by the peer/server in response to the GetNonBoundaryNodesRequest message.
For the each requested Node, it contains the non-boundary Nodes present just one level below it in the Prolly tree.
Client can then use these intermediate Nodes to traverse the tree further and find the missing Nodes.

# Implementation of Sync using Prolly tree (PoC version)

<!-- This whole section can be removed once the Spec of Prolly tree or any foundation unit of Sync mechanism is ready -->

This Section describes a proof-of-concept (PoC) implementation of the Sync protocol using Prolly tree.

The PoC code is written in Python and can be found [here](https://github.com/ABresting/Prolly-Tree-Waku-Message).
One can also run the code using the README.md guide.

The Sync Process works in following steps:

Step 1: A key-value store is created using the Store protocol where the key is the timestamp of a message, and the value is the `message_hash` attribute which uniquely identifies a message in Waku network.

Step 2: A Prolly tree is populated using the key-value store.
This construction is similar at both client and server side.

Step 3: A client node sends a request to the server node to get the root node of the Prolly tree.
Client sends its own local root node of the Prolly tree.
With client's root node, server can calculate the height inside the Prolly tree from where client wants to Sync.

Step 4: The server node responds with the root node of the Prolly tree present at same height as client's root.
4a) If the root height at server is less than the root height requested by the client, then the server responds with the root node of the Prolly tree at the maximum height present at the server.
4b) If the root height at server is greater than the root height requested by the client, then the server responds with the root node at the requested height by traversing down to that height inside it's local Prolly tree.

Step 5: The client node starts traversing the Prolly tree from the root node to the leaf nodes and finds the missing nodes or nodes with different merkle hash at every level of the tree starting from the top.
5a) If the client detects the missing key or difference of merkle hash at a key of a certain level, then it requests the server node to get the intermediate child node belonging to that key.
5b) And the keys with no difference in merkle hash are skipped.

This step continues iteratively until the client reaches the leaf nodes of the tree.

Step 6: The client eventually computes the missing message hashes and sends a request to server to get the missing Waku messages corresponding to the computed message hashes.
Server responds with the requested Waku messages.

When a server receives the `ExchangeRootRequest` message from the client, it can also detect if server also needs to Sync message hashes with the client.
This step helps reducing the bandwidth usage and the number of messages exchanged between the client and the server nodes in longer run.

Upon receiving the missing messages that are present in the peer's Prolly tree, the client node can request the onward (messages with greater timestamp than last message in the Prolly tree) messages using the existing Store [method](https://github.com/waku-org/nwaku/blob/master/waku/waku_archive/driver.nim#L36) since these messages are for sure can be missing from the client Prolly tree.

# Security/Privacy Considerations

Sync protocol on an abstract level provides an interface to synchronize Store of Waku messages with peer nodes in the network.
The security and privacy considerations are limited to the storage of messages on the server node, not disclosing any other information about the origin of the messages on server.

# Limitations and Future Work

This document is intentionally simplified in its initial version.
It is to showcase the working pieces of the Sync protocol and how it can be used to synchronize the nodes in the network.
The PoC of the Sync store is in itself feature complete but not sufficient for production code without sanitizing it to work with Waku Store.

The following ideas may be explored in future:

- Batch sequenced insertions/deletions: The design and the advantage of the Waku Prolly tree over others is such that it can be used to insert/delete multiple sequenced messages at once without affecting the Merkle hashes of the Nodes already present in the Prolly tree i.e. left side of the tree.

- It can itself be served as a Store of messages in light weight nodes with limited storage capacity.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## informative

RFCs of request-response protocols:

- [13/WAKU2-STORE](/spec/13)
