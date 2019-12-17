# Specifications

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

## Status

The entire vac protocol is under active development, each specification has its own `status` which is reflected through the version number at the top of every document. We use [semver](https://semver.org/) to version these specifications.

## Protocols

These protocols define various components of the [vac](https://vac.dev) stack.

 - [mvds](./mvds.md) - Data Synchronization protocol for unreliable transports.
 - [remote log](./remote-log.md) - Remote replication of local logs.
 - [mdf](./mdf.md) - Metadata field for [MVDS](./mvds.md) messages. 

### Waku

Waku is a protocol to substituting [EIP-627](https://eips.ethereum.org/EIPS/eip-627).

 - [waku](./waku/waku.md) - ÐΞVp2p wire protocol, substituting [EIP-627](https://eips.ethereum.org/EIPS/eip-627).
 - [wedf](./waku/wedf.md) - [Waku](./waku.md) envelope data field specification.
 - [wms](./waku/wms.md) - Mailserver specification for archiving and delivering historical [waku](./waku.md) envelopes on demand.

## Style guide

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`. Both the source and generated image should be in source control. For ease of readability, the generated image is embedded inside the main spec document.

## Meta

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).
