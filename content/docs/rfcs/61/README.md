---
slug: 61
title: 61/STATUS-Community-History-Archives
name: Status Community History Archives
status: raw
category: Standards Track
description: Explains how new members of a Status community can request historical messages from nodes archiving old data.
editor: Jimmy Debe <jimmy@status.im>
contributors:
  - Sanaz Taheri <sanaz@status.im>
  - John Lea <john@status.im>
  - r4bbit <r4bbit@status.im>
---

# Abstract

Messages are stored permanently by store nodes ([13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)) for up to a certain configurable period of time, limited by the overall storage provided by a store node.
Messages older than that period are no longer provided by store nodes, making it impossible for other nodes to request historical messages that go beyond that time range. 
This raises issues in the case of Status communities, where recently joined members of a community are not able to request complete message histories of the community channels.

This specification describes how **Control Nodes** (which are specific nodes in Status communities) archive historical message data of their communities, beyond the time range limit provided by Store Nodes using the [BitTorrent](https://bittorrent.org) protocol.
It also describes how the archives are distributed to community members via the Status network, so they can fetch them and gain access to a complete message history.

# Terminology

The following terminology is used throughout this specification. Notice that some actors listed here are nodes that operate in Waku networks only, while others operate in the Status communities layer):

| Name                 | References |
| -------------------- | --- |
| Waku node            | An Waku node ([10/WAKU2](https://rfc.vac.dev/spec/10/)) that implements [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/)|
| Store node           | A Waku node that implements [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) |
| Waku network         | A group of Waku nodes forming a graph, connected via [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/) |
| Status user          | An Status account that is used in a Status consumer product, such as Status Mobile or Status Desktop |
| Status node          | A Status client run by a Status application |
| Control node      | A Status node that owns the private key for a Status community |
| Community member     | A Status user that is part of a Status community, not owning the private key of the community |
| Community member node| A Status node with message archive capabilities enabled, run by a community member |
| Live messages        | Waku messages received through the Waku network |
| BitTorrent client    | A program implementing the [BitTorrent](https://bittorrent.org) protocol |
| Torrent/Torrent file | A file containing metadata about data to be downloaded by BitTorrent clients |
| Magnet link          | A link encoding the metadata provided by a torrent file ([Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme)) |

# Requirements / Assumptions

This specification has the following assumptions:

- Store nodes([13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)) are available 24/7, ensuring constant live message availability.
- The storage time range limit is 30 days.
- Store nodes have enough storage to persist historical messages for up to 30 days.
- No store nodes have storage to persist historical messages older than 30 days.
- All nodes are honest.
- The network is reliable.

Furthermore, it assumes that:

- Control nodes have enough storage to persist historical messages older than 30 days.
- Control nodes provide archives with historical messages **at least** every 30 days.
- Control nodes receive all community messages.
- Control nodes are honest.
- Control nodes know at least one store node from which it can query historical messages.

These assumptions are less than ideal and will be enhanced in future work. This [forum discussion](https://forum.vac.dev/t/status-communities-protocol-and-product-point-of-view/114) provides more details.

# Overview

The following is a high-level overview of the user flow and features this specification describes. For more detailed descriptions, read the dedicated sections in this specification.

## Serving community history archives

Control nodes go through the following (high level) process to provide community members with message histories:

1. Community owner creates a Status community (previously known as [org channels](https://github.com/status-im/specs/pull/151)) which makes its node a Control node.
2. Community owner enables message archive capabilities (on by default but can be turned off as well - see [UI feature spec](https://github.com/status-im/feature-specs/pull/36)).
3. A special type of channel to exchange metadata about the archival data is created, this channel is not visible in the user interface.
4. Community owner invites community members.
5. Control node receives messages published in channels and stores them into a local database.
6. After 7 days, the control node exports and compresses last 7 days worth of messages from database and bundles it together with a [message archive index](#waku-message-archive-index) into a torrent, from which it then creates a magnet link ([Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme), [Extensions for Peers to Send Metadata Files](https://www.bittorrent.org/beps/bep_0009.html)). 
7. Control node sends the magnet link created in step 6 to community members via special channel created in step 3 through the Waku network.
8. Every subsequent 7 days, steps 6 and 7 are repeated and the new message archive data is appended to the previously created message archive data.

## Serving archives for missed messages

If the control node goes offline (where "offline" means, the control node's main process is no longer running), it MUST go through the following process:

1. Control node restarts
2. Control node requests messages from store nodes for the missed time range for all channels in their community
3. All missed messages are stored into control node's local message database
4. If 7 or more days have elapsed since the last message history torrent was created, the control node will perform step 6 and 7 of [Serving community history archives](#serving-community-history-archives) for every 7 days worth of messages in the missed time range (e.g. if the node was offline for 30 days, it will create 4 message history archives)

## Receiving community history archives

Community member nodes go through the following (high level) process to fetch and restore community message histories:

1. User joins community and becomes community member (see [org channels spec](https://rfc.vac.dev/spec/56/))
2. By joining a community, member nodes automatically subscribe to special channel for message archive metadata exchange provided by the community
3. Member node requests live message history (last 30 days) of all the community channels including the special channel from store nodes
4. Member node receives Waku message ([14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/)) that contains the metadata magnet link from the special channel
5. Member node extracts the magnet link from the Waku message and passes it to torrent client
6. Member node downloads [message archive index](#message-history-archive-index) file and determines which message archives are not downloaded yet (all or some)
7. Member node fetches missing message archive data via torrent
8. Member node unpacks and decompresses message archive data to then hydrate its local database, deleting any messages for that community that the database previously stored in the same time range as covered by the message history archive

# Storing live messages

For archival data serving, the control node MUST store live messages as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/).
This is in addition to their database of application messages. 
This is required to provide confidentiality, authenticity, and integrity of message data distributed via the BitTorrent layer, and later validated by community members when they unpack message history archives.

Control nodes SHOULD remove those messages from their local databases once they are older than 30 days and after they have been turned into message archives and distributed to the BitTorrent network.

## Exporting messages for bundling

Control nodes export Waku messages from their local database for creating and bundling history archives using the following criteria:

- Waku messages to be exported MUST have a `contentTopic` that match any of the topics of the community channels
- Waku messages to be exported MUST have a `timestamp` that lies within a time range of 7 days

The `timestamp` is determined by the context in which the control node attempts to create a message history archives as described below:

1. The control node attempts to create an archive periodically for the past seven days (including the current day). In this case, the `timestamp` has to lie within those 7 days.
2. The control node has been offline (control node's main process has stopped and needs restart) and attempts to create archives for all the live messages it has missed since it went offline. In this case, the `timestamp` has to lie within the day the latest message was received and the current day.

Exported messages MUST be restored as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/) for bundling. Waku messages that are older than 30 days and have been exported for bundling can be removed from the control node's database (control nodes still maintain a database of application messages).

# Message history archives

Message history archives are represented as `WakuMessageArchive` and created from Waku messages exported from the local database. 
Message history archives are implemented using the following protocol buffer.

## WakuMessageHistoryArchive

The `from` field SHOULD contain a timestamp of the time range's lower bound.
The type parallels the `timestamp` of [WakuMessage](https://rfc.vac.dev/spec/14/#payload-encryption).

The `to` field SHOULD contain a timestamp of the time range's the higher bound.

The `contentTopic` field MUST contain a list of all communiity channel topics.

The `messages` field MUST contain all messages that belong into the archive given its `from`, `to` and `contentTopic` fields.

The `padding` field MUST contain the amount of zero bytes needed so that the overall byte size of the protobuf encoded `WakuMessageArchive` is a multiple of the `pieceLength` used to divide the message archive data into pieces.
This is needed for seamless encoding and decoding of archival data in interation with BitTorrent as explained in [creating message archive torrents](#creating-message-archive-torrents).

```
syntax = "proto3"

message WakuMessageArchiveMetadata {
  uint8 version = 1
  uint64 from = 2
  uint64 to = 3
  repeated string contentTopic = 4
}

message WakuMessageArchive {
  uint8 version = 1
  WakuMessageArchiveMetadata metadata = 2
  repeated WakuMessage messages = 3 // `WakuMessage` is provided by 14/WAKU2-MESSAGE
  bytes padding = 4
}
```

# Message history archive index

Control nodes MUST provide message archives for the entire community history.
The entirey history consists of a set of `WakuMessageArchive`'s where each archive contains a subset of historical `WakuMessage`s for a time range of seven days. 
All the `WakuMessageArchive`s are concatenated into a single file as a byte string (see [Ensuring reproducible data pieces](#ensuring-reproducible-data-pieces)).

Control nodes MUST create a message history archive index (`WakuMessageArchiveIndex`) with metadata that allows receiving nodes to only fetch the message history archives they are interested in.

## WakuMessageArchiveIndex

A `WakuMessageArchiveIndex` is a map where the key is the KECCAK-256 hash of the `WakuMessageArchiveIndexMetadata` derived from a 7-day archive and the value is an instance of that `WakuMessageArchiveIndexMetadata` corresponding to that archive.

The `offset` field MUST contain the position at which the message history archive starts in the byte string of the total message archive data. This MUST be the sum of the length of all previously created message archives in bytes (see [Creating message archive torrents](#creating-message-archive-torrents)).

```
syntax = "proto3"

message WakuMessageArchiveIndexMetadata {
  uint8 version = 1
  WakuMessageArchiveMetadata metadata = 2
  uint64 offset = 3
  uint64 num_pieces = 4
}

message WakuMessageArchiveIndex {
  map<string, WakuMessageArchiveIndexMetadata> archives = 1
}
```

The control node MUST update the `WakuMessageArchiveIndex` every time it creates one or more `WakuMessageArchive`s and bundle it into a new torrent.
For every created `WakuMessageArchive`, there MUST be a `WakuMessageArchiveIndexMetadata` entry in the `archives` field `WakuMessageArchiveIndex`.

# Creating message archive torrents

Control nodes MUST create a torrent file ("torrent") containing metadata to all message history archives.
To create a torrent file, and later serve the message archive data in the BitTorrent network, control nodes MUST store the necessary data in dedicated files on the file system.

A torrent's source folder MUST contain the following two files:

- `data` - Contains all protobuf encoded `WakuMessageArchive`'s (as bit strings) concatenated in ascending order based on their time
- `index` - Contains the protobuf encoded `WakuMessageArchiveIndex`

Control nodes SHOULD store these files in a dedicated folder that is identifiable via the community id.

## Ensuring reproducible data pieces

The control node MUST ensure that the byte string resulting from the protobuf encoded `data` is equal to the byte string `data` from the previously generated message archive torrent, plus the data of the latest 7 days worth of messages encoded as `WakuMessageArchive`.
Therefore, the size of `data` grows every seven days as it's append only.

The control nodes also MUST ensure that the byte size of every individual `WakuMessageArchive` encoded protobuf is a multiple of `pieceLength: ???` (**TODO**) using the `padding` field.
If the protobuf encoded 'WakuMessageArchive` is not a multiple of `pieceLength`, its `padding` field MUST be filled with zero bytes and the `WakuMessageArchive` MUST be re-encoded until its size becomes multiple of `pieceLength`.

This is necessary because the content of the `data` file will be split into pieces of `pieceLength` when the torrent file is created, and the SHA1 hash of every piece is then stored in the torrent file and later used by other nodes to request the data for each individual data piece.

By fitting message archives into a multiple of `pieceLength` and ensuring they fill possible remaining space with zero bytes, control nodes prevent the **next** message archive to occupy that remaining space of the last piece, which will result in a different SHA1 hash for that piece.

### **Example: Without padding**

Let `WakuMessageArchive` "A1" be of size 20 bytes:

```
 0 11 22 33 44 55 66 77 88 99
10 11 12 13 14 15 16 17 18 19 
```

With a `pieceLength` of 10 bytes, A1 will fit into `20 / 10 = 2` pieces:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
```

### **Example: With padding**

Let `WakuMessageArchive` "A2" be of size 21 bytes:

```
 0 11 22 33 44 55 66 77 88 99
10 11 12 13 14 15 16 17 18 19
20
```

With a `pieceLength` of 10 bytes, A2 will fit into `21 / 10 = 2` pieces. The remainder will introduce a third piece:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20                            // piece[2] SHA1: 0x789
```

The next `WakuMessageArchive` "A3" will be appended ("#3") to the existing data and occupy the remaining space of the third data piece. The piece at index 2 will now produce a different SHA1 hash:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[2] SHA1: 0xeef
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[3]
```

By filling up the remaining space of the third piece with A2 using its `padding` field, it is guaranteed that its SHA1 will stay the same:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20  0  0  0  0  0  0  0  0  0 // piece[2] SHA1: 0x999
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[3]
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[4]
```

# Seeding message history archives

The control node MUST seed the [generated torrent](#creating-message-archive-torrents) until a new `WakuMessageArchive` is created.

The control node SHOULD NOT seed torrents for older message history archives. Only one torrent at a time should be seeded.

# Creating magnet links

Once a torrent file for all message archives is created, the control node MUST derive a magnet link following the [Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme) using the underlying BitTorrent protocol client.

# Message archive distribution

Message archives are available via the BitTorrent network as they are being [seeded by the control node](#seeding-message-history-archives).
Other community member nodes will download the message archives from the BitTorrent network once they receive a magnet link that contains a message archive index.

The control node MUST send magnet links containing message archives and the message archive index to a special community channel. 
The topic of that special channel follows the following format:

```
/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}
```

All messages sent with this topic MUST be instances of `ApplicationMetadataMessage` ([62/PAYLOAD](https://rfc.vac.dev/spec/62)) with a `payload` of `CommunityMessageArchiveIndex`.

Only the control node MAY post to the special channel. Other messages on this specified channel MUST be ignored by clients.
Community members MUST NOT have permission to send messages to the special channel.
However, community member nodes MUST subscribe to special channel to receive Waku messages containing magnet links for message archives.

# Canonical message histories

Only control nodes are allowed to distribute messages with magnet links via the special channel for magnet link exchange.
Community members MUST NOT be allowed to post any messages to the special channel.

Status nodes MUST ensure that any message that isn't signed by the control node in the special channel is ignored.

Since the magnet links are created from the control node's database (and previously distributed archives), the message history provided by the control node becomes the canonical message history and single source of truth for the community.

Community member nodes MUST replace messages in their local databases with the messages extracted from archives within the same time range.
Messages that the control node didn't receive  MUST be removed and are no longer part of the message history of interest, even if it already existed in a community member node's database.

# Fetching message history archives

Generally, fetching message history archives is a three step process:

1. Receive [message archive index](#message-history-archive-index) magnet link as described in [Message archive distribution], download `index` file from torrent, then determine which message archives to download
2. Download individual archives

Community member nodes subscribe to the special channel that control nodes publish magnet links for message history archives to. 
There are two scenarios in which member nodes can receive such a magnet link message from the special channel:

1. The member node receives it via live messages, by listening to the special channel 
2. The member node requests messages for a time range of up to 30 days from store nodes (this is the case when a new community member joins a community)

## Downloading message archives
When member nodes receive a message with a `CommunityMessageHistoryArchive` ([62/PAYLOAD](https://rfc.vac.dev/spec/62)) from the aforementioned channnel, they MUST extract the `magnet_uri` and pass it to their underlying BitTorrent client so they can fetch the latest message history archive index, which is the `index` file of the torrent (see [Creating message archive torrents](#creating-message-archive-torrents)).

Due to the nature of distributed systems, there's no guarantee that a received message is the "last" message. This is especially true when member nodes request historical messages from store nodes. 

Therefore, member nodes MUST wait for 20 seconds after receiving the last `CommunityMessageArchive` before they start extracting the magnet link to fetch the latest archive index.

Once a message history archive index is downloaded and parsed back into `WakuMessageArchiveIndex`, community member nodes use a local lookup table to determine which of the listed archives are missing using the KECCAK-256 hashes stored in the index.

For this lookup to work, member nodes MUST store the KECCAK-256 hashes of the `WakuMessageArchiveIndexMetadata` provided by the `index` file for all of the message history archives that have been downlaoded in their local database.

Given a `WakuMessageArchiveIndex`, member nodes can access individual `WakuMessageArchiveIndexMetadata` to download individual archives.

Community member nodes MUST choose one of the following options:

1. **Download all archives** - Request and download all data pieces for `data` provided by the torrent (this is the case for new community member nodes that haven't downloaded any archives yet)
2. **Download only the latest archive** - Request and download all pieces starting at the `offset` of the latest `WakuMessageArchiveIndexMetadata` (this the case for any member node that already has downloaded all previous history and is now interested in only the latst archive)
3. **Download specific archives** - Look into `from` and `to` fields of every `WakuMessageArchiveIndexMetadata` and determine the pieces for archives of a specific time range (can be the case for member nodes that have recently joined the network and are only interested in a subset of the complete history)

# Storing historical messages

When message archives are fetched, community member nodes MUST unwrap the resulting `WakuMessage` instances into `ApplicationMetadataMessage` instances and store them in their local database.
Community member nodes SHOULD NOT store the wrapped `WakuMessage` messages.

All message within the same time range MUST be replaced with the messages provided by the message history archive.

Community members nodes MUST ignore the expiration state of each archive message.

# Considerations

The following are things to cosider when implementing this specification.

## Control node honesty

This spec assumes that all control nodes are honest and behave according to the spec. Meaning they don't inject their own messages into, or remove any messages from historic archives.

## Bandwidth consumption

Community member nodes will download the latest archive they've received from the archive index, which includes messages from the last seven days. Assuming that community members nodes were online for that time range, they have already downloaded that message data and will now download an archive that contains the same.

This means there's a possibility member nodes will download the same data at least twice.

## Multiple community owners

It is possible for control nodes to export the private key of their owned community and pass it to other users so they become control nodes as well.
This means, it's possible for multiple control nodes to exist.

This might conflict with the assumption that the control node serves as a single source of thruth. Multiple control nodes can have different message histories.

Not only will multiple control nodes multiply the amount of archive index messages being distributed to the network, they might also contain different sets of magnet links and their corresponding hashes.

Even if just a single message is missing in one of the histories, the hashes presented in archive indices will look completely different, resulting in the community member node to download the corresponding archive (which might be identical to an archive that was already downloaded, except for that one message).

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References
* [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)
* [BitTorrent](https://bittorrent.org)
* [10/WAKU2](https://rfc.vac.dev/spec/10/)
* [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/)
* [Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme)
* [forum discussion](https://forum.vac.dev/t/status-communities-protocol-and-product-point-of-view/114)
* [org channels](https://github.com/status-im/specs/pull/151)
* [UI feature spec](https://github.com/status-im/feature-specs/pull/36))
* [Extensions for Peers to Send Metadata Files](https://www.bittorrent.org/beps/bep_0009.html)
* [org channels spec](https://rfc.vac.dev/spec/56/)
* [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/)
* [62/PAYLOAD](https://rfc.vac.dev/spec/62)

