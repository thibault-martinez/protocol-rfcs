+ Feature name: `white-flag`
+ Start date: 2020-03-06
+ RFC PR: [iotaledger/protocol-rfcs#0005](https://github.com/iotaledger/protocol-rfcs/pull/5)

# Summary

This RFC is part of a set of protocol changes, [Chrysalis](https://roadmap.iota.org/chrysalis), aiming at improving the
network before [Coordicide](https://coordicide.iota.org/) is complete.

The feature presented in this RFC, White Flag, allows milestones to confirm conflicting bundles by enforcing
deterministic ordering of the Tangle and applying only the first bundle(s) that does not violate the ledger state.

The content of this RFC is based on [Conflict white flag: Mitigate conflict spamming by ignoring conflicts](https://iota.cafe/t/conflict-white-flag-mitigate-conflict-spamming-by-ignoring-conflicts/233).

# Motivation

<!-- TODO -->

The main motivations:

- Defend against censorship attacks - Conflicts will no longer block bundles from being approved.
- Make reattachments unnecessary - As long as the network is not saturated, theoretically all bundles should be
approved. And no bundle will be left behind.
- Increase TPS - Due to easy node tip selection the network throughput should increase.
- Increase CTPS - Due to the above, increase in TPS and no left-behinds, we expect CTPS to increase as well.

# Detailed design

<!-- TODO -->

Definitions:
- confirm
- ignored
- applied
- approve
- reference / indirectly

Let's define a conflicting bundle as a bundle that leads to a negative balance on an address if applied to the current
ledger state.

In case of conflicting bundles with White Flag, nodes will apply only one bundle to the ledger state and ignore all the
others. For this to work, nodes need to be sure they are all applying the same bundle; hence, the need for a
deterministic ordering of the Tangle.

First, this RFC propose a deterministic ordering of the Tangle, then it explain which bundle is selected in case of
conflict.

## Deterministically ordering the Tangle

When a new milestone is broadcasted to the network, nodes will need to order the set of bundles it confirms that are
was not previously confirmed by any other milestone.

A subset of the Tangle can be ordered depending on many of its properties (e.g. alphanumeric sort of the bundle hashes);
however, to compute the ledger state, a graph traversal has to be done so we can use it to order the bundles in a
deterministic order with no extra overhead.

This ordering is then defined as the [topological ordering](https://en.wikipedia.org/wiki/Topological_sorting) generated
by a post-order [Depth-First Search (DFS)](https://en.wikipedia.org/wiki/Depth-first_search) starting from a milestone
and by going first through trunk bundles, then branch bundles and finally root bundles. Since only a subset of bundles
is considered, the stopping condition of the DFS is reaching bundles that are already confirmed by another milestone.

![][Tangle]

In this example, the topological ordering of the set of bundles confirmed by milestone `V` (purple set) is then
`{D, G, J, M, L, I, K, O, N, S, R, V}`.

## Applying first bundle(s) that does not violate the ledger state

If a conflict is occurring in the set of bundles confirmed by a milestone, nodes have to apply the first (with regards
to the order previously proposed) of the conflicting bundles to the ledger and ignore all the others.

Once a bundle is marked as ignored, this is final and can't be changed by a later milestone.

Since the ledger state is maintained from one milestone to another, a bundle conflicting with another bundle already
confirmed by a previous milestone would also obviously be ignored.

![][Tangle-conflict]

In this example, bundles `G` and `O` both confirmed by milestone `V` are conflicting. Since in the topologically ordered
set `{D, G, J, M, L, I, K, O, N, S, R, V}`, `G` appears before `O`, `G` is applied to the ledger state and `O` is
ignored.

## Pseudo-code

The following algorithm describes the process of updating the ledger state which is usually triggered by the arrival of
a new milestone confirming many new bundles.

Even though the tangle is a graph made of transactions, for the sake of this pseudo-code it is considered as a graph of
bundles. For this reason, when we talk about the `trunk` or the `branch` of a bundle, we are actually referring to the
`trunk` and `branch` of the last transaction of the bundle.

There are multiple topological orders for each DAG, and in order to avoid conflicting
ledger states it is requisite that all nodes will apply bundles in the exact same order

To avoid ambiguities we define the order which will be followed,
upon the arrival of a new milestone, we will span the cone of transactions
in a DFS manner, we will visit bundles from root to trunk and then to branch,
we will then apply the bundles in an order reversed from the order in which we visited 
the bundles when we DFS-ed

```
// ledger is the global ledger state
// newMilestone is a bundle containing the new milestone transactions
UpdateLedgerState(ledger, new_milestone) {

    let seen_bundles = an empty stack
    let post_order_seen_bundles = an empty stack

    seen_bundles.push(new_milestone)
    post_order_seen_bundles.push(newMilestone)
    mark new_milestone as visited

    while (seen_bundles is not empty) {
        let curr_bundle = seen_bundles.pop()

        trunk = the next bundle pointed by the trunk of the last transaction in curr_bundle
        if (trunk is not visited and not confirmed by any milestone previous to newMilestone) {
            seen_bundles.push(trunk)
            post_order_seen_bundles.push(trunk)
            mark trunk as visited
        }

        branch = the next bundle pointed by the branch of any transaction in curr_bundle
        if (branch is not visited and not confirmed by any miilestone previous to newMilestone) {
            seen_bundles.push(branch)
            post_order_seen_bundles.push(branch)
            mark branch as visited
        }
    }
    
    while (post_order_seen_bundles is not empty){
    
            let curr_bundle = post_order_seen_bundles.pop()
    
            if (!ledger.conflicts(curr_bundle)) {
                ledger.apply(curr_bundle)
            }

    }
}
```

# Drawbacks

<!-- TODO -->

- If we ever want to supply a proof that a value transfer is valid and approved, we can't do so by merely supplying the
path from the bundle to the approving milestone as before. A proof will require to have all the transactions that are in
the past cone of the approving milestone to the genesis. However, up until now we never required to give such proofs.
If we want to do easy proofs we can create merkle trees from approved bundle hashes and add them to milestones.
- Everything that is seen is part of the Tangle, including double-spend attempts. Meaning we define that possibly
malicious data will be saved as part of the consensus set of the Tangle.

# Rationale and alternatives

A transaction that chooses tips to approve, first performs tip selection to get these tips,
then in order for that transaction to be valid it must not approve two different tips which 
conflicts each other, asserting that two tips are not conflicting requires validating the ledger state
upon applying each of the transactions in the cone spanned by the selected tips, which is expensive
since tip selection is done also by the coordinator it affects the CTPS, discarding this check
by allowing conflicting bundles to coexist should result in CTPS increase

# Unresolved questions

<!-- TODO -->

Since nodes will try to sort out conflicts themselves perhaps it will be wise to add more protection against forks.
In a separate RFC we can maybe define additional data that can be added to milestones to prevent that.

[Tangle]: img/tangle.svg
[Tangle-conflict]: img/tangle-conflict.svg
