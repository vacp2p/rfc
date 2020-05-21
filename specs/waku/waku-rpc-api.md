---
title: Waku RPC API
version: 1.0.0
status: Draft
authors: Dean Eigenmann <dean@status.im>
---

## Table of Contents

1. [Abstract](#abstract)
2. [Methods](#methods)
3. [Copyright](#copyright)

## Abstract

In this specification we describe the RPC API that Waku nodes SHOULD adhere to. The unified API allows clients to easily
be able to connect to any node implementation. 

## Methods

### `waku_addSymKey`

The `waku_addSymKey` method stores a symmetric key on the node and returns its ID.

### `waku_post`

The `waku_post` method creates a [waku message](./waku-1.md#messages) and disseminates it to the network.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
