# HyperBFT

HotStuff-inspired BFT consensus used by [[Hyperliquid]].

## Properties
- **Block time**: ~70ms (14 blocks/second)
- **Finality**: Instant (no reorgs after commit)
- **Validators**: 24 active (top by stake)
- **Epoch**: **CORRECTED**: Time-based (`epoch_duration_seconds`), NOT round-based. Previously documented as "100,000 rounds (~90 minutes)" but this is incorrect. The Staking struct field `epoch_duration_seconds` controls epoch length.
- **Proposer**: Weighted round-robin (`RoundRobinTtl`)

## Message Types
From `node/src/consensus/` in [[Binary Structure]]:

| Message | Purpose |
|---------|---------|
| `BlockTimeout` | View change trigger |
| `Heartbeat` | Liveness probe |
| `HeartbeatAck` | Liveness response |
| `Tx` | Transaction forwarding |
| `Proposal(7)` | Block proposal from leader |
| `CommitProof` | QC with 2/3+ stake signatures |

## Consensus Flow
1. Leader selected by `RoundRobinTtl` (stake-weighted)
2. Proposal broadcast to validators
3. Validators validate: proposer, QC, parent_round, tx validity
4. 2/3+ stake votes → [[Quorum Certificate]] forms
5. Block committed, round advances

## Quorum Certificate (QC)
- `next_proposer` — who proposes next
- `suspect` — suspected validator for jailing
- `round_to_jailed_validators` — jailing tracking
- Formed when 2/3+ validators by stake agree

## Epoch Management (CONFIRMED time-based, 2026-04-02)
- [[EpochTracker]](5): epoch_states, active_epoch, cur_epoch_state, cur_epoch, epoch_duration_seconds
- **CONFIRMED**: Epochs are **time-based** via `epoch_duration_seconds` field on the Staking struct, NOT round-based
- Validator set recomputed at epoch boundaries
- Stakes locked for the epoch
- Epoch transitions trigger reward distribution via `distribute_reward_bucket_guard`

## Heartbeat Tracking
- `validator_latency_ema` — exponential moving average
- `latency_ema_jail_threshold` — jailing threshold
- `HeartbeatSnapshot` + `ValidatorSetSnapshot`

## Jailing
- Via `VoteJail` consensus action
- Jailed validators forward messages but don't vote
- `Can only manually unjail once every N`
- `round_to_jailed_validators` tracking

## Upgrade Flow ([[App Hash]])
1. `scheduled_freeze_height` set via governance
2. Chain freezes at that height
3. Validators vote `VoteAppHash` with their computed hash
4. 2/3+ agreement → hardfork confirmed
5. Visor downloads new binary, restarts
6. `"observed qc for newer hardfork, crashing"` forces laggards

## Links
- [[Exchange State]] — state that consensus commits
- [[Gossip Protocol]] — how validators communicate
- [[LtHash]] — how state hash is computed
- [[Guards and Limits]] — rate limiting

#consensus #bft #hotstuff
