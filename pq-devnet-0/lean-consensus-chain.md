# The Lean Consensus Chain

## Introduction

This document represents the specification for the pq-devnet chain.

### Key differences from Beacon Chain specs

- Removed signature aggregation data e.g. `aggregation_bits`
- Removed checkpoints & FFG information, e.g. `source` and `target` checkpoints

## Preset

*Note*: The below configuration is bundled as a preset: a bundle of
configuration variables which are expected to differ between different modes of
operation, e.g. testing, but not generally between different networks.

## Configuration

*Note*: The default mainnet configuration values are included here for
illustrative purposes. Testnets and other types of chain instances may
use a different configuration.

### Time parameters

| Name                               | Value                        |  Unit   |   Duration   |
| ---------------------------------- | ---------------------------- | :-----: | :----------: |
| `SECONDS_PER_SLOT`                 | `uint64(4)`                  | seconds | 4 seconds    |

### Beacon operations

#### `Attestation`

```python
class Attestation(Container):
    data: AttestationData
    signature: BLSSignature
```

#### `AttestationData`

```python
class AttestationData(Container):
    slot: Slot
    attester_index: ValidatorIndex
    lean_block_root: Root
```

### Beacon state accessors

#### `get_active_validator_indices`

TODO: Update this to a static indices

```python
def get_active_validator_indices(state: BeaconState, epoch: Epoch) -> Sequence[ValidatorIndex]:
    """
    Return the sequence of active validator indices at ``epoch``.
    """
    return [
        ValidatorIndex(i) for i, v in enumerate(state.validators) if is_active_validator(v, epoch)
    ]
```
