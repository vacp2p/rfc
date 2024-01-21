---
slug: "75"
title: 75/WAKU2-MESSAGE-RELIABILITY
name: Message reliability for Waku Store
status: raw
category: Standards Track
tags:
  - message-reliability
editor: Abhimanyu Rawat <abhi@status.im>
contributors:
---

# Abstract

Sync protocol enables a node to synchronize itself with messages that it may have missed due to any reason, from one of its peers. This is an efficient method for the nodes present in the network to stay synchronized while sharing minimal information with each other.

This document describes the Sync store protocol and how it is used to synchronize the nodes in the network. It also explains the various components of the protocol and how they interact with each other. The detailed explanation of the protocol is given in the [research doc](https://github.com/waku-org/research/issues/78). In a Sync request-response protocol, a client receives the messages through Sync mechanism using their peer nodes only. Store is planned to work in conjunction with Sync to provide a reliable message delivery mechanism.

# Background / Rationale / Motivation

Nodes who want to synchronize with each other should be able to do so with minimal amount of communication and bandwidth usage. Existing methods present in Waku Store protocol are based on full message delivery using time range query where there is no way to know apart from a given timestamp range if other messages are also missing, making them unreliable and bandwidth consuming. Node that wants to synchronize sends a request to the peer node, and in return peer node send the response to the requesting node. After a brief low-bandwidth consuming back and flow exchange, resulting will be a list values (message_hashes) that the requesting node needs to sync with the peer node. These hashes can be used by other parts of the Store protocol to make sure the related full messages are wired to the requesting node and stored in the Store, making the requesting node fully synchronized with the peer node.

A detailed view in the process can be tracked using the research [issue](https://github.com/waku-org/research/issues/62).

# Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Consider a request-response protocol with two roles: a client and a server.
Client and Server both MUST be peers supporting the Waku Sync protocol. There is no other eligibility for the node at this point in time to be a client or a server who can synchronize with each other.

# Wire Format Specification / Syntax

```

    syntax = "proto3";

    package prollytree;

    message Node {
        bytes data = 1;              // The data stored in the node
        int64 timestamp = 2;          // Timestamp or other unique identifier
        bytes node_hash = 3;         // Hash of the node
        int32 level = 4;              // Level of the node in the tree
        bytes merkel_hash = 5;       // Merkel hash of the node
        bool is_tail = 6;             // Flag indicating if it's a tail node

        // Relationships (IDs or indices of related nodes)
        // Note: Relationships like left, right, up, and down are typically represented using references
        // (like indices or IDs) rather than directly embedding the objects to avoid deep nesting.
        // In our case we can use the key of the node itself and figure out the level based on the current node level.
        int32 left = 7;               // Reference to the left node
        int32 right = 8;              // Reference to the right node
        int32 up = 9;                 // Reference to the parent node
        int32 down = 10;              // Reference to the child node
    }

    // Request message for GetRoot RPC. No fields needed for this example.
    message GetRootRequest {
    }

    // Response message for GetRoot RPC
    message GetRootResponse {
        Node root = 1;  // The root node of the ProllyTree
        // Add additional fields if necessary, e.g., a status message or an error code.
    }

    // Request message for GetIntermediateNode to resolve the reference indices as mentioned above
    message GetIntermediateNodeRequest {
        int32 node_id = 1;  // ID of the node to fetch
        int32 level = 2;    // Level of the node, if necessary for retrieval
    }

    // Response message for GetIntermediateNode
    message GetIntermediateNodeResponse {
        Node node = 1;  // The requested node
    }


    // Request message for GetRootAtHeight RPC
    message GetRootAtHeightRequest {
        int32 height = 1;  // The desired height
    }

    // Request message for non boundary nodes
    message GetNonBoundaryNodesRequest {
        repeated Node start_nodes = 1;  // Nodes from which to start searching
    }

    // Response message for GetNonBoundaryNodes RPC
    message GetNonBoundaryNodesResponse {
        repeated Node non_boundary_nodes = 1;  // Non-boundary nodes found
    }
```

A brief description of each message request-response is described as follows:

### GetRootRequest

This message is sent by the node that wants to sync with the peer node. It contains no fields. The peer node if using the Sync store protocol will respond with the GetRootResponse message. It sends the root node of the tree to the requesting node.

### GetRootResponse

This message is sent by the peer node in response to the GetRootRequest message. It contains the root node of the tree. The requesting node can then use this node to traverse the tree and find the nodes that it needs to sync with the peer node.

### GetRootAtHeightRequest

This message is sent by the node that wants to sync with the peer node. It contains the height of the tree that the requesting node wants to sync with the peer node. The peer node if using the Sync store protocol will respond with the GetRootResponse message. It sends the root node of the tree to the requesting node.

### GetIntermediateNodeRequest

This message is sent by the node that wants to sync with the peer node. It contains the node id/key and the level of the Prolly tree, uisng this information peer node searches and in response send back the subjected node. The peer node if using the Sync store protocol will respond with the GetIntermediateNodeResponse message. It sends the node of the tree to the requesting node.

### GetIntermediateNodeResponse

This message is sent by the peer node in response to the GetIntermediateNodeRequest message. It contains an intermediate node of the tree. The requesting node can then use this node to traverse the tree and find the missing nodes.

### GetNonBoundaryNodesRequest

This message is sent by the node that wants to sync with the peer node. It contains the start nodes from which the peer node will start searching for the non boundary nodes. The peer node if using the Sync store protocol will respond with the GetNonBoundaryNodesResponse message. It sends the non boundary nodes of the tree to the requesting node.

### GetNonBoundaryNodesResponse

This message is sent by the peer node in response to the GetNonBoundaryNodesRequest message. It contains the non boundary nodes of the tree. The requesting node can then use this nodes to traverse the tree and find the missing nodes.

# Implementation in of Sync using Prolly tree (PoC version)

This Section describes a proof-of-concept (PoC) implementation of the Sync protocol using Prolly tree.

The PoC code is written in Python and can be found [here](https://github.com/ABresting/Prolly-Tree-Waku-Message). One can also run the code using the README.md guide.

The Sync Process works in following steps:

Step 1: A key-value store is created using the Store protocol where the key is the timestamp of a message, and the value is the `message_hash` attribute which uniquely identifies a message in Waku network.

Step 2: A Prolly tree is populated using the key-value store. This construction is similar at both client and server side.

Step 3: A client node sends a request to the server node to get the root node of the Prolly tree. The server node responds with the root node of the Prolly tree.

Step 4: As Prolly tree is designed to compare diff based on equal hieght so if the height of the root of the Prolly tree is different then the client node sends a request to the server node to get the root node of the Prolly tree at the height of the root of the client's Prolly tree. The server node responds with the root node of the Prolly tree at the height of the root of the client's Prolly tree.

Step 5: The client node then traverses the Prolly tree from the root node to the leaf nodes and finds the missing nodes at every level of the tree starting from the top. The missing intermediate Prolly tree Node is requested by client to the server. Server responds with the requested nodes.

Step 6: The client eventually computes the missing `message_hash`es and sends a request to server to get the missing messages. Server responds with the requested messages.

Upon receiving the missing messages that are present in the Prolly tree, the client node request the onward messages using the existing Store [method](https://github.com/waku-org/nwaku/blob/master/waku/waku_archive/driver.nim#L36) since these nodes are for sure missing from the client Prolly tree.

# Security/Privacy Considerations

Sync protocol on an abstract level provides an interface to synchronize Store of Waku messages with peer nodes in the network. The security and privacy considerations are limited to the storage of messages on the server node, not disclosing any other information about the origin of the messages on server.

# Limitations and Future Work

This document is intentionally simplified in its initial version.
It is to showcase the working pieces of the Sync protocol and how it can be used to synchronize the nodes in the network. The PoC of the Sync store is initself feature complete but not sufficient for production code without sanitizing it to work with Waku Store.

The following ideas may be explored in future:

- Batch sequenced insertions: The design and the advantage of the Waku Prolly tree over others is such that it can be used to insert multiple sequenced messages at once without affection the Merkel hashes of the messages already present in the tree i.e. left side of the tree.

- It can itself be served as a Store of messages in light weight nodes with limited storage capacity.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## informative

RFCs of request-response protocols:

- [13/WAKU2-STORE](/spec/13)
