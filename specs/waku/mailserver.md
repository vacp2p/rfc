---
title: Waku Mailserver
version: 0.2.0
status: Draft
authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Oskar Thorén <oskar@status.im>
redirect_from:
  - /waku/mailserver.html
---

## Table of Contents

1. [Abstract](#abstract)
2. [Specification](#specification)
    1. [Requesting Historic Envelopes](#requesting-historic-envelopes)
    2. [Receiving Historic Envelopes](#receiving-historic-envelopes)
    3. [Protocol Buffers](#protocol-buffers)
    4. [Security considerations](#security-considerations)
3. [Copyright](#copyright)

## Abstract

In this specification, we describe Mailservers. These are nodes responsible for archiving envelopes and delivering them to peers on-demand.

## Specification

A node which wants to provide mailserver functionality MUST store envelopes from incoming Messages packets (Waku packet-code `0x01`). The envelopes can be stored in any format, however they MUST be serialized and deserialized to the Waku envelope format.

A mailserver SHOULD store envelopes for all topics to be generally useful for any peer, however for specific use cases it MAY store envelopes for a subset of topics.

### Requesting Historic Envelopes

In order to request historic envelopes, a node MUST send a packet P2P Request (`0x7e`) to a peer providing mailserver functionality. This packet requires one argument which MUST be a Waku envelope.

In the Waku envelope's payload section, there MUST be RLP-encoded information about the details of the request:

```abnf
; UNIX time in seconds; oldest requested envelope's creation time
lower  = 4OCTET

; UNIX time in seconds; newest requested envelope's creation time
upper  = 4OCTET

; array of Waku topics encoded in a bloom filter to filter envelopes
bloom  = 64OCTET

; unsigned integer limiting the number of returned envelopes
limit  = 4OCTET

; array of a cursor returned from the previous request (optional)
cursor = *OCTET

; List of topics interested in
topics = "[" *1000topic "]"

; 4 bytes of arbitrary data
topic = 4OCTET

payload-without-topic = "[" lower upper bloom limit [ cursor ] "]"

payload-with-topic = "[" lower upper bloom limit cursor [ topics ] "]"

payload = payload-with-topic | payload-without-topic
```

The `Cursor` field SHOULD be filled in if a number of envelopes between `Lower` and `Upper` is greater than `Limit` so that the requester can send another request using the obtained `Cursor` value. What exactly is in the `Cursor` is up to the implementation. The requester SHOULD NOT use a `Cursor` obtained from one mailserver in a request to another mailserver because the format or the result MAY be different.

The envelope MUST be encrypted with a symmetric key agreed between the requester and Mailserver.

If `Topics` is used the `Cursor` field MUST be specified for the argument order to be unambiguous. However, it MAY be set to `null`. `Topics` is used to specify which topics a node is interested in. If `Topics` is not empty, a mailserver MUST only send envelopes that belong to a topic from `Topics` list and `Bloom` value MUST be ignored.

### Receiving Historic Envelopes

Historic envelopes MUST be sent to a peer as a packet with a P2P Message code (`0x7f`) followed by an array of Waku envelopes. A Mailserver MUST limit the amount of messages sent, either by the `Limit` specified in the request or limited to the maximum [RLPx packet size](./waku#maximum-packet-size), whichever limit comes first.

In order to receive historic envelopes from a mailserver, a node MUST trust the selected mailserver, that is allow to receive expired packets with the P2P Message code. By default, such packets are discarded.

Received envelopes MUST be passed through the Whisper envelope pipelines so that they are picked up by registered filters and passed to subscribers.

For a requester, to know that all envelopes have been sent by mailserver, it SHOULD handle P2P Request Complete code (`0x7d`). This code is followed by a list with:

```abnf
; array with a Keccak-256 hash of the envelope containing the original request.
request-id = 32OCTET

; array with a Keccak-256 hash of the last sent envelope for the request. 
last-envelope-hash = 32OCTET

; array of a cursor returned from the previous request (optional)
cursor = *OCTET

payload = "[" request-id last-envelope-hash [ cursor ] "]"
```

If `Cursor` is not empty, it means that not all envelopes were sent due to the set `Limit` in the request. One or more consecutive requests MAY be sent with `Cursor` field filled in in order to receive the rest of the envelopes.

### Protocol Buffers

```protobuf
message Cursor {
  bytes before = 1;
  bytes after = 2;
}

message Query {
  int32 from = 1;
  int32 to = 2;
  bytes bloom = 3;
  int32 limit = 4;
  repeated bytes topics = 5;
  Cursor cursor = 6;
}

message Envelope {
  int32 expiry = 1;
  int32 ttl = 2;
  bytes nonce = 3;
  bytes topic = 4;
  bytes data = 5;
}

message Response {
  repeated Envelope envelopes = 1;
  Cursor cursor = 2;
}

service Mailserver {
  rpc Fetch (Query) returns (Response);
}
```

**Either `bloom` or `topics` MUST be present.**

### Security considerations

There are several security considerations to take into account when running or interacting with Mailservers. Chief among them are: scalability, DDoS-resistance and privacy.

**Mailserver High Availability requirement:**

A mailserver has to be online to receive envelopes for other nodes, this puts a high availability requirement on it.

**Mailserver client privacy:**

A mailserver client has to trust a mailserver, which means they can send direct traffic. This reveals what topics / bloom filter a node is interested in, along with its peerID (with IP).

**Mailserver trusted connection:**

A mailserver has a direct TCP connection, which means they are trusted to send traffic. This means a malicious or malfunctioning mailserver can overwhelm an individual node.

## Changelog

| Version | Comment |
| :-----: | ------- |
| 0.2.0 (current)  | Add topic interest to reduce bandwidth usage |
| [0.1.0](https://github.com/vacp2p/specs/blob/06d4c736c920526472a507e5d842212843a112ed/wms.md)  | Initial Release |

### Difference between wms 0.1 and wms 0.2

- `topics` option

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
