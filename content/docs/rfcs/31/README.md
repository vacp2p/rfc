---
slug: 31
title: 31/WAKU2-TRANSPORT
name: Waku v2 Transport Specification
status: draft
editor: Reeshav Khan <reeshav@status.im>
contributors:
---

`31/WAKU2-TRANSPORT` describes the specification for Protocols supported by Waku v2. While Waku v2 strives to be transport agonistic, however any underlying transport layer MUST be IETF standardized transport protocol, this document underlines the baseline for interoperability between different Waku v2 clients.

# Transport Protocol Specification 

Waku client implementations MUST support the TCP transport, and it MUST be enabled for both dialing and listening even if other transports are available. During negotiation upgrade protocol MAY be chosen over TCP. 
The underlying transports SHOULD support websocket and secure websockets for bidirectional communication streams and browser applications. 


# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
