---
slug: 38
title: 38/LOGOS-CONSENSUS
name: Initial Logos Consensus Description
status: raw
category: informative
tags: logos
editor: Mark Evenson <mark.evenson@status.im>
contributors:
- Mark Evenson <mark.evenson@status.im>
---

# Abstract

We sketch the needs of Logos for the ability to provide a consensus
mechanism for virtualized speech-act communities upon which consensual
meaning can be arbitrated.  We sketch a simplistic architecture by
which a base binary consensus model can be used to vote on the
evolution of a distributed computation expressed as a directed acyclic
graph (DAG).  We outline the support for vision of how such DAGs can
express arbitrary consensual execution models including unspent
transaction output (UTXO) transactions, NFT ownership transfer, up to
resource constrained Turing machine execution environments such as the
Ethereum Virtual Machine (EVM).  We then provide the interoperability
semantics for the Glacier binary consensus algorithim, which is an
improvement of the Snow* family under certain adversarial conditions.
We then outline the BFT adversarial models we have considered to
motivate the construction of Glacier.

# Background 

## Logos 

The Logos organization researches, develops, and promotes the
technological public good infrastructures necessary for the evolution
of emerging a-territorial startup societies, incompletely theorized as
"Network states".  

One of the most fundamental public goods would be the ability for
participants to achieve consensus at a base level, an

the affinity groups in network states.  

Such a community has the following characteristics:

We expect the number of participants in a given community to vary
widely from the smallest of around 10 nodes to contemporary "blockchain
limits" of around 10000 nodes.

## Requirements 

# Consensus

We seek a layered approach to creating mechanisms for consensus: a
low-level binary consensus protocol upon which we can create varying
execution layers.

## Probabilistic Binary Consensus

The Snow* family of agorithims quickly finalize a binary consensus in
a leaderless manner with O(n) message transfer


## Execution Layer

The Dag 

# Binary Consensus Semantics

We motivate and describe the Glacier binary consensus algorithim.

# Interoperability 
## Wire Formats Specification / Syntax
At this point, we do not commit to a syntax but rather to a semantics
of message exchange between nodes.


# Colophon
## In Media Res

Although we have provided the interoperable semantics of an
implementation,  the actual agorithim utilized by Logos may be
different as we haven't fully worked out what is possible given a set
of requirements.  As such RFC is expected to be superseded by a more rigorous
structure.


## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

[logos]:   https://
[blake]:   https://genius.com/William-blake-the-marriage-of-heaven-and-hell-annotated
[vaneigm]: http://library.nothingness.org/articles/SI/en/display/10
