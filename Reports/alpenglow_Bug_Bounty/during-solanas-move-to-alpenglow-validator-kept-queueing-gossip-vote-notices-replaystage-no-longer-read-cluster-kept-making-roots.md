
# During Solana's move to Alpenglow, a validator kept queueing gossip vote notices that ReplayStage no longer read while the cluster kept making roots

By 0xarno

During Solana's move from TowerBFT to Alpenglow, a validator could keep queueing staked gossip vote notices into a channel that nothing read anymore. `ReplayStage` drained that channel only on the Tower path. Once Alpenglow turned on, that path stopped running, and the queued vote notices sat in memory until the process restarted. The cluster kept making roots. I reported this through the Anza Alpenglow bounty. Anza fixed the leftover Tower channels in [agave#14404](https://github.com/anza-xyz/agave/pull/14404), merged on 7 August 2026.

It is also a clean example of how a migration fails: the old producer stays alive, and the new consumer forgets the queue.

## The old producer

Alpenglow replaces TowerBFT voting with Votor. The new work is in the `votor` crate and the certificate path. The channel predates Alpenglow. `ClusterInfoVoteListener` sends `ReplayStage` a small tuple, `(vote pubkey, slot, hash)`, for each new staked gossip vote. Before Alpenglow, `ReplayStage` used those tuples to feed fork choice with votes seen on gossip but not yet replayed inside a block. In production, the sender sits on an unbounded channel.

## The Alpenglow branch did not drain the channel

`ReplayStage` picks its work from the migration status. On Tower, it drains the gossip vote-hash receiver, records the votes for fork choice, and discards whatever is left. When `migration_status.is_alpenglow_enabled()` is true, it takes the Alpenglow arm instead.

That arm does not drain the receiver, and the vote listener is not shut down at migration. The send in `process_last_vote_for_optimistic_confirmation` keeps firing, and the tuples sit in the channel buffer.

The send has no Alpenglow check. It fires when the vote came from gossip, is new, and the voter has stake:

```rust
if is_gossip_vote && is_new && stake > 0 {
    let _ = notifiers.gossip_verified_vote_hash_sender.send((
        *vote_pubkey,
        last_vote_slot,
        last_vote_hash,
    ));
}
```

The bank-notification path next to this send is already gated on migration status. The hash send is not.

## When the queue can grow

The growth window is the mixed migration: the `AlpenglowEnabled` phase, before the cluster reaches a full Alpenglow epoch.

In a full Alpenglow epoch, gossip drops `CrdsData::Vote` on the way in and on the way out. New remote Tower votes stop arriving.

Only a staked vote account can feed the channel. Its transaction must be signed by the epoch's authorized voter and carry a valid legacy Tower vote into gossip during that window. An unstaked vote account never reaches the send.

Holding the cluster on Tower does not extend the window. What sticks is the residue from the phase in between.

Votor does not read these Tower-era tuples. Leaving them in the channel does not change certificates, finality, or rewards.

Existing gossip limits still apply. A voter cannot invent unlimited distinct hashes for one slot, and old slots fall out of the range the listener accepts.

## What happened in the local cluster

I ran the migration on a small local cluster. Before the switch, the old path was clearing the queue. After Alpenglow turned on, a few vote notices were still in it. Ten seconds later they were still there. The cluster kept making roots.

A longer mixed window would have left more in the queue. I did not measure memory over a long migration, so I am not putting a size on the pile.

## The basket analogy

Picture an office that is moving. One clerk's job was to empty a basket. That clerk has gone to the new floor, and nobody told the slot in the wall. Account holders can still drop letters in. The letters land in the basket and stay there. The rest of the office keeps working. When the move is finished they close the slot, but nobody walks back to empty the basket. The letters sit until the office restarts and the basket is thrown out.

## Impact

The leftover notices sit in validator memory until the process restarts. New ones arrive only during the mixed window, and only from a staked voter. Certificates, finality, and rewards stay as they were. The cluster keeps making roots. It is a small retention bug.

## The fix

The pull request does both: [agave#14404](https://github.com/anza-xyz/agave/pull/14404) stops the send once Alpenglow is enabled and drains the gossip vote-hash receiver inside the Alpenglow arm of `ReplayStage`. It also clears the other leftover Tower channels in that arm.
