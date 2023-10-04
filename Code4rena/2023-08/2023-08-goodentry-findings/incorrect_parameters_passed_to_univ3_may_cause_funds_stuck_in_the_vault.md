## Tags

- bug
- 2 (Med Risk)
- grade-b
- primary issue
- selected for report
- M-02

# [Incorrect parameters passed to UniV3 may cause funds stuck in the vault](https://github.com/code-423n4/2023-08-goodentry-findings/issues/397) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/TokenisableRange.sol#L306-L313


# Vulnerability details

## Impact
Note: this issue happened on the deployed version of GoodEntry and was discovered when using https://alpha.goodentry.io

Due to incorrect parameters and validation when working with UniV3 LP the vault may enter a state where rebalancing reverts. This means any deposits and withdrawals from the vault become unavailable.

### Code walkthrough

When rebalancing a vault, the existing positions need to be removed from Uni. This is done in [removeFromTick](https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/GeVault.sol#L321) function.
```solidity
    if (aBal > 0){
      lendingPool.withdraw(address(tr), aBal, address(this));
      tr.withdraw(aBal, 0, 0);
    }
```
Here, zeros are passed as `amount0Min` and `amount1Min` arguments. The execution continues in [TokenisableRange.withdraw](https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/TokenisableRange.sol#L292) function. `decreaseLiquidity` is called to remove liquidity from Uni.
```solidity
    (removed0, removed1) = POS_MGR.decreaseLiquidity(
      INonfungiblePositionManager.DecreaseLiquidityParams({
        tokenId: tokenId,
        liquidity: uint128(removedLiquidity),
        amount0Min: amount0Min,
        amount1Min: amount1Min,
        deadline: block.timestamp
      })
    );
```
Here there is an edge-case that for really small change in liquidity the returned values `removed0` and `removed1` can be 0s (will be explained at the end of a section).

Then, `collect` is called and `removed0`,`removed1` are passed as arguments.
```solidity
    POS_MGR.collect( 
      INonfungiblePositionManager.CollectParams({
        tokenId: tokenId,
        recipient: msg.sender,
        amount0Max: uint128(removed0),
        amount1Max: uint128(removed1)
      })
    );
```
However, `collect` reverts when both of these values are zeros - https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L316

As a result, any deposit/withdraw/rebalancing of the vault will revert when it will be attempting to remove existing liquidity.

### When can decreaseLiquidity return 0s

This edge-case is possible to achieve as it happened in the currently deployed alpha version of the product. The sponsor confirmed that the code deployed is the same as presented for the audit.

The tick that caused the revert has less than a dollar of liquidity. Additionally, that tick has outstanding debt and so the `aBal` value was small enough to cause the issue. In the scenario that happened on-chain aBal is only *33446*.
```solidity
    uint aBal = ERC20(aTokenAddress).balanceOf(address(this));
    uint sBal = tr.balanceOf(aTokenAddress);

    // if there are less tokens available than the balance (because of outstanding debt), withdraw what's available
    if (aBal > sBal) aBal = sBal;
    if (aBal > 0){
      lendingPool.withdraw(address(tr), aBal, address(this));
      tr.withdraw(aBal, 0, 0);
    }
```

## Proof of Concept
The issue can be demonstrated on the contracts deployed on the arbitrum mainnet. This is the foundry test
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

interface IGeVault {
    function withdraw(uint liquidity, address token) external returns (uint amount);
}

contract GeVaultTest is Test {
    IGeVault public vault;

    function setUp() public {
        vault = IGeVault(0xdcc16DEfe27cd4c455e5520550123B4054D1b432);
        // 0xdcc16DEfe27cd4c455e5520550123B4054D1b432 btc vault
    }

    function testWithdraw() public {
        // Account that has a position in the vault
        vm.prank(0x461F5f86026961Ee7098810CC7Ec07874077ACE6);

        // Trying to withdraw 1 USDC
        vault.withdraw(1e6, 0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8);
    }
}

```

The test can be executed by forking the arbitrum mainnet
```
forge test -vvv --fork-url <your_arb_rpc> --fork-block-number 118634741
```

The result is an error in UniV3 `NonfungiblePosition.collect` method
```
   │   │   │   ├─ [1916] 0xC36442b4a4522E871399CD717aBDD847Ab11FE88::collect((697126, 0xdcc16DEfe27cd4c455e5520550123B4054D1b432, 0, 0)) 
    │   │   │   │   └─ ← "EvmError: Revert"
```

## Tools Used

E2E testing, then code review

## Recommended Mitigation Steps

It is unclear why this `collect` call is needed because the fees are already collected a few lines above in `claimFees`. I suggest removing the second `collect` call altogether. If it's needed then perhaps only collect if one of removed0/removed1 is non-zero.


## Assessed type

Uniswap