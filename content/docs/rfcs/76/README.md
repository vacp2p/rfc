---
slug: "76"
title: 76/WAKU2-PROLLY-TREE
name: Prolly Tree
status: raw
category: Standards Track
tags:
  - sync-protocol
  - prolly-tree
editor: Abhimanyu Rawat <abhi@status.im>
contributors:
---

# Abstract

Prolly Tree is a probabilistic data structure that can be used to synchronize data, which in our case is the Waku messages in a peer-to-peer network.
Prolly trees are itself not a data storage layer but can be used to build/assist one.
This document describes how the Prolly Tree is designed.
It also describes components which can be used to synchronize data between two Nodes.
Prolly trees are of different types, but the one used in this document is a Merkle tree with a probabilistic data structure with some difference than other implementations such as [Canvas](https://docs.canvas.xyz/blog/2023-05-04-merklizing-the-key-value-store.html), [dolthub](https://www.dolthub.com/blog/2020-04-01-how-dolt-stores-table-data/#prolly-trees), [MST](https://inria.hal.science/hal-02303490/document), etc.

# Background / Rationale / Motivation

Prolly trees, or probabilistically balanced trees, are a specialized data structure used for efficient data storage and synchronization.
They are particularly effective in environments where data needs to be frequently updated or synchronized across different systems.
By maintaining a balanced tree structure, prolly trees ensure quick and efficient data operations like search, insert, and delete.
This structure is crucial for handling large volumes of data, providing scalability and performance benefits.
Additionally, Prolly trees are adept at managing data integrity during synchronization processes, making them ideal for distributed systems where consistency and up-to-date data are paramount.

# Specification

A Prolly tree constitutes of the following components:

- **Node**: It is the simplest unit of data storage in Prolly tree.
  It has a key and a value at Level 0.
  At Non-zero levels, only the key is used to traverse the tree.
  Each Node stores a rolling Merkle hash of the subtree rooted at that Node.

- **Level**: A level is a collection of Nodes present at the same height in the Prolly tree.
  Level 0 is the leaf level of the tree where Nodes have both key and value filled in them.
  At Non-zero levels, only the key is used to traverse the tree.
  At the highest level, there is only one Node present which is the root of the tree.

- **Root**: It is the topmost Node of the tree.
  Its `merkle_hash` denotes the state of the whole tree.
  At each level there is a root i.e. the right most Node at that level.
  It is also called a Tail/Anchor Node.

- **Boundary Node**: It is a simple Node as mentioned above but its `node_hash` attribute falls below a certain threshold.
  Due to this it is promoted to the next level in the Prolly tree.
  This Node contains the rolling Merkel-hash of the non-boundary Nodes which are present in the level just below it until the first boundary Node or None is seen when moving from right to left.
  It is also called a Promoted Node.

- **Height**: It is the number of levels present in the Prolly tree starting from 0.
  At each height there is a root Node, which is the right most Node (Tail/Anchor Node) at that level.
  Two Prolly tree containing same sets of leaf Nodes will have same height, but not the other way around.

- **Tail/Anchor Node**: It is the last/right-most Node of the Prolly Tree situated at each level.
  It can also be termed as a Fake Node.
  It acts as a backbone for the Prolly tree.
  It has the property that it is always a boundary Node by default.
  Incoming Nodes that are inserted in the tree (most recent in the network) and they are non-boundary Nodes, then we need some Node to store their Merkel hash as well so that they can also be tracked during the Sync process.
  It stores the Merkel hash of the incoming non-boundary Nodes.
  At a height H, the Tail/Anchor Node also can act as a root for the sub tree from height H-1 to 0, this property is useful when comparing two Prolly trees with different Height.

- **Bucket/Chunk**: It is a collection of Nodes that are present at the same level in the Prolly tree and on either end there are boundary Nodes.
  The bucket/chunk boundary starts from the last nodeseen boundary Node in a sequence until the first seen Boundary Node.
  For. eg. at level 0, there are 5 Nodes in sequence with keys 1, 2, 3, 4, 5 and 4 is the boundary Node and 1, 2, 3 are non-boundary Nodes, then the chunk will be (1, 2, 3, 4).
  A chunk is also represented using the rolling Merkle hash of Nodes that are present in it.
  The Merkle hash of the chunk is stored in the boundary Node (last Node of the chunk that was promoted) present at the level just above it.
  In this example in Node(4) of Level 1.
  Using this Merkle hash one can verify the integrity of the chunk/bucket.

- **Threshold Mechanism**: Using this the structure of the Prolly tree is kept in check.
  It is used to maintain an average number of Nodes present in a chunk/bucket.
  It works in a probabilistic manner, where the probability of a Node being a boundary Node is checked against a customizable threshold value.
  If the has of a Node is less than the threshold value, then it is promoted to the next level in the Prolly tree.

- **Node Hash**: It is an attribute of the Node.
  If the Node is a leaf Node then, it is calculated as the SHA-256 hash of its contents i.e. key/value pair, timestamp and `message_hash` of the Waku message.
  If the Node is promoted then it is calculated as the rehashing of it's previous `node_hash` attribute.

## Node Structure

A [node](https://github.com/ABresting/Prolly-Tree-Waku-Message/blob/main/prolly_tree.py#L24) in the Prolly tree has following attributes:

| Attribute     | Description                            |
| ------------- | -------------------------------------- |
| `data`        | The data stored in the Node            |
| `timestamp`   | Timestamp or other unique identifier   |
| `node_hash`   | Hash of the Node                       |
| `level`       | Level of the Node in the tree          |
| `merkle_hash` | Merkle hash of the Node/sub-tree/chunk |
| `is_tail`     | Flag indicating if it's a tail Node    |
| `left`        | Reference to the left Node             |
| `right`       | Reference to the right Node            |
| `up`          | Reference to the parent Node           |
| `down`        | Reference to the child Node            |

## How a Prolly Tree is populated

A Prolly Tree is populated using a key-value store.
The key is the timestamp of a Waku message, and the value is the `message_hash` attribute which uniquely identifies a message in Waku network.
Nodes are ordered based on timestamp of the Waku message.
Prolly tree is populated from the scratch from Level 0.

Following steps are takes to populate the Prolly tree:

**Step 1:** A level in a Prolly tree is populated using timestamped ordered Nodes from left to right.
While populating level 0, the Nodes to populate come from the Key-value store.
While populating non-zero levels, the Nodes to populate come from the previous levels i.e. promoted Nodes from the previous level.
**1a)** If the `node_hash` attribute of the Node is less than the threshold value then it is promoted to the next level, this Node will be called Boundary Node.
**1b)** If the `node_hash` attribute of the Node is greater than the threshold value then it is kept at the same level and next Node is checked, this Node will be called a non-Boundary Node.

**Step 2:** Once a Node is promoted to the next level, it contains the `merkle_hash` of the chunk/bucket this Node belongs to.

**Step 3:** This process continues until all the Nodes are populated at that level.

**Step 4:** While finishing the population of Nodes at a given level, if the last populated Nodes are non-boundary Nodes then those Nodes become a part of the chunk where the Tail Node acts as a boundary Node.
For eg. if 1, 2, 3, 4, 5, 6, 7 are the Nodes at a level and Node 3 and 5 are boundary Node, and 6 and 7 are non-boundary Nodes, then first two chunks will be (1,2,3), (4,5) and the last chunk will be (6,7, Tail).
Here the Tail Node will act as a boundary Node for the last chunk and will store the Merkle hash of the chunk.

**Step 5:** Step 1-4 continues while there are still Nodes that were promoted from the previous level.
If no Nodes are are promoted for the next level, then the next level contains only one Node which is the Tail Node of the Prolly tree which also acts as a root of the whole Prolly Tree.

## How insertion happens inside a Prolly tree

If there is an already populated Prolly tree and we want to insert a new Node with key K inside it.
Starting from the root of the Prolly tree.
There is only one Node, i.e. Tail/Anchor which is also the Root of the Tree.

Following steps are taken:

**Step 1:** Go 1 level down and check to the left which Node has a key smaller than K.

**Step 2:** Either find a key smaller than K or reach to the left most of the Node of level which is None.

**Step 3:** Continue the step 1-2 until reach the level 0.

**Step 4:** At level 0, check to the left if the key of the Node is smaller than K or not.
**4a)** If find the a Node with key smaller than K, insert the K ahead of the found Node.
**4b)** If reach either a boundary Node or a None Node, then insert the new Node at that position.

**Step 5:** After insertion, check if the new Node is a boundary Node or not.
**5a)** If the new Node is a non-boundary Node, then update the Merkle hash of the chunk/bucket it belongs to and following buckets until reach the top of the tree.
**5b)** If the new Node is a boundary Node, then create a new chunk from the existing chunk and update the Merkle hash of the new chunk and following buckets til the top of the tree.

While adding the new Node it is possible that the Height of the Tree increases.
One can add new levels to the tree as and when required.

## How deletion happens inside a Prolly tree

Follow the same steps as insertion to find the Node to delete.
Instead of inserting the new Node, delete the Node from the Prolly tree and update the Merkle hashes.
In this case the levels can decrease as well, remove the levels from the top of the tree as and when required.

## How to compare two Prolly trees

Two Prolly trees can be compared using their Merkle hashes.
Each Node in Prolly Tree contains the Merkle hash of the subtree rooted at that Node.
Two Prolly trees can be compared provided they have the same height.
When comparing two Prolly tree min(PTree_1_height,PTree_2_height), minimum of the two hight is selected.
At the bigger Prolly tree, go to the given height/level, and choose the Tail Node.
This Tail Node will act as a Root when comparing with the one with smaller height.
Upon getting two Prolly Trees with same height, following steps should be taken:

**Step 1:** If the Root Node `merkel_hash` at of both the Trees match then no Sync required.
If the `merkel_hash` doesn't match, then [get](https://github.com/ABresting/Prolly-Tree-Waku-Message/blob/main/prolly_tree.py#L347) the child Nodes of Root Node.

**Step 2:** Upon receiving the List of child/chunk Nodes, compares the `merkel_hash` of each and discard the once which match.

**Step 3:** Mismatching Nodes are again requested from the other Prolly Tree and Step 2 repeated until Level 0 is reached.

**Step 4:** Upon [receiving](https://github.com/ABresting/Prolly-Tree-Waku-Message/blob/main/prolly_tree.py#L367) the Nodes of level 0, extract the `data` attribute i.e. `message_hash`.
This is the diff of the the Prolly Trees.

# Theory / Semantics

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

Consider a request-response protocol with two roles: a client and a server.
Client and Server both MUST be peers supporting the Waku Sync protocol.
Each operator (client/server) MUST contain empty or populated Prolly tree of Waku message hashes.
There is no other eligibility for the operators at this point in time to be a client or a server who can synchronize with each other.

# Security/Privacy Considerations

Prolly Tree provides the the Sync service based on the subjected data, it doesn't reveal anything other than the original datagram.
A Prolly tree can not disclose what other operators in the system has quired from it in the past.
To prevent DDoSing a Prolly tree with requests another layer of security measures needs to be taken.

# Limitations and Future Work

This document is intentionally simplified in its initial version.
It is to showcase the working pieces of the Prolly tree and how it can be used to synchronize the data in a p2p network.

The following ideas may be explored in future:

- Batch sequenced insertions/deletions: The design and the advantage of the Prolly tree over others is such that it can be used to insert/delete multiple sequenced messages at once without affecting the Merkle hashes of the Nodes already present in the Prolly tree i.e. left side of the tree.

- It can itself be served as a Store of messages in lightweight Nodes with limited storage capacity.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## informative

RFCs of request-response protocols:

- [75/WAKU2-SYNC](/spec/75)
