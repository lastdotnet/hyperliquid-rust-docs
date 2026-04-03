# Action Types

[[Hyperliquid]] has ~90 action types that mutate the [[Exchange State]]: 80 HlAction enum variants + ~10 internal/system variants in the binary.

**UPDATED 2026-04-03**: Testnet binary (commit `331bef9b`, built 2026-04-03) reveals new action variants embedded in `l1/src/time.rs` rodata. The binary also contains complete `VoteGlobalAction`, `SpotDeployAction`, `Hip3DeployAction`, `OutcomeDeploy`, `SetBole`, and `CValidatorAction` sub-variant lists.

## Distribution (from 2.3hr mainnet sample)
| Action | Count | % |
|--------|-------|---|
| order | 1,422,571 | 68.2% |
| cancelByCloid | 348,407 | 16.7% |
| cancel | 219,274 | 10.5% |
| evmRawTx | 36,189 | 1.7% |
| noop | 23,817 | 1.1% |
| batchModify | 18,728 | 0.9% |
| scheduleCancel | 5,216 | 0.3% |
| updateLeverage | 2,952 | 0.1% |
| SetGlobalAction | 1,271 | 0.1% |
| ValidatorSignWithdrawal | 1,177 | 0.1% |

## Categories

### Trading (variants 0-15)
- `order` — place orders (with grouping: na, normalTpsl, positionTpsl)
- `cancel` — batch cancel by `{a, o}` pairs
- `cancelByCloid` — cancel by client order ID
- `modify` — cancel + replace
- `batchModify` — multiple modify
- `scheduleCancel` — [[Guards and Limits|volume-gated]] timed cancel

### Account Management
- `updateLeverage` — change leverage
- `updateIsolatedMargin` — adjust margin
- `topUpStrictIsolatedMargin` — top up strict mode
- `approveAgent` — API key/agent management
- `approveBuilderFee` — builder fee approval
- `setReferrer` — referral code
- `setDisplayName` — display name

### Transfers
- `usdSend` — USDC transfer
- `usdClassTransfer` — USD class transfer  
- `spotSend` — spot token transfer
- `sendAsset` — general asset transfer
- `withdraw3` — withdrawal v3

### Vaults
- `createVault` — create new vault
- `vaultTransfer` — deposit/withdraw
- `vaultDistribute` — distribute funds
- `vaultModify` — modify settings

### Staking
- `tokenDelegate` — delegate/undelegate
- `claimRewards` — claim staking rewards
- `linkStakingUser` — link staking user

### Governance
- `SetGlobalAction` — oracle, margin tables, fees, etc.
  - Sub-types: setOracle, insertMarginTable, setFeeRecipient, haltTrading, setOpenInterestCaps, setFundingMultipliers, setMarginModes, etc.
- `govPropose` — governance proposal
- `govVote` — vote on proposal

### Bridge ([[Bridge]])
- `VoteEthDepositAction` — vote to confirm ETH deposit
- `ValidatorSignWithdrawalAction` — sign withdrawal
- `VoteEthFinalizedValidatorSetUpdateAction` — validator set update

### EVM ([[HyperEVM]])
- `evmRawTx` — raw EVM transaction
- `sendToEvmWithData` — L1→EVM with calldata (11 fields confirmed, see [[HyperEVM]])

### Advanced
- `twapOrder` / `twapCancel` — TWAP orders
- `subAccountTransfer` / `subAccountSpotTransfer` — sub-accounts
- `multiSig` / `convertToMultiSigUser` — multi-sig
- `perpDeploy` — deploy new perp market
- `borrowLend` — [[BOLE]] lending
- `userOutcome` — [[Outcomes]] prediction markets
- `userDexAbstraction` / `agentSetAbstraction` — DEX abstraction
- `userPortfolioMargin` — portfolio margin

## New Action Variants (2026-04-03 testnet binary)
- `GossipPriorityBidAction` — gossip priority bidding
- `ReserveRequestWeightAction` — BOLE reserve request weighting
- `SystemAlignedQuoteSupplyDeltaAction` — system-level aligned quote supply delta
- `DeployerSendToEvmForFrozenUserAction` — deployer can send to EVM for frozen users
- `SystemUsdClassTransferAction` — system USD class transfer
- `FinalizeEvmContractAction` — finalize EVM contract deployment
- `SubAccountSpotTransferAction` — sub-account spot transfers

## VoteGlobalAction Sub-Variants (complete from binary)
- `SetPerpDexDailyPxRange` | `SetReserveAccumulator` | `SetOutcomeFeeScale`
- `ModifyCStakingPeriods` | `SetPerpDexMaxNUsersWithPositions`
- `UserIsUnrewarded` | `SetPerpDexTransferLimits` | `SetPerpDexOpenInterestCap`
- `HaltPerpTrading` | `ModifyReferrer` | `UnjailSigner`
- `DisableCValidator` | `UserCanLiquidate` | `ModifyNonCirculatingSupply`
- `CancelUserOrders` | `DisableNodeIp` | `SetMaxOiPerSecond`
- `QuarantineUser` | `ResetHip3PxAlert` | `SetEvmBlockDuration`
- `SetEvmPrecompileEnabled` | `SetHip3BackstopLiquidatorParams`
- `DeployGasAuctionChange` | `CValidator` | `SetCoreWriterActionEnabled`
- `ModifyBroadcaster`

## SpotDeployAction Sub-Variants (complete from binary)
- `RegisterHyperliquidity` (5 fields: maxSupply, startPx, orderSz, nOrders, nSeededLevels)
- `RegisterSpot` | `SetFullName` | `UserGenesis` (4 fields: freeze, userAndWei, existingTokenAndWei, blacklistUsers)
- `RegisterToken2` (3 fields) | `SetDeployerTradingFeeShare`
- `RevokeFreezePrivilege` | `FreezeUser` (3 fields) | `RequestEvmContract` (3 fields: evmExtraWeiDecimals)
- `Genesis` (3 fields) | `EnableFreezePrivilege` | `EnableAlignedQuoteToken` | `EnableQuoteToken`

## SetBole Sub-Variants (from binary)
- `BackstopLiquidatorTokenParams` (4 fields)
- `Adl` (2 fields) | `TestnetAction` (2 fields) | `RateLimit` (7 fields)
- `ResetIndex` (2 fields) | `Reserve` (9 fields)

## OutcomeDeploy Sub-Variants (from binary)
- `RegisterQuestion` (3 fields: tokenAndNames, quoteToken, namedOutcome)
- `RegisterOutcome` (4 fields: outcome, maxGas, fallbackOutcome, settleFraction)
- `RegisterNamedOutcome` | `ChangeOutcomeDescription` | `SettleOutcome` (3 fields: details)
- `ChangeQuestionDescription` | `RegisterOutcomeToken` | `RegisterTokensAndStandaloneOutcome`

## CValidatorAction Sub-Variants (from binary)
- `Register` (3 fields: disable_delegations, profile, initial_wei)
- `ChangeProfile` (7 fields)

## Broadcaster Constraint
All user actions wrapped in `SignedActionWrapper`:
- Must come through one of 8 whitelisted broadcaster addresses
- `Tx unexpected broadcaster` — rejected if not whitelisted
- `Tx has invalid broadcaster nonce` — nonce validation

## Links
- [[Exchange State]] — state mutated by actions
- [[Matching Engine]] — processes order/cancel/modify
- [[Guards and Limits]] — volume-gating, rate limits
- [[Binary Structure]] — action dispatch at offset ~0x1F36320

#actions #protocol #trading
