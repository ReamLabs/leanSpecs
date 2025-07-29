# The Lean Consensus Chain

## Introduction

This document represents the specification for the pq-devnet chain.

## Preset

*Note*: The below configuration is bundled as a preset: a bundle of
configuration variables which are expected to differ between different modes of
operation, e.g. testing, but not generally between different networks.

### Time parameters

| Name                               | Value                        |  Unit  |   Duration   |
| ---------------------------------- | ---------------------------- | :----: | :----------: |
| `SLOTS_PER_EPOCH`                  | `uint64(1000000)`            | slots  | 46.3 days    |

## Configuration

*Note*: The default mainnet configuration values are included here for
illustrative purposes. Testnets and other types of chain instances may
use a different configuration.

### Time parameters

| Name                               | Value                        |  Unit   |   Duration   |
| ---------------------------------- | ---------------------------- | :-----: | :----------: |
| `SECONDS_PER_SLOT`                 | `uint64(12)`                 | seconds | 12 seconds   |

### Misc dependencies

#### `AttestationData`

```python
class AttestationData(Container):
    slot: Slot
    index: CommitteeIndex
    beacon_block_root: Root
    source: Checkpoint
    target: Checkpoint
```

### Beacon operations

#### `Attestation`

```python
class Attestation(Container):
    aggregation_bits: Bitlist[MAX_VALIDATORS_PER_COMMITTEE]
    data: AttestationData
    signature: BLSSignature
```
