# Phase 0 -- Beacon Chain Fork Choice

## Introduction

This document is the beacon chain fork choice spec, part of Phase 0. It assumes
the [beacon chain state transition function spec](./beacon-chain.md).

## Protocols

### `ExecutionEngine`

*Note*: The `notify_forkchoice_updated` function is added to the
`ExecutionEngine` protocol to signal the fork choice updates.

The body of this function is implementation dependent. The Engine API may be
used to implement it with an external execution engine.

#### `notify_forkchoice_updated`

This function performs three actions *atomically*:

- Re-organizes the execution payload chain and corresponding state to make
  `head_block_hash` the head.
- Updates safe block hash with the value provided by `safe_block_hash`
  parameter.
- Applies finality to the execution state: it irreversibly persists the chain of
  all execution payloads and corresponding state, up to and including
  `finalized_block_hash`.

Additionally, if `payload_attributes` is provided, this function sets in motion
a payload build process on top of `head_block_hash` and returns an identifier of
initiated process.

```python
def notify_forkchoice_updated(
    self: ExecutionEngine,
    head_block_hash: Hash32,
    safe_block_hash: Hash32,
    finalized_block_hash: Hash32,
    payload_attributes: Optional[PayloadAttributes],
) -> Optional[PayloadId]: ...
```

*Note*: The `(head_block_hash, finalized_block_hash)` values of the
`notify_forkchoice_updated` function call maps on the `POS_FORKCHOICE_UPDATED`
event defined in the
[EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#definitions). As per
EIP-3675, before a post-transition block is finalized,
`notify_forkchoice_updated` MUST be called with
`finalized_block_hash = Hash32()`.

*Note*: Client software MUST NOT call this function until the transition
conditions are met on the PoW network, i.e. there exists a block for which
`is_valid_terminal_pow_block` function returns `True`.

*Note*: Client software MUST call this function to initiate the payload build
process to produce the merge transition block; the `head_block_hash` parameter
MUST be set to the hash of a terminal PoW block in this case.

##### `safe_block_hash`

The `safe_block_hash` parameter MUST be set to return value of
[`get_safe_execution_block_hash(store: Store)`](../../fork_choice/safe-block.md#get_safe_execution_block_hash)
function.

##### `should_override_forkchoice_update`

If proposer boost re-orgs are implemented and enabled (see `get_proposer_head`)
then additional care must be taken to ensure that the proposer is able to build
an execution payload.

If a beacon node knows it will propose the next block then it SHOULD NOT call
`notify_forkchoice_updated` if it detects the current head to be weak and
potentially capable of being re-orged. Complete information for evaluating
`get_proposer_head` _will not_ be available immediately after the receipt of a
new block, so an approximation of those conditions should be used when deciding
whether to send or suppress a fork choice notification. The exact conditions
used may be implementation-specific, a suggested implementation is below.

Let `validator_is_connected(validator_index: ValidatorIndex) -> bool` be a
function that indicates whether the validator with `validator_index` is
connected to the node (e.g. has sent an unexpired proposer preparation message).

```python
def should_override_forkchoice_update(store: Store, head_root: Root) -> bool:
    head_block = store.blocks[head_root]
    parent_root = head_block.parent_root
    parent_block = store.blocks[parent_root]
    current_slot = get_current_slot(store)
    proposal_slot = head_block.slot + Slot(1)

    # Only re-org the head_block block if it arrived later than the attestation deadline.
    head_late = is_head_late(store, head_root)

    # Shuffling stable.
    shuffling_stable = is_shuffling_stable(proposal_slot)

    # FFG information of the new head_block will be competitive with the current head.
    ffg_competitive = is_ffg_competitive(store, head_root, parent_root)

    # Do not re-org if the chain is not finalizing with acceptable frequency.
    finalization_ok = is_finalization_ok(store, proposal_slot)

    # Only suppress the fork choice update if we are confident that we will propose the next block.
    parent_state_advanced = store.block_states[parent_root].copy()
    process_slots(parent_state_advanced, proposal_slot)
    proposer_index = get_beacon_proposer_index(parent_state_advanced)
    proposing_reorg_slot = validator_is_connected(proposer_index)

    # Single slot re-org.
    parent_slot_ok = parent_block.slot + 1 == head_block.slot
    proposing_on_time = is_proposing_on_time(store)

    # Note that this condition is different from `get_proposer_head`
    current_time_ok = head_block.slot == current_slot or (
        proposal_slot == current_slot and proposing_on_time
    )
    single_slot_reorg = parent_slot_ok and current_time_ok

    # Check the head weight only if the attestations from the head slot have already been applied.
    # Implementations may want to do this in different ways, e.g. by advancing
    # `store.time` early, or by counting queued attestations during the head block's slot.
    if current_slot > head_block.slot:
        head_weak = is_head_weak(store, head_root)
        parent_strong = is_parent_strong(store, parent_root)
    else:
        head_weak = True
        parent_strong = True

    return all(
        [
            head_late,
            shuffling_stable,
            ffg_competitive,
            finalization_ok,
            proposing_reorg_slot,
            single_slot_reorg,
            head_weak,
            parent_strong,
        ]
    )
```

*Note*: The ordering of conditions is a suggestion only. Implementations are
free to optimize by re-ordering the conditions from least to most expensive and
by returning early if any of the early conditions are `False`.

In case `should_override_forkchoice_update` returns `True`, a node SHOULD
instead call `notify_forkchoice_updated` with parameters appropriate for
building upon the parent block. Care must be taken to compute the correct
`payload_attributes`, as they may change depending on the slot of the block to
be proposed (due to withdrawals).

If `should_override_forkchoice_update` returns `True` but `get_proposer_head`
later chooses the canonical head rather than its parent, then this is a
misprediction that will cause the node to construct a payload with less notice.
The result of `get_proposer_head` MUST be preferred over the result of
`should_override_forkchoice_update` (when proposer reorgs are enabled).

## Fork choice

The head block root associated with a `store` is defined as `get_head(store)`.
At genesis, let `store = get_forkchoice_store(genesis_state, genesis_block)` and
update `store` by running:

- `on_tick(store, time)` whenever `time > store.time` where `time` is the
  current Unix time
- `on_block(store, block)` whenever a block `block: SignedBeaconBlock` is
  received
- `on_attestation(store, attestation)` whenever an attestation `attestation` is
  received
- `on_attester_slashing(store, attester_slashing)` whenever an attester slashing
  `attester_slashing` is received

Any of the above handlers that trigger an unhandled exception (e.g. a failed
assert or an out-of-range list access) are considered invalid. Invalid calls to
handlers must not modify `store`.

*Notes*:

1. **Leap seconds**: Slots will last `SECONDS_PER_SLOT + 1` or
   `SECONDS_PER_SLOT - 1` seconds around leap seconds. This is automatically
   handled by [UNIX time](https://en.wikipedia.org/wiki/Unix_time).
2. **Honest clocks**: Honest nodes are assumed to have clocks synchronized
   within `SECONDS_PER_SLOT` seconds of each other.
3. **Eth1 data**: The large `ETH1_FOLLOW_DISTANCE` specified in the
   [honest validator document](./validator.md) should ensure that
   `state.latest_eth1_data` of the canonical beacon chain remains consistent
   with the canonical Ethereum proof-of-work chain. If not, emergency manual
   intervention will be required.
4. **Manual forks**: Manual forks may arbitrarily change the fork choice rule
   but are expected to be enacted at epoch transitions, with the fork details
   reflected in `state.fork`.
5. **Implementation**: The implementation found in this specification is
   constructed for ease of understanding rather than for optimization in
   computation, space, or any other resource. A number of optimized alternatives
   can be found [here](https://github.com/protolambda/lmd-ghost).

### Custom types

| Name        | SSZ equivalent | Description                              |
| ----------- | -------------- | ---------------------------------------- |
| `PayloadId` | `Bytes8`       | Identifier of a payload building process |

### Constant

| Name                 | Value       |
| -------------------- | ----------- |
| `INTERVALS_PER_SLOT` | `uint64(3)` |

### Configuration

| Name                                  | Value         |
| ------------------------------------- | ------------- |
| `PROPOSER_SCORE_BOOST`                | `uint64(40)`  |
| `REORG_HEAD_WEIGHT_THRESHOLD`         | `uint64(20)`  |
| `REORG_PARENT_WEIGHT_THRESHOLD`       | `uint64(160)` |
| `REORG_MAX_EPOCHS_SINCE_FINALIZATION` | `Epoch(2)`    |

- The proposer score boost and re-org weight threshold are percentage values
  that are measured with respect to the weight of a single committee. See
  `calculate_committee_fraction`.

### Helpers

#### `PayloadAttributes`

Used to signal to initiate the payload build process via
`notify_forkchoice_updated`.

```python
@dataclass
class PayloadAttributes(object):
    timestamp: uint64
    prev_randao: Bytes32
    suggested_fee_recipient: ExecutionAddress
    # [New in Capella]
    withdrawals: Sequence[Withdrawal]
    # [New in Deneb:EIP4788]
    parent_beacon_block_root: Root
```

#### `PowBlock`

```python
class PowBlock(Container):
    block_hash: Hash32
    parent_hash: Hash32
    total_difficulty: uint256
```

#### `is_data_available`

*[New in Deneb:EIP4844]*

The implementation of `is_data_available` will become more sophisticated during
later scaling upgrades. Initially, verification requires every verifying actor
to retrieve all matching `Blob`s and `KZGProof`s, and validate them with
`verify_blob_kzg_proof_batch`.

The block MUST NOT be considered valid until all valid `Blob`s have been
downloaded. Blocks that have been previously validated as available SHOULD be
considered available even if the associated `Blob`s have subsequently been
pruned.

*Note*: Extraneous or invalid Blobs (in addition to KZG expected/referenced
valid blobs) received on the p2p network MUST NOT invalidate a block that is
otherwise valid and available.

```python
def is_data_available(
    beacon_block_root: Root, blob_kzg_commitments: Sequence[KZGCommitment]
) -> bool:
    # `retrieve_blobs_and_proofs` is implementation and context dependent
    # It returns all the blobs for the given block root, and raises an exception if not available
    # Note: the p2p network does not guarantee sidecar retrieval outside of
    # `MIN_EPOCHS_FOR_BLOB_SIDECARS_REQUESTS`
    blobs, proofs = retrieve_blobs_and_proofs(beacon_block_root)

    return verify_blob_kzg_proof_batch(blobs, blob_kzg_commitments, proofs)
```

#### `get_pow_block`

Let `get_pow_block(block_hash: Hash32) -> Optional[PowBlock]` be the function
that given the hash of the PoW block returns its data. It may result in `None`
if the requested block is not yet available.

*Note*: The `eth_getBlockByHash` JSON-RPC method may be used to pull this
information from an execution client.

#### `is_valid_terminal_pow_block`

Used by fork-choice handler, `on_block`.

```python
def is_valid_terminal_pow_block(block: PowBlock, parent: PowBlock) -> bool:
    is_total_difficulty_reached = block.total_difficulty >= TERMINAL_TOTAL_DIFFICULTY
    is_parent_total_difficulty_valid = parent.total_difficulty < TERMINAL_TOTAL_DIFFICULTY
    return is_total_difficulty_reached and is_parent_total_difficulty_valid
```

#### `validate_merge_block`

```python
def validate_merge_block(block: BeaconBlock) -> None:
    """
    Check the parent PoW block of execution payload is a valid terminal PoW block.

    Note: Unavailable PoW block(s) may later become available,
    and a client software MAY delay a call to ``validate_merge_block``
    until the PoW block(s) become available.
    """
    if TERMINAL_BLOCK_HASH != Hash32():
        # If `TERMINAL_BLOCK_HASH` is used as an override, the activation epoch must be reached.
        assert compute_epoch_at_slot(block.slot) >= TERMINAL_BLOCK_HASH_ACTIVATION_EPOCH
        assert block.body.execution_payload.parent_hash == TERMINAL_BLOCK_HASH
        return

    pow_block = get_pow_block(block.body.execution_payload.parent_hash)
    # Check if `pow_block` is available
    assert pow_block is not None
    pow_parent = get_pow_block(pow_block.parent_hash)
    # Check if `pow_parent` is available
    assert pow_parent is not None
    # Check if `pow_block` is a valid terminal PoW block
    assert is_valid_terminal_pow_block(pow_block, pow_parent)
```

#### `LatestMessage`

```python
@dataclass(eq=True, frozen=True)
class LatestMessage(object):
    epoch: Epoch
    root: Root
```

#### `Store`

The `Store` is responsible for tracking information required for the fork choice
algorithm. The important fields being tracked are described below:

- `justified_checkpoint`: the justified checkpoint used as the starting point
  for the LMD GHOST fork choice algorithm.
- `finalized_checkpoint`: the highest known finalized checkpoint. The fork
  choice only considers blocks that are not conflicting with this checkpoint.
- `unrealized_justified_checkpoint` & `unrealized_finalized_checkpoint`: these
  track the highest justified & finalized checkpoints resp., without regard to
  whether on-chain ***realization*** has occurred, i.e. FFG processing of new
  attestations within the state transition function. This is an important
  distinction from `justified_checkpoint` & `finalized_checkpoint`, because they
  will only track the checkpoints that are realized on-chain. Note that on-chain
  processing of FFG information only happens at epoch boundaries.
- `unrealized_justifications`: stores a map of block root to the unrealized
  justified checkpoint observed in that block.

```python
@dataclass
class Store(object):
    time: uint64
    genesis_time: uint64
    justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    unrealized_justified_checkpoint: Checkpoint
    unrealized_finalized_checkpoint: Checkpoint
    proposer_boost_root: Root
    equivocating_indices: Set[ValidatorIndex]
    blocks: Dict[Root, BeaconBlock] = field(default_factory=dict)
    block_states: Dict[Root, BeaconState] = field(default_factory=dict)
    block_timeliness: Dict[Root, boolean] = field(default_factory=dict)
    checkpoint_states: Dict[Checkpoint, BeaconState] = field(default_factory=dict)
    latest_messages: Dict[ValidatorIndex, LatestMessage] = field(default_factory=dict)
    unrealized_justifications: Dict[Root, Checkpoint] = field(default_factory=dict)
```

#### `get_forkchoice_store`

The provided anchor-state will be regarded as a trusted state, to not roll back
beyond. This should be the genesis state for a full client.

*Note* With regards to fork choice, block headers are interchangeable with
blocks. The spec is likely to move to headers for reduced overhead in test
vectors and better encapsulation. Full implementations store blocks as part of
their database and will often use full blocks when dealing with production fork
choice.

```python
def get_forkchoice_store(anchor_state: BeaconState, anchor_block: BeaconBlock) -> Store:
    assert anchor_block.state_root == hash_tree_root(anchor_state)
    anchor_root = hash_tree_root(anchor_block)
    anchor_epoch = get_current_epoch(anchor_state)
    justified_checkpoint = Checkpoint(epoch=anchor_epoch, root=anchor_root)
    finalized_checkpoint = Checkpoint(epoch=anchor_epoch, root=anchor_root)
    proposer_boost_root = Root()
    return Store(
        time=uint64(anchor_state.genesis_time + SECONDS_PER_SLOT * anchor_state.slot),
        genesis_time=anchor_state.genesis_time,
        justified_checkpoint=justified_checkpoint,
        finalized_checkpoint=finalized_checkpoint,
        unrealized_justified_checkpoint=justified_checkpoint,
        unrealized_finalized_checkpoint=finalized_checkpoint,
        proposer_boost_root=proposer_boost_root,
        equivocating_indices=set(),
        blocks={anchor_root: copy(anchor_block)},
        block_states={anchor_root: copy(anchor_state)},
        checkpoint_states={justified_checkpoint: copy(anchor_state)},
        unrealized_justifications={anchor_root: justified_checkpoint},
    )
```

#### `get_slots_since_genesis`

```python
def get_slots_since_genesis(store: Store) -> int:
    return (store.time - store.genesis_time) // SECONDS_PER_SLOT
```

#### `get_current_slot`

```python
def get_current_slot(store: Store) -> Slot:
    return Slot(GENESIS_SLOT + get_slots_since_genesis(store))
```

#### `get_current_store_epoch`

```python
def get_current_store_epoch(store: Store) -> Epoch:
    return compute_epoch_at_slot(get_current_slot(store))
```

#### `compute_slots_since_epoch_start`

```python
def compute_slots_since_epoch_start(slot: Slot) -> int:
    return slot - compute_start_slot_at_epoch(compute_epoch_at_slot(slot))
```

#### `get_ancestor`

```python
def get_ancestor(store: Store, root: Root, slot: Slot) -> Root:
    block = store.blocks[root]
    if block.slot > slot:
        return get_ancestor(store, block.parent_root, slot)
    return root
```

#### `calculate_committee_fraction`

```python
def calculate_committee_fraction(state: BeaconState, committee_percent: uint64) -> Gwei:
    committee_weight = get_total_active_balance(state) // SLOTS_PER_EPOCH
    return Gwei((committee_weight * committee_percent) // 100)
```

#### `get_checkpoint_block`

```python
def get_checkpoint_block(store: Store, root: Root, epoch: Epoch) -> Root:
    """
    Compute the checkpoint block for epoch ``epoch`` in the chain of block ``root``
    """
    epoch_first_slot = compute_start_slot_at_epoch(epoch)
    return get_ancestor(store, root, epoch_first_slot)
```

#### `get_proposer_score`

```python
def get_proposer_score(store: Store) -> Gwei:
    justified_checkpoint_state = store.checkpoint_states[store.justified_checkpoint]
    committee_weight = get_total_active_balance(justified_checkpoint_state) // SLOTS_PER_EPOCH
    return (committee_weight * PROPOSER_SCORE_BOOST) // 100
```

#### `get_weight`

```python
def get_weight(store: Store, root: Root) -> Gwei:
    state = store.checkpoint_states[store.justified_checkpoint]
    unslashed_and_active_indices = [
        i
        for i in get_active_validator_indices(state, get_current_epoch(state))
        if not state.validators[i].slashed
    ]
    attestation_score = Gwei(
        sum(
            state.validators[i].effective_balance
            for i in unslashed_and_active_indices
            if (
                i in store.latest_messages
                and i not in store.equivocating_indices
                and get_ancestor(store, store.latest_messages[i].root, store.blocks[root].slot)
                == root
            )
        )
    )
    if store.proposer_boost_root == Root():
        # Return only attestation score if ``proposer_boost_root`` is not set
        return attestation_score

    # Calculate proposer score if ``proposer_boost_root`` is set
    proposer_score = Gwei(0)
    # Boost is applied if ``root`` is an ancestor of ``proposer_boost_root``
    if get_ancestor(store, store.proposer_boost_root, store.blocks[root].slot) == root:
        proposer_score = get_proposer_score(store)
    return attestation_score + proposer_score
```

#### `get_voting_source`

```python
def get_voting_source(store: Store, block_root: Root) -> Checkpoint:
    """
    Compute the voting source checkpoint in event that block with root ``block_root`` is the head block
    """
    block = store.blocks[block_root]
    current_epoch = get_current_store_epoch(store)
    block_epoch = compute_epoch_at_slot(block.slot)
    if current_epoch > block_epoch:
        # The block is from a prior epoch, the voting source will be pulled-up
        return store.unrealized_justifications[block_root]
    else:
        # The block is not from a prior epoch, therefore the voting source is not pulled up
        head_state = store.block_states[block_root]
        return head_state.current_justified_checkpoint
```

#### `filter_block_tree`

*Note*: External calls to `filter_block_tree` (i.e., any calls that are not made
by the recursive logic in this function) MUST set `block_root` to
`store.justified_checkpoint`.

```python
def filter_block_tree(store: Store, block_root: Root, blocks: Dict[Root, BeaconBlock]) -> bool:
    block = store.blocks[block_root]
    children = [
        root for root in store.blocks.keys() if store.blocks[root].parent_root == block_root
    ]

    # If any children branches contain expected finalized/justified checkpoints,
    # add to filtered block-tree and signal viability to parent.
    if any(children):
        filter_block_tree_result = [filter_block_tree(store, child, blocks) for child in children]
        if any(filter_block_tree_result):
            blocks[block_root] = block
            return True
        return False

    current_epoch = get_current_store_epoch(store)
    voting_source = get_voting_source(store, block_root)

    # The voting source should be either at the same height as the store's justified checkpoint or
    # not more than two epochs ago
    correct_justified = (
        store.justified_checkpoint.epoch == GENESIS_EPOCH
        or voting_source.epoch == store.justified_checkpoint.epoch
        or voting_source.epoch + 2 >= current_epoch
    )

    finalized_checkpoint_block = get_checkpoint_block(
        store,
        block_root,
        store.finalized_checkpoint.epoch,
    )

    correct_finalized = (
        store.finalized_checkpoint.epoch == GENESIS_EPOCH
        or store.finalized_checkpoint.root == finalized_checkpoint_block
    )

    # If expected finalized/justified, add to viable block-tree and signal viability to parent.
    if correct_justified and correct_finalized:
        blocks[block_root] = block
        return True

    # Otherwise, branch not viable
    return False
```

#### `get_filtered_block_tree`

```python
def get_filtered_block_tree(store: Store) -> Dict[Root, BeaconBlock]:
    """
    Retrieve a filtered block tree from ``store``, only returning branches
    whose leaf state's justified/finalized info agrees with that in ``store``.
    """
    base = store.justified_checkpoint.root
    blocks: Dict[Root, BeaconBlock] = {}
    filter_block_tree(store, base, blocks)
    return blocks
```

#### `get_head`

```python
def get_head(store: Store) -> Root:
    # Get filtered block tree that only includes viable branches
    blocks = get_filtered_block_tree(store)
    # Execute the LMD-GHOST fork choice
    head = store.justified_checkpoint.root
    while True:
        children = [root for root in blocks.keys() if blocks[root].parent_root == head]
        if len(children) == 0:
            return head
        # Sort by latest attesting balance with ties broken lexicographically
        # Ties broken by favoring block with lexicographically higher root
        head = max(children, key=lambda root: (get_weight(store, root), root))
```

#### `update_checkpoints`

```python
def update_checkpoints(
    store: Store, justified_checkpoint: Checkpoint, finalized_checkpoint: Checkpoint
) -> None:
    """
    Update checkpoints in store if necessary
    """
    # Update justified checkpoint
    if justified_checkpoint.epoch > store.justified_checkpoint.epoch:
        store.justified_checkpoint = justified_checkpoint

    # Update finalized checkpoint
    if finalized_checkpoint.epoch > store.finalized_checkpoint.epoch:
        store.finalized_checkpoint = finalized_checkpoint
```

#### `update_unrealized_checkpoints`

```python
def update_unrealized_checkpoints(
    store: Store,
    unrealized_justified_checkpoint: Checkpoint,
    unrealized_finalized_checkpoint: Checkpoint,
) -> None:
    """
    Update unrealized checkpoints in store if necessary
    """
    # Update unrealized justified checkpoint
    if unrealized_justified_checkpoint.epoch > store.unrealized_justified_checkpoint.epoch:
        store.unrealized_justified_checkpoint = unrealized_justified_checkpoint

    # Update unrealized finalized checkpoint
    if unrealized_finalized_checkpoint.epoch > store.unrealized_finalized_checkpoint.epoch:
        store.unrealized_finalized_checkpoint = unrealized_finalized_checkpoint
```

#### Proposer head and reorg helpers

_Implementing these helpers is optional_.

##### `is_head_late`

```python
def is_head_late(store: Store, head_root: Root) -> bool:
    return not store.block_timeliness[head_root]
```

##### `is_shuffling_stable`

```python
def is_shuffling_stable(slot: Slot) -> bool:
    return slot % SLOTS_PER_EPOCH != 0
```

##### `is_ffg_competitive`

```python
def is_ffg_competitive(store: Store, head_root: Root, parent_root: Root) -> bool:
    return (
        store.unrealized_justifications[head_root] == store.unrealized_justifications[parent_root]
    )
```

##### `is_finalization_ok`

```python
def is_finalization_ok(store: Store, slot: Slot) -> bool:
    epochs_since_finalization = compute_epoch_at_slot(slot) - store.finalized_checkpoint.epoch
    return epochs_since_finalization <= REORG_MAX_EPOCHS_SINCE_FINALIZATION
```

##### `is_proposing_on_time`

```python
def is_proposing_on_time(store: Store) -> bool:
    # Use half `SECONDS_PER_SLOT // INTERVALS_PER_SLOT` as the proposer reorg deadline
    time_into_slot = (store.time - store.genesis_time) % SECONDS_PER_SLOT
    proposer_reorg_cutoff = SECONDS_PER_SLOT // INTERVALS_PER_SLOT // 2
    return time_into_slot <= proposer_reorg_cutoff
```

##### `is_head_weak`

```python
def is_head_weak(store: Store, head_root: Root) -> bool:
    justified_state = store.checkpoint_states[store.justified_checkpoint]
    reorg_threshold = calculate_committee_fraction(justified_state, REORG_HEAD_WEIGHT_THRESHOLD)
    head_weight = get_weight(store, head_root)
    return head_weight < reorg_threshold
```

##### `is_parent_strong`

```python
def is_parent_strong(store: Store, parent_root: Root) -> bool:
    justified_state = store.checkpoint_states[store.justified_checkpoint]
    parent_threshold = calculate_committee_fraction(justified_state, REORG_PARENT_WEIGHT_THRESHOLD)
    parent_weight = get_weight(store, parent_root)
    return parent_weight > parent_threshold
```

##### `get_proposer_head`

```python
def get_proposer_head(store: Store, head_root: Root, slot: Slot) -> Root:
    head_block = store.blocks[head_root]
    parent_root = head_block.parent_root
    parent_block = store.blocks[parent_root]

    # Only re-org the head block if it arrived later than the attestation deadline.
    head_late = is_head_late(store, head_root)

    # Do not re-org on an epoch boundary where the proposer shuffling could change.
    shuffling_stable = is_shuffling_stable(slot)

    # Ensure that the FFG information of the new head will be competitive with the current head.
    ffg_competitive = is_ffg_competitive(store, head_root, parent_root)

    # Do not re-org if the chain is not finalizing with acceptable frequency.
    finalization_ok = is_finalization_ok(store, slot)

    # Only re-org if we are proposing on-time.
    proposing_on_time = is_proposing_on_time(store)

    # Only re-org a single slot at most.
    parent_slot_ok = parent_block.slot + 1 == head_block.slot
    current_time_ok = head_block.slot + 1 == slot
    single_slot_reorg = parent_slot_ok and current_time_ok

    # Check that the head has few enough votes to be overpowered by our proposer boost.
    assert store.proposer_boost_root != head_root  # ensure boost has worn off
    head_weak = is_head_weak(store, head_root)

    # Check that the missing votes are assigned to the parent and not being hoarded.
    parent_strong = is_parent_strong(store, parent_root)

    if all(
        [
            head_late,
            shuffling_stable,
            ffg_competitive,
            finalization_ok,
            proposing_on_time,
            single_slot_reorg,
            head_weak,
            parent_strong,
        ]
    ):
        # We can re-org the current head by building upon its parent block.
        return parent_root
    else:
        return head_root
```

*Note*: The ordering of conditions is a suggestion only. Implementations are
free to optimize by re-ordering the conditions from least to most expensive and
by returning early if any of the early conditions are `False`.

#### Pull-up tip helpers

##### `compute_pulled_up_tip`

```python
def compute_pulled_up_tip(store: Store, block_root: Root) -> None:
    state = store.block_states[block_root].copy()
    # Pull up the post-state of the block to the next epoch boundary
    process_justification_and_finalization(state)

    store.unrealized_justifications[block_root] = state.current_justified_checkpoint
    update_unrealized_checkpoints(
        store, state.current_justified_checkpoint, state.finalized_checkpoint
    )

    # If the block is from a prior epoch, apply the realized values
    block_epoch = compute_epoch_at_slot(store.blocks[block_root].slot)
    current_epoch = get_current_store_epoch(store)
    if block_epoch < current_epoch:
        update_checkpoints(store, state.current_justified_checkpoint, state.finalized_checkpoint)
```

#### `on_tick` helpers

##### `on_tick_per_slot`

```python
def on_tick_per_slot(store: Store, time: uint64) -> None:
    previous_slot = get_current_slot(store)

    # Update store time
    store.time = time

    current_slot = get_current_slot(store)

    # If this is a new slot, reset store.proposer_boost_root
    if current_slot > previous_slot:
        store.proposer_boost_root = Root()

    # If a new epoch, pull-up justification and finalization from previous epoch
    if current_slot > previous_slot and compute_slots_since_epoch_start(current_slot) == 0:
        update_checkpoints(
            store, store.unrealized_justified_checkpoint, store.unrealized_finalized_checkpoint
        )
```

#### `on_attestation` helpers

##### `validate_target_epoch_against_current_time`

```python
def validate_target_epoch_against_current_time(store: Store, attestation: Attestation) -> None:
    target = attestation.data.target

    # Attestations must be from the current or previous epoch
    current_epoch = get_current_store_epoch(store)
    # Use GENESIS_EPOCH for previous when genesis to avoid underflow
    previous_epoch = current_epoch - 1 if current_epoch > GENESIS_EPOCH else GENESIS_EPOCH
    # If attestation target is from a future epoch, delay consideration until the epoch arrives
    assert target.epoch in [current_epoch, previous_epoch]
```

##### `validate_on_attestation`

```python
def validate_on_attestation(store: Store, attestation: Attestation, is_from_block: bool) -> None:
    target = attestation.data.target

    # If the given attestation is not from a beacon block message, we have to check the target epoch scope.
    if not is_from_block:
        validate_target_epoch_against_current_time(store, attestation)

    # Check that the epoch number and slot number are matching
    assert target.epoch == compute_epoch_at_slot(attestation.data.slot)

    # Attestation target must be for a known block. If target block is unknown, delay consideration until block is found
    assert target.root in store.blocks

    # Attestations must be for a known block. If block is unknown, delay consideration until the block is found
    assert attestation.data.beacon_block_root in store.blocks
    # Attestations must not be for blocks in the future. If not, the attestation should not be considered
    assert store.blocks[attestation.data.beacon_block_root].slot <= attestation.data.slot

    # LMD vote must be consistent with FFG vote target
    assert target.root == get_checkpoint_block(
        store, attestation.data.beacon_block_root, target.epoch
    )

    # Attestations can only affect the fork choice of subsequent slots.
    # Delay consideration in the fork choice until their slot is in the past.
    assert get_current_slot(store) >= attestation.data.slot + 1
```

##### `store_target_checkpoint_state`

```python
def store_target_checkpoint_state(store: Store, target: Checkpoint) -> None:
    # Store target checkpoint state if not yet seen
    if target not in store.checkpoint_states:
        base_state = copy(store.block_states[target.root])
        if base_state.slot < compute_start_slot_at_epoch(target.epoch):
            process_slots(base_state, compute_start_slot_at_epoch(target.epoch))
        store.checkpoint_states[target] = base_state
```

##### `update_latest_messages`

```python
def update_latest_messages(
    store: Store, attesting_indices: Sequence[ValidatorIndex], attestation: Attestation
) -> None:
    target = attestation.data.target
    beacon_block_root = attestation.data.beacon_block_root
    non_equivocating_attesting_indices = [
        i for i in attesting_indices if i not in store.equivocating_indices
    ]
    for i in non_equivocating_attesting_indices:
        if i not in store.latest_messages or target.epoch > store.latest_messages[i].epoch:
            store.latest_messages[i] = LatestMessage(epoch=target.epoch, root=beacon_block_root)
```

### Handlers

#### `on_tick`

```python
def on_tick(store: Store, time: uint64) -> None:
    # If the ``store.time`` falls behind, while loop catches up slot by slot
    # to ensure that every previous slot is processed with ``on_tick_per_slot``
    tick_slot = (time - store.genesis_time) // SECONDS_PER_SLOT
    while get_current_slot(store) < tick_slot:
        previous_time = store.genesis_time + (get_current_slot(store) + 1) * SECONDS_PER_SLOT
        on_tick_per_slot(store, previous_time)
    on_tick_per_slot(store, time)
```

#### `on_block`

```python
def on_block(store: Store, signed_block: SignedBeaconBlock) -> None:
    """
    Run ``on_block`` upon receiving a new block.
    """
    block = signed_block.message
    # Parent block must be known
    assert block.parent_root in store.block_states
    # Blocks cannot be in the future. If they are, their consideration must be delayed until they are in the past.
    assert get_current_slot(store) >= block.slot

    # Check that block is later than the finalized epoch slot (optimization to reduce calls to get_ancestor)
    finalized_slot = compute_start_slot_at_epoch(store.finalized_checkpoint.epoch)
    assert block.slot > finalized_slot
    # Check block is a descendant of the finalized block at the checkpoint finalized slot
    finalized_checkpoint_block = get_checkpoint_block(
        store,
        block.parent_root,
        store.finalized_checkpoint.epoch,
    )
    assert store.finalized_checkpoint.root == finalized_checkpoint_block

    # [New in Deneb:EIP4844]
    # Check if blob data is available
    # If not, this block MAY be queued and subsequently considered when blob data becomes available
    # *Note*: Extraneous or invalid Blobs (in addition to the expected/referenced valid blobs)
    # received on the p2p network MUST NOT invalidate a block that is otherwise valid and available
    assert is_data_available(hash_tree_root(block), block.body.blob_kzg_commitments)

    # Check the block is valid and compute the post-state
    # Make a copy of the state to avoid mutability issues
    state = copy(store.block_states[block.parent_root])
    block_root = hash_tree_root(block)
    state_transition(state, signed_block, True)

    # Add new block to the store
    store.blocks[block_root] = block
    # Add new state for this block to the store
    store.block_states[block_root] = state

    # Add block timeliness to the store
    time_into_slot = (store.time - store.genesis_time) % SECONDS_PER_SLOT
    is_before_attesting_interval = time_into_slot < SECONDS_PER_SLOT // INTERVALS_PER_SLOT
    is_timely = get_current_slot(store) == block.slot and is_before_attesting_interval
    store.block_timeliness[hash_tree_root(block)] = is_timely

    # Add proposer score boost if the block is timely and not conflicting with an existing block
    is_first_block = store.proposer_boost_root == Root()
    if is_timely and is_first_block:
        store.proposer_boost_root = hash_tree_root(block)

    # Update checkpoints in store if necessary
    update_checkpoints(store, state.current_justified_checkpoint, state.finalized_checkpoint)

    # Eagerly compute unrealized justification and finality.
    compute_pulled_up_tip(store, block_root)
```

#### `on_attestation`

```python
def on_attestation(store: Store, attestation: Attestation, is_from_block: bool = False) -> None:
    """
    Run ``on_attestation`` upon receiving a new ``attestation`` from either within a block or directly on the wire.

    An ``attestation`` that is asserted as invalid may be valid at a later time,
    consider scheduling it for later processing in such case.
    """
    validate_on_attestation(store, attestation, is_from_block)

    store_target_checkpoint_state(store, attestation.data.target)

    # Get state at the `target` to fully validate attestation
    target_state = store.checkpoint_states[attestation.data.target]
    indexed_attestation = get_indexed_attestation(target_state, attestation)
    assert is_valid_indexed_attestation(target_state, indexed_attestation)

    # Update latest messages for attesting indices
    update_latest_messages(store, indexed_attestation.attesting_indices, attestation)
```

#### `on_attester_slashing`

*Note*: `on_attester_slashing` should be called while syncing and a client MUST
maintain the equivocation set of `AttesterSlashing`s from at least the latest
finalized checkpoint.

```python
def on_attester_slashing(store: Store, attester_slashing: AttesterSlashing) -> None:
    """
    Run ``on_attester_slashing`` immediately upon receiving a new ``AttesterSlashing``
    from either within a block or directly on the wire.
    """
    attestation_1 = attester_slashing.attestation_1
    attestation_2 = attester_slashing.attestation_2
    assert is_slashable_attestation_data(attestation_1.data, attestation_2.data)
    state = store.block_states[store.justified_checkpoint.root]
    assert is_valid_indexed_attestation(state, attestation_1)
    assert is_valid_indexed_attestation(state, attestation_2)

    indices = set(attestation_1.attesting_indices).intersection(attestation_2.attesting_indices)
    for index in indices:
        store.equivocating_indices.add(index)
```
