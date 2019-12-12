# Waku Mailserver

> Version 0.1.0
>
> Authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Table of Contents

1. [Abstract](#abstract)
2. [Specification](#specification)
    1. [Requesting messages](#requesting-messages)
    2. [Receiving historic messages](#receiving-historic-messages)
    3. [Security considerations](#security-considerations)
3. [Copyright](#copyright)

## Abstract

In this specification, we describe Mailservers. These are nodes responsible for archiving messages and delivering them too peers on-demand.

## Specification

A node which wants to provide mailserver functionality MUST store envelopes from incoming message packets (Waku packet-code 0x01). The envelopes can be stored in any format, however they MUST be serialized and deserialized to the Waku envelope format.

A mailserver SHOULD store envelopes for all topics to be generally useful for any peer, however for specific use cases it MAY store envelopes for a subset of topics.

### Requesting messages

In order to request historic messages, a node MUST send a packet P2P Request (`0x7e`) to a peer providing mailserver functionality. This packet requires one argument which MUST be a Waku envelope.

In the Waku envelope's payload section, there MUST be RLP-encoded information about the details of the request:

```
[ Lower, Upper, Bloom, Limit, Cursor ]
```

| Field    |   Description                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------ |
| `Lower`  |   4-byte wide unsigned integer (UNIX time in seconds; oldest requested envelope's creation time) |
| `Upper`  |   4-byte wide unsigned integer (UNIX time in seconds; newest requested envelope's creation time) |
| `Bloom`  |   64-byte wide array of Waku topics encoded in a bloom filter to filter envelopes                |
| `Limit`  |   4-byte wide unsigned integer limiting the number of returned envelopes                         |
| `Cursor` |   32-byte wide array of a cursor returned from the previous request (optional)                   |

The `Cursor` field SHOULD be filled in if a number of envelopes between `Lower` and `Upper` is greater than `Limit` so that the requester can send another request using the obtained `Cursor` value. What exactly is in the `Cursor` is up to the implementation. The requester SHOULD NOT use a `Cursor` obtained from one mailserver in a request to another mailserver because the format or the result MAY be different.

The envelope MUST be signed with a symmetric key agreed between the requester and Mailserver.

### Receiving historic messages

Historic messages MUST be sent to a peer as a packet with a P2P Message code (`0x7f`) followed by an array of Waku envelopes.

In order to receive historic messages from a mailserver, a node MUST trust the selected mailserver, that is allow to receive expired packets with the P2P Message code. By default, such packets are discarded.

Received envelopes MUST be passed through the Waku envelopes pipelines so that they are picked up by registered filters and passed to subscribers.

### Security considerations

There are several security considerations to take into account when running or interacting with Mailservers. Chief among them are: scalability, DDoS-resistance and privacy.

**Mailserver High Availability requirement:**

A mailserver has to be online to receive messages for other nodes, this puts a high availability requirement on it.

**Mailserver client privacy:**

A mailserver client has to trust a mailserver, which means they can send direct traffic. This reveals what topics / bloom filter a node is interested in, along with its peerID (with IP).

**Mailserver trusted connection:**

A mailserver has a direct TCP connection, which means they are trusted to send traffic. This means a malicious or malfunctioning mailserver can overwhelm an individual node.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
