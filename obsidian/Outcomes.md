# Outcomes

Native L1 prediction market system in [[Hyperliquid]]. Not an EVM contract.

## Types (from [[Binary Structure]])
- **OutcomeSpec(4)**: settleFraction, details, questions, outcome
- **OutcomeSideSpec(3)**
- **QuestionSpec(5)**
- **OutcomeTracker(11)**: manages all outcome state
- **SettledOutcomeSpec(3)**, **SettledQuestionSpec(4)**, **SettledOutcomeHistory(3)**

## User Operations (variant 70: `userOutcome`)
- `splitOutcome` — split collateral into outcome tokens
- `mergeOutcome` — merge tokens back to collateral
- `mergeQuestion` — merge question-level tokens
- `negateOutcome` — take opposite position

## Governance Operations (via [[Action Types|SetGlobalAction]])
- `registerOutcomeToken` — register new token
- `registerOutcome` — register new market
- `registerTokensAndStandaloneOutcome` — combined registration
- `registerNamedOutcome` — register with name
- `settleOutcome` — settle resolved market
- `changeOutcomeDescription` / `changeQuestionDescription` — update text

## Design
- Follows Conditional Token Framework patterns (like Polymarket)
- Uses the same clearinghouse/margin infrastructure as perps
- Settlement is governance-controlled
- Token splitting/merging for position management
- `MainOrHip3::Main(4)` vs `MainOrHip3::Hip3(3)` distinguishes asset types

## Links
- [[Exchange State]] — outcome state in exchange
- [[Action Types]] — userOutcome variant
- [[Clearinghouse]] — margin infrastructure shared

#predictions #outcomes #markets
