## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor disputed
- M-01

# [Router.getOrCreatePoolAndAddLiquidity can be frontrunned which leads to price manipulation](https://github.com/code-423n4/2022-12-Stealth-Project-findings/issues/22) 

# Lines of code

https://github.com/code-423n4/2022-12-Stealth-Project/blob/main/router-v1/contracts/Router.sol#L277-L287


# Vulnerability details

## Impact
Router.getOrCreatePoolAndAddLiquidity can be frontrunned which allows to set different initial price.

## Proof of Concept
When new Pool is created it is already initialized with [active tick](https://github.com/code-423n4/2022-12-Stealth-Project/blob/main/maverick-v1/contracts/models/Pool.sol#L76). This allows the pool creator to provide price for assets.
So when first LP calls Pool.addLiquidity, this active tick [is used]( to determine how many assets LP should deposit.

Router.getOrCreatePoolAndAddLiquidity function allows caller to add liquidity to the pool that can be not created yet. In such case it will create it with initial active tick provided by sender and then it will provide user's liquidity to the pool. In case if pool already exists, function will just add liquidity to it.

https://github.com/code-423n4/2022-12-Stealth-Project/blob/main/router-v1/contracts/Router.sol#L277-L287
```solidity
    function getOrCreatePoolAndAddLiquidity(
        PoolParams calldata poolParams,
        uint256 tokenId,
        IPool.AddLiquidityParams[] calldata addParams,
        uint256 minTokenAAmount,
        uint256 minTokenBAmount,
        uint256 deadline
    ) external payable checkDeadline(deadline) returns (uint256 receivingTokenId, uint256 tokenAAmount, uint256 tokenBAmount, IPool.BinDelta[] memory binDeltas) {
        IPool pool = getOrCreatePool(poolParams);
        return addLiquidity(pool, tokenId, addParams, minTokenAAmount, minTokenBAmount);
    }
```

1.Pool of tokenA:tokenB doens't exist.
2.User calls Router.getOrCreatePoolAndAddLiquidity function to create pool with initial active tick and provide liquidity in same tx.
3.Attacker frontruns tx and creates such pool with different active tick.
4.Router.getOrCreatePoolAndAddLiquidity is executed and new Pool was not created, and liquidity was added to the Pool created by attacker.
5.Attacker swapped provided by first depositor tokens for a good price.
6.First depositor lost part of funds.

## Tools Used
VsCode
## Recommended Mitigation Steps
Do not allow to do such actions together in one tx. Do it in 2 tx. First, create Pool. And in second tx add liquidity.