## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-43

# [Deploy flow of Talos is  broken](https://github.com/code-423n4/2023-05-maia-findings/issues/75) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/base/TalosBaseStrategy.sol#L83


# Vulnerability details

## Impact
Talos protocol can't be deployed in a right way.

## Proof of Concept
TalosBaseStrategy needs TalosManager to be passed in constructor:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/base/TalosBaseStrategy.sol#L79-L95
```solidity
    constructor(
        IUniswapV3Pool _pool,
        ITalosOptimizer _optimizer,
        INonfungiblePositionManager _nonfungiblePositionManager,
        address _strategyManager,
        address _owner
    ) ERC20("TALOS LP", "TLP", 18) {
        ...

        strategyManager = _strategyManager;
        
        ...
    }
```

But TalosManager needs Strategy to be passed in constructor:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/talos/TalosManager.sol#L44-L56
```solidity
   constructor(
        address _strategy,
        int24 _ticksFromLowerRebalance,
        int24 _ticksFromUpperRebalance,
        int24 _ticksFromLowerRerange,
        int24 _ticksFromUpperRerange
    ) {
        strategy = ITalosBaseStrategy(_strategy);
        
        ...
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add setters for complete deploy, or initializing function


## Assessed type

DoS