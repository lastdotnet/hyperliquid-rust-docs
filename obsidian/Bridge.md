# Bridge

ETH↔Hyperliquid bridge system. Version 2 ("Bridge2") is the current active bridge.

## Bridge2 (10 fields)
From [[Exchange State]]:
- `eth_id_to_deposit_votes` — deposit confirmation tracking
- `finished_deposits_data` — completed deposits
- `withdrawal_signatures` — withdrawal sigs
- `withdrawal_finalized_votes` — finalization votes
- `finished_withdrawal_to_time` — withdrawal timing
- `validator_set_signatures` — validator set update sigs
- `validator_set_finalized_votes` — validator set votes
- `bal` — bridge balance tracking
- `last_pruned_deposit_block_number` — deposit pruning watermark
- `oaw` — (abbreviated, pending full decode)

## Signing Model (CONFIRMED 2026-04-02)

**CONFIRMED**: Bridge signing uses the **same validator signer keys** as consensus signing. There are NOT separate EOA signing keys for bridge operations. The consensus signer key (the key used for HyperBFT voting) is the same key that signs bridge withdrawal messages and validator set updates.

### Two-Tier Model on Ethereum (Arbitrum Bridge2.sol)
The Ethereum-side Bridge2.sol contract uses a **hot/cold wallet architecture**:
- **Hot signers**: validator signer keys that sign withdrawal messages (same as consensus keys)
- **Cold signers**: higher-security keys for validator set updates and emergency operations
- Hot and cold sets can differ, but the hot set mirrors the consensus signer set

### Threshold
- **2/3+ stake-weighted threshold** required for both deposits and withdrawals
- Same threshold as HyperBFT consensus (not a separate quorum)

### Dispute Period
- **CONFIRMED**: Withdrawals have a **dispute period** before finalization on Ethereum
- During the dispute period, validators can `invalidateWithdrawals` to cancel
- After the dispute period expires, funds can be claimed on Ethereum

## Deposit Flow
1. User deposits ETH/tokens on Ethereum L1 (Arbitrum)
2. Validators observe the deposit on-chain
3. Each validator submits `VoteEthDepositAction(3)`: deposit_id, chain, validator
4. 2/3+ stake-weighted validator votes confirm the deposit
5. Funds credited on Hyperliquid

## Withdrawal Flow
1. User submits `withdraw3` action
2. Validators sign the withdrawal using their **consensus signer keys**
3. `ValidatorSignWithdrawalAction(5)`: signature, chain, nonce, withdrawal_hash, validator
4. 2/3+ stake-weighted signatures collected
5. Withdrawal enters dispute period on Ethereum
6. After dispute period, funds released on Ethereum

## Governance
- `InvalidateWithdrawals` — cancel pending withdrawals during dispute period (no escape hatch for users!)
- Bridge validator set changes apply instantly (no timelock)
- `bridge2_withdraw_fee` — configurable fee

## Security Concerns (CONFIRMED 2026-04-02)

### Critical Findings
1. **Broadcaster centralization**: Single broadcaster (or small set) controls action ordering and inclusion. MEV extraction possible.
2. **InvalidateWithdrawals**: Validators can cancel ANY pending withdrawal during the dispute period. No escape hatch for users if validators collude.
3. **No escape hatch**: Users cannot force-withdraw if the validator set becomes adversarial. The bridge is fully trust-dependent on the active validator set.
4. **Broadcaster MEV**: The broadcaster sees all pending transactions before they are included in blocks and can front-run, sandwich, or censor.

### High-Severity Findings
- Bridge validator set is modifiable via `ModifyBroadcaster` (instant, no timelock)
- Single `oracle_updater` (broadcaster) controls all perp oracle prices
- Validator set update on Ethereum side can be changed with hot key quorum

### Known Incidents
- **$38M+ confirmed losses** across 5 documented incidents (see Security section in Project Status)

## sendToEvmWithData (CONFIRMED 2026-04-02, 11 fields)

L1-to-EVM transfer with optional calldata. See [[HyperEVM]] for full struct definition.

Fields: token, amount, sourceDex, destinationRecipient, toPerp, destinationChainId, time, data, nonce, signatureChainId, hyperliquidChain

## Links
- [[Exchange State]] — bridge2 field
- [[Action Types]] — VoteEthDeposit, ValidatorSignWithdrawal, sendToEvmWithData
- [[HyperBFT]] — validators vote on bridge events
- [[Guards and Limits]] — withdrawal limits
- [[HyperEVM]] — sendToEvmWithData details

#bridge #ethereum #crosschain #security
