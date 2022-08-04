---
slug: 41
title: 41/LOGOS-REPUTATION 
name: Logos Node Reputation 
status: raw
category: Informational
tags: logos/reputation
editor: Mark Evenson <mark.evenson@status.im>
contributors:
    - Alvaro
---

# Abstract

We present Ininkgut, a deceptively simple algorithm for computing the
reputation of other nodes in their performance of a shared alogrithm.


# Background

The following document describes the Ikingut reputation algorithm,
designed to be used to augment the Logos consensus layer for various
purposes.

The use of objectives of defining a system for ongoing maintainance of
node reputation is to
 
   - Strengthen the Byzantine Fault Tolerance properties of the
     consensus mechanism, possibly making the algorithm more resilient
     to a higher number of byzantine nodes, or more aggressive
     strategies.

   - Providing a mechanism for bootstrapping networks, with a low
     number of nodes.

# Theory / Semantics

## Ikingut Algorithm

### Concept

The Ikingut algorithm for reputation is a local heuristic computed
from a single source of information: *how many correct responses did a
node give to queries*. The challenge in this metric is the definition
of *correct response*. We define a correct response as the decision
the node achieves on the bit value (by means of Glacier).

This in essence summarizes the algorithm. There are implementation
details and parameters that require some elaboration, as we will see
below in further detail. Later on we will provide an experimental
background to the specific design decisions.

Other relevant characteristics of the algorithm are:

- It’s simple. The complete implementation can be described in two
  functions.
- Lightweight. It doesn’t perform long computations nor use a lot of
  memory.
- Pluggable into Glacier and Snowball. It’s designed to fit well with
  swarm-like consensus algorithms. Moreover, it augments the consensus
  algorithm without altering it.
- Adaptive. It processes network and behavior changes fast.
- Dynamic. It introduces new nodes consistently and fairly.
- It is robust against strategic adversaries. We will see these in more detail below.
- No transitive trust. This avoids collusion, greatly simplifies. The
  tradeoff is more time to build the reputation list.
- Reasonable time to bootstrap reputation. With 10.000 nodes and k ~=
  9 is in the order of 8-10 min.

### Algorithm

The Ikingut algorithm follows two parts, which run at every iteration
of the consensus algorithm.

1. On every agent vote:
    1. Accumulate +1 for positive
    2. Accumulate -1 for negative
    3. Accumulate +1 in separate counter for *no response*
2. On consensus decision finality. On every agent:
    1. If the decision is NO, we need to invert the accumulated
       reputation.
    2. If the resulting reputation is positive, add it to the trust
       score. A constant can be used for the reputation increasing
       operation.
    3. If the resulting reputation is negative, add it to the trust
       score, using the following equation:
    
    $$
    t_i := min(at_i, t_i + br_i)\\
    \text{where}\ 0 < a < 1 \ \text{and} \ b > 1
    $$
    
    Where $a$ and $b$ are constants used to manipulate how much
    momentum is applied.

**Pseudocode:**

```python
def ikingut_agent_trust(self, a, vote):
		if vote == ConsensusAgent.YES:
		    self.agent_votes_accum[a.unique_id] += 1
    elif vote == ConsensusAgent.NO:
        self.agent_votes_accum[a.unique_id] -= 1
    else:
        self.agent_votes_no_resp[a.unique_id] += 1

def ikingut_model_trust(self):
    # Compute new reputations
    for (agent_id, rep_delta) in enumerate(self.agent_votes_accum):
		    # If our decision is NO, we need to invert the accumulator
		    if self.color == ConsensusAgent.NO:
		        rep_delta = -rep_delta
			  # No response is always negative score, so we substract it after
		    # setting the right sign on the accumulator
		    if rep_delta > 0:
				    self.trust_scores[agent_id] += rep_delta
		    elif rep_delta < 0:
				    # Only apply momentum for punishment.
				    self.trust_scores[agent_id] = min(self.trust_scores[agent_id] // 2,
				                                      self.trust_scores[agent_id] + rep_delta)
```

**Min-multiplicative reputation punishment**

The idea behind using a min function is that the punishment is much
harsher if the current reputation of the node is high. The min
function will select the highest reduction of reputation resulting
from these two operations:

- Multiplicative. This provides resiliency against patient attackers
  that have built reputation long-term. Some back-of-the-envelope
  calculations:
    - In one year at 1000 tx/s (network size: 10000 nodes, k=9), the
      expectation is around 0.9 req/s to each node. In 31536000
      secs/year, the max positive score: 28,382,400 (all
      reqs/node/year).
    - With a 2x punishment: 25 reqs to lose of all reputation
    - With a 3x punishment: 16 reqs to lose of all reputation
    
    Note that these requests don’t need to be consecutive, just close
    enough in time for the multiplicative punishment to take effect.
    
- Linear. This ensures the following properties:
    - Allow for new nodes to have a neutral score of zero.
    - Bring the opportunity of building reputation faster.
    - Ensure accumulation of negative scores for misbehaving nodes.

# Security/Privacy Considerations


# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

## informative

#. 38/LOGOS-CONSENSUS
#. Rocket, Team, Maofan Yin, Kevin Sekniqi, Robbert van Renesse, and Emin Gün Sirer. “Scalable and Probabilistic Leaderless BFT Consensus through Metastability.” arXiv, August 24, 2020. https://doi.org/10.48550/arXiv.1906.08936.
