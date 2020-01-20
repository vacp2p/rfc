# Waku Mailserver

> Version 0.2.0
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
topic-interest   = "[" *1000topic "]"

; 4 bytes of arbitrary data
topic = 4OCTET

payload_without_topic = "[" lower upper bloom limit [ cursor ] "]"

payload_with_topic = "[" lower upper bloom limit cursor [ topic-interest ] "]"

payload = payload_without_topic | payload_with_topic
```

The `Cursor` field SHOULD be filled in if a number of envelopes between `Lower` and `Upper` is greater than `Limit` so that the requester can send another request using the obtained `Cursor` value. What exactly is in the `Cursor` is up to the implementation. The requester SHOULD NOT use a `Cursor` obtained from one mailserver in a request to another mailserver because the format or the result MAY be different.

The envelope MUST be encrypted with a symmetric key agreed between the requester and Mailserver.

If `topic-interest` is used the `Cursor` field MUST be specified for the argument order to be unambiguous. However, it MAY be set to null. `topic-interest` is used to specify which topics a node is interested in. If this is specified, a mailserver MUST NOT send messages that aren't in in that topic. When `topic-interest` is set (even if it an empty array), the `bloom` filter parameter should be ignored.

### Receiving historic messages

Historic messages MUST be sent to a peer as a packet with a P2P Message code (`0x7f`) followed by an array of Waku envelopes.

In order to receive historic messages from a mailserver, a node MUST trust the selected mailserver, that is allow to receive expired packets with the P2P Message code. By default, such packets are discarded.

Received envelopes MUST be passed through the Whisper envelope pipelines so that they are picked up by registered filters and passed to subscribers.

For a requester, to know that all messages have been sent by mailserver, it SHOULD handle P2P Request Complete code (`0x7d`). This code is followed by a list with:

```abnf
; array with a Keccak-256 hash of the envelope containing the original request.
request-id = 32OCTET

; array with a Keccak-256 hash of the last sent envelope for the request. 
last-envelope-hash = 32OCTET

; array of a cursor returned from the previous request (optional)
cursor = *OCTET

payload = "[" request-id last-envelope-hash [ cursor ] "]"
```

If `Cursor` is not empty, it means that not all messages were sent due to the set `Limit` in the request. One or more consecutive requests MAY be sent with `Cursor` field filled in in order to receive the rest of the messages.

### Security considerations

There are several security considerations to take into account when running or interacting with Mailservers. Chief among them are: scalability, DDoS-resistance and privacy.

**Mailserver High Availability requirement:**

A mailserver has to be online to receive messages for other nodes, this puts a high availability requirement on it.

**Mailserver client privacy:**

A mailserver client has to trust a mailserver, which means they can send direct traffic. This reveals what topics / bloom filter a node is interested in, along with its peerID (with IP).

**Mailserver trusted connection:**

A mailserver has a direct TCP connection, which means they are trusted to send traffic. This means a malicious or malfunctioning mailserver can overwhelm an individual node.

## Changelog

| Version | Comment |
| :-----: | ------- |
| 0.2.0 (current)  | Add topic interest to reduce bandwidth usage |
| 0.1.0  | Initial Release |

### Difference between wms 0.1 and wms 0.2

- `topic-interest` option

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
