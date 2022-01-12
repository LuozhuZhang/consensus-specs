# Optimistic Sync

## Introduction

In order to provide a syncing execution engine with a partial view of the head
of the chain, it may be desirable for a consensus engine to import beacon
blocks without verifying the execution payloads. This partial sync is called an
*optimistic sync*.

## Constants

|Name|Value|Unit
|---|---|---|
|`SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY`| `128` | slots

*Note: the `SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY` must be user-configurable. See
[Fork Choice Poisoning](#fork-choice-poisoning).*

## Helpers

Let `head_block: BeaconBlock` be the result of calling of the fork choice
algorithm at the time of block production.

Let `optimistic_roots: Set[Root]` be the set of `hash_tree_root(block)` for all
optimistically imported blocks which have yet to receive an `INVALID` or
`VALID` designation from an execution engine.

Let `optimistic_roots: Set[Root]` be the set of `hash_tree_root(block)` for all
optimistically imported blocks which have only received a `SYNCING` designation
from an execution engine (i.e., they are not known to be `INVALID` or `VALID`).

```python
@dataclass
class Store(object):
    optimistic_roots: Set[Root]
    head_block_root: Root
    blocks: Dict[Root, BeaconBlock]
    block_states: Dict[Root, BeaconState]
```

```python
def is_optimistic(store: Store, block: BeaconBlock) -> bool:
    return hash_tree_root(block) in store.optimistic_roots
```

```python
def latest_verified_ancestor(store: Store, block: BeaconBlock) -> BeaconBlock:
    while True:
        if not is_optimistic(store, block) or block.parent_root == Root():
            return block
        block = get_block(block.parent_root)
```

```python
def is_execution_block(block: BeaconBlock) -> BeaconBlock:
    return block.body.execution_payload != ExecutionPayload()
```

```python
def should_optimistically_import_block(store: Store, current_slot: Slot, block: BeaconBlock) -> bool:
	justified_root = store.block_states[store.head_block_root].current_justified_checkpoint.root
	justifed_is_verified = is_execution_block(store.blocks[justified_root])
    block_is_deep = block.slot + SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY <= current_slot
    return justified_is_verified or block_is_deep
```

Let only a node which returns `is_optimistic(head) is True` be an *optimistic
node*. Let only a validator on an optimistic node be an *optimistic validator*.

When this specification only defines behaviour for an optimistic
node/validator, but *not* for the non-optimistic case, assume default
behaviours without regard for optimistic sync.

## Mechanisms

### When to optimistically import blocks

A block MAY be optimistically imported when either of the following
conditions are met:

1. The justified checkpoint has execution enabled. I.e.,
   `is_execution_block(get_block(get_state(head_block).current_justified_checkpoint.root))`
1. The current slot (as per the system clock) is at least `SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY` ahead of
   the slot of the block being imported. I.e., `should_optimistically_import_block(current_slot, block) == True`.

*See [Fork Choice Poisoning](#fork-choice-poisoning) for the motivations behind
these conditions.*

### How to optimistically import blocks

To optimistically import a block:

- The [`execute_payload`](../specs/merge/beacon-chain.md#execute_payload) function MUST return `True` if the execution
	engine returns `SYNCING` or `VALID`. An `INVALID` response MUST return `False`.
- The [`validate_merge_block`](../specs/merge/fork-choice.md#validate_merge_block) function MUST NOT raise an assertion if both the
`pow_block` and `pow_parent` are unknown to the execution engine.
- The parent of the block MUST NOT have an INVALID execution payload.

In addition to this change to validation, the consensus engine MUST be able to
ascertain, after import, which blocks returned `SYNCING` and which returned
`VALID`.

Optimistically imported blocks MUST pass all verifications included in
`process_block` (withstanding the modifications to `execute_payload`).

A consensus engine MUST be able to retrospectively (i.e., after import) modify
the status of `SYNCING` blocks to be either `VALID` or `INVALID` based upon responses
from an execution engine. I.e., perform the following transitions:

- `SYNCING` -> `VALID`
- `SYNCING` -> `INVALID`

When a block transitions from `SYNCING` -> `VALID`, all *ancestors* of the
block MUST also transition from `SYNCING` -> `VALID`. Such a block is no longer
considered "optimistically imported".

When a block transitions from `SYNCING` -> `INVALID`, all *descendants* of the
block MUST also transition from `SYNCING` -> `INVALID`.

When a block transitions from the `SYNCING` state it is removed from the set of
`store.optimistic_roots`.

### Execution Engine Errors

When an execution engine returns an error or fails to respond to a payload
validity request for some block, a consensus engine:

- MUST NOT optimistically import the block.
- MUST NOT apply the block to the fork choice store.
- MAY queue the block for later processing.

### Assumptions about Execution Engine Behaviour

This specification assumes execution engines will only return `SYNCING` when
there is insufficient information available to make a `VALID` or `INVALID`
determination on the given `ExecutionPayload` (e.g., the parent payload is
unknown). Specifically, `SYNCING` responses should be fork-specific, in that
the search for a block on one chain MUST NOT trigger a `SYNCING` response for
another chain.

### Re-Orgs

The consensus engine MUST support any chain reorganisation which does *not*
affect the justified checkpoint. The consensus engine MAY support re-orgs
beyond the justified checkpoint.

If the justified checkpoint transitions from `SYNCING` -> `INVALID`, a
consensus engine MAY choose to alert the user and force the application to
exit.

## Fork Choice

Consensus engines MUST support removing blocks from fork choice that transition
from `SYNCING` to `INVALID`. Specifically, a block deemed `INVALID` at any
point MUST NOT be included in the canonical chain and the weights from those
`INVALID` blocks MUST NOT be applied to any `VALID` or `SYNCING` ancestors.

### Fork Choice Poisoning

During the merge transition it is possible for an attacker to craft a
`BeaconBlock` with an execution payload that references an
eternally-unavailable `body.execution_payload.parent_hash` (i.e., the parent
hash is random bytes). In rare circumstances, it is possible that an attacker
can build atop such a block to trigger justification. If an optimistic node
imports this malicious chain, that node will have a "poisoned" fork choice
store, such that the node is unable to produce a block that descends from the
head (due to the invalid chain of payloads) and the node is unable to produce a
block that forks around the head (due to the justification of the malicious
chain).

If an honest chain exists which justifies a higher epoch than the malicious
chain, that chain will take precedence and revive any poisoned store.
Therefore, the poisoning attack is temporary if >= 2/3rds of the network is
honest and non-faulty.

The `SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY` parameter assumes that the network
will justify a honest chain within some number of slots. With this assumption,
it is acceptable to optimistically import transition blocks during the sync
process. Since there is an assumption that an honest chain with a higher
justified checkpoint exists, any fork choice poisoning will be short-lived and
resolved before that node is required to produce a block.

However, the assumption that the honest, canonical chain will always justify
within `SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY` slots is dubious. Therefore,
clients MUST provide the following command line flag to assist with manual
disaster recovery:

- `--safe-slots-to-import-optimistically`: modifies the
	`SAFE_SLOTS_TO_IMPORT_OPTIMISTICALLY`.

## Checkpoint Sync (Weak Subjectivity Sync)

A consensus engine MAY assume that the `ExecutionPayload` of a block used as an
anchor for checkpoint sync is `VALID` without necessarily providing that
payload to an execution engine.

## Validator assignments

An optimistic node is *not* a full node. It is unable to produce blocks, since
an execution engine cannot produce a payload upon an unknown parent. It cannot
faithfully attest to the head block of the chain, since it has not fully
verified that block.

### Block Production

An optimistic validator MUST NOT produce a block (i.e., sign across the
`DOMAIN_BEACON_PROPOSER` domain).

### Attesting

An optimistic validator MUST NOT participate in attestation (i.e., sign across the
`DOMAIN_BEACON_ATTESTER`, `DOMAIN_SELECTION_PROOF` or
`DOMAIN_AGGREGATE_AND_PROOF` domains).

### Participating in Sync Committees

An optimistic validator MUST NOT participate in sync committees (i.e., sign across the
`DOMAIN_SYNC_COMMITTEE`, `DOMAIN_SYNC_COMMITTEE_SELECTION_PROOF` or
`DOMAIN_CONTRIBUTION_AND_PROOF` domains).

## Ethereum Beacon APIs

Consensus engines which provide an implementation of the [Ethereum Beacon
APIs](https://github.com/ethereum/beacon-APIs) must take care to avoid
presenting optimistic blocks as fully-verified blocks.

### Helpers

Let the following response types be defined as any response with the
corresponding HTTP status code:

- "Success" Response: Status Codes 200-299.
- "Not Found" Response: Status Code 404.
- "Syncing" Response: Status Code 503.

### Requests for Optimistic Blocks

When information about an optimistic block is requested, the consensus engine:

- MUST NOT respond with success.
- MAY respond with not found.
- MAY respond with syncing.

### Requests for an Optimistic Head

When `is_optimistic(head) is True`, the consensus engine:

- MUST NOT return an optimistic `head`.
- MAY substitute the head block with `latest_valid_ancestor(block)`.
- MAY return syncing.

### Requests to Validators Endpoints

When `is_optimistic(head) is True`, the consensus engine MUST return syncing to
all endpoints which match the following pattern:

- `eth/*/validator/*`