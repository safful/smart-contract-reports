## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-02

# [Unused funds are not returned and not counted in `GeVault`](https://github.com/code-423n4/2023-08-goodentry-findings/issues/325) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L407


# Vulnerability details

## Impact
Users can lose a portion of their deposited funds if some of their funds haven't been deposited to the underlying Uniswap pools. There's always a chance of such event since Uniswap pools take balanced token amounts when liquidity is added but `GeVault` doesn't pre-compute balanced amounts. As a result, depositing and withdrawing can result in a partial loss of funds.
## Proof of Concept
The [GeVault.deposit()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L247) function is used by users to deposits funds into ticks and underlying Uniswap pools. The function takes funds from the caller and calls `rebalance()` to distribute the funds among the ticks. The [GeVault.rebalance()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L202) function first removes liquidity from all ticks and then deposits the removed assets plus the user assets back in to the ticks:
```solidity
function rebalance() public {
    require(poolMatchesOracle(), "GEV: Oracle Error");
    removeFromAllTicks();
    if (isEnabled) deployAssets();
}
```

The `GeVault.deployAssets()` function calls the [GeVault.depositAndStash()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L404) function, which actually deposits tokens into a `TokenisableRange` contract by calling the [TokenisableRange.deposit()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L222). The function deposits tokens into a Uniswap V3 pool and returns unspent tokens to the caller:
```solidity
(uint128 newLiquidity, uint256 added0, uint256 added1) = POS_MGR.increaseLiquidity(
    ...
);

...

_mint(msg.sender, lpAmt);
TOKEN0.token.safeTransfer( msg.sender, n0 - added0);
TOKEN1.token.safeTransfer( msg.sender, n1 - added1);
```

However, the `GeVault.depositAndStash()` function doesn't handle the returned unspent tokens. Since Uniswap V3 pools take balanced token amounts (respective to the current pool price) and since the funds deposited into ticks are not balanced ([`deployAssets()` splits token amounts in halves](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L353-L358)), there's always a chance that the `TokenisableRange.deposit()` function won't consume all specified tokens and will return some of them to the `GeVault` contract. However, `GeVault` won't return the unused tokens to the depositor.

Moreover, the contract won't include them in the TVL calculation:
1. The [GeVault.getTVL()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L392) function computes the total LP token balance of the contract (`getTickBalance(k)`) and the price of each LP token (`t.latestAnswer()`), to compute the total value of the vault.
1. The [GeVault.getTickBalance()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L420) function won't count the unused tokens because it only returns the amount of LP tokens deposited into the lending pool. I.e. only the liquidity deposited to Uniswap pools is counted.
1. The [TokenisableRange.latestAnswer()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L361) function computes the total value ([TokenisableRange.sol#L355](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L355)) of the liquidity deposited into the Uniswap pool ([TokenisableRange.sol#L338](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/TokenisableRange.sol#L338)). Thus, the unused tokens won't be counted here as well.
1. The [GeVault.getTVL()](https://github.com/code-423n4/2023-08-goodentry/blob/4b785d455fff04629d8675f21ef1d1632749b252/contracts/GeVault.sol#L220-L223) function is used to compute the amount of tokens to return to the depositor during withdrawal.

Thus, the unused tokens will be locked in the contract until they're deposited into ticks. However, rebalancing and depositing of tokens can result in new unused tokens that won't be counted in the TVL.

## Tools Used
Manual review
## Recommended Mitigation Steps
In the `GeVault.deposit()` function, consider returning unspent tokens to the depositor. Extra testing is needed to guarantee that rebalancing doesn't result in unspent tokens, or, alternatively, such tokens could be counted in a storage variable and excluded from the balance of unspent tokens during depositing.
Alternatively, consider counting `GeVault`'s token balances in the `getTVL()` function. This won't require returning unspent tokens during depositing and will allow depositors to withdraw their entire funds.


## Assessed type

Token-Transfer