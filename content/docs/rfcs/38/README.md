---
slug: 38
title: 38/LOGOS-CONSENSUS
name: Logos Glacier Consensus Protocol
status: raw
category: informative
tags: logos/consensus,rust-implementation
editor: Mark Evenson <mark.evenson@status.im>
contributors:
    - Álvaro Castro-Castilla 

---

# Abstract

We propose to replace Nakomoto consensus mechanisms with ones which
afford secure leaderless, decentralized execution.  We sketch a
two-layer model for the practical execution of such a distributed
consensus, in which an underlying binary decision mechanism is
utilized to used to vote on the construction of a distributed,
directed, acyclic graph.  We present the Glacier algorithm which
provides a Byzantine fault tolerant implementation of the base binary
decision mechanism.  We outline a taxonomy of Byzantine adversaries
that seek to thwart this computation's "correctness".

# Consensus Model

Given an underlying binary consensus mechanism, one may quickly vote
on the distributed, directed acyclic ledger graph of transactions.  If
the underlying binary consensus mechanism is Byzantine fault tolerant,
then one essentially gets the computation of the trust of the graph of
transactions "for free".  The finalization of shared confidence in the
values in a given sequence along this graph is isomorphic to the trace
of a shared, trusted state machine evolution.  Such a state machine
may perform an arbitrary computation from the class of "smart
contract" artifacts by implementing a
suitable model of transaction.  

# Glacier 

The Glacier consensus algorithm computes a yes/no decision via a set
of distributed computational nodes.  Glacier is a leaderless
probabilistic binary consensus algorithm with fast finality that
provides good reliability for network and Byzantine fault tolerance.

## Algorithm 

Given a set of $n$ distributed computational nodes, a gossip-based
protocol is presumed to exist which which allows members to discover,
join, and leave a possibly maximally connected graph.  Joining this
graph allows each node to view a possibly incomplete node membership
list of all other nodes.  This view may change as the protocol
advances, as nodes join and leave.  


### Pre-requisites

A proposal is formulated to which consensus of truth or falsity is
desired.  Each node that participates starts the protocol with an
opinion on the proposal, represented in the sequel as **YES**, **NO**,
and **UNDECIDED**.
    
The algorithm proceeds in rounds for each node.

### Setup

The node initializes the following constants variables 

    ;;; The following values are constants chosen with justification from experiments
    ;;; performed with the adversarial models
    
    ;; constant look ahead parameter, 
    look_ahead <-- 20
    
    ;; first order confidence smoothing parameter
    $alpha_1$ <-- 0.8
    
    ;; second order confidence smoothing parameter
    $alpha_2$ <-- 0.4
    
    ;;; The following variables are initialized
    
    ;; number of nodes to query 
    k <-- 20      
    
    ;; total number of votes
    total_votes <-- 0    

    ;; total number of positive nodes
    total_positive <-- 0
    
    ;; threshold multiple 
    k_multiplier <-- 2

    ;; maximal threshold multiplier
    max_k_multiplier <-- 2

###  Query 

A node selects $k$ nodes randomly from the locally known complete set
of peers in the network.

Each node replies with their current opinion on the proposal.

When the query finishes, the node now initializes the following two
values:

    new_votes <-- vote replies received in this round
    positive_votes <-- YES votes received 
    
### Computation    

The node runs the `new_votes` and `positive_votes` parameters received
in the query round through the following algorithm:

    total_votes += new_votes
    total_positive += positive_votes
    confidence <- total_votes / (total_votes + look_ahead) 
    evidence_accumulated <- total_positive / total_votes
    evidence_round <- positive_votes / new_votes
    evidence <- evidence_round * ( 1 - confidence ) + evidence_accumulated * ( confidence )
    alpha <- alpha_1 * ( 1 - confidence ) + alpha_2 * confidence 
    
### Opinion 

The node updates its local opinion on the consensus proposal:

    IF 
       $evidence$ > $alpha$ 
    THEN 
       The node adopts the opinion YES on the proposal

    IF 
        $evidence$ is less than $1 - \alpha$ 
    THEN 
       The node adopts the opinion NO on the proposal 


###  Decision 

If `confidence` exceeds a threshold derived from the network size and
directly related to the total votes received, the node marks the
decision as final, and always returns this opinion is response to
further queries from other nodes on the network.
    
Otherwise, a new query is initiated by going back to the `QUERY`
step. 

## Questions 

In the query step, the node is envisioned as packing information into
the query to cut down on the communication overhead  a query to each of this $k$
nodes containing the node's own current opinion on the proposal
("YES", "NO", or "UNDECIDED").

## Optional Features

### Weighted Node values

The view of network peers participants may optionally have a weighting
assigned for each node consisting of a real number on the interval [0,
1].  This weight is used in each query round when selecting the $k$
peers so the probability of selecting nodes is proportional to their
weight.  

$$
P(i) = \frac{w_i}{\sum_{j=0}^{j=N} w_j}
$$ 

where $w_i$ is the weight of the {i}th peer.

This weighted value would be used to reflect decisions mediating

- Staking
- Heuristic reputation
- Manual reputation

### Problems 

If the view of other nodes is incomplete, then the sum of the optional
weighting won't be a probability distribution normalized to 1.

The current algorithm doesn't describe how the initial opinions are formed.


# Adversarial  Models 


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

No current wire protocol.  Nodes are advised to use WAKU messages to
include their own metadata in serializations as needed.  

## TODO Semantics

    { -1, +1, 0 }
    YES, NO, UNDECIDED 

# Implementation status

logos.co is prepared to share a Rust implementation which contains an
efficiently multi-threaded implementation utilitized by both a
simulator and a multi-node constructions.  

# Sovereignty Considerations

## Privacy

## Security

Adversarial models exist for which the values for current parameters
of Glacier have been tuned.  Exposition of the justification of this
tuning need to be completed.

### Local Strategies

#### Random Adversary

A random adversary optionally chooses to respond to all queries with a
random decision.

#### Naive Opposite Adversary

An naive oppositional adversary responds with the opposite vote on an
opinion.

### Omniscient Coordinated Behavior Adversaries

An omniscient adversary controls $f$ of $N$ nodes and may inspect,
delay, and drop arbitrary messages in the gossip layer, and utilizes
this to corrupt consensus.


# Future Directions

Although we have proposed a normative description of the
implementation of the underlying binary consensus algorithm (Glacier),
we believe we have analyzed its adversarial performance in a manner
that is ammendable to replacement by another member of the [snow*][]
family.  

We have presumed the existence of a general family of algorithms that
can be counted on to vote on nodes in the DAG in a fair manner.
Avalanche provides an example of the construction of votes on UTXO
transactions.  One can express all state machine, i.e. account-based
models as checkpoints anchored in UTXO trust, so we believe that this
presupposition has some justification.

# Informative References

#. [On BFT Consensus Evolution: From Monolithic to DAG](https://dahliamalkhi.github.io/posts/2022/06/dag-bft/)
#. [snow-ipfs](https://ipfs.io/ipfs/QmUy4jh5mGNZvLkjies1RWM4YuvJh5o2FYopNPVYwrRVGV) 
#. [snow*](https://https://doi.org/10.48550/arXiv.1906.08936) Rocket, Team, Maofan Yin, Kevin Sekniqi, Robbert van Renesse, and Emin Gün Sirer. “Scalable and Probabilistic Leaderless BFT Consensus through Metastability.” arXiv, August 24, 2020.

# Dumpster
 
 , a .  This execution mechanism
 consists of transitions in a state machine representation, each one a
 potential transaction.  These transactions are gossiped to active
 (i.e. online) participants.  The participants vote on whether a given
 transaction should be counted as valid which reaches an eventual
 consistent state in which the transactions are said to have been
 finalized.




# Colophon
## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Format

This document is currently utilizing Pandoc 2.18's notion of Markdown.
