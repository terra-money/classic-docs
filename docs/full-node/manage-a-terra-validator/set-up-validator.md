# Register a validator

This is a detailed step-by-step guide for setting up a Terra validator. Please be aware that while it is easy to set up a rudimentary validating node, running a production-quality validator node with a robust architecture and security features requires an extensive setup.

For more information on setting up a validator, see [additional resources](README.md#additional-resources).

## Prerequisites

- You have completed [how to run a full Terra node](../run-a-full-terra-node/README.md), which outlines how to install, connect, and configure a node.
- You are familiar with [terrad](../../develop/how-to/terrad/README.md).
- You have read through [the validator FAQ](faq.md)
- You understand the [different keys](faq.md#what-are-the-different-types-of-keys) of a validator in the FAQ

## 1. Retrieve your PubKey

The Consensus PubKey of your node is required to create a new validator. Run:

```bash
--pubkey=$(terrad tendermint show-validator)
```

## 2. Create a new validator

   :::{admonition} Get tokens
   :class: tip
   In order for Terrad to recognize a wallet address it must contain tokens. For the testnet, use [the faucet](https://faucet.terra.money/) to send Luna to your wallet. If you are on mainnet, send funds from an existing wallet. 1-3 luna are sufficient for most setup processes.
   :::

   ::: {note}
   Due to changes in classic-core v0.5.19 (and Terra's cosmos-sdk v0.44) on May 12th, 2022, messages that modified validator staking power on Terra were prevented, and the cosmos-sdk functions CreateValidator(), and Delegate() were blocked after block height 7603700.  This behavior has changed with classic-core v0.5.21 (via a dependence on, and changes made in, Terra's cosmos-sdk v0.44).  The changes made do the following:  re-enables the creation of validators (beginning at block height 9988390, approximately Oct 26, 2022), re-enables the ability to delegate stake to existing validators (beginning at block height 9109990, approximately Aug 26, 2022)(while preventing catch-up node consensus failure), and prevents any one validator from obtaining more than 20% of the staking power until future governance decides otherwise (as a preventative security measure).  You may find it interesting to look at the [Staking](../../develop/module-specifications/spec-staking.md) module, which covers the logic for staking and validators.
   :::

To create the validator and initialize it with a self-delegation, run the following command. `key-name` is the name of the Application Operator Key that is used to sign transactions.

```bash
terrad tx staking create-validator \
    --amount=5000000uluna \
    --pubkey=$(<your-consensus-PubKey>) \
    --moniker="<your-moniker>" \
    --chain-id=<chain_id> \
    --from=<key-name> \
    --commission-rate="0.10" \
    --commission-max-rate="0.20" \
    --commission-max-change-rate="0.01" \
    --min-self-delegation="1"
```

::: {warning}
When you specify commission parameters, the `commission-max-change-rate` is measured as a percentage-point change of the `commission-rate`. For example, a change from 1% to 2% is a 100% rate increase, but the `commission-max-change-rate` is measured as 1%.
:::

## 3. Confirm your validator is active

If running the following command returns something, your validator is active:

```bash
terrad query tendermint-validator-set | grep "$(terrad tendermint show-validator)"
```

You are looking for the `bech32` encoded `address` in the `~/.terra/config/priv_validator.json` file.

::: {note}
Only the top 130 validators in voting power are included in the active validator set.
:::

## 4. Secure your keys and have a backup plan

In general a validator needs to do three things well

- Sign and commit blocks (using the Tendermint Consensus key)
- Provide Oracle FX rates via a feeder (using an Application Oracle Feeder key)
- Conduct on-chain operations such as voting on Governance proposals (using an Application Operator Key)

Protecting and having a contingency backup plan for all your [keys](faq.md#what-are-the-different-types-of-keys) will help mitigate catastrophic hardware or software failures of the node.
It is a good practice to test your backup plan on a testnet node in case of node failure.

:::{include} restore.md
:::
