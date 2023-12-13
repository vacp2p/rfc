---
slug: 67
title: 67/STATUS-WAKU2-USAGE
name: Status Waku2 Usage
status: raw
category: Standards Track
description: Get familiar with all the Waku protocols used by the Status application.
editor: Jimmy Debe <jimmy@status.im>
contributors: 
- Aaryamann Challani <aaryamann@status.im>

---

# Abstract

Status is a chat application which has several features, including, but not limited to -
- Private 1:1 chats, described by [55/STATUS-1TO1-CHAT](/spec/55)
- Large scale group chats, described by [56/STATUS-COMMUNITIES](/spec/56)

This specification describes how a Status implementation will make use of the underlying infrastructure, 
Waku, which is described in [10/WAKU2](/spec/10).

# Background 

The Status application aspires to achieve censorship resistance and incorporates specific privacy features, 
leveraging the comprehensive set of protocols offered by Waku to enhance these attributes. 
Waku protocols provide secure communication capbailites over decentralized networks. 
Once integrated, an application will benifit from privacy preserving, 
censorship resistance and spam protected communcation. 

Since Status uses a large set of Waku protocols, 
it is imperative to describe how each are used. 

# Definitions

| Terminology  | Description |
| --------------- | --------- |
| `RELAY`| This refers to the Waku Relay protocol, described in [11/WAKU2-RELAY](/spec/11) |
|`FILTER` | This refers to the Waku Filter protocol, described in [12/WAKU2-FILTER](/spec/12) |
| `STORE` | This refers to the Waku Store protocol, described in [13/WAKU2-STORE](/spec/13) |
| `MESSAGE` | This refers to the Waku Message format, described in [14/WAKU2-MESSAGE](/spec/14) |
| `Pubsub Topic` / `Content Topic` | This refers to the routing of messages within the Waku network, described in [23/WAKU2-TOPICS](/spec/23/) |
| `LIGHTPUSH` | This refers to the Waku Lightpush protocol, described in [19/WAKU2-LIGHTPUSH](/spec/19) |

# Semantics
Waku Node : A server running Waku software configured with a set of protocols

Light Client: Status clients that operate within resource constrained environments, and uses only a subset of Waku Protocols.


# Protocol Usage

The following is a list of Waku Protocols used by the Status application.


## 1. `RELAY`

> Note: This protocol MUST not be used by Status light clients.

Waku Relay is used to broadcast messages from a Status Client.
All Status messages are transformed into Waku Messages, [14/WAKU2-MESSAGE](/spec/14), which are sent over the wire.
All Status message types are described in [62/STATUS-PAYLOAD](/spec/62).

Status Clients MUST transform the following object into a `MESSAGE` as described below -

```go
StatusSendMessage {
	SymKey[]     []byte // [optional] The symmetric key used to encrypt the message
	PublicKey    []byte // [optional] The public key to use for asymmetric encryption
	Sig          string // [optional] The private key used to sign the message
	PubsubTopic  string // The Pubsub topic to publish the message to     
	ContentTopic string // The Content topic to publish the message to
	Payload      []byte // A serialized representation of a Status message to be sent
	Padding      []byte // Padding that must be applied to the Payload
	TargetPeer   string // [optional] The target recipient of the message
	Ephemeral    bool   // If the message is not to be stored, this is set to `true` 
}
```

1. A user MUST only provide either a Symmetric key OR an Asymmetric keypair to encrypt the message.
If both are received, the implementation MUST throw an error.
2. `WakuMessage.Payload` MUST be set to `StatusMessage.Payload` 
3. `WakuMessage.Key` MUST be set to `StatusMessage.SymKey`
<!-- 
Curious about the key that is being set within the message here -
https://github.com/status-im/status-go/blob/dc6fe5613a0b59249b11d14477f581ed636a8153/wakuv2/api.go#L217-L219
are we sharing the symmetric key for each message?
-->
4. `WakuMessage.Version` MUST be set to `1`
5. `WakuMessage.Ephemeral` MUST be set to `StatusMessage.Ephemeral`
6. `WakuMessage.ContentTopic` MUST be set to `StatusMessage.ContentTopic`
7. `WakuMessage.Timestamp` MUST be set to the current Unix epoch timestamp (in nanosecond precision)

## 2. `STORE`

> Note: This protocol MUST remain optional according to the user's preferences,
it MAY be enabled on Light clients as well.

Messages received via Waku Relay, are stored in a database as described in [13/WAKU2-STORE](/spec/13).
The messages that have the `ephemeral` flag set to true will not be stored.

The Status Client SHOULD provide a method to prune the database of older records to save storage.

## 3. `FILTER`

> Note: This protocol SHOULD be enabled on Light clients.

This protocol SHOULD be used to filter messages based on a given criteria, such as the Content Topic of a Waku Message.

This allows a reduction in bandwidth consumption by the Status Client.

Status Clients SHOULD apply a filter for all the Content Topics they are interested in, 
such as Content Topics derived from -
1. 1:1 chats with other users, described in [55/STATUS-1TO1-CHAT](/spec/55)
2. Group chats
3. Community Channels, described in [56/STATUS-COMMUNITIES](/spec/56)

## 4. `LIGHTPUSH`

> Note: This protocol MUST be enabled solely on Light clients.

This protocol allows Status Light Clients to publish messages to the Waku Network,
without running a full-fledged Waku Relay protocol.

When a Status Client is publishing a message, 
it MUST check if Light mode is enabled,
and if so, it MUST publish the message via `LIGHTPUSH`.

## 5. `DISCOVERY`

> Note: This protocol MUST be supported by Light clients and Full clients

Status Clients SHOULD make use of the following peer discovery methods that are provided by Waku,
such as -

1. [EIP-1459: DNS-Based Discovery](https://eips.ethereum.org/EIPS/eip-1459)
2. [33/WAKU2-DISCV5](/spec/33)
3. [34/WAKU2-PEER-EXCHANGE](/spec/34)

Status Clients MAY use any combination of the above peer discovery methods, 
which is suited best for their implementation.

# Security/Privacy Considerations

This specification inherits the security and privacy considerations from the following specifications -

1. [10/WAKU2](/spec/10)
2. [11/WAKU2-RELAY](/spec/11)
3. [12/WAKU2-FILTER](/spec/12)
4. [13/WAKU2-STORE](/spec/13)
5. [14/WAKU2-MESSAGE](/spec/14)
6. [23/WAKU2-TOPICS](/spec/23)
7. [19/WAKU2-LIGHTPUSH](/spec/19)
8. [55/STATUS-1TO1-CHAT](/spec/55)
9. [56/STATUS-COMMUNITIES](/spec/56)
10. [62/STATUS-PAYLOAD](/spec/62)
11. [EIP-1459: DNS-Based Discovery](https://eips.ethereum.org/EIPS/eip-1459)
12. [33/WAKU2-DISCV5](/spec/33)
13. [34/WAKU2-PEER-EXCHANGE](/spec/34)

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## normative

- [10/WAKU2](/spec/10)
- [11/WAKU2-RELAY](/spec/11)
- [12/WAKU2-FILTER](/spec/12)
- [13/WAKU2-STORE](/spec/13)
- [14/WAKU2-MESSAGE](/spec/14)
- [23/WAKU2-TOPICS](/spec/23)
- [19/WAKU2-LIGHTPUSH](/spec/19)

## informative
- [55/STATUS-1TO1-CHAT](/spec/55)
- [56/STATUS-COMMUNITIES](/spec/56)
- [62/STATUS-PAYLOAD](/spec/62)
- [33/WAKU2-DISCV5](/spec/33)
- [34/WAKU2-PEER-EXCHANGE](/spec/34)
