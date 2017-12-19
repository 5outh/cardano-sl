# Establishing trust in a snapshot

This document describes the problem of establishing trust in a
snapshot and proposes two ways to solve it. It is needed
for [fast synchronization](#README.md).

## The problem

Suppose there is a node which is launched for the first time. It has a
way to download a snapshot necessary to process new blocks, figure out
wallets' balances, make transactions and all other activities which we
want to support. The problem is to check whether this snapshot is
correct, i. e. corresponds to a valid sequence of blocks. There are
several properties we want to optimize:
* How possible is that incorrect snapshot will be considered correct?
  How costly is it?
* How much data needs to be downloaded (as a function of time and
  maybe other parameters)?
* How many local computations should be performed?

## Why the problem is harder for PoS

In PoW system the problem can be solved in the following way:
* Put hash of snapshot into each `n`-th block.
* Download hash from block with depth in range `[n .. 2 · n)`.
* Obtain actual snapshot `S` corresponding to that hash somehow (out scope
  of this document).
* Download all other blocks after `B`.
* Verify these blocks according to snapshot `S`.
* If blocks are valid, we can trust snapshot `S`.

It works fine in PoW system, because creating `n` valid blocks is very
expensive (assuming `n` is large enough).

However, creating blocks in PoS system is very cheap. So one can
create a snapshot according to which they own all stake and then
create a lot of blocks on top of that snapshot very quickly.

## Solution with subscribable checkpoints

TBD

## Blyad po-chelovechesky

This solution is decentralized. The idea is to build another chain
which will be much smaller than the actual chain, but will contain
enough data to verify its validity.

### Description

First of all let's introduce a constant `l ≤ k`. It will be used later.

Apart from actual blocks each full node stores another sequence of
blocks. Let's call them _tiny_ blocks. One tiny block corresponds to one
epoch (`e`) and contains the following data:
* List of stakeholders elected to create last `2 · l` slots in epoch
  `e`. Note that in presence of heavyweight delegation it's not the
  same as list of slot leaders (if Alice is a leader and delegated to
  Bob, she can't create a block). Stakeholders are represented by
  hashes of their public keys.
* Signatures of this list issued by creators of blocks from last `2 · l`
  slots from epoch `e - 1`.
* Public keys corresponding to signatures (i. e. which can be used to
  verify signatures) from the previous bullet point.

We should also put some extra data into the actual blockchain to be
able to compute tiny blocks. We require all blocks `b` in epoch `e`
after slot `10 · k - 2 · l - 1` until the end of the epoch to have a
signature `sig_e`. This signature must be issued by the creator of
`b`. They should sign list of leaders of last `2 · l` slots in epoch
`e + 1` (which can be computed from blocks prior to `b`). If
stakeholder creates more than one block in this interval, only the
first one should contain the signature.

Another extra data which should be put into the actual blockchain is
hash of recent snapshot. For example, in each epoch we can put
snapshot corresponding to the state before `6 · k`-th slot into the
first block after `8 · k`-th slot.

It's not necessary to put signature `sig_e` or snapshot hash into
header, they can be put into block (and their hash should be part of
header for integrity).

#### Maintenance of tiny blocks

Full nodes which have actual blocks can easily maintain tiny blocks. A
tiny block for epoch `e` can be created when `e` starts. At that
moment we know leaders of new epoch and have last `2 · l` blocks from
epoch `e - 1`.  Those blocks contain signatures of list of new leaders
and corresponding public keys. So there is all necessary data to
create tiny block for epoch `e`.

#### Fast synchronization procedure

If node lacks a lot of blocks and wants to quickly obtain a recent
snapshot, it does the following:
1. Send few most recent `HeaderHash`es of last accepted blocks to one
   or more peers, just like it's done for regular synchronization.
2. The peer finds the most recent header among received ones in their
   blockchain. Suppose this header is from epoch `e`. Note that it's
   important to send more than one hash, because node could be stopped
   before fork happened and may have newest blocks which are not part
   of the main chain. So the node has all blocks from all epochs
   before `e`.
3. Then the peer sends a bunch of tiny blocks starting from one for
   epoch `e`.
4. When the node receives a tiny block for epoch `e`, it knows who can
   create blocks in that epoch. So it can verify that signatures from
   the tiny block are legitimate and provided by actual slot leaders.
5. Now the node knows who can create blocks in last `2 · l` slots in
   epoch `e` (this information is available from tiny block for
   epoch `e`). It can then verify signatures from tiny block for
   epoch `e + 1`.
6. Repeat step 5 for all other tiny blocks. Request more tiny blocks
   after the first bunch is processed. A request may include hash of
   last processed tiny block, for example, or just epoch
   number. Download and process all available tiny blocks.
7. If current epoch is `x`, last tiny block will be for epoch
   `x`. Request headers from last `2 · l` slots of
   epoch `x - 1`.
8. Verify that headers are consecutive and issued by proper
   stakeholders. If that's the case, request the block corresponding
   to the oldest header and take hash of snapshot (`H`) from it.
9. Download snapshot somehow. The mechanism to download a snapshot is
   not described in this document. Check it's correctness according
   to `H`.
10. Download all blocks after that snapshot and verify them fully
    according to the snapshot.
11. After all recent blocks are processed, the node is fully
    functional. It still makes sense to download all other blocks to
    be 100% sure the whole chain is valid.

#### Choice of `l`

The choice of `l` affects performance requirements and security
guarantees. There is a certain trade-off.

The biggest reasonable value of `l` is `k` (security parameter of
Ouroboros protocol). It's the best value from security point of
view. However, it might be not feasible for real system for
performance reasons. Specifically, there are two problems:
1. It's not trivial to compute slot leaders for an epoch, because it
   requires iterating over whole UTXO or stakes map which might be
   big. It's not yet clear how much time it will take in practice, but
   probably not just few seconds. If it takes a minute and `l = k`, first three
   slots among last `2 · k` will be missed, because their leaders will
   be computing slot leaders (because it's mandatory to put a
   signature of this list into new block).
2. If leaders must be computed before creating a block for `8 · k`-th
   slot, it's enough to force rollback of only one block to require
   nodes to recalculate slot leaders. It's not very expensive to make
   a fork of depth 1 (or 2, 3). However, computing slot leaders is an
   expensive operation, so we'd like to avoid doing it more than once
   for epoch.

If we pick `l` slightly smaller than `k`, both problems will be
resolved with high probability. In Cardano SL `k = 2160`. Suppose we
say `l = 2100`. Then:
1. Each node will have 40 minutes to compute list of slot
   leaders. 20 minutes should be enough for everybody.
2. Rollback of 60-120 (depending on chain quality) blocks is very
   unlikely to happen, so almost always it will be enough to compute
   slot leaders only once.
3. However, we will assume that there is at least one honest block in
   2100 blocks, not in 2160. Probably it's not a big problem, because
   probability of all blocks being invalid decreases exponentially, so
   there should be significant difference between 2100 and 2160 (in
   both cases probability is negligible).

### Properties

#### Correctness of the snapshot

The probability of the snapshot being incorrect is the same as the
probability of a fork in Ouroboros protocol with depth more than
`l`. If `l` is almost same as `k`, this probability is negligible.

Recall the steps from
[fast synchronization procedure](#fast-synchronization-procedure). The
node can easily verify tiny block for `e`, because it knows leaders
for epoch `e` (it has all blocks before `e`). For tiny block for
epoch `e + 1` it's only possible to verify that signatures are valid
and correspond to actual slot leaders, but it's impossible to verify
list of slot leaders put into that tiny block. However, it's easy to
see that if signatures are valid, list of leaders should also be valid
if Common Prefix property with parameter `l` is not violated. In this
case among last `2 · l` slots of epoch `e` there are at least `l`
blocks and at least one of them must be produced by an honest
stakeholder according to Common Prefix property. It implies that at
least one signature in the tiny block for epoch `e + 1` is produced by
an honest stakeholder, so the whole list must be correct. Verification
of subsequent tiny blocks is the same.

If all tiny blocks are correct, headers received at step 8 are
guaranteed to be created by real slot leaders. According to Common
Prefix property, at least the oldest block must be created by an
honest stakeholder, hence it must contain the hash of the correct
snapshot.

#### Data to download

#### Local computations

### Possible optimizations

Using some optimizations it's possible to improve some properties at
the cost of making the design and/or implementation a bit more
complex.

* Aggregate signature
* Compression
* Public keys instead of `StakeholderId`s.
* Put at most `k` signatures.

### A note about delegation

If we preserve heavyweight delegation, we must make a change to put
heavweight delegation into effect only in the next epoch after
certificates is put into block. Currently if there is a certificate in
block `X` from Alice to Bob, all blocks after `X` must be created by
Bob instead of Alice. But proposed solution assumes that we know who
will create blocks in all slots of epoch `e` when `e` starts.

There is no such problem for lightweight delegation, because we don't
need to check whether lightweight delegation certificate is in
the blockchain.

### Pros and cons


### Inclusion into Cardano SL
