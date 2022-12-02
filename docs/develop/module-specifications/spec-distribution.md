# Distribution

::: {important}
Terra's distribution module inherits from Cosmos SDK's [`distribution`](https://docs.cosmos.network/master/modules/distribution/) module. This document is a stub and covers mainly important Terra-specific notes about how it is used.
:::

The distribution module describes a mechanism that tracks collected fees and passively distributes them to validators and delegators. Additionally, the distribution module defines the [community pool](#community-pool), which is a pool of funds under the control of on-chain governance.

## Concepts

### Validator and delegator rewards

:::{important}
Passive distribution means that validators and delegators need to manually collect their fee rewards by [submitting withdrawal transactions](../how-to/terrad/subcommands.md#tx-distribution-withdraw-rewards).
:::

Collected rewards are pooled globally and distrubuted to validators and delegators. Each validator has the opportunity to charge delegators commission on the rewards collected on behalf of the delegators. Fees are collected directly into a global reward pool and a validator proposer-reward pool. Due to the nature of passive accounting, whenever changes to parameters which affect the rate of reward distribution occur, withdrawal of rewards must also occur.

### Community pool

The community pool is a reserve of tokens designated for funding projects that promote further adoption and stimulate growth for the Terra economy. The comminuty pool used to be funded by seigniorage. As of Columbus-5, all seigniorage is burned, and the community pool no longer receives funding.

## State

> This section was taken from the official Cosmos SDK docs, and placed here for your convenience to understand the distribution module's parameters and genesis variables.

### FeePool

All globally tracked parameters for distribution are stored within
`FeePool`. Rewards are collected and added to the reward pool and
distributed to validators/delegators from here.

Note that the reward pool holds decimal coins (`DecCoins`) to allow
for fractions of coins to be received from operations like inflation.
When coins are distributed from the pool they are truncated back to
`sdk.Coins` which are non-decimal.

### Validator distribution

The following validator distribution information for the relevant validator is updated each time:

- Delegation amount to a validator is updated,
- A validator successfully proposes a block and receives a reward.
- Any delegator withdraws from a validator or the validator withdraws its commission.

### Delegation distribution

Each delegation distribution only needs to record the height at which it last
withdrew fees. Because a delegation must withdraw fees each time its
properties change (aka bonded tokens etc.), its properties will remain constant
and the delegator's accumulation factor can be calculated passively knowing
only the height of the last withdrawal and its current properties.

## Message types

### MsgSetWithdrawAddress

```go
type MsgSetWithdrawAddress struct {
	DelegatorAddress sdk.AccAddress `json:"delegator_address" yaml:"delegator_address"`
	WithdrawAddress  sdk.AccAddress `json:"withdraw_address" yaml:"withdraw_address"`
}
```


### MsgWithdrawDelegatorReward

```go
// msg struct for delegation withdraw from a single validator
type MsgWithdrawDelegatorReward struct {
	DelegatorAddress sdk.AccAddress `json:"delegator_address" yaml:"delegator_address"`
	ValidatorAddress sdk.ValAddress `json:"validator_address" yaml:"validator_address"`
}
```


### MsgWithdrawValidatorCommission

```go
type MsgWithdrawValidatorCommission struct {
	ValidatorAddress sdk.ValAddress `json:"validator_address" yaml:"validator_address"`
}
```


### MsgFundCommunityPool

```go
type MsgFundCommunityPool struct {
	Amount    sdk.Coins      `json:"amount" yaml:"amount"`
	Depositor sdk.AccAddress `json:"depositor" yaml:"depositor"`
}
```


## Proposals

### CommunityPoolSpendProposal

The distribution module defines a special proposal that, upon being passed, disburses the coins specified in `Amount` to the `Recipient` account using funds from the community pool.

```go
type CommunityPoolSpendProposal struct {
	Title       string         `json:"title" yaml:"title"`
	Description string         `json:"description" yaml:"description"`
	Recipient   sdk.AccAddress `json:"recipient" yaml:"recipient"`
	Amount      sdk.Coins      `json:"amount" yaml:"amount"`
}
```

## Transitions

### Begin-Block

> This section derives from the official Cosmos SDK docs, and is placed here for your convenience to understand the distribution module's parameters.

At the beginning of each block, the distribution module will set the proposer for determining distribution during endblock and distribute rewards for the previous block.

The fees received are transferred to the Distribution `ModuleAccount`, which tracks the flow of coins in and out of the module. Fees are also allocated to the proposer, community fund, and global pool:

- Proposer: When a validator is the proposer of a round, that validator and its delegators receive 1-5% of the fee rewards.
- Community fund: The reserve community tax is charged and distributed to the community pool. As of Columbus-5, this tax is no longer charged and the community pool no longer receives funding.
- Global pool: The remainder of the funds is allocated to the global pool, where they are distributed proportionally by voting power to all bonded validators independent of whether they voted. This allocation is called social distribution. Social distribution is applied to the proposer validator in addition to the proposer reward.

The proposer reward is calculated from precommits Tendermint messages to incentivize validators to wait and include additional precommits in the block. All provision rewards are added to a provision reward pool, which each validator holds individually (`ValidatorDistribution.ProvisionsRewardPool`).

```go
func AllocateTokens(feesCollected sdk.Coins, feePool FeePool, proposer ValidatorDistribution,
              sumPowerPrecommitValidators, totalBondedTokens, communityTax,
              proposerCommissionRate sdk.Dec)

     SendCoins(FeeCollectorAddr, DistributionModuleAccAddr, feesCollected)
     feesCollectedDec = MakeDecCoins(feesCollected)
     proposerReward = feesCollectedDec * (0.01 + 0.04
                       * sumPowerPrecommitValidators / totalBondedTokens)

     commission = proposerReward * proposerCommissionRate
     proposer.PoolCommission += commission
     proposer.Pool += proposerReward - commission

     communityFunding = feesCollectedDec * communityTax
     feePool.CommunityFund += communityFunding

     poolReceived = feesCollectedDec - proposerReward - communityFunding
     feePool.Pool += poolReceived

     SetValidatorDistribution(proposer)
     SetFeePool(feePool)
```

## Parameters

::: {note}
On June 15, 2022, the Terra community passed [governance proposal 4080](https://station.terra.money/proposal/4080) which changed the following parameters via a parameter proposal:  "CommunityTax":"0.5"; "BaseProposerReward":"0.03"; "BonusProposerReward":"0.12".  You may find it interesting to also look at the Cosmos [distribution](https://docs.cosmos.network/v0.44/modules/distribution/03_begin_block.html#reward-to-the-community-pool) module documentation regarding these parameters.
:::

The subspace for the distribution module is `distribution`.

```go
type GenesisState struct {
	...
	CommunityTax        sdk.Dec `json:"community_tax" yaml:"community_tax"`
	BaseProposerReward	sdk.Dec `json:"base_proposer_reward" yaml:"base_proposer_reward"`
	BonusProposerReward	sdk.Dec	`json:"bonus_proposer_reward" yaml:"bonus_proposer_reward"`
	WithdrawAddrEnabled bool 	`json:"withdraw_addr_enabled"`
	...
}
```

### CommunityTax

- type: `Dec`

### BaseProposerReward

- type: `Dec`

### BonusProposerReward

- type: `Dec`

### WithdrawAddrEnabled

- type: `bool`
