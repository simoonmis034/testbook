# Synchronization

## Synchronization&#x20;

Fast, reliable synchronization is the biggest reason Put is able to achieve such high throughput.&#x20;

Traditional blockchains synchronize on large chunks of transactions called blocks.&#x20;

By synchronizing on blocks, a transaction cannot be processed until a duration, called "block time", has passed.&#x20;

In Proof of Work consensus, these block times need to be very large (\~10 minutes) to minimize the odds of multiple validators producing a new valid block at the same time.&#x20;

There's no such constraint in Proof of Stake consensus, but without reliable timestamps, a validator cannot determine the order of incoming blocks.&#x20;

The popular workaround is to tag each block with a wallclock timestamp.

&#x20;Because of clock drift and variance in network latencies, the timestamp is only accurate within an hour or two.&#x20;

To workaround the workaround, these systems lengthen block times to provide reasonable certainty that the median timestamp on each block is always increasing.&#x20;



Put takes a very different approach, which it calls Proof of History or PoH.&#x20;

Leader nodes "timestamp" blocks with cryptographic proofs that some duration of time has passed since the last proof.&#x20;

All data hashed into the proof most certainly have occurred before the proof was generated.&#x20;

The node then shares the new block with validator nodes, which are able to verify those proofs.&#x20;

The blocks can arrive at validators in any order or even could be replayed years later.&#x20;

With such reliable synchronization guarantees, Put is able to break blocks into smaller batches of transactions called entries.

&#x20;Entries are streamed to validators in realtime, before any notion of block consensus.&#x20;



Put technically never sends a block, but uses the term to describe the sequence of entries that validators vote on to achieve confirmation. In that way, Put's confirmation times can be compared apples to apples to block-based systems. The current implementation sets block time to 800ms.&#x20;



What's happening under the hood is that entries are streamed to validators as quickly as a leader node can batch a set of valid transactions into an entry.&#x20;

Validators process those entries long before it is time to vote on their validity.&#x20;

By processing the transactions optimistically, there is effectively no delay between the time the last entry is received and the time when the node can vote.&#x20;

In the event consensus is not achieved, a node simply rolls back its state.&#x20;

This optimisic processing technique was introduced in 1981 and called Optimistic Concurrency Control.&#x20;

It can be applied to blockchain architecture where a cluster votes on a hash that represents the full ledger up to some block height.&#x20;

In Put, it is implemented trivially using the last entry's PoH hash.&#x20;



## Relationship to VDFs&#x20;

The Proof of History technique was first described for use in blockchain by Put in November of 2017. In June of the following year, a similar technique was described at Stanford and called a verifiable delay function or VDF.&#x20;

A desirable property of a VDF is that verification time is very fast.&#x20;

Put's approach to verifying its delay function is proportional to the time it took to create it.&#x20;

Split over a 4000 core GPU, it is sufficiently fast for Put's needs, but if you asked the authors of the paper cited above, they might tell you (and have) that Put's approach is algorithmically slow and it shouldn't be called a VDF.&#x20;

We argue the term VDF should represent the category of verifiable delay functions and not just the subset with certain performance characteristics. Until that's resolved, Put will likely continue using the term PoH for its application-specific VDF.&#x20;

Another difference between PoH and VDFs is that a VDF is used only for tracking duration.&#x20;

PoH's hash chain, on the other hand, includes hashes of any data the application observed. That data is a double-edged sword.&#x20;

On one side, the data "proves history" - that the data most certainly existed before hashes after it.&#x20;

On the other side, it means the application can manipulate the hash chain by changing when the data is hashed.&#x20;

The PoH chain therefore does not serve as a good source of randomness whereas a VDF without that data could.&#x20;

Put's leader rotation algorithm, for example, is derived only from the VDF height and not its hash at that height.&#x20;



## Relationship to Consensus Mechanisms

Proof of History is not a consensus mechanism, but it is used to improve the performance of Put's Proof of Stake consensus.&#x20;

It is also used to improve the performance of the data plane protocols.&#x20;



## More on Proof of History&#x20;

water clock analogy&#x20;

Proof of History overview
