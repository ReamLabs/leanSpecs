# pqdevnet-0 Fork Choice

## Introduction

This specs intend to be a bare-minimal fork choice rule that enables pqdevnet validators
to attest to the same chain.

It is originally based on the combined Beacon Chain fork choice specs from Phase0 to
Electra. Fulu is excluded as its specs is a work in progress at the time of writing.

### Why minimal Beacon fork choice than 3SF?

- Beacon fork choice is battle-tested and well understood, therefore is ideal as a
control factor for pqdevnet.
- 3SF, on the other hand, is an entirely new mechanism with many non-battle-tested
subcomponents (e.g. Majority Fork Choice function, modified TOB-SVD, a new finality
protocol, etc.) that easily warrants its own devnet(s).

### Key differences from Beacon Chain specs

- Removed `ExecutionEngine` related mechanisms, e.g. `notify_forkchoice_updated()`,
  `should_override_forkchoice_update`, `PayloadAttributes`, etc.
- Removed slashings e.g. `on_attester_slashing()`, `equivocating_indices`, etc.
- Removed The Merge-related logic (that was not explicitly removed in Capella)
- Removed blobs
- Removed epochs, checkpoints, justifications and finalizations e.g. `compute_pulled_up_tip()`, `get_checkpoint_block()`, `update_checkpoints()`, etc.

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

Any of the above handlers that trigger an unhandled exception (e.g. a failed
assert or an out-of-range list access) are considered invalid. Invalid calls to
handlers must not modify `store`.

*Notes*:

1. **Leap seconds**: Slots will last `SECONDS_PER_SLOT + 1` or
   `SECONDS_PER_SLOT - 1` seconds around leap seconds. This is automatically
   handled by [UNIX time](https://en.wikipedia.org/wiki/Unix_time).
2. **Honest clocks**: Honest nodes are assumed to have clocks synchronized
   within `SECONDS_PER_SLOT` seconds of each other.
3. **Implementation**: The implementation found in this specification is
   constructed for ease of understanding rather than for optimization in
   computation, space, or any other resource. A number of optimized alternatives
   can be found [here](https://github.com/protolambda/lmd-ghost).

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

- The proposer score boost and re-org weight threshold are percentage values
  that are measured with respect to the weight of a single committee. See
  `calculate_committee_fraction`.

### Helpers

#### `LatestMessage`

```python
@dataclass(eq=True, frozen=True)
class LatestMessage(object):
    root: Root
```

#### `Store`

The `Store` is responsible for tracking information required for the fork choice
algorithm. The important fields being tracked are described below:

- `justified_checkpoint`: the justified checkpoint used as the starting point
  for the LMD GHOST fork choice algorithm.
- `finalized_checkpoint`: the highest known finalized checkpoint. The fork
  choice only considers blocks that are not conflicting with this checkpoint.
- `unrealized_justifications`: stores a map of block root to the unrealized
  justified checkpoint observed in that block.

```python
@dataclass
class Store(object):
    time: uint64
    genesis_time: uint64
    genesis_root: Root
    justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    proposer_boost_root: Root
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
        genesis_root=anchor_state.genesis_root,
        justified_checkpoint=justified_checkpoint,
        finalized_checkpoint=finalized_checkpoint,
        proposer_boost_root=proposer_boost_root,
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
def get_proposer_score(state: LeanState) -> Gwei:
    total_attesters_weight = get_total_active_balance(state)
    return (total_attesters_weight * PROPOSER_SCORE_BOOST) // 100
```

#### `get_weight`

```python
def get_weight(store: Store, root: Root) -> Gwei:
    state = store.checkpoint_states[store.justified_checkpoint] # TODO: store should have the latest state
    active_validator_indices = get_active_validator_indices(state)

    attestation_score = Gwei(
        sum(
            state.validators[i].effective_balance
            for i in active_validator_indices
            if (
                i in store.latest_messages
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
        proposer_score = get_proposer_score(state)
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

    # Instead of checking for justified/finalized checkpoints, check that the leaf block is a descendant of the genesis block.
    if get_ancestor(store, block_root, 0):
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

#### Proposer head and reorg helpers

_Implementing these helpers is optional_.

##### `is_head_late`

```python
def is_head_late(store: Store, head_root: Root) -> bool:
    return not store.block_timeliness[head_root]
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
```

#### `on_attestation` helpers

##### `validate_on_attestation`

```python
def validate_on_attestation(store: Store, attestation: Attestation) -> None:
    # Attestations must be for a known block. If block is unknown, delay consideration until the block is found
    assert attestation.data.beacon_block_root in store.blocks
    # Attestations must not be for blocks in the future. If not, the attestation should not be considered
    assert store.blocks[attestation.data.beacon_block_root].slot <= attestation.data.slot

    # Attestations can only affect the fork choice of subsequent slots.
    # Delay consideration in the fork choice until their slot is in the past.
    assert get_current_slot(store) >= attestation.data.slot + 1
```

##### `update_latest_message`

```python
def update_latest_message(store: Store, attestation: Attestation) -> None:
    lean_block_root = attestation.data.lean_block_root
    attester_index = attestation.data.index

    if attester_index not in store.latest_messages:
        store.latest_messages[i] = LatestMessage(root=lean_block_root)
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
```

#### `on_attestation`

```python
def on_attestation(store: Store, attestation: Attestation) -> None:
    """
    Run ``on_attestation`` upon receiving a new ``attestation`` from the wire.
    """
    validate_on_attestation(store, attestation, is_from_block)

    # Get state at the `target` to fully validate attestation
    target_state = store.checkpoint_states[attestation.data.target]
    indexed_attestation = get_indexed_attestation(target_state, attestation)
    assert is_valid_indexed_attestation(target_state, indexed_attestation)

    # Update latest message for the attestation
    update_latest_message(store, attestation)
```
