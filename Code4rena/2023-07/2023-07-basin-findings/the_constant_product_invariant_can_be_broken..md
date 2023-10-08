## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-05

# [The constant product invariant can be broken.](https://github.com/code-423n4/2023-07-basin-findings/issues/191) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/main/src/functions/ConstantProduct2.sol#L65-L66
https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L590-L598
https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L695-L702
https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L541
https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L562
https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L582


# Vulnerability details

## description
Let `reserves` returned by `Well._getReserves()` as x, y and `Well.tokenSupply()` as k. 
They must maintain the invariant `x * y * EXP_PRECISION = k ** 2`.
However, the reserves can increase without updating the token supply if a user transfers one token of the well and call `Well.sync()`.
We can sync the reserves and balances using `Well.sync`, but there is no way to sync `Well.tokenSupply() ** 2` to `x * y * EXP_PRECISION`.

## impact
`ConstantProduct2.calcReserve` assumes `Well.tokenSupply` equals to `reserves[0] * reserves[1]`.
This exception breaks the assumption and reverts normal transactions.
For example, when `Well.totalSupply` is less than `reserves[0] * reserves[1]`, [Well.removeLiquidityImbalanced](https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L562) may revert.

1. Comment out minting initial liquidity in `TestHelper.sol`. https://github.com/code-423n4/2023-07-basin/blob/main/test/TestHelper.sol#L107
```solidity
function setupWell(Call memory _wellFunction, Call[] memory _pumps, IERC20[] memory _tokens) internal {
    // ...
    // @audit comment out the line 107 to see apparently
    // Add initial liquidity from TestHelper
    // addLiquidityEqualAmount(address(this), initialLiquidity);
}
```

2. Add the test in `Well.AddLiquidity.t.sol` as below. https://github.com/code-423n4/2023-07-basin/blob/main/test/Well.AddLiquidity.t.sol#L9
```solidity
contract WellAddLiquidityTest is LiquidityHelper {
    function setUp() public {
        setupWell(2);
    }
    
    // @audit add this test
    function test_tokenSupplyError() public {
        IERC20[] memory tokens = well.tokens();
        Balances memory userBalance;
        Balances memory wellBalance = getBalances(address(well), well);

        console.log(wellBalance.lpSupply); // 0

        mintTokens(user, 10000000e18);

        vm.startPrank(user);
        tokens[0].transfer(address(well), 100);
        tokens[1].transfer(address(well), 100);
        vm.stopPrank();

        userBalance = getBalances(user, well);
        console.log(userBalance.lp); // 0

        addLiquidityEqualAmount(user, 1);

        userBalance = getBalances(user, well);
        console.log(userBalance.lp); // 1e6

        well.sync(); // reserves = [101, 101]

        uint256[] memory amounts = new uint256[](tokens.length);
        amounts[0] = 1;

        // FAIL: Arithmetic over/underflow
        vm.prank(user);
        well.removeLiquidityImbalanced(type(uint256).max, amounts, user, type(uint256).max);
    }
}
```

3. I commented the reason of underflow. https://github.com/code-423n4/2023-07-basin/blob/main/src/Well.sol#L562
```solidity
function removeLiquidityImbalanced(
    uint256 maxLpAmountIn,
    uint256[] calldata tokenAmountsOut,
    address recipient,
    uint256 deadline
) external nonReentrant expire(deadline) returns (uint256 lpAmountIn) {
    IERC20[] memory _tokens = tokens();
    uint256[] memory reserves = _updatePumps(_tokens.length);

    for (uint256 i; i < _tokens.length; ++i) {
        _tokens[i].safeTransfer(recipient, tokenAmountsOut[i]);
        reserves[i] = reserves[i] - tokenAmountsOut[i];
    }

    // @audit
    // 1e6 - sqrt(101 * (101 - 1) * 1000000 ** 2)
    // <=> 1e6 - 100498756
    lpAmountIn = totalSupply() - _calcLpTokenSupply(wellFunction(), reserves);
    if (lpAmountIn > maxLpAmountIn) {
        revert SlippageIn(lpAmountIn, maxLpAmountIn);
    }
    _burn(msg.sender, lpAmountIn);

    _setReserves(_tokens, reserves);
    emit RemoveLiquidity(lpAmountIn, tokenAmountsOut, recipient);
}
```

## tools used
Manual review.

## recommended mitigation steps
In `Well.sync()`, mint `(reserves[0] * reserves[1] * ConstantProduct2.EXP_PRECISION).sqrt() - totalSupply()` Well tokens to `msg.sender`.
This keeps the invariant that `Well.tokenSupply() ** 2` equals to `reserves[0] * reserves[1] * ConstantProduct2.EXP_PRECISION` as long as the swap fee is 0.


## Assessed type

Under/Overflow