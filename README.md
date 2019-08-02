# Specifications

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

## Status

The entire vac protocol is under active development, each specification has its own `status` which is reflected through the version number at the top of every document. We use [semver](https://semver.org/) to version these specifications.

## Protocols

These protocols define various components of the [vac](https://vac.dev) stack.

 - [mvds](./mvds.md) - Ensure reliable messaging between peers across an unreliable peer-to-peer (P2P) network where they may be unreachable or unresponsive.
 - [mdsc](./mdsc.md) - A minimal data sync client that utilizes [mvds](./mvds.md) to distribute messages amongst participants.
 
<!-- @Todo put this in a better place !-->
All specs follow [RFC-2119](https://tools.ietf.org/html/rfc2119).

## Contributing
