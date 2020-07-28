![CI](https://github.com/vacp2p/specs/workflows/CI/badge.svg)

This repository contains the specs for [vac](https://vac.dev), a modular peer-to-peer messaging stack, with a focus on secure messaging. A detailed explanation of the vac and its design goals can be found [here](https://vac.dev/vac-overview).

## Status

The entire vac protocol is under active development, each specification has its own `status` which is reflected through the version number at the top of every document. We use [semver](https://semver.org/) to version these specifications.

## Protocols

These protocols define various components of the [vac](https://vac.dev) stack.

 - [mvds](./specs/mvds.md) - Data Synchronization protocol for unreliable transports.
 - [remote log](./specs/remote-log.md) - Remote replication of local logs.
 - [mvds metadata](./specs/mvds-metadata.md) - Metadata field for [MVDS](./specs/mvds.md) messages. 

### Waku

Waku is a protocol that substitutes [EIP-627](https://eips.ethereum.org/EIPS/eip-627).

 - [waku](./specs/waku/waku.md) - ÐΞVp2p wire protocol, substituting [EIP-627](https://eips.ethereum.org/EIPS/eip-627).
 - [envelope data format](./specs/waku/envelope-data-format.md) - [waku](./specs/waku/waku.md) envelope data field specification.
 - [mailserver](./specs/waku/mailserver.md) - Mailserver specification for archiving and delivering historical [waku](./specs/waku/waku.md) envelopes on demand.
 - [rpc api](./specs/waku/waku-rpc-api.md) - The RPC API for interacting with a Waku node.

Waku v2 is in early development. Here you can read more about the [plan for
it](https://vac.dev/waku-v2-plan). Here's the current raw spec:

- [waku v2](https://specs.vac.dev/specs/waku/waku-v2.html)

## Style guide

Sequence diagrams are generated using [Mscgen](http://www.mcternan.me.uk/mscgen/) like this: `mscgen -T png -i input.msc -o output.png`. Both the source and generated image should be in source control. For ease of readability, the generated image is embedded inside the main spec document.
The lifecycle of the specs follows the [COSS Lifecycle](https://rfc.unprotocols.org/spec:2/COSS/)

## Meta

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Linting

### Spellcheck

To run the spellchecker locally, you must install [pyspelling](https://facelessuser.github.io/pyspelling/).

It can then be run with the following command:

```console
pyspelling -c .pyspelling.yml
```

Words that should be ignored or are unrecognized must be added to the [wordlist](./wordlist.txt).

### Markdown Verification

We use [remark](https://remark.js.org/) to verify our markdown. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run lint
```

### Textlint

We use [textlint](https://textlint.github.io/) for extra markdown verification. You can easily run this tool simply by using our `npm` package:

```console
npm install
npm run textlint
```
