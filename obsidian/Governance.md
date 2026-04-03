# Governance

On-chain governance system tied to the CStaking contract.

## Actions

- **`GovPropose`** (action variant 41): `struct GovProposeAction with 6 elements` — title, description + 4 more
- **`GovVote`** (action variant 42): `struct GovVoteAction with 5 elements`

## Structures

```
struct Proposal with 7 elements — title, end, votes, quorum + 3
enum ProposalCategory
struct Vote with 2 elements — choice, weight
enum VoteChoice
tuple struct VoteWeight
```

## Validation Rules

- Max active proposals limited ("Too many active proposals. Limit is N")
- Title: 1-50 characters
- Description: 1-1000 characters
- Proposer needs sufficient delegations ("Insufficient delegations to propose")
- No duplicate proposals
- No voting after end time

## CStaking Integration

Proposals live inside the `CStaking` struct (23 elements). Voting is delegation-weighted.

CStaking fields:
```
stakes, allow_all_validators, allowed_validators, broadcasters,
validator_to_profile, validator_to_last_signer_change_time,
delegations, proposals, stakes_decay_bucket_guard,
jailed_signers, jail_vote_tracker, jail_until,
signer_to_last_manual_unjail_time, native_token_reserve,
stage_reward_bucket_guard, validator_to_staged_rewards,
distribute_reward_bucket_guard, pending_withdrawals,
self_delegation_requirement, disabled_validators, disabled_node_ips,
validator_to_state, gversion
```

## Info API

`GovProposals` table in the info server.

#governance #staking #voting
