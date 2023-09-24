## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [Many `create` methods are suspicious of the reorg attack](https://github.com/code-423n4/2023-05-maia-findings/issues/861) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyStaked.sol#L28


# Vulnerability details

## Proof of Concept

There are many instance of this, but to understand things better, taking the example of `createTalosV3Strategy` method.

The `createTalosV3Strategy` function deploys a new `TalosStrategyStaked` contract using the create, where the address derivation depends only on the arguments passed.

At the same time, some of the chains like Arbitrum and Polygon are suspicious of the reorg attack.

```solidity
File: TalosStrategyStaked.sol

  function createTalosV3Strategy(
        IUniswapV3Pool pool,
        ITalosOptimizer optimizer,
        BoostAggregator boostAggregator,
        address strategyManager,
        FlywheelCoreInstant flywheel,
        address owner
    ) public returns (TalosBaseStrategy) {
        return new TalosStrategyStaked( // @audit-issue Reorg Attack
                pool,
                optimizer,
                boostAggregator,
                strategyManager,
                flywheel,
                owner
            );
    }

```
[Link to Code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyStaked.sol#L28)

Even more, the reorg can be couple of minutes long. So, it is quite enough to create the `TalosStrategyStaked` and transfer funds to that address using `deposit` method, especially when someone uses a script, and not doing it by hand.

Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation.

Same issue can affect factory contracts in Ulysses omnichain contracts as well with more severe consequences.

Can refer this an issue previously report [here](https://code4rena.com/reports/2023-04-frankencoin#m-14-re-org-attack-in-factory) to have more understanding about it.

## Impact

Exploits involving Stealing of funds.

## Tools Used

VS Code

## Recommended Mitigation Steps

Deploy such contracts via `create2` with `salt`.


## Assessed type

Other