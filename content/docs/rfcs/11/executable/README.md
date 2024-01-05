# Build a Waku Node: Waku Relay

## Introduction

This is the first guide in the tutorial demonstatraing how to build your own Waku node using python. 
Waku provides a collection of protocols on top of libp2p to create mesage anonymity.
These guides will use the core protocols of Waku, as described in [10/WAKU2](https://rfc.vac.dev/spec/10/), 
other protocols can be added to your node after completing the tutorial.
This tutorial will focus on the Waku Relay protocol.

## Configuration

Nodes must use configurations detailed in [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/).
The [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/) is the most important protocol to implement,
as a Waku node should be a running a relay, as detailed (here)[].
Since this is the first 

Let's set up the basic libp2p modules that are need for a Waku Relay. 
First, lets create a directory for our new project:

``` bash

# create a new directory and make sure you change into it
> mkdir waku-node
> cd waku-node

```
In your new directory, download the supported py-libp2p from the github repository.

> Note: py-libp2p is still under development and should not be used for production envirnoments.

``` bash

> git clone git@github.com:libp2p/py-libp2p.git

```

## Configuration
A Waku node is Publish/Subscribe which allows peers to communicate with each other.
Publish/Subscribe allows peers to join topics, within a network,
that they are interseted in.
Once joined, they are able to send and recieve messages within the topic.

The gossipsub protocol is the Publish/Subscribe protocol used in a Waku node.


``` python
# import the necessary py-libp2p
from libp2p.pubsub import pubsub
from libp2p.pubsub import gossipsub
from libp2p.peer.id import ID

```

