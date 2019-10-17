# Specifications

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

## Status

The entire vac protocol is under active development, each specification has its own `status` which is reflected through the version number at the top of every document. We use [semver](https://semver.org/) to version these specifications.

## Protocols

These protocols define various components of the [vac](https://vac.dev) stack.

 - [mvds](./mvds.md) - Data Synchronization protocol for unreliable transports.
 - [remote log](./remote-log.md) - Remote replication of local logs.


## Style guide
<!-- @Todo put this in a better place !-->
All specs follow [RFC-2119](https://tools.ietf.org/html/rfc2119).

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`. Both the source and generated image should be in source control. For ease of readability, the generated image is embedded inside the main spec document.
