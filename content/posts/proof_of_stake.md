---
title: 'Proof of Stake: how does it work anyway'
date: 2022-09-02T19:12:46-03:00
draft: false
meta_img: "images/image.png"
tags:
  - "pos"
  - "merge"
description: "My investigation into proof of stake"
math: true
---
With **The Merge** around the corner (I've read posts from 2014 starting with exactly the same sentence) I thought it would be a good time [to share something I  wrote when researching proof of stake](https://github.com/joaquinlpereyra/blog/raw/master/content/pdf/Proof%20of%20Stake.pdf).

This is basically my version of _what the heck is proof of stake and how does it work anyway_, particularly the how does it work part. It goes over Tendermint but focuses on Gasper, which is what Ethereum will implement.

[The piece](https://github.com/joaquinlpereyra/blog/raw/master/content/pdf/Proof%20of%20Stake.pdf) is intended to be kind-of-readable but still very in depth for people who already know how blockchains and proof of work function. 

It is in PDF, because damn me there are some formulas in there which I don't know how to markdown.

It is far from perfect, and at times it skews too much into _just notes for myself_. If you have any questions about it just ping me at @0xmatebabe on Twitter, we can discuss them.

A [[Consensus]] mechanism for descentralized, distributed systems. The major competition to [[Proof of Work]].

**Sources:**
- [Cosmo Network comparison of Casper and Tendermint](account_tx_rate_limit_algorithm)
* [Combining GHOST and Casper (Vitalik et. al.)](https://arxiv.org/pdf/2003.03052.pdf)

**Open questions**

_Q_: Why is it always _1/3_ of validators?
A: Answer can be found in the paper _Asynchronous consensus and broadcast protocols_ dealing with _Practical Byzantine Fault Tolerance_.

In proof of work, creating a block is extremely expensive. In proof of stake, it is essentially free: you only need to sign it. All the effort in proof of stake is in making it costly to create an invalid or malicious block.

> Instead of using computational power to propose blocks, proposing blocks is essentially free. In exchange, we need an additional layer of mathematical theory to prevent perverse incentives that arise when we make proposing blocks “easy.”
> Combining GHOST and Casper

Main ingredients: 
1. A [[Fork Choice Rule]]: a function $fork()$ that when given a _view_  $G$ (ie, a series of blocks) can identify a canonical chain with its single leaf node (ie: latest block) $B$. Intuitively: a rule for validators to choose the right chain when presented with conflicting blocks.
2. A [[Finality]] definition: a function $fork$ that given $G$ will return a set $F(G)$ of _finalized_ blocks which will never be removed from the canonical chain.
3. Slashing conditions: conditions that honest validators will never break and dishonest validators can be provably identified. 

Note that a consensus mechanism can have:

1. **Safety**: if the set $F(G)$ of finalized blocks can never contain conflicting blocks.
2. **Liveness**: if the $F(G)$ actually _grows_. Liveness can be plausible (impossible to become deadlocked) or probabilistic (if it liveness needs some probabilistic assumptions).

# Pitfalls of PoS
- **Nothing at stake**: validators have no incentive to vote for a specific blob of data (in [[Blockchain]], a block). If several choices are present, the rational behavior is to vote for all of them to be assured I get the reward for voting on the winning choice. Solved by [[#Slashing]].
- **Long range attacks**: if I can remove my deposit, I in collution with other validators remove it... then create a fork from the point where I was still a validator (why not? what are you gonna slash?). Here enters the embrace of [weak subjectivity](https://blog.ethereum.org/2014/11/25/proof-stake-learned-love-weak-subjectivity/) by PoS, which introduces _subjectivty_ into the protocol: the biggest chain _will not be always that with most work or stake_, but some other considerations are taken into account: validators node must currently have a stake, etc.
- **Cartel formation**: few validators with a lot of capital will outweight smaller ones. Tendermint basically accepts this. Casper protocol is the only construction which tries to fight this with in-consensus censorship-resistance incentives _(which ones?)_
# Implementation strategies
Two main strategies to implement PoS. 
## Byzantine Fault Tolerant Proof of Work
Chooses consistency over availability: some transactions may not get processed, but you sure as hell will get a consistent state. [[Tendermint]] is an example of this type of [[Consensus]].

A validator is assigned the right to propose a block at random. Commiting that block needs a supermajority of 2/3 of validators (_why this number?_). 

As an example of consistency over availability: if more than 2/3 of validators in [[Tendermint]] are offline, the network may halt: there's not enough voters to vote for the next block.

There are no forks on Tendermint based blockchains: forks are the result of two miners finding a block at the same time. With a single validator proposing a node, this is just impossible.

This may seem like a silly problem, but actually it is quite bad: if a network upgrade makes 2/3 of validators offline, the rest of validators can't continue with the old chain!

## Chain-based Proof of Work
They simulator more directly [[Proof of Work]]. The new block is hash-linked to the parent block. [[Casper Protocol]] is of this kind. It favor availability over consistency.

As of 2021, I don't see many differences with Tendermint. Seems like they have converged? Maybe.

# GASPER

The objective of this piece is to gain intuition on how Ethereum's [[Beacon Chain]] works for the reader familiar with proof of work blockchains, such as current Ethereum (as of April 2022) or Bitcoin. 

### Time
A key difference is the introduction of _time_. In [[Proof of Work]], the blockchain itself can be seen as a malfunctioning clock, where each block is the quantum of time. Event are measured in their block-time. This works because [[Proof of Work]] requires _work_, and because that work should be relatively difficult, that _work_ requires physical time. 

In [[Proof of Stake]], creating blocks is trivial. You actually just ensemble the data and off it goes. So we actually need to introduce clocks into the blockchain. This is, to me, is one of the main differences one should keep in mind.

As mentioned above, Gasper introduces time with _slots_. 

* Slots are 12 second long, and slots _may_ have a block associated. An empty slot is possible if no one proposed a block.
* _Epochs_ are rounds of 32 blocks, where a committee of validators are chosen to _attest_ for their head of the chain.

### Checkpoints
Another very important difference, related to the introduction of natural time, is that we now have _special_ blocks: the blocks that begin each epoch. In traditional proof of work, we had as a special case only the Genesis block. Now, every block that starts an epoch is special, and is called a _epoch boundary block_.

### Block publishing
Every time an _epoch_ starts, a new committee is chosen at random from the set of validators.

Every time an _slot_ starts, one committee member must propose a block by computing the canonical head of the chain at that slot. The block itself is fairly similar to those in proof of work, but for two _details_:

1. It contains a filed _newttests_, attestations that the proposer has accepted and haven't appeared in their view of the blockchain yet.
2. Arbitrary data. This is actually a fairly important detail, not for the protocol, but for Ethereum: this arbitrary data is, in the [[Beacon Chain]], the hash of the head of the block of the execution layer! (See [Eth1Data](https://beaconcha.in/block/3529978) on the Beacon Chain Explorer)

### Attestations
So, what are exactly attestations? Intuitively, attestations are messages containing a block from each validators view. But they also contain a so-called _checkpoint edge_, a transition from _epoch boundaries_. These are used for justification and finalization.

We can see attestations published with the blocks [in the explorer](https://beaconcha.in/block/3530362#attestations). Note that these attestations are [_aggregated_](https://kb.beaconcha.in/attestation) so as not to overload the block proposer with messages. 

Note that we can attest not only for the previous block published, but for even older blocks than that, although attestation rewards are best when the delay is kept at a minimum. 

### Justification and finalization
Justification is a new concept from those of us used to traditional Ethereum, while finalization takes a different form.

We say that a checkpoint $B$ (ie: an epoch boundary) is _justified_ when more than 2/3 of the stake has attested for the transition between a previously justified checkpoint and $B$. The genesis block is always justified, allowing us to loop out of the recursion.

As far as I know, _justification_ is a concept useful only to build up to _finality_. While in traditional proof of work blockchains we have _probabilistic finality_, in GASPER it is baked into the protocol. 

For those unfamiliar with _finality_, a _message_ (transaction, block) is considered final when it can't be reversed. In Bitcoin and other proof of work blockchains, finality is always probabilistic: it is _extremely unlikely_ that after six confirmations your block will get reversed, but it is _possible_ in theory. 

In GASPER, _finality_ is provable: we can definitively say that a message is final and the protocol will _reject_ any attempt to publish a message which conflicts with a finalized one. Finality is achieved in GASPER when a checkpoint block is justified and it is used to justify the next block.

So, in ideal conditions, _justification_ of a checkpoint happens as soon as the next epoch starts (everyone voted to pass from the state A to B and A was justified, so B is justified) and _finalization_ of a checkpoint happens after two epochs (everyone voted to pass from B to C and B was justified, so B is now finalized).

### Consistency
Another interesting point to analyze are forks. Not all proof of stake protocols have forks, for example  [Tendermint consensus](https://docs.tendermint.com/master/tendermint-core/consensus/) only publishes blocks if enough attestations have been gathered for a block. In GASPER, though, forks are possible.

This should be surprising for a distracted reader like me, as I always assume forks happens because of two miners finding the solution for a block at nearly the same time; and Gasper uses only _one_ block proposer, so this is impossible.

But the real world is a messy place, and network latency exists: we only need to imagine two validators with different views of the blockchain proposing blocks with different parents. The fork choice rule of GASPER is tricky, but its intuition its simple. As proof of works blockchain prefer the chain with the highest amount of work put into them, GASPER will prefer the chain with the highest amount of validations in it.

GASPER makes the strategic choice to prefer to be always live instead of always consistent. Allowing forks is a sacrifice needed to be able to forego setting strict timeouts. 

### Liveness 
The GASPER network favors availability and will continue publishing blocks even though not enough validators are online, **but** blocks will never be justified or finalized. Once this situation corrects itself though, the blockchain can continue! Of course it is possible that things don't actually correct _themselves_ and a hardfork to modify the list of validators is needed.

This is again different from [Tendermint consensus](https://docs.tendermint.com/master/tendermint-core/consensus/), which only guarantees liveness in case a supermajority of validators are online.

Another important point is that absolutely no timeouts is assumed to guarantee the consistency promises made, and only a _partially synchronous_ assumption is made to discuss liveness. That is, it only needs for timeouts to _exists_ but they are not set beforehand.

All in all, I think this is a good base to start forming an intuition on how Ethereum is gonna work after The Merge. The [[Beacon Chain]] has been working for a while under this model already and it is very interesting to [explore its blocks](https://beaconcha.in/). The rest of this doc are my notes while reading the [Combining GHOST And Casper](https://arxiv.org/pdf/2003.03052.pdf) paper by Vitalik et al. As notes tend to go, I think they are fairly organized, but there may be mistakes in my understanding of some topics and repeated information. You have been warned. They are probably best read while reading the paper itself. 

## CASPER  FFG
Or **Casper the Friendly Finality Gadget**. Defined _justification_ and _finalization_.

### Definitions
- Every block has a _height_.
- _Checkpoint_ blocks are defined which are blocks whose height is a multiple of a constant $H$ (in [[Ethereum2]], $H = 100$). Checkpoint height $\frac{h(B)}{H}$is always an integer. Note that the subsets of checkpoint blocks also forms a subtree.
- _Attestations_ are signed messsages (ie: blocks) containing _checkpoint edges_ $A \rightarrow B$ where A and B are checkpoint blocks. Each _attestation_ is a _vote_ to move from $A$ to $B$. Each attestation has a _weight_, the stake of the validator.
- In each view of the blockchain $G$, there is a set of _justified_ and _finalized_ checkpoint blocks $J(G)$ and $F(G)$ where $F(G) \subset J(G) \subset G$. The genesis is always both justified and finalized.
- A checkpoint block _B_ is _justified_ (by a _justified_ checkpoint block $A$) if _A_ is justified and there are _attestations_ voting for $A \rightarrow B$ with total weight 2/3 of the total stake. This is also called a _supermajority link_ from A to B.
* A checkpoint block $B$ is _finalized_ if it is _justified_ and there's a _supermajority link_ $B \rightarrow C$.

**Note (!)**: _attestations_ are defined over _checkpoint blocks_, not all blocks!
**Note**: _justification_ is dependent of the view of the node, because in your view you may not have seen all attestation or even the to-be-justified checkpoint block $B$

## GHOST
Intuitively: the chain with most attestations is the correct one.

## Combining [[#GHOST]] and [[#CASPER FFG]] to form [[#GASPER]]

Back to [[#GASPER]] then: even though [[#CASPER FFG]] does not mention _slots_ or _epochs_, [[#GASPER]] does. Another important difference is that the same block may appear as a _checkpoint_ more than once (for different epochs). To disambiguate, one formally must refer to a [[#GASPER]] checkpoint as a pair $(B, e)$ where $B$ is the block and $e$ is the epoch.

Another change is that [[#GASPER]] calls divides its validators _committees_ in each _epoch_, and one _committee_ is used per slot. In each slot, one _validator_ from the _committee_ proposes a block, and all member of the _committee_ attest to what they see as the head of the chain (in the best case, the proposed block) using the fork-choice rule _HLMD GHOST_ (a slight variation of [[#GHOST]]).

### Epoch boundary

**Intuition**: the $aep$ is the _attestation epoch_ and is the period during which attestations for a block can be gathered. The _epoch_ is objective, but the $aep$ of a block depends on your view of the chain.

The _latest epoch boundary block_ of $B$  ($LEBB$) is the latest _checkpoint block_ before $B$ or, if none, the latest block before $B$. 

The $jEBB$ is the epoch-adjusted epoch boundary of a block. You _go back_ $j$ epochs.

- For every block $B$, $EBB(B, 0)$ is the genisis.
- $slot(B) = jC$ for some epoch $j$, $B$ will be an epoch boundary block in every chain that includes it (ie: a checkpoint is always an epoch boundary block, if it exists).

**Opinion**: non intuitive concept. They key is that _every block_ needs a _EBB_, and if you cannot find it, _make one up_. So if I have a chain `63 <- 64 <- 65` and `64` marks the beginning of an epoch, then $EBB(65) = 64$. But if there's a fork an another validator proposes the block `66` like `63 <- 66`, then $EBB(66) = 63$, even though 63 was not a checkpoint block. 

Now, with the _epoch boundary pairs_ (which are the $B, j$ pair identifying the epoch boundary of a block) we define the _attestation epoch $j$_ for $P = (B,j)$ using the notation $aep(B) = j$ which is not necessarily the same as $ep(B)$.

These _epoch boundary pairs_ are used as the _checkpoints_. 

**Note**: the _epoch_ is absolute, the _epoch boundary_ is not. Even in the view `63 <- 66`, `63` nodes not begin an _epoch_, even though it serves as _EBB_ of 66.

**Note**: why make it difficult it with pairs? The paper discussed this. It gives two main reasons: (1) it makes fewer assumptions about latency, as nodes may not have produced a block for the slot which is supposed to be a checkpoint, so the pair represent a best-approximation (2) we can sync time in proof of stake, so let's use that! we can use the concept that blocks will be produced at regular intervals and requrires both data (represented in the blockchain) and time (represented in _epochs_).

### Committees
Fairly intuitive. In each _epoch_, the work of is divided in roughly equal sized _committees_ for each _slot_.

### Blocks and attestation 
Two kinds of _committee_ work:
1. On person proposes a block
2. Everyone in the committee attests to _their_ head of the chain

**Note**: see how this allows forking! Everyone attests to _their_ head othe chain, not for the proposed block!

Let's do the rounds of a slot $i$ from the point of view of _validator_ $V$:

1. The _proposer_ (the first in the list of the committee) uses the [[Fork Choice Rule]] to find $B'$.
2. The _proposer_ proposes a block $B$ which is a message containing $slot(B) = i$ , $P(B) = B'$, $newattests(B)$ a set of pointer to _all_ attestations $V$ has accepted but have not been included in any previous blocks and some _implementation specific data_ (see **Note on fancyness**).
3. Each _validator_ in the _committee_ computes $B'$ and publishes an _attestation $\alpha$_  which is a message containing: $slot(\alpha)=i$, $block(\alpha)=B'$ and a _checkpoint edge_ $LJ(\alpha) \rightarrow LE(\alpha)$. $LJ$ and $LE$ are _epoch boundary pairs_ in the view of the validator at time _i_ plus some amount of time due to delay. These functions are defined later.

**Note on fancyness**: the paper wants to be fancy and say it doesn't care about the data, but as users we do. This is the `Eth1Data` found in the [[Beacon Chain]] implementations, and is extremely important as it tracks the execution layer results! So basically _what we now know as [[Ethereum]]_ is all in there.

### Justification

**Definitions**:
- $view(B)$ the view consisting of $B$ and all its ancestors. 
- $ffgview(B)$, the _FFG View of B_, to be $view(LEBB(B))$ (ie:the view since the last epoch boundary block)
* $(A, j') \rightarrow (B, j)$ is a _supermajority link_ from pair $(A, j')$ to pair $(B, j)$ if the attestations with checkpoint edge $(A, j') \rightarrow (B, j)$ have a total weight of more than 2/3 of the stake.
* Given a view $G$, we define _justified of G_, $J(G)$, as the genesis plus all the blocks such that if $(A, j') \in J(G)$ and $(A, j') \rightarrow (B, j)$  with _supermajority_, the $(B, j) \in J(G)$ as well. Meaning: if there's a supermajority vote for the state transition into $(B, j)$ and the previous state was justified, then the new state is justified.

Time to define $LJ$ and $LE$! Finally. Remember from [[#Blocks and attestation]]? 

Given an attestation $\alpha$ and $B = LEBB(block(\alpha))$:
* $LJ(\alpha)$, the _last justified pair of $\alpha$_, the highest attestation epoch justified pair in $ffgview(block(\alpha)) = view(B)$ . 
* $LE(\alpha)$, the _last epoch boundary pair of $\alpha$, to be $(B, ep(slot(\alpha)))$.

Daunting. OK. Intuitively, $LJ$ is then the latest justified epoch boundary which is in view of $B$. $LE$ is just the last epoch boundary. OK. Not so hard.

### Finalization
Core concept: once a block is finalized,  _no view_ will have a conflicting block with it, unless the blockchain is (1/3)-slashable (ie: everything is broken, someone controls more than 1/3 of the stake). 

Finalization happens for a block and epoch pair $(B_0, j)$ if:

* $(B_0,j)$ is the genesis at epoch zero.
* Or if there's a $k \geq 1$ such that $(B_0, j),...,(B_{k-1}, j+k-1)$   which are justified and there's a supermajority to transition $(B_0, j) \rightarrow (B_k, j+k)$. 

The last condition is confusing, but in plain English it states that the finalized block must be justified _and_ there must be a supermajority that votes to transition from it to another state.

**Note**: the difference between finalization and justification is not as intuitive as one may wish but looking at the formulas, but is easy intuitively: a block is _justified_ when the supermajority voted to transition into its state, a block is _finalized_ when it is used as the base to go into another state. 

In even plainer English: in the happy case, a justified block will always be the current  epoch boundary and the finalized block the previous epoch boundary. 

**Note**: in ideal conditions, $k = 1$, but the paper allows for $k \gt 1$ to account for network latency and other implementation issues. Note that $k \gt 1$ only implies that the justifications could skip blocks, for example: supermajority could vote to go from $B_0 \rightarrow B_2$ (this is $k = 2$). 

![[Supermajority and finalization.png]]

### LMD GHOST
The [[Fork Choice Rule]]. It has many _ifs_ and _buts_ which may be read on the paper, but the idea is that presented with a fork, the chain with the most _validations_ wins.

### Slashing conditions
The slashing conditions are only two, and are quite simple to understand:

1. No validator makes two distinct attestations $\alpha_1$ and $\alpha_2$ with $ep(\alpha_1) = ep(\alpha_2)$. Meaning: no validator attests two blocks in the same epoch! 
2. No validator makes two distinct attestations $\alpha_1$ and $\alpha_2$ with $aep(LJ(\alpha_1)) \lt aep(LJ(\alpha_2)) \lt aep(LE(\alpha_1)) \lt aep(LE\alpha_2)$.

Condition (1) is fairly intuitive: validators are only required to attest once per epoch (committees are formed one per epoch). It is also equivalent to saying $aep(LE(\alpha_1)) = aep(LE(\alpha_2))$. Remember $aep$ is the _attestation boundary epoch_ and $LE$ is the last epoch boundary of a block. 

Condition (2) is more complicated. The paper goes a bit more into details specially regarding why an honest validator can't incur in this behavior. _I think_ the intuition is that this asks validators on epochs $r \lt s \lt t \lt u$ to be consistent: if they voted to transition from $(B_2, s) \rightarrow (B_3, t)$ they can now not vote for $(B_1, r) \rightarrow (B_4, u)$. They must have been lying at some point! Why did they vote for the first one knowing that later they would vote for the second one? 


