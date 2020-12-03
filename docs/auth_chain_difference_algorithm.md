# Auth Chain Difference Algorithm

The auth chain difference algorithm is used by V2 state resolution, where a
naive implementation can be a significant source of CPU and DB usage.

The auth chain difference of a set of state sets is the union minus the
intersection of the sets of auth chains corresponding to the state sets, i.e an
event is in the auth chain difference if it is reachable by walking the auth
event graph from at least one of the state sets but not from *all* of the state
sets.

## Chain Cover Index

Synapse computes auth chain differences by pre-computing a "chain cover" index
for the auth chain in a room, allowing efficient reachability queries like "is
event A in the auth chain of event B". This is done by assigning every event a
*chain ID* and *sequence number* and having map of *links* such that A is
reachable by B (i.e. `A` is in the auth chain of `B`) if and only if either:

1. A and B have the same chain ID and `A`'s sequence number is less than `B`'s
   sequence number; or
2. there is a link `L` between `B`'s chain ID and `A`'s chain ID such that
   `L.seq_no` <= `B.seq_no` and `A.seq_no` <= `L.seq_no`.

There are actually two variants, one where we store links from each chain to
every other reachable chain (the transitive closure of the links graph), and one
where we remove redundant links (the transitive reduction of the links graph)
e.g. if we have chains `C3 -> C2 -> C1` then the link `C3 -> C1` would not be
stored. Synapse uses the former variant so that it doesn't need to recurse to
test reachability between chains.

### Example

An example auth graph would look like the following, where chains have been
formed based on type/state_key and are denoted by colour and are labelled with
`(chain ID, sequence number)`. Links are denoted by the arrows (links in grey
are those that would be remove in the second variant described above).

![Example](auth_chain_diff.dot.png)

Note that we don't add links between every event and its auth events, as that is
redundant (under both variants), e.g. all events point to the create event, but
each chain only needs the one link from it's base to the create event.

## Using the Index

This index can be used to calculate the auth chain difference of the state sets
by looking at the chain ID and sequence numbers reachable from each state set:

1. For every state set lookup the chain ID/sequence numbers of each state event
2. Use the index to find all chains and the maximum sequence number reachable
   from each state set.
3. The auth chain difference is then all events in each chain that have sequence
   numbers between the maximum sequence number reachable from *any* state set and
   the minimum reachable by *all* state sets (if any).

### Worked Examplee

For example, if we take the above graph and try and get the difference between
state sets consisting of:

1. `S1`: Alice's invite `(4,1)` and Bob's second join `(2,2)`; and
2. `S2`: Alice's second join `(4,3)` and Bob's first join `(2,1)`.

Using the index we see that the following chains are reachable from each:
1. `S1`: `(1,1)`, `(2,2)`, `(3,1)` & `(4,1)`
2. `S2`: `(1,1)`, `(2,1)`, `(3,2)` & `(4,3)`

And so, for each the ranges that are in the auth chain difference:
1. Chain 1: None, (since everything can reach the create event).
2. Chain 2: The range `(1, 2]` (i.e. just `2`), as `1` is reachable by all state
   sets and the maximum reachable is `2` (corresponding to Bob's second join).
3. Chain 3: Similarly the range `(1, 2]` (corresponding to the second power
   level).
4. Chain 4: The range `(1, 3]` (corresponding to both of Alice's joins).

So the final result is: Bob's second join, the second power level and both of
Alice's joins.