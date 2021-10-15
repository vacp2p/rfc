---
slug: 31
title: 31/WAKU2-ENR
name: Waku v2 usage of ENR
status: raw
editor: Franck Royer <franck@status.im>
contributors:
---

# Abstract

This RFC describes the usage of the ENR (Ethereum Node Records) format for Waku v2 purposes.
The ENR format is defined in [EIP-778](https://eips.ethereum.org/EIPS/eip-778).

This RFC is an extension of EIP-778, ENR used in Waku v2 MUST adhere to both EIP-718 and 31/WAKU2-ENR.

# Motivation

EIP-1459 with the usage of ENR has been implemented [1] [2] as a discovery protocol for Waku v2.

EIP-778 specifies a number of pre-defined keys.
However, the usage of these keys alone does not allow for certain transport capabilities to be encoded,
such as Websocket.

Currently, Waku v2 nodes running in a Browser only support websocket transport protocol.a

Hence, new ENR keys needs to be defined to allow for the encoding of transport protocol other than raw TCP.

# `multiaddr` ENR key

TODO

# References

- [1] https://github.com/status-im/nim-waku/pull/690 
- [2] https://github.com/vacp2p/rfc/issues/462#issuecomment-943869940 
