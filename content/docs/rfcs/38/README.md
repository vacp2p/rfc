---
slug: 38
title: 38/LOGOS-CONSENSUS-GLACIER
name: Logos Glacier Consensus Protocol
status: raw
category: Standards Track
tags: logos/consensus
editor: Mark Evenson <mark.evenson@status.im>
created: 01-JUL-2022
revised: <2022-08-26 Fri 11:17>
uri: <https://rdf.logos.co/protocol/glacier/1/0/0#<2022-08-19%20Fri$2009:40Z>
contributors:
    - Álvaro Castro-Castilla 
---

# Abstract

This document specifies Glacier: a Byzantine tolerant binary decision
agreement algorithm that utilizes bounded memory for its execution.
Glacier is a novel member of the Snow family providing a probabilistic
leaderless BFT consensus algorithm that achieves metastablity via
network sub-sampling.  We present an application context of the use of
Glacier in an efficient, leaderless, probabilistic permission-less
consensus mechanism.  We outline a simple taxonomy of Byzantine
adversaries, leaving explicit explorations of to subsequent
publication.

# One Possible Logos Manifesto

Logos seeks to develop components for a composable infrastructure for
any Network State.  One of these components is the ability to come to
consensus on a given proposal.  We seek to replace resource intensive
Nakomoto consensus mechanisms with those which belong to the class of
more efficient ones rooted in leaderless decentralization.  Being less
constrained by the demands of Proof-of-Work (PoW) in which all nodes
must reproduce all computations, such consensus mechanisms can be
considerably more adaptive to a given network's requirements.

By documenting Glacier--a component for shared consensus composable
with models of the finalization of shared data structure--Logos starts
the construction of a "value-free" Network State infrastructure.
Subsequently, we will connect the implementation of Glacier to an
appropriately secure execution of a explicitly desired security model.
As a member of the Snow family, Glacier provides a tractably
computable probabilistic measure of safety of the finalization of
states of shared data structures.

We start with simple two-level composition model for the practical
execution of such a fairly distributed consensus mechanism, in which
an underlying leaderless binary decision mechanism votes on the
finalization of nodes distributed in a directed, acyclic graph.  In
the sequel, we present a pseudo-code specification the Glacier
algorithm which provides a Byzantine fault tolerant implementation of
the base binary decision mechanism.  We start to outline a taxonomy of
Byzantine adversaries that seek to thwart this computation's correct
honesty, to which subsequent results need to be annotated.

The algorithm we specify here, Glacier, may be extended to a DAG
structure in order to achieve a leaderless Byzantine fault tolerance.

## Leaderless Consensus Model Application to Shared State 

The consensus model specified in this document is based on the
observation that given an underlying underlying binary consensus
mechanism, one may quickly vote on the distributed, directed acyclic
ledger graph of transactions.  If the underlying binary consensus
mechanism is Byzantine fault tolerant, then one essentially gets the
computation of the trust of the graph of transactions "for free".  The
finalization of shared confidence in the values in a given sequence
along this graph is isomorphic to the trace of a shared, trusted state
machine evolution.  Such a state machine may perform an arbitrary
computation from the class of "smart contract" artifacts by
implementing a given model of transaction.

This execution mechanism consists of transitions in a state machine
representation, each one a potential transaction.  These transactions
are gossiped to active (i.e. online) participants.  The participants
vote on whether a given transaction should be counted as valid which
reaches an eventual consistent state in which the transactions are
said to have been finalized.

Like the Prisoner Dilemma, the execution of Glacier on a single
proposal doesn't have very complicated game theoretics.  Simply
conceived, multiple runs of Glacier could track a linear chain of
consensus like any other contemporary implementation. But in practice,
leaderless consensus may be used contribute finalization of multiple
proposals arranged in a directed acyclic graph.  Nodes would naturally
choose to compute the consensus of directed graph of the proposals
that they are interested in.  This autosharding property supersedes
any possible Nakomoto consensus mechanism, as any proof of work
mechanism would by definition need to increase its the work necessary
for its continued constant security of all chains of interest.

# Glacier Algorithm Specification

The Glacier consensus algorithm computes a boolean decision on a
proposition via a set of distributed computational nodes.  Glacier is
a leaderless probabilistic binary consensus algorithm with fast
finality that provides good reliability for network and Byzantine
fault tolerance.

## Algorithm 

The algorithm begins with considering the truth status of a given
proposition.

A proposal is formulated to which consensus of truth or falsity is
desired.  Each node that participates starts the protocol with an
opinion on the proposal, represented in the sequel as `no`, `none`,
and `yes`.

A new proposition is discovered either by local creation or in
response to a query, a node checks its local opinion.  If the node can
compute a justification of the proposal, it sets its opinion to one of
`yes` or `no`.  If it cannot form an opinion, it leaves its opinion as
`none`.

The node then participates in a number of query rounds in which it
solicits other node's opinion in query rounds.  Given a set of `n`
leaderless computational nodes, a gossip-based protocol is presumed to
exist which allows members to discover, join, and leave a weakly
transitory maximally connected graph.  Joining this graph allows each
node to view a possibly incomplete node membership list of all other
nodes.  This view may change as the protocol advances, as nodes join
and leave.  Under generalized Internet conditions, the membership of
the graph would experience a churn rate varying across different
time-scales, as the protocol rounds progress.  As such, a given node
may not have a view on the complete members participating in the
consensus on a proposal in a given round.


### Proposal Identification

The node has a semantics and serialization of the proposal, of which
it sets an initial opinion:

     opinion
        <-- initial opinion on truth of the proposal
            as one of: {NO, NONE, YES}
    
The proposal proceeds in asynchronous rounds, in which each node
queries `k` randomly sampled nodes for their opinions until a decision
about local finality is achieved.  The liveness of the algorithm is
severely constrained in the absence of timeouts for a round to
proceed.  When a given node has finalized its decision on the
proposal, it enters a quiescent state in which it optionally discards
all information gathered during the query process retaining only the
final opinion on the truth of the proposal.

### Setup Parameters

The node initializes the following integer ratios as constants:
```
;;; The following values are constants chosen with justification from experiments
;;; performed with the adversarial models

;; FIXME: find and justify an empirical confidence threshold 
confidence_threshold
  <-- 1  ;; BOGUS:  this should be a function of the network size
         ;; and the current confidence in the observer majority as 
         ;; a function of the current round.  With a value of `1`
         ;; the algorithm will never terminate

;;; constant look ahead for number of rounds we expect to finalize a
;;; decision.  Could be set dependent on number of nodes 
;;; visible in the current gossip graph.
look_ahead 
  <-- 19
;;; These need to be justified empirically via modeling and simulation.
certainty ;; the confidence weighting parameter (aka alpha_1)
  <-- 4 / 5  
doubt ;; the lack of confidence weighting parameter (aka alpha_2)
  <-- 2 / 5 

k_multiplier     ;; neighbor threshold multiplier
  <-- 2

;;; maximal threshold multiplier, i.e. we will never exceed 
;;; questioning k_initial * k_multiplier ^ max_k_multiplier_power peers
max_k_multiplier_power 
  <-- 4
    
;;; Initial numbers of nodes queried in a round
k_initial 
  <-- 7

;;; maximum query rounds before termination
max_rounds
   <-- 997 ;; TODO justify
```
      
The following variables are needed to keep the state of Glacier:

```
;; current number of nodes to attempt to query in a round
k 
  <-- k_original
  
;; total number of votes examined over all rounds
total_votes 
   <-- 0 
;; total number of YES (i.e. positive) votes for the truth of the proposal
total_positive 
   <-- 0
;; the current query round, an integer starting from zero
round
  <-- 0
```


###  Query 

A node selects `k` nodes randomly from the current locally known
complete set of peers in the network.

A query is sent to each neighbor with the node's current `opinion` of
the proposal.

Each node replies with their current opinion on the proposal.

See [the wire protocol Interoperability section](#wire-protocol) for
details on the semantics and syntax of the "on the wire"
representation of this query.

When the query finishes, the node now initializes the following two
values:

    new_votes 
      <-- |total vote replies received in this round to the current query|
    positive_votes 
      <-- |YES votes received from the query| 
    
### Computation

The node runs the `new_votes` and `positive_votes` parameters received
in the query round through the following algorithm:

    total_votes 
      +== new_votes
    total_positive 
      +== positive_votes
    confidence 
      <-- total_votes / (total_votes + look_ahead) 
    total_evidence 
      <-- total_positive / total_votes
    new_evidence 
      <-- positive_votes / new_votes
    evidence 
      <-- new_evidence * ( 1 - confidence ) + total_evidence * confidence 
    alpha 
      <-- doubt * ( 1 - confidence ) + certainty * confidence 
    
### Opinion 

The node updates its local opinion on the consensus proposal by
examining the relationship between the evidence accumulated for a
proposal with the confidence encoded in the `alpha` parameter:

    IF
      evidence > alpha
    THEN 
      opinion <-- YES
    ELSE IF       
      evidence < 1 - alpha
    THEN 
      opinion <-- NO
       
If the opinion of the node is `NONE` after evaluating the relation
between `evidence` and `alpha`, adjust the number of uniform randomly
queried nodes by multiplying the neighbors `k` by the `k_multiplier`
up to the limit of `k_max_multiplier_power` query size increases.
    
    ;; possibly increase number nodes to uniformly randomly query in next round
    WHEN
         opinion is NONE
      AND 
         k < k_original * k_multiplier ^ max_k_multiplier_power
    THEN 
       k <-- k * k_multiplier

###  Decision 

After the OPINION phase is executed, the current value of `confidence`
is considered: if `confidence` exceeds a threshold derived from the
network size and directly related to the total votes received, an
honest node marks the decision as final, and always returns this
opinion is response to further queries from other nodes on the
network.
 
    IF 
      confidence > confidence_threshold
    OR 
      round > max_rounds
    THEN
      finalized <-- T
      QUERY LOOP TERMINATES
    ELSE 
      round +== 1
      QUERY LOOP CONTINUES

Thus, after the decision phase, either a decision has been finalized
and the local node becomes quiescent never initiating a new query, or
it initiates a [new query](#query).

### Termination

A local round of Glacier terminates in one of the following following
execution model considerations:


1.  No queries are received for any newly initiated round for temporal
    periods observed via a locally computed passage of time.  See [the
    following point on local time](#clock).

2.  The confidence on the proposal exceeds our threshold for
    finalization.
    
3.  The number of `rounds` executed would be greater than
    `max_rounds`. 
    
#### Quiescence

After a local node has finalized an opinion, it enters a quiescent
state whereby it never solicits new votes on the proposal.  The local
node MUST reply with the currently finalized majority opinion.

#### Clock

The algorithm only requires that nodes have computed the drift of
observation of the passage of local time, not that that they have
coordinated an absolute time with their peers.  For an implementation
of a phase locked-loop feedback to measure local clock drift see
[NTP](https://www.rfc-editor.org/rfc/rfc5905.html).

## Further points
    
#### Modeling 

[[ TODO map to sync/async arguments ]]

### Node receives information during round

In the query step, the node is envisioned as packing information into
the query to cut down on the communication overhead a query to each of
this `k` nodes containing the node's own current opinion on the
proposal ("YES", "NO", or "NONE").  The algorithm does not currently
specify how a given node utilizes this incoming information.  A
possible use may be to count unsolicited votes towards a currently
active round, and discard the information if the node is in a
quiescent state.

### Weighted Node values

The view of network peers participants may optionally have a weighting
assigned for each node consisting of a real on the interval inclusive
of `0` but exclusive of `1`.  This weight is used in each query round
when selecting the `k` peers so the probability of selecting nodes is
proportional to their weight.

Node weighting probability
$$
P(i) = \frac{w_i}{\sum_{j=0}^{j=N} w_j}
$$ 

where `w_i` is the weight of the `i`th peer.

A given nodes weight can be calculated in a multitude of ways and is
left as an implementation detail or further modification. Such a
weight could be derived from a multitude of values including:

- Staking
- Heuristic reputation
- Manual reputation

#### Problems with Weighting Node Value of Opinions

If the view of other nodes is incomplete, then the sum of the optional
weighting won't be a probability distribution normalized to 1.

The current algorithm doesn't describe how the initial opinions are formed.

# Implementation status

logos.co prepares to share an implementations in Rust which contains
an efficiently multi-threaded implementation utilized by both a local
simulator and actual network construction.  Expressions of Glacier in
Python and Common Lisp are also in limited public review.


# Wire Protocol 

For interoperability we present a wire protocol semantics by requiring
the validity of the following statements expressed in Notation3 (aka
`n3`) about any query performed by a query node:


```n3
@prefix rdf:         <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs:        <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd:         <http://www.w3.org/2001/XMLSchema#> .

@prefix glacier      <https://rdf.logos.co/protocol/glacier#> .

glacier:query
  :holds (
    :_0 [ rdfs:label "round";
          rdfs:comment """
The current round of this query 

This starts with zero.
""" ;
          a xsd:postitiveInteger; ],
    :_1 [ rdfs:label "uri";
          rdfs:comment """
A unique URI for the proposal.

It should be possible to examine the proposal by resolving this resource.
""" ;
          a xsd:anyURI ],
    :_2 [ rdfs:label "opinion";
          rdfs:comment """
The opinion on the proposal

One of the strings "YES" "NO" or "NONE".
""" ;
          # TODO constrain as an enumeration on three values efficiently
          a xsd:string ] 
    ) .
```

Nodes are advised to use Waku messages to include their own
metadata in serializations as needed.  

## Syntax

The semantic description presented above can be reliably round-tripped
through a suitable serialization mechanism.  JSON-LD provides a
canonical mapping to UTF-8 JSON.

The message exchanged are a simple enumeration of three values is not
currently analyzed reflecting the opinion on the given proposal:

    { NO, NONE, YES }

When represented via integers, such as choosing 
 
     { -1, 0, +1 }

the parity summations across network invariants often become easier to
manipulate.

# Security Considerations


## Privacy

In practice, each honest node gossips its current opinion which
reduces the number of messages that need to be gossiped for a given
proposal.  The resulting impact on the privacy of the node's opinion
is not currently analyzed.

## Security with respect to various Adversarial Models 

Adversarial models have been tested for which the values for current
parameters of Glacier have been tuned.  Exposition of the
justification of this tuning need to be completed.

### Local Strategies

#### Random Adversaries

A random adversary optionally chooses to respond to all queries with a
random decision.  Note that this adversary may be in some sense
Byzantine but not malicious.  The random adversary also models some
software defects involved in not "understanding" how to derive a truth
value for a given proposition.

#### Infantile Adversary

Like a petulant child, an infantile adversary responds with the
opposite vote of the honest majority on an opinion.

### Omniscient Adversaries

Omniscient adversaries have somehow gained an "unfair" participation in
consensus by being able to control `f` of `N` nodes with a out-of-band
"supra-liminal" coordination mechanism.  Such adversaries use this
coordinated behavior to delay or sway honest majority consensus.

#### Passive Gossip Adversary

The passive network omniscient adversary is fully aware at all times
of the network state. Such an adversary can always chose to vote in
the most efficient way to block the distributed consensus from
finalizing.

#### Active Gossip Adversary

An omniscient gossip adversary somehow not only controls `f` of `N`
nodes, but has also has corrupted communications between nodes such
that she may inspect, delay, and drop arbitrary messages.  Such an
adversary uses capability to corrupt consensus away from honest
decisions to ones favorable to itself.  This adversary will, of
course, choose to participate in an honest manner until defecting is
most advantageous.

# Future Directions

Although we have proposed a normative description of the
implementation of the underlying binary consensus algorithm (Glacier),
we believe we have prepared for analysis its adversarial performance
in a manner that is amenable to replacement by another member of the
[snow*](#snow*) family.

We have presumed the existence of a general family of algorithms that
can be counted on to vote on nodes in the DAG in a fair manner.
Avalanche provides an example of the construction of votes on UTXO
transactions.  One can express all state machine, i.e. account-based
models as checkpoints anchored in UTXO trust, so we believe that this
presupposition has some justification.  We can envision a need for
tooling abstraction that allow one to just program the DAG itself, as
they should be of stable interest no matter if Glacier isn't. 



# Appendix A: Alvaro's Exposition of Glacier

In this appendix, we present Alvaro's original notion of Glacier,
adorned with suitable edits that attempt to preserve the original
flavor of the exposition.

## Phase One: Querying

A node selects `k` nodes randomly from the complete pool of peers in the
network. This query is can optionally be weighted, so the probability
of selecting nodes is proportional to their 

Node Weighting
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
use the *alpha* threshold defined below. This adaptive growth of
the query size is done as follows:

Every time the threshold is not reached, we multiply *k* by a
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
parameter *l*.) Evidence accumulated keeps the ratio of total positive
votes vs the total votes received (positive and negative), whereas the
evidence per round stores the ratio of the current round only.

Parameters
$$
\begin{array}{lc}
\text{Look-ahead parameter}      & l = 20 \newline
\text{First evidence parameter}  & \alpha_1 = 0.8 \newline
\text{Second evidence parameter} & \alpha_2 = 0.5 \newline
\end{array}
$$

Computation
$$
\begin{array}{lc}
\text{Confidence}                & c_{accum} \impliedby \frac{total\ votes}{total\ votes + l} \newline
\text{Total accumulated evidence}& e_{accum} \impliedby \frac{total\ positive\ votes}{total\ votes} \newline
\text{Evidence per round}        & e_{round} \impliedby \frac{round\ positive\ votes}{round\ votes} \newline
\end{array}
$$


## Phase Three: Transition function 

In order to eliminate the need for a step function (a conditional in
the code), we introduce a transition function from one regime to the
other. Our interest in removing the step function is twofold:

1. Simplify the algorithm. With this change the number of branches is
   reduced, and everything is expressed as a set of equations.

2. The transition function makes the regime switch smooth,
   making it harder to potentially exploit the sudden regime change in
   some unforeseen manner. Such a swift change in operation mode could
   potentially result in a more complex behavior than initially
   understood, opening the door to elaborated attacks. The transition
   function proposed is linear with respect to the confidence.

Transition Function
$$
\begin{array}{cl}
evidence & \impliedby e_{round} (1 - c_{accum}) + e_{accum} c_{accum} \newline
\alpha &  \impliedby \alpha_1 (1 - c_{accum}) + \alpha_2 c_{accum} \newline
\end{array} 
$$
    
Since the confidence is modeled as a ratio that depends on the
constant `l`, we can visualize the transition function at
different values of `l`. Recall that this constant encapsulates
the idea of “near future” in the frequentist certainty model: the
higher it is, the more distant in time we consider the next
valuable input of evidence to happen.

We have observed via expeiment that for a transition function to be
useful, we need establish two requirements:

1.  The change has to be balanced and smooth, giving an
    opportunity to the first regime to operate and not jump directly
    to the second regime.

2.  The convergence to 1.0 (fully operating in the second regime)
    should happen within a reasonable time-frame. We’ve set this
    time-frame experimentally at 1000 votes, which is in the order of
    ~100 queries given a *k* of 9.

[[ Note: elaborate on the selection of k, diameter of graph, etc. ]]

[[ Note: Avalanche uses k = 20, as an experimental result from their
deployment. Due to the fundamental similarities between the
algorithms, it’s a good start for us. ]]
  
## Phase Four: Opinion and Decision  

The next step is a simple one: change our opinion if the threshold
*alpha* is reached. This needs to be done separately for the yes/no
decision, checking both boundaries. The last step is then to *decide*
on the current opinion. For that, a confidence threshold is
employed. This threshold is derived from the network size, and is
directly related to the number of total votes received.
  
Decision
$$
\begin{array}{cl}
evidence > \alpha & \implies \text{opinion YES} \newline
evidence < 1 - \alpha & \implies \text{opinion NO} \newline
if\ \text{confidence} > c_{target} & THEN \ \text{finalize decision} \newline
\end{array}
$$

Note: elaborate on `c_{target}` selection.

# Colophon

# Informative References

0. [Logos](<https://logos.co/>)

1. [On BFT Consensus Evolution: From Monolithic to
   DAG](<https://dahliamalkhi.github.io/posts/2022/06/dag-bft/>)

2. [snow-ipfs](<https://ipfs.io/ipfs/QmUy4jh5mGNZvLkjies1RWM4YuvJh5o2FYopNPVYwrRVGV>)

3. [snow*](<https://www.avalabs.org/whitepapers>) The Snow family of
   algorithms
   
4. [Move](<https://cloud.google.com/composer/docs/how-to/using/writing-dags>)
    Move: a Language for Writing DAG Abstractions 

5. [rdf](<http://www.w3.org/1999/02/22-rdf-syntax-ns#>)

6. [rdfs](<http://www.w3.org/2000/01/rdf-schema#>)

7. [xsd](<http://www.w3.org/2001/XMLSchema#>) 

8. [n3-w3c-notes](<https://www.w3.org/TeamSubmission/n3/>)

9. [ntp](<https://www.ntp.org/downloads.html>)

## Normative References

0. [glacier](<https://rdf.logos.co/protocol/glacier/1/0/0/raw>)

1. [n3](<https://www.w3.org/DesignIssues/Notation3.html>)

2. [json-ld](<https://json-ld.org/>)

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

## Format

This document attempts to be formatted in polyglot Markdown.

This document was utilizing Pandoc 2.18's notion of Markdown, but is
now trying to co-exist with SVG via MathJax 3.2 or Github.
