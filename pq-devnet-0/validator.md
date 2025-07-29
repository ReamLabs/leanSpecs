# Honest Validator

This is an accompanying document to [The Lean Consensus Chain](./lean-consensus-chain.md),
which describes the expected actions of a "validator" participating in Ethereum's
Lean Consensus protocol.

## Key differences from the Beacon Chain specs

- Removed slashings logic
- Attestation is broadcasted to a single, global `pqdevnet_attestation` pubsub topic.

## Validator responsibilities

A validator has two primary responsibilities to the beacon chain: proposing blocks
and creating attestations.

### Block proposal

A validator is expected to propose a `SignedBlock` at the beginning of
any `slot` during which `is_proposer(state, validator_index)` returns `True`.

To propose, the validator selects a `BeaconBlock`, `parent` using this process:

1. Compute fork choice's view of the head at the start of `slot`, after running
   `on_tick` and applying any queued attestations from `slot - 1`. Set
   `head_root = get_head(store)`.
2. Compute the _proposer head_, which is the head upon which the proposer SHOULD
   build in order to incentivise timely block propagation by other validators.
   Set `parent_root = get_proposer_head(store, head_root, slot)`. A proposer may
   set `parent_root == head_root` if proposer re-orgs are not implemented or
   have been disabled.
3. Let `parent` be the block with `parent_root`.

The validator creates, signs, and broadcasts a `block` that is a child of
`parent`. Note that the parent's slot must be strictly less than the slot
of the block about to be proposed, i.e. `parent.slot < slot`.

*Note*: In this section, `state` is the state of the slot for the block proposal
_without_ the block yet applied. That is, `state` is the `previous_state`
processed through any empty slots up to the assigned slot using
`process_slots(previous_state, slot)`.

#### Preparing for a `BeaconBlock`

To construct a `BeaconBlockBody`, a `block` (`BeaconBlock`) is defined with the
necessary context for a block proposal:

##### Slot

Set `block.slot = slot` where `slot` is the current slot at which the validator
has been selected to propose. The `parent` selected must satisfy that
`parent.slot < block.slot`.

*Note*: There might be "skipped" slots between the `parent` and `block`. These
skipped slots are processed in the state transition function without per-block
processing.

##### Proposer index

Set `block.proposer_index = validator_index` where `validator_index` is the
validator chosen to propose at this slot. The private key mapping to
`state.validators[validator_index].pubkey` is used to sign the block.

##### Parent root

Set `block.parent_root = hash_tree_root(parent)`.

#### Constructing the `BeaconBlockBody`

##### Eth1 Data

The `block.body.eth1_data` field is for block proposers to vote on recent Eth1
data. This recent data contains an Eth1 block hash as well as the associated
deposit root (as calculated by the `get_deposit_root()` method of the deposit
contract) and deposit count after execution of the corresponding Eth1 block. If
over half of the block proposers in the current Eth1 voting period vote for the
same `eth1_data` then `state.eth1_data` updates immediately allowing new
deposits to be processed. Each deposit in `block.body.deposits` must verify
against `state.eth1_data.eth1_deposit_root`.

##### Attestations

Up to `MAX_ATTESTATIONS`, aggregate attestations can be included in the `block`.
The attestations added must satisfy the verification conditions found in
[attestation processing](./beacon-chain.md#attestations). To maximize profit,
the validator should attempt to gather aggregate attestations that include
singular attestations from the largest number of validators whose signatures
from the same epoch have not previously been added on chain.

#### Packaging into a `SignedBeaconBlock`

##### State root

Set `block.state_root = hash_tree_root(state)` of the resulting `state` of the
`parent -> block` state transition.

*Note*: To calculate `state_root`, the validator should first run the state
transition function on an unsigned `block` containing a stub for the
`state_root`. It is useful to be able to run a state transition function
(working on a copy of the state) that does _not_ validate signatures or state
root for this purpose:

```python
def compute_new_state_root(state: BeaconState, block: BeaconBlock) -> Root:
    temp_state: BeaconState = state.copy()
    signed_block = SignedBeaconBlock(message=block)
    state_transition(temp_state, signed_block, validate_result=False)
    return hash_tree_root(temp_state)
```

##### Signature

`signed_block = SignedBeaconBlock(message=block, signature=block_signature)`,
where `block_signature` is obtained from:

```python
def get_block_signature(state: BeaconState, block: BeaconBlock, privkey: int) -> BLSSignature:
    domain = get_domain(state, DOMAIN_BEACON_PROPOSER, compute_epoch_at_slot(block.slot))
    signing_root = compute_signing_root(block, domain)
    return bls.Sign(privkey, signing_root)
```

### Attesting

A validator is expected to create, sign, and broadcast an attestation during
each epoch. The `committee`, assigned `index`, and assigned `slot` for which the
validator performs this role during an epoch are defined by
`get_committee_assignment(state, epoch, validator_index)`.

A validator should create and broadcast the `attestation` to the associated
attestation subnet when either (a) the validator has received a valid block from
the expected block proposer for the assigned `slot` or (b)
`1 / INTERVALS_PER_SLOT` of the `slot` has transpired
(`SECONDS_PER_SLOT / INTERVALS_PER_SLOT` seconds after the start of `slot`) --
whichever comes _first_.

*Note*: Although attestations during `GENESIS_EPOCH` do not count toward FFG
finality, these initial attestations do give weight to the fork choice, are
rewarded, and should be made.

#### Attestation data

First, the validator should construct `attestation_data`, an
[`AttestationData`](./beacon-chain.md#attestationdata) object based upon the
state at the assigned slot.

- Let `head_block` be the result of running the fork choice during the assigned
  slot.
- Let `head_state` be the state of `head_block` processed through any empty
  slots up to the assigned slot using `process_slots(state, slot)`.

##### General

- Set `attestation_data.slot = slot` where `slot` is the assigned slot.
- Set `attestation_data.index = index` where `index` is the index associated
  with the validator's committee.

##### LMD GHOST vote

Set `attestation_data.beacon_block_root = hash_tree_root(head_block)`.

##### FFG vote

- Set `attestation_data.source = head_state.current_justified_checkpoint`.
- Set
  `attestation_data.target = Checkpoint(epoch=get_current_epoch(head_state), root=epoch_boundary_block_root)`
  where `epoch_boundary_block_root` is the root of block at the most recent
  epoch boundary.

*Note*: `epoch_boundary_block_root` can be looked up in the state using:

- Let `start_slot = compute_start_slot_at_epoch(get_current_epoch(head_state))`.
- Let
  `epoch_boundary_block_root = hash_tree_root(head_block) if start_slot == head_state.slot else get_block_root(state, get_current_epoch(head_state))`.

#### Construct attestation

Next, the validator creates `attestation`, an
[`Attestation`](./lean-chain.md#attestation) object.

##### Data

Set `attestation.data = attestation_data` where `attestation_data` is the
`AttestationData` object defined in the previous section,
[attestation data](#attestation-data).

##### Aggregate signature

Set `attestation.signature = attestation_signature` where `attestation_signature`
is obtained from:

```python
def get_attestation_signature(
    state: BeaconState, attestation_data: AttestationData, privkey: int
) -> BLSSignature:
    domain = get_domain(state, DOMAIN_BEACON_ATTESTER, attestation_data.target.epoch)
    signing_root = compute_signing_root(attestation_data, domain)
    return bls.Sign(privkey, signing_root)
```

#### Broadcast attestation

The validator broadcasts `attestation` to the `pqdevnet_attestation` pubsub topic.

### Attestation aggregation

No attestation aggregation for this devnet.
