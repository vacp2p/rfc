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

The DAG

# Binary Consensus Semantics

We motivate and describe the Glacier binary consensus algorithim.

The algorithm is divided into 4 phases:

1. **Querying.** We select *k* nodes from the complete pool of peers
   in the network. This query is weighted, so the probability of
   selecting nodes is proportional to their weight.

$$ P(i) = \frac{w_i}{\sum_{j=0}^{j=N}w_j} $$

The list of nodes is maintained by a separate protocol (the network
layer), and eventual consistency of this knowledge in the network
suffices. Even if there are slight divergences in the network view
from different nodes, the algorithm is resilient to those.

*Adaptive querying*. An additional optimization in the query consists
of adaptively growing the *k* constant in the event of *high
confusion*. We define high confusion as the situation in which neither
opinion is strongly held in a query (*i.e.* a threshold is not reached
for either yes or no). For this, we will use the $\alpha$ threshold
defined below. This adaptive growth of the query size is done as
follows:

- Every time the threshold is not reached, we multiply *k* by a
  constant. In our experiments, we found that a constant of 2 works
  well, but what really matter is that it stays within that order of
  magnitude.

- The growth is capped at 4 times the initial *k* value. Again, this
  is an experimental value, and could potentially be increased. This
  depends mainly on complex factors such as the size of the query
  messages, which could saturate the node bandwidth if the number of
  nodes queried is too high.


1. **Computing the confidence, evidence and accumulated evidence.**
   These three ratios are used later on to compute the transition
   function and the opinion forming. Confidence encapsulates the
   notion of how much we know (as a node) in relation to how much we
   will know in the near future (this being encoded in the parameter
   $l$. Evidence accumulated keeps the ratio of total positive votes
   vs the total votes received (positive and negative), whereas the
   evidence per round stores the ratio of the current round only.

$$
l: \text{look-ahead parameter} \\
\alpha_1 = 0.8, \ \alpha_2 = 0.5 \\
\text{confidence}: c_{accum} = \frac{total\ votes}{total\ votes + l} \\
\text{evidence accumulated}: e_{accum} = \frac{total\ positive\ votes}{total\ votes} \\
\text{evidence per round}: e_{round} = \frac{round\ positive\ votes}{round\ votes}
$$

1. **Transition function.** In order to eliminate the need for a step
   function (a conditional in the code), we introduce a transition
   function from one regime to the other. Our interest in removing the
   step function is twofold:

- Simplify the algorithm. With this change the number of branches is
  reduced, and everything is expressed as a set of equations.

- Second, the transition function makes the regime switch smooth,
  making it harder to potentially exploit the sudden regime change in
  some unforeseen manner. Such a swift change in operation mode could
  potentially result in a more complex behavior than initially
  understood, opening the door to elaborated attacks. The transition
  function proposed is linear with respect to the confidence.
    
    $$
    e = e_{round} (1-c_{accum}) + e_{accum} c_{accum} \\ \alpha = \alpha_1 (1-c_{accum}) + \alpha_2 c_{accum}
    $$
    
    Since the confidence is modeled as a ratio that depends on the
    constant $l$, we can visualize the transition function at
    different values of $l$. Recall that this constant encapsulates
    the idea of “near future” in the frequentist certainty model: the
    higher it is, the more distant in time we consider the next
    valuable input of evidence to happen.
    
    In the following graph four different values of $l$ have been
    plotted. We observe that for a transition function to be useful,
    we need establish two requirements:
    
    - First, the change has to be balanced and smooth, giving an
      opportunity to the first regime to operate and not jump directly
      to the second regime.
    
    - Second, the convergence to 1.0 (fully operating in the second
      regime) should happen within a reasonable time-frame. We’ve set
      this time-frame experimentally at 1000 votes, which is in the
      order of ~100 queries given a k=9.
    
    Note: elaborate on the selection of k, diameter of graph, etc.
    
    Note: Avalanche uses k = 20, as an experimental result from their
    deployment. Due to the fundamental similarities between the
    algorithms, it’s a good start for us.
    
    
    
2. **Opinion and decision.** The next step is a simple one: change our
   opinion if the threshold $\alpha$ is reached. This needs to be done
   separately for the yes/no decision, checking both boundaries. The
   last step is then to *decide* on the current opinion. For that, a
   confidence threshold is employed. This threshold is derived from
   the network size, and is directly related to the number of total
   votes received.
    
    $$
    e > \alpha \implies \text{opinion YES} \\
    e < 1-\alpha \implies \text{opinion NO} \\
    
    if\ \text{confidence} > c_{target} \implies  \text{decide}
    $$
    
    Note: elaborate on $c_{target}$ selection.
    

**Code**

The following Python code outlines the algorithm 

```python
def init(self):
        self.evidence = (0, 0)
        self.confidence = (0, L)

def step(self):
        if self.color == ConsensusAgent.NONE:
            return

        vote_count = self.query_nodes()
        total_votes = sum(vote_count)
        if total_votes == 0:
            return
        self.evidence = (self.evidence[0] + vote_count[0], self.evidence[1] + total_votes)
        self.confidence = (self.confidence[0] + total_votes, self.confidence[1] + total_votes)

        conf = self.confidence[0] / self.confidence[1]
        e1 = vote_count[0] / total_votes
        e2 = self.evidence[0] / self.evidence[1]
        e = e1 * (1-conf) + e2 * conf
        alpha = self.evidence_alpha * (1-conf) + self.evidence_alpha2 * conf

        if e > alpha:
            self.color = ConsensusAgent.YES
        elif e < 1 - alpha:
            self.color = ConsensusAgent.NO
        else:
            self.k = min(int(self.k * 2), self.initial_k * 4)
```


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


## Rendering 

Via use of a local `pandoc` installation, a PDF rendering may be created via:

```shell
pandoc -f commonmark_x README.md -o README.pdf
```

## References

[pandoc]:  https://pandoc.org/
[logos]:   https://logos.co
[blake]:   https://genius.com/William-blake-the-marriage-of-heaven-and-hell-annotated
[vaneigm]: http://library.nothingness.org/articles/SI/en/display/10




