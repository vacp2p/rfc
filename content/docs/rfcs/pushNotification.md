---
slug: XX
title: 
name: Status Push Notification Server
status: raw
category: Standards Track
editor: Jimmy Debe
---


# Abstract
Push notification server implementation for IOS devices and Android devices. This implementation is a set of functions that will allow clients to have push notification services in restricting mobile environments.

# Background

Mobile devices using iOS and some android devices have restrictions for applications that want to use push notifications. Apple devices can use the APN service, Apple Push Notifications, or Firebase for Android devices. For some Android devices, foreground services are restricted, require a user to grant the foreground notifications, seen as dangerous. Apple iOS devices, restrict notifications to some internal functions, that not every application can use. Applications request to use background but does not schedule background resources if the application was forced quit and is not background services are not responsive envoy for push notification systems.

# Specification
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119.

### Definitions:
client = A node that implements the Status specification. <br />
User = The owner of a device that runs a client<br />
topic = <br />
server = A service that performs push notifications<br />
Token? = <br />
Mailserver = A Waku node that decides to provide functionality to store messages permanently and deliver the messages to requesting clients.<br />

### Components:
gorush Instance = 
Only used by push notification servers and MUST be publicly available.<br />

Push Notification Server =
Used by clients to register for receiving and sending notifications.<br />

Registering Client =
A client that want to receive push notifications.<br />

Sending Client =
A client that want to send push notifications.<br />

Requirements = The party releasing the app MUST possess a certificate for the Apple Push Notification service and its has to run a [gorush](https://github.com/appleboy/gorush) publicly accessible server for sending the actual notification. The party releasing the app, Status in this case, needs to run its own [gorush](https://github.com/appleboy/gorush).<br />



## Push Notification Server Flow
### Registration Process:

![image](https://github.com/jimstir/rfc/assets/91767824/00cae4bf-6eb5-4e7b-9377-3a5d01b041a1)

### Sending and Receiving Notification Process:

![image](https://github.com/jimstir/rfc/assets/91767824/cd33c21d-26f1-4137-834c-d0e1a536600e)

## Registering Client

Registering a client with a push notification service.

- A client MAY register with one or more push notification services in order to increase availability.
- A client SHOULD make sure that all the notification services they registered with have the same information about their tokens.
- A `PNR message` (Push Notification Registration) MUST be sent to the [partitioned topic](https://specs.status.im/spec/10#partitioned-topic) for the public key of the node, encrypted with this key.
- The message MUST be wrapped in a `ApplicationMetadataMessage` with type set to `PUSH_NOTIFICATION_REGISTRATION`.
- The marshaled protobuf payload MUST also be encrypted with AES-GCM using the Diffie–Hellman key generated from the client and server identity. This is done in order to ensure that the extracted key from the signature will be considered invalid if it can’t decrypt the payload.

The content of the message MUST contain the following [protobuf record](https://developers.google.com/protocol-buffers/):





