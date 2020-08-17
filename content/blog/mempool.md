+++
title=" The Urgency of Rearchitecting Bitcoin's Mempool"
date='2020-08-01'
author="Jeremy Rubin"
front_pic='/img/complex-mempool-sm.png'
summary="""A deep dive into understanding some of the architectural challenges in Bitcoin's mempool
and where future research may make improvements."""
+++

### *If this post excites you let's [work together](/join/). Or, if you see the value of this work and are able to dedicate financial resources, [please reach out](/join/) to support more developers working on this project.*

------------

Bitcoin is a complicated software artifact which requires active investment in discovering its flaws
to harden and strengthen its implementations against fatal attacks and insidious bugs. Over the last
year I've been investing significant efforts to put Bitcoin's Mempool under the microscope to
categorise potential bugs and vulnerabilities and come up with plans and patches to fix them.

The fruits of this labor so far have been promising in their ability to address substantial
stability issues in Bitcoin, but there is still much work to be done to fully implement the
necessary changes and tackle some of the harder architectural challenges. The Mempool serves as the
main gatekeeper for submitting transactions to the network for inclusion in a block, which makes it
one of the main ways all types of users interact with Bitcoin, and drives the urgency of completing
this security critical work in a manner that respects the myriad of currently supported uses.


Judica exists in part to build a team to tackle the hard problems facing Bitcoin. In the last two
months, [100x/BitMEX](/blog/100x-grant/) and [ACINQ](/blog/acinq-grant) have issued grants to
support Judica's work improving Bitcoin's Mempool over the next year. To grow our team, I plan to
use these grants to sponsor other developers (either in a full or part time capacity) to contribute
to the Mempool efforts.  One of the chief bottlenecks in Bitcoin development is tight review and
feedback loops on ambitious projects like this, so having more developers engaged under contract to
complete this project as a team will help us deliver results for the Bitcoin community.

Judica is not the only organization working on improving the Mempool, Bitcoin developers have paid
serious attention to it over the years and have made vast improvements and contributions. Other
contributors are focused on issues like package relay and rebroadcasting. However, with this new
effort I'm hoping to assemble the resources to make sustained progress on the  goals described below and
increase the number of developers who understand the Mempool functionality fully enough to provide
high-quality review.


In the remainder of this blog post, I'm going to explain the tensions between implementing an
idealized functionality of the Mempool and making it secure against resource exhaustion attacks.

## The Mempool: Bitcoin's MVP

The Mempool is one of the most critical subsystems in the Bitcoin network, yet its function is
poorly understood by many businesses and projects. What role does the Mempool play?

Before transactions are mined in a block, they are typically submitted to the network to be relayed
and stored until block inclusion. A Bitcoin miner uses the Mempool to select which transactions they
wish to include into a block. A perfectly rational node would accept and store all transactions it
had ever seen and pick the best transactions (i.e., those paying the highest fees) for mining. 

Incidentally, this is why the Mempool is such an important data structure in Bitcoin. With a less
rational Mempool, miners would produce less valuable blocks, and there would be less incentive to
mine. All of Bitcoin would suffer for having worse security.

It's not practical to store, process, and consider all transactions, so instead nodes make decisions
on what to accept based on a number of heuristics. The Mempool is used for more than just mining,
network nodes use the Mempool to figure out what is likely to be included in a block in the future
to decrease block propagation latency and to relay other profitable transactions across the network,
improving privacy (not all transactions from a node are yours) and improving the user experience of
transaction broadcast.

The Mempool has a number of artificial constraints on what types of transactions it will accept by
default. These limitations include static checks (e.g., the size of the transaction in bytes or
"weight units") and contextual checks (e.g., is this transaction mineable immediately, does it pay
more fee than other transactions, or does it have more unconfirmed parent or grandparent
transactions than permitted or cause another transaction to have more children transactions than
permitted).

These rules are artificial in the sense that they are locally decided policies that are not
generally enforced. In other words, when you mine a block, these Mempool restrictions are not
considered. Why does the Mempool need to have stricter rules than a block?

### It's Complicated!

Many of the Mempool algorithms are "superlinearly algorithmically complex". This means that as the
Mempool grows, if these restrictions were not enforced, the amount of work to select transactions
for a block to mine could grow as well, which can lead to resource exhaustion, denial-of-service,
and cost miners money with a loss of uptime. Imagine it taking more time for a miner to decide what
should go into a block than it takes to mine the block itself! Therefore we need stricter rules in
the mempool to prevent these blowouts.

{{< figure src="/img/less-complex-mempool.png" width="100%" caption="Visualization of Mempool with 800 transactions. See [this PR](https://github.com/bitcoin/bitcoin/pull/17292#issuecomment-547592769) for more details.">}}

Unfortunately, a number of algorithms in the Mempool are suboptimal, meaning where a small amount
of operations might be sufficient, a much larger amount is done. Suboptimal often also means that
while the overall work is similar in terms of the number of operations, the algorithm's operations
are inefficient to run (maybe allocating too much memory or just doing something slowly).
{{< figure src="/img/complex-mempool.png" width="100%" caption="Visualization of Mempool with 10,000 transactions. See [this PR](https://github.com/bitcoin/bitcoin/pull/17292#issuecomment-547592769) for more details. This makes it easy to see how the work becomes harder as more entries are added.">}}

These algorithms performance mean that we have to determine safe limitations for what we will accept
into the Mempool that fits within our computational budgets, and if the performance is suboptimal,
we end up with stricter limitations than necessary, or even end up requiring a limitation where it
would be fine not to have any. Developers often determine safe limits experimentally (as opposed to
by a rigorous proof), which means we have to include an additional safety margin in case we haven't
found the true worst case behavior.

### What's the worst that can happen?

Because we make a best effort, and not a perfect one, Mempool implementations have to make some
tough decisions about which transactions to process and which to ignore in order to prevent resource
exhaustion or other denial-of-service vulnerabilities. Even with the guards we have in place, it's
possible that a series of carefully crafted but otherwise normal input could cause all nodes on the
network to crash or cease to be functional.

Fully understanding the gap between perfectly rational Mempool behavior and computationally limited
Mempool behavior can be quite hard and full of surprising edge cases, which is the bane of many
protocol developers and infrastructure operators.

As an example, there's a generic issue called "mempool pinning" which impacts protocols like the
Lightning Network or when exchanges payout via batches. The issue comes up when a transaction in the
Mempool has two outputs owned by two different parties, one good, one evil. The evil party attaches
a bunch of low-fee transactions to their output. Now, because of the Mempool's limitations on how
many child spends an unconfirmed transaction may have, the good party is unable to spend from their
unconfirmed output. This seems like a minor problem, but the issue can be more severe in contexts
that rely on Child Pays for Parent (CPFP) for paying for fees and have time lock conditions.

### So what're we gonna do about that?!

 Don't fret. Judica is working to improve the Mempool. The Mempool Project is an effort that has multiple
components aimed at improving Mempool performance with the goal of reducing the maximum memory the
Mempool requires to operate and improving performance. This work may ultimately lead to being able
to remove some restrictions previously required or loosen them up. This makes application developers
happy, improves mining revenue, and hopefully helps protocols be more robust against attacks.

To complete this work, multiple components must be developed simultaneously:

1) Epoch Mempool --  a base level algorithmic improvement across the Mempool with minimal
architectural changes that has already shown major performance benefits (2-3x faster on difficult
benchmarks) but requires more review and testing.

2) Mempool Stress Tests -- identifying and assembling a catalogue of worst-case exploitable
behaviors (with and without current limitations enforced and other non standard node flags). This
makes us more confident in the changes we're making and can also discover and prevent regressions
introducing new Denial-of-Service issues.

3) Major architectural re-design -- reconsideration of the general design of the Mempool to improve
how transactions are selected for mining, relay, and eviction to fully address issues like pinning
and improve rationality.


