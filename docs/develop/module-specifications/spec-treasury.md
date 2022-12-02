---
sidebarDepth: 2
---

# Treasury

:::{admonition} Important
:class: danger
The treasury module logic is no longer effectively used by the Terra protocol, with the exception of the burn tax mentioned below. On March 3rd, 2021, the Terra community passed [governance proposal 43](https://station.terra.money/proposal/43), updating the seigniorage reward weight to burn all seigniorage. On January 6th, 2022, the Terra community passed [proposal 172](https://station.terra.money/proposal/172), which changed the stability fee tax rate to zero. Neither seigniorage nor the tax rate are currently used. The information in this section is kept as reference. Although the rates and parameters used in this section no longer have any effect on the protocol or transactions, they are still calculated as their logic is intact. The effective rates of seigniorage and stability fees are calculated as zero.  

On June 10th, 2022, the Terra community passed [governance proposal 3568](https://station.terra.money/proposal/3568), creating a burn tax.  The burn tax, a tax that is sent to burn, as developed in the code in both Terra Auth and Treasury modules, applies to all unbonded denominations (beginning at block height 9346889, approximately Sept 12, 2022).  It is an extension of the stability fee tax, and uses each of the PolicyConstraints keys within the TaxPolicy parameter (with the intention that all rates are set to the same value, and the cap is set to an appropriate value).
:::


The Treasury module acts as the "central bank" of the Terra economy, measuring macroeconomic activity by [observing indicators](#observed-indicators) and adjusting [monetary policy levers](#monetary-policy-levers) to modulate miner incentives toward stable, long-term growth.

:::{Important}
While the Treasury stabilizes miner demand by adjusting rewards, the [`market`](./spec-market.md) module is responsible for Terra price stability through arbitrage and the market maker.
:::

## Concepts

### Observed indicators

The treasury observes three macroeconomic indicators for each epoch and keeps [indicators](#indicators) of their values during previous epochs:

- **Tax Rewards**: $T$, the income generated from transaction and stability fees during an epoch.
- **Seigniorage Rewards**: $S$, the amount of seigniorage generated from Luna swaps to Terra during an epoch which is destined for ballot rewards inside the `Oracle` rewards. As of Columbus-5, all seigniorage is burned.
- **Total Staked Luna**: $\lambda$, the total amount of Luna staked by users and bonded to their delegated validators.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

These indicators can be used to derive two other values, the **Tax Reward per unit Luna** represented by $\tau = T / \lambda$, used in [Updating Tax Rate](#kupdatetaxpolicy), and **total mining rewards** $R = T + S$: the sum of the Tax Rewards and the Seigniorage Rewards, used in [Updating Reward Weight](#kupdaterewardpolicy).

The protocol can compute and compare the short-term ([`WindowShort`](#windowshort)) and the long-term ([`WindowLong`](#windowlong)) rolling averages of the above indicators to determine the relative direction and velocity of the Terra economy.

### Monetary policy levers

- **Tax Rate**: $r$, adjusts the amount of income gained from Terra transactions, limited by [_tax cap_](#tax-caps).

::: {admonition} Note
:class: warning
As of [proposal 172](https://station.terra.money/proposal/172), the stability fee tax rate is zero.
:::

- **Reward Weight**: $w$, the portion of seigniorage allocated to the reward pool for [`Oracle`](spec-oracle.md) vote winners. This is given to validators who vote within the reward band of the weighted median exchange rate.

:::{Important}
As of Columbus-5, all seigniorage is burned and no longer funds the community pool or the oracle reward pool. Validators are rewarded for faithful oracle votes through swap fees.
:::

### Updating policies


Both [Tax Rate](#tax-rate) and [Reward Weight](#reward-weight) are stored as values in the `KVStore` and can have their values updated through [governance proposals](#proposals) after they have passed. The Treasury recalibrates each lever once per epoch to stabilize unit returns for Luna, ensuring predictable mining rewards from staking:

- For tax rate, to ensure that unit-mining rewards do not stay stagnant, the treasury adds a [`MiningIncrement`](#miningincrement) so mining rewards increase steadily over time, as described in [k.updatetaxpolicy](#kupdatetaxpolicy).

- For reward weight, the treasury observes the portion of seigniorage needed to bear the overall reward profile, [`SeigniorageBurdenTarget`](#seigniorageburdentarget) and raises rates accordingly, as described in [k.updaterewardpolicy](#kupdaterewardpolicy). The current reward weight is `1`.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::
### Probation

A probationary period specified by the [`WindowProbation`](#windowprobation) prevents the network from updating the tax rate and reward weight during the first epochs after genesis to allow the blockchain to first obtain a critical mass of transactions and a mature, reliable history of indicators.


## Data

### Policy constraints

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

Policy updates from governance proposals and automatic calibration are constrained by the [`TaxPolicy`](#taxpolicy) and [`RewardPolicy`](#rewardpolicy) parameters, respectively. `PolicyConstraints` specifies the floor, ceiling, and max periodic changes for each variable.

```go
// PolicyConstraints defines constraints around updating a key Treasury variable
type PolicyConstraints struct {
    RateMin       sdk.Dec  `json:"rate_min"`
    RateMax       sdk.Dec  `json:"rate_max"`
    Cap           sdk.Coin `json:"cap"`
    ChangeRateMax sdk.Dec  `json:"change_max"`
}
```

The logic for constraining a policy lever update is done by `pc.Clamp()`.

```go
// Clamp constrains a policy variable update within the policy constraints
func (pc PolicyConstraints) Clamp(prevRate sdk.Dec, newRate sdk.Dec) (clampedRate sdk.Dec) {
	if newRate.LT(pc.RateMin) {
		newRate = pc.RateMin
	} else if newRate.GT(pc.RateMax) {
		newRate = pc.RateMax
	}

	delta := newRate.Sub(prevRate)
	if newRate.GT(prevRate) {
		if delta.GT(pc.ChangeRateMax) {
			newRate = prevRate.Add(pc.ChangeRateMax)
		}
	} else {
		if delta.Abs().GT(pc.ChangeRateMax) {
			newRate = prevRate.Sub(pc.ChangeRateMax)
		}
	}
	return newRate
}
```

## Proposals

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

The Treasury module defines special proposals which allow the [Tax Rate](#tax-rate) and [Reward Weight](#reward-weight) values in the `KVStore` to be voted on and changed accordingly, subject to the [policy constraints](#policy-constraints) imposed by `pc.Clamp()`.

### TaxRateUpdateProposal

```go
type TaxRateUpdateProposal struct {
	Title       string  `json:"title" yaml:"title"`             // Title of the Proposal
	Description string  `json:"description" yaml:"description"` // Description of the Proposal
	TaxRate     sdk.Dec `json:"tax_rate" yaml:"tax_rate"`       // target TaxRate
}
```

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

## State

### Tax rate

- type: `Dec`
- min: .1%
- max: 1%

The value of the tax rate policy lever for the current epoch.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### Reward Weight

- type: `Dec`
- default: `1`

The value of the reward weight policy lever for the current epoch. As of Columbus-5, the reward weight is `1`.

### Tax caps

- type: `map[string]Int`

The treasury keeps a `KVStore` that maps a denomination `denom` to an `sdk.Int`, which represents the maximum income that can be generated from taxes on a transaction in that same denomination. It is updated every epoch with the equivalent value of [`TaxPolicy.Cap`](#taxpolicy) at the current exchange rate.

For example, if a transaction's value is 100 SDT with a tax rate of 5% and a tax cap of 1 SDT, the income generated is 1 SDT, not 5 SDT.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### Tax Proceeds

- type: `Coins`

The tax rewards $T$ for the current epoch.

### Epoch initial issuance

- type: `Coins`

The total supply of Luna at the beginning of the current epoch. This value is used in [`k.SettleSeigniorage()`](#ksettleseigniorage) to calculate the seigniorage distributed at the end of each epoch. As of Columbus 5, all seigniorage is burned.

Recording the initial issuance automatically uses the supply module to determine the total issuance of Luna. Peeking returns the epoch's initial issuance of µLuna as `sdk.Int` instead of `sdk.Coins` for clarity.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### Indicators

The Treasury keeps track of the following indicators for the present and previous epochs:

#### Tax rewards

- type: `Dec`

The Tax Rewards $T$ for each `epoch`.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

#### Seigniorage Rewards

- type: `Dec`

The seigniorage rewards $S$ for each `epoch`.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

#### Total Staked Luna

- type: `Int`

The total staked Luna $\lambda$ for each `epoch`.

## Functions

### `k.UpdateIndicators()`

```go
func (k Keeper) UpdateIndicators(ctx sdk.Context)
```

At the end of each epoch $t$, this function records the current values of tax rewards $T$, seigniorage rewards $S$, and total staked Luna $\lambda$ before moving to the next epoch $t+1$.

- $T_t$ is the current value in [`TaxProceeds`](#tax-proceeds).
- $S_t = \Sigma * w$, with epoch seigniorage $\Sigma$ and reward weight $w$.
- $\lambda_t$ is the result of `staking.TotalBondedTokens()`.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### `k.UpdateTaxPolicy()`

```go
func (k Keeper) UpdateTaxPolicy(ctx sdk.Context) (newTaxRate sdk.Dec)
```

At the end of each epoch, this function calculates the next value of the tax rate monetary lever.

Using $r_t$ as the current tax rate and $n$ as the [`MiningIncrement`](#miningincrement) parameter:

1. Calculate the rolling average $\tau_y$ of tax rewards per unit Luna over the last year `WindowLong`.

2. Calculate the rolling average $\tau_m$ of tax rewards per unit Luna over the last month `WindowShort`.

3. If $\tau_m = 0$, no tax revenue occurred in the last month. The tax rate should be set to the maximum permitted by the tax policy, subject to the rules of `pc.Clamp()`. For more information, see [constraints](#policy-constraints).

4. If $\tau_m > 0$, the new Tax Rate is $r_{t+1} = (n r_t \tau_y)/\tau_m$, subject to the rules of `pc.Clamp()`. For more information, see [constraints](#policy-constraints).

When monthly tax revenues dip below the yearly average, the treasury increases the tax rate. When monthly tax revenues go above the yearly average, the treasury decreases the tax rate.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### `k.UpdateRewardPolicy()`

```go
func (k Keeper) UpdateRewardPolicy(ctx sdk.Context) (newRewardWeight sdk.Dec)
```

At the end of each epoch, this function calculates the next value of the Reward Weight monetary lever.

Using $w_t$ as the current reward weight, and $b$ as the [`SeigniorageBurdenTarget`](#seigniorageburdentarget) parameter:

1. Calculate the sum $S_m$ of seigniorage rewards over the last month `WindowShort`.

2. Calculate the sum $R_m$ of total mining rewards over the last month `WindowShort`.

3. If $R_m = 0$ and $S_m = 0$, no mining or seigniorage rewards occurred in the last month. The reward weight should be set to the maximum permitted by the reward policy, subject to the rules of `pc.Clamp()`. For more information, see [constraints](#policy-constraints).

4. If $R_m > 0$ or $S_m > 0$, the new Reward Weight is $w_{t+1} = b w_t S_m / R_m$, subject to the rules of `pc.Clamp()`. For more information, see [constraints](#policy-constraints).


::: {Important}
As of Columbus-5, all seigniorage is burned and no longer funds the community or reward pools.
:::

### `k.UpdateTaxCap()`

```go
func (k Keeper) UpdateTaxCap(ctx sdk.Context) sdk.Coins
```

This function is called at the end of an epoch to compute the Tax Caps for every denomination for the next epoch.

For every denomination in circulation, the new Tax Cap for each denomination is set to be the global Tax Cap defined in the [`TaxPolicy`](#taxpolicy) parameter, at current exchange rates.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### `k.SettleSeigniorage()`

```go
func (k Keeper) SettleSeigniorage(ctx sdk.Context)
```

This function is called at the end of an epoch to compute seigniorage and forwards the funds to the [oracle module](spec-oracle.md) module for ballot rewards and the [distribution module](spec-distribution.md) for the community pool.

1. The seigniorage $\Sigma$ of the current epoch is calculated by taking the difference between the Luna supply at the start of the epoch ([epoch initial issuance](#epoch-initial-issuance)) and the Luna supply at the time of calling.

    $\Sigma > 0$ when the current Luna supply is lower than it was at the start of the epoch because the Luna had been burned from Luna swaps into Terra. For more information, see [seigniorage](spec-market.md#seigniorage).

2. The reward weight $w$ is the percentage of the seigniorage designated for ballot rewards. Amount $S$ of new Luna is minted, and the [oracle module](spec-oracle.md) receives $S = \Sigma * w$ of the seigniorage.

3. The remainder of the coins $\Sigma - S$ is sent to the [distribution module](spec-distribution.md) , where it is allocated into the community pool.

::: {Important}
As of Columbus-5, all seigniorage is burned and no longer funds the community pool or the oracle reward pool. Validators are rewarded for faithful oracle votes through swap fees.
:::

## Transitions

### End-Block

If the blockchain is at the final block of the epoch, the following procedure is run:

1. Update all the indicators with [`k.UpdateIndicators()`](#kupdateindicators)

2. If the current block is under [probation](#probation), skip to step 6.

3. [Settle seigniorage](#ksettleseigniorage) accrued during the epoch and make funds available to ballot rewards and the community pool during the next epoch. As of Columbus-5, all seigniorage is burned.

4. Calculate the [Tax Rate](#kupdatetaxpolicy), [Reward Weight](#kupdaterewardpolicy), and [Tax Cap](#kupdatetaxcap) for the next epoch. As of Columbus-5, all seigniorage is burned, and the reward weight is `1`.

5. Emit the `policy_update` event, recording the new policy lever values.

6. Finally, record the Luna issuance with [`k.RecordEpochInitialIssuance()`](#epoch-initial-issuance). It will be used to calculate the seigniorage for the next epoch.


| Type          | Attribute Key | Attribute Value |
| ------------- | ------------- | --------------- |
| policy_update | tax_rate      | {taxRate}       |
| policy_update | reward_weight | {rewardWeight}  |
| policy_update | tax_cap       | {taxCap}        |


::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

## Parameters

The subspace for the Treasury module is `treasury`.

```go
type Params struct {
	TaxPolicy               PolicyConstraints `json:"tax_policy" yaml:"tax_policy"`
	RewardPolicy            PolicyConstraints `json:"reward_policy" yaml:"reward_policy"`
	SeigniorageBurdenTarget sdk.Dec           `json:"seigniorage_burden_target" yaml:"seigniorage_burden_target"`
	MiningIncrement         sdk.Dec           `json:"mining_increment" yaml:"mining_increment"`
	WindowShort             int64             `json:"window_short" yaml:"window_short"`
	WindowLong              int64             `json:"window_long" yaml:"window_long"`
	WindowProbation         int64             `json:"window_probation" yaml:"window_probation"`
}
```

### TaxPolicy

- type: `PolicyConstraints`
- default:

```go
DefaultTaxPolicy = PolicyConstraints{
    RateMin:       sdk.NewDecWithPrec(5, 4), // 0.05%
    RateMax:       sdk.NewDecWithPrec(1, 2), // 1%
    Cap:           sdk.NewCoin(core.MicroSDRDenom, sdk.OneInt().MulRaw(core.MicroUnit)), // 1 SDR Tax cap
    ChangeRateMax: sdk.NewDecWithPrec(25, 5), // 0.025%
}
```

Constraints for updating the [tax rate](#tax-rate) monetary policy lever.

### RewardPolicy

- type: `PolicyConstraints`
- default:

```go
DefaultRewardPolicy = PolicyConstraints{
    RateMin:       sdk.NewDecWithPrec(5, 2), // 5%
    RateMax:       sdk.NewDecWithPrec(90, 2), // 90%
    ChangeRateMax: sdk.NewDecWithPrec(25, 3), // 2.5%
    Cap:           sdk.NewCoin("unused", sdk.ZeroInt()), // UNUSED
}
```

Constraints for updating the [reward weight](#reward-weight) monetary policy lever.

### SeigniorageBurdenTarget

- type: `sdk.Dec`
- default: 67%

Multiplier specifying the portion of burden seigniorage needed to bear the overall reward profile for Reward Weight updates during epoch transition.

::: {admonition} Note
:class: warning
As of proposals [43](https://station.terra.money/proposal/43) and [172](https://station.terra.money/proposal/172), all seigniorage is burned, and the stability fee tax rate is zero.   
:::

### MiningIncrement

- type: `sdk.Dec`
- default: 1.07 growth rate, 15% CAGR of $\tau$

Multiplier determining an annual growth rate for tax rate policy updates during epoch transition.

### WindowShort

- type: `int64`
- default: `4` (month = 4 weeks)

A number of epochs that specifies a time interval for calculating the short-term moving average.

### WindowLong

- type: `int64`
- default: `52` (year = 52 weeks)

A number of epochs that specifies a time interval for calculating the long-term moving average.

### WindowProbation

- type: `int64`
- default: `18`

A number of epochs that specifies a time interval for the probationary period.
