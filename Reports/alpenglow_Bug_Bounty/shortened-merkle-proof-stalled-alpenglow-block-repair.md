# A shortened Merkle proof could stall Alpenglow block repair

By 0xarno

I found a proof-validation gap in Alpenglow block-ID repair. A `FecSetRoot` response could contain a shortened Merkle proof built from an internal tree node. The verifier could still accept it because it checked the proof's shape and final root, but not the depth required by the committed FEC-set tree.

The result is an availability vulnerability. A malicious repair peer can make a victim key follow-up shred requests to the wrong root for one block. Honest shreds for the real FEC-set leaf then fail verification, and the victim does not complete that block through block-ID repair during that repair episode. No wrong block is accepted. Cluster certificates continue under the honest supermajority.

## The missing depth check

Alpenglow uses `BlockIdRepairType::FecSetRoot` to help a node recover the FEC-set roots that belong to a block. The response carries a root and a Merkle proof. The receiving node's response check rejects an empty proof and passes the remaining bytes to the generic Merkle verifier.

That verifier checks that the proof is made of complete entries, folds the supplied siblings into a hash, requires the final tree index to be zero, and compares the result with the requested block ID. It does not know how many leaves the tree was meant to contain. A proof with fewer levels can therefore be valid for the same block ID if its claimed leaf is an internal node from the original tree.

The neighboring `ParentAndFecSetCount` repair path already has the missing information. It carries the FEC-set count and checks the exact proof length derived from that count. The `FecSetRoot` request contains only the slot, block ID, and FEC-set index. By the time the request reaches the repair handler, the FEC-set count is no longer available for an equivalent check. Honest servers still send the full proof, so the gap is in verification.

## How a shortened proof passes

The simplest case uses FEC-set index zero. Let `R0` and `R1` be neighboring FEC-set roots, and let `A` be their parent in the Merkle tree. A malicious peer can claim `A` as the FEC-set root and send only the proof suffix above `A`.

The verifier starts with the claimed root `A`, folds the remaining sibling nodes, reaches the block ID, and ends at index zero. It has no committed tree depth with which to reject the shorter path. The node has accepted a genuine internal hash as though it were the requested leaf.

The missing depth check is a general `FecSetRoot` validation gap, rather than a rule for index zero only. A one-set tree is an incidental exception because shortening its proof would leave an empty proof, which the handler rejects. That does not repair the general validation rule.

## The repair cascade

Once the response is accepted, the metadata request is consumed and the victim asks for shreds under the claimed root. Those requests are keyed to `A`, while honest shreds are rooted at `R0`. Their `ShredForBlockId` checks fail, so the victim cannot assemble the block from the honest responses.

The failed requests are retried with the same poisoned root. The repair state also keeps the block marked as requested, which suppresses a fresh block fetch. The victim remains stuck on that block for the lifetime of the repair episode.

This does not make the victim accept a different block. The block ID came from consensus, and the malicious root is an internal node from that block's own tree. The failure is that the victim cannot finish recovering the block it already expects.

## Why the issue is Alpenglow-specific

Block-ID repair is used for Alpenglow slots. That makes the proof gap relevant during the Alpenglow path even though the underlying Merkle verifier is shared. The attack needs a repair peer that can answer the victim's `FecSetRoot` request with the shortened response.

The peer does not need to change consensus data. It only needs to win the repair interaction for the target block. In a production cluster, peer selection makes the hit rate depend on the attacker's presence in the victim's repair sample. 

## Impact and recovery

Under sustained attack, a Byzantine repair peer can keep a victim from completing selected blocks through block-ID repair. If the missing block is needed for a bank switch or a later Alpenglow action, replay waits for the missing data. The cluster can still produce certificates and make progress under the honest supermajority.

The poison is held in process state. Restarting clears the repair state, but a malicious peer can answer the next request with the same shortened proof and create the stall again. If the attack stops, a restart or a newly selected honest peer can supply a full-depth proof and let repair recover.

I reproduced the stall in a local Alpenglow cluster. The leader continued producing blocks while the victim stopped completing the affected block through block-ID repair. Restarting the victim while the malicious repair behavior remained enabled produced the same stall again.

## The fix

The landed fix is [agave#14359](https://github.com/anza-xyz/agave/pull/14359). It carries the authenticated FEC-set count into the local `FecSetRoot` request state, validates the index and count, and requires the exact Merkle proof length for every FEC-set index, as the `ParentAndFecSetCount` path already does. 

**Disclaimer:** Two other reports describing the same issue landed at the same time. The team made an exception and decided to reward all three reports.


