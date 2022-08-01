---
slug: 38
title: 38/LOGOS-CONSENSUS
name: Initial Logos Consensus Description
status: raw
category: informative
tags: logos
editor: Mark Evenson <mark.evenson@status.im>
contributors:
    - Álvaro Castro-Castilla
    - Moh Jalalzai <moh@status.im>

---

# Abstract

We sketch a two-layer model of distributed consensus, whereby a base
layer provides a binary decision mechanism which is used to vote on
the construction of a distributed, directed, acyclic graph
corresponding to an unfolding, shared computation.  We outline a
taxonomy of Byzantine adversaries that seek to thwart this
computation's "correctness".  We then present and  define the Glacier
algorithm which provides a Byzantine and fault tolerant implementation
of the base binary decision mechanism.  

# Consensus Model

# Adversaries

[[ Maybe do this after Consensus ]]

# Glacier Consensus

Given a set of $n$ distributed computational nodes, a gossip-based
protocol is presumed to which which allows members to discover, join,
and leave a possibly maximally connected graph.  Joining this graph
allows each node to view a possibly incomplete node membership list of
all other nodes.  This view may changes as the protocol advances.
This view may optionally have a weighting assigned for each node
consisting of a real number on the interval [0, 1].

0.  A proposal is formulated to which consensus of truth or falsity is
    desired.  Each node that participates starts the protocol with an
    opinion on the proposal, represented in the sequel as **YES**, **NO**, and
    **UNDECIDED**.  
    
The algorithim proceeds in rounds for each node.    

1. Setup

The node initializes the following variables 

    k <- 20             ;; nodes to query 
    total_votes <- 0 
    look_ahead <- 20    ;; look ahead parameter
    $alpha_1$ <- 0.8    ;; first order confidence smoothing parameter
    $alpha_2$ <- 0.4    ;; second order confidence smoothing parameter
    total_votes <- 0
    total_positive <- 0

2.  Query 
    A node selects $k$ nodes randomly from the locally known
    complete set of peers in the network.  This query can be weighted,
    so the probability of selecting nodes is proportional to their
    weight.  $$ P(i) = \frac{w_i}{\sum_{j=0}^{j=N} w_j} $$ where $w_i$
    is the weight of the {i}th peer.

    The node then sends a query to each of this $k$ nodes containing
    the nodes own current opinion on the proposal ("YES", "NO", or
    "UNDECIDED").  Each node replies with their current opinion on the
    proposal.

The node now runs the following two inputs 

    new_votes <- vote replies received in round
    positive_votes <- YES votes received 
    
3. Computation    

The node runs these parameters through the following algorithm:

    total_votes += new_votes
    total_positive += positive_votes
    confidence <- total_votes / (total_votes + look_ahead) 
    evidence_accumulated <- total_positive / total_votes
    evidence_round <- positive_votes / new_votes
    evidence <- evidence_round * ( 1 - confidence ) + evidence_accumulated ( confidence )
    alpha <- alpha_1 * ( 1 - confidence ) + alpha_2 * confidence 
    
4. Opionion

The node updates its local opinion on the consensus proposal:

If $evidence$ is greater $alpha$ than the node adopts the opinion YES
on the proposal.

If $evidence$ is less than $1 - \alpha$ the node adopts the opinion NO
on the proposal.

4.  Decision 

If $confidence$ exceeds a threshold derived from the network size and
directly related to the total votes received, the node marks the
decision as final, and always returns this opinion is response to
further queries from other nodes on the network.
    
Otherwise, a new query is initiated by going to step 2.


# Analysis





# Abstract

We sketch the needs of Logos for the ability to provide a consensus
mechanism for virtualized speech-act communities upon which consensual
meaning can be arbitrated.  We sketch a simplistic architecture by
which a base binary consensus model can be used to vote on the
evolution of a distributed computation expressed as a directed acyclic
graph (DAG).  We outline the support for our vision of how such DAGs
can express arbitrary consensual execution models including unspent
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
of emerging a-territorial startup societies, increasingly theorized
under the notion "Network States".

One of the most fundamental public goods would be the ability for
participants to achieve consensus within the affinity groups in network
states.

## Use case 
    
Logos conensus infrastructure supports the bootstrapping and
maintaining **Common Interest Communities (CIC)**, which provides a
concise anticipatio of the needs for the bureaucratic infrastructure
for the construction and maintenance of network States.

Common Interest Communities have the following key characteristics:
    
- A "code is law" approach towards rules requiring a State Machine
  Replication service (i.e. "smart contracts".
- We expect the range of network sizes to be wide. We can expect
  some CICs to be in the scale ~10 nodes, while others could
  (potentially) scale up to thousands.
- CICs will be operating independently. But will interact with
  each other if required.
- The logos network will act as a secure anchor for CIC
  networks. To minimize CIC interactions with the Logos network,
  CICs can store their checkpoints regularly in the Logos
  network. Moreover, some randomly may be selected Logos nodes can
  store the state of the CIC network without actually taking part
  in the consensus of the CIC. During an event of a dispute, the
  checkpoints and/or the Logos nodes can be used to perform
  dispute resolution. Furthermore, the proofs in checkpoints or
  Logos nodes with the CIC state can be used to punish (by
  slashing stake) the malicious actors and/or recover the lost
  state.
- Some CICs will exist as part of the Status Network, while others
  could exist completely independently as a separate
  infrastructure. But in either case security of these networks
  can be strengthened(guaranteed?) through the Logos network.
- We also need to prepare a business plan including an incentive
  plan which will be part of tokenomics. What will be the
  incentives of Logos nodes to provide security for CIC networks?
  We need to define this clearly. Whatever the incentive
  mechanism, it will be employed over the Logos network.


## Revised Requirements 

#. Safety
    #. If a transaction/proposal is committed, by an honest node, it
       must never be revoked. In more formal terms, if a proposal $p$ is
       committed by a node $u$ in the network in the sequence $s$, then no
       other honest node $v$ will commit another proposal $v’$ in the same
       sequence $s$.
    #. In some cases, a probabilistic safety/liveness is acceptable
       given that it improves other properties i.e, scalability.
    
#. Scalable:
    
    #. The protocol should minimize--or preferrably avoid--a
       bottleneck around the proposal of new transactions.
    #. Low message/bit complexity
    #. Low cost of cryptographic operations
    #. Should be able to easily support a network of thousands of nodes
    #. Have a low commit latency
    #. Should support SMR (State-Machine-Replication)
    #. It should be able to operate in an asynchronous or a partially
       synchronous network. The protocol should not wait for a maximum
       delay time period to improve/guarantee liveness.
#. Liveness
    #. The protocol should make progress and must not stall indefinitely.
#. Security
    #. The protocol should be resilient against Byzantine attacks
    #. The protocol should be resilient against Performance
       attacks. Performance attacks are classes of attacks that
       greatly reduce the protocol throughput and/or significantly
       increase protocol latency. Though these attacks do not break
       the safety or liveness guarantees.
#. Fairness

Initially, we want to assume rewarding the successful block
proposers (producers) whose blocks have been added to the
blockchain. Other cases that can be considered to be rewarded
include participation in voting, reporting malicious behavior with
valid proof etc., can be added later (as these cases are more
complex).

We assume that each node receives a fee for proposing a block that
is eventually added to the blockchain. Informally, with fairness,
our goal is that every node gets an equal number of opportunities
to append blocks to the blockchain. Hence, every node will achieve
equal rewards over the long term. But since we assume the
incentive/reward mechanism will be implemented over the top of the
consensus protocol, we generalize the definition of fairness to
allow **stake-based fairness**. Stake-based fairness ensures that
every node gets a number of opportunities relative to their stake
weight to append a block to the blockchain.

Let $B_i(m)$ be the number of blocks appended in the blockchain by
the node $i$ in $m$ rounds. $B_i(m)$ can be a random variable due
to the randomness of the proposer/block selection
mechanism. $B(m)$ is the total number of blocks appended to the
blockchain in $m$ rounds. Then the fraction of blocks added into
the blockchain by node $i$ in $m$ rounds is given by:

$$
{\lim_{m \rightarrow \infty}{\frac{B_i(m)}{B(m)}}}.
$$

The stake weight of node $i$ is the ratio of its stake $S_i$ to
the total stake of the network $S = \sum_{i=1}^{i=n}S_i$ in $m$
rounds:

$$
  {\lim_{m \rightarrow \infty}{\frac{S_i(m)}{S(m)}}}. 
$$

A protocol guarantees stake-based fairness if 

$\forall i \in N_h$ we have

$$
{\lim_{m \rightarrow \infty}{\frac{B_i(m)}{B(m)}}} \ge {\lim_{m \rightarrow \infty}{\frac{S_i(m)}{S(m)}}}.
$$

where $N_h$ is the set of all honest nodes in the network.


# Glacier 

We motivate and describe the Glacier binary consensus algorithim.


Glacier is a leaderless probablistic binary consensus algorithm that
has fast finality that provides high tolerance for network and
Byzantine fault tolerance. 


## Phase One: Querying

A node selects $k$ nodes randomly from the complete pool of peers in the
network. This query is can optionally be weighted, so the probability
of selecting nodes is proportional to their weight.  [[Explain
weighting needs]].

$$ 
P(i) = \frac{w_i}{\sum_{j=0}^{j=N} w_j} 
$$

The list of nodes is maintained by a separate protocol (the network
layer), and eventual consistency of this knowledge in the network
suffices. Even if there are slight divergences in the network view
from different nodes, the algorithm is resilient to those.

*Adaptive querying*. An additional optimization in the query
consists of adaptively growing the *k* constant in the event of
*high confusion*. We define high confusion as the situation in
which neither opinion is strongly held in a query (*i.e.* a
threshold is not reached for either yes or no). For this, we will
use the $\alpha$ threshold defined below. This adaptive growth of
the query size is done as follows:

Every time the threshold is not reached, we multiply $k$ by a
constant. In our experiments, we found that a constant of 2 works
well, but what really matter is that it stays within that order of
magnitude.

The growth is capped at 4 times the initial *k* value. Again, this
is an experimental value, and could potentially be increased. This
depends mainly on complex factors such as the size of the query
messages, which could saturate the node bandwidth if the number of
nodes queried is too high.


## Phase Two: Compute the confidence, evidence and accumulated evidence

When the query returns, three ratios are used later on to compute the
transition function and the opinion forming. Confidence encapsulates
the notion of how much we know (as a node) in relation to how much we
will know in the near future (this being encoded in the look-ahead
parameter $l$.) Evidence accumulated keeps the ratio of total positive
votes vs the total votes received (positive and negative), whereas the
evidence per round stores the ratio of the current round only.

$$
l = 20 \text{look-ahead parameter} \\
\alpha_1 = 0.8 \text{first evidence parameter} \\
\alpha_2 = 0.5 \text{second evidence parameter} \\
\text{confidence}: c_{accum} = \frac{total\ votes}{total\ votes + l} \\
\text{evidence accumulated}: e_{accum} = \frac{total\ positive\ votes}{total\ votes} \\
   \text{evidence per round}: e_{round} = \frac{round\ positive\ votes}{round\ votes}
$$


## Phase Three: Transition function 

In order to eliminate the need for a step function (a conditional in
the code), we introduce a transition function from one regime to the
other. Our interest in removing the step function is twofold:

   #. Simplify the algorithm. With this change the number of branches is
   reduced, and everything is expressed as a set of equations.

   #. The transition function makes the regime switch smooth,
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

[[ Note: elaborate on the selection of k, diameter of graph, etc. ]]

[[ Note: Avalanche uses k = 20, as an experimental result from their
deployment. Due to the fundamental similarities between the
algorithms, it’s a good start for us. ]]
    
    
  
## Phase Four: Opinion and Decision  

The next step is a simple one: change our opinion if the threshold
$\alpha$ is reached. This needs to be done separately for the yes/no
decision, checking both boundaries. The last step is then to *decide*
on the current opinion. For that, a confidence threshold is
employed. This threshold is derived from the network size, and is
directly related to the number of total votes received.
    
$$
e > \alpha \implies \text{opinion YES} \\
e < 1-\alpha \implies \text{opinion NO} \\
if\ \text{confidence} > c_{target} \implies  \text{decide}
$$

Note: elaborate on $c_{target}$ selection.


# Interoperability 
## Wire Formats Specification / Syntax

At this point, we do not commit to a syntax but rather to a semantics
of message exchange between nodes.


# Colophon
## In Media Res

Although we have provided the interoperable semantics of an
implementation,  the actual agorithim utilized by Logos may be
different as we haven't fully worked out what is possible given a set
of requirements.  As such this RFC is expected to be superseded by a more rigorous
structure.  It serves to document our current progress.


## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Format

This document is currently utilizing Pandoc 2.18's notion of Markdown.

## References

#. [pandoc](https://pandoc.org/)
#. [logos](https://logos.co)
#. [blake](https://genius.com/William-blake-the-marriage-of-heaven-and-hell-annotated)
#. [vaneigm](http://library.nothingness.org/articles/SI/en/display/10)
#. [The Network State](https://thenetworkstate.com)



