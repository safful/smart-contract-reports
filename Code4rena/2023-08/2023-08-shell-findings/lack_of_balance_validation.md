## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- upgraded by judge
- sufficient quality report
- H-01

# [Lack of Balance Validation](https://github.com/code-423n4/2023-08-shell-findings/issues/57) 

# Lines of code

https://github.com/code-423n4/2023-08-shell/blob/main/src/proteus/EvolvingProteus.sol#L563-L595


# Vulnerability details

## Description

The pool's ratio of y to x must be within the interval `[MIN_M, MAX_M)`, which will be checked by the `_checkBalances()` function.
External view functions will call `_swap()`, `_reserveTokenSpecified()` or `_lpTokenSpecified()` functions to get the specified result.
However, `_checkBalances()` is only used in the `_swap()` and `_lpTokenSpecified()` functions. There is no balance validation for `depositGivenInputAmount()` and `withdrawGivenOutputAmount()` functions, which use `_reserveTokenSpecified()` function.

## Impact
If there's no other validation outside these two functions, user deposits/withdraws may break the invariant, i.e. the pool's ratio of y to x is outside the interval `[MIN_M, MAX_M)`.

## Proof of Concept
Add the following code in test/EvolvingProteusProperties.t.sol file EvolvingProteusProperties contract, and run `forge test --mt RatioOutsideExpectedInterval`.

```js
function testDepositRatioOutsideExpectedInterval(uint256 x0, uint256 y0, uint256 s0, uint256 depositedAmount) public {
  int128 MIN_M = 0x00000000000002af31dc461;
  uint256 INT_MAX_SQRT = 0xb504f333f9de6484597d89b3754abe9f;

  vm.assume(x0 >= MIN_BALANCE && x0 <= INT_MAX_SQRT);
  vm.assume(y0 >= MIN_BALANCE && y0 <= INT_MAX_SQRT);
  vm.assume(s0 >= MIN_BALANCE && s0 <= INT_MAX_SQRT);
  vm.assume(depositedAmount >= MIN_OPERATING_AMOUNT && depositedAmount < INT_MAX_SQRT && depositedAmount >= 2 * uint256(FIXED_FEE));
  vm.assume(y0/x0 <= MAX_BALANCE_AMOUNT_RATIO);
  vm.assume(x0/y0 <= MAX_BALANCE_AMOUNT_RATIO);
  vm.assume(int256(y0).divi(int256(x0) + int256(depositedAmount)) < MIN_M);   // breaks the invariant
  SpecifiedToken depositedToken = SpecifiedToken.X;
  
  vm.expectRevert();  // There should be at least one case that call did not revert as expected
  DUT.depositGivenInputAmount(
      x0,
      y0,
      s0,
      depositedAmount,
      depositedToken
  );
}

function testWithdrawRatioOutsideExpectedInterval(uint256 x0, uint256 y0, uint256 s0, uint256 withdrawnAmount) public {
  int128 MIN_M = 0x00000000000002af31dc461;
  uint256 INT_MAX_SQRT = 0xb504f333f9de6484597d89b3754abe9f;

  vm.assume(x0 >= MIN_BALANCE && x0 <= INT_MAX_SQRT);
  vm.assume(y0 >= MIN_BALANCE && y0 <= INT_MAX_SQRT);
  vm.assume(s0 >= MIN_BALANCE && s0 <= INT_MAX_SQRT);
  vm.assume(withdrawnAmount >= MIN_OPERATING_AMOUNT && withdrawnAmount < INT_MAX_SQRT && withdrawnAmount >= 2 * uint256(FIXED_FEE));
  vm.assume(y0/x0 <= MAX_BALANCE_AMOUNT_RATIO);
  vm.assume(x0/y0 <= MAX_BALANCE_AMOUNT_RATIO);
  vm.assume(withdrawnAmount < y0);    // no more than balance
  vm.assume((int256(y0) - int256(withdrawnAmount)).divi(int256(x0)) < MIN_M);   // breaks the invariant
  SpecifiedToken withdrawnToken = SpecifiedToken.Y;
  
  vm.expectRevert();  // There should be at least one case that call did not revert as expected
  DUT.withdrawGivenOutputAmount(
      x0,
      y0,
      s0,
      withdrawnAmount,
      withdrawnToken
  );
}
```

## Tools Used
Manual
## Recommended Mitigation Steps
It's recommended to add `_checkBalances(xi + specifiedAmount, yi)` after [L579](https://github.com/code-423n4/2023-08-shell/blob/main/src/proteus/EvolvingProteus.sol#L579) and add `_checkBalances(xi, yi + specifiedAmount)` after [L582](https://github.com/code-423n4/2023-08-shell/blob/main/src/proteus/EvolvingProteus.sol#L582).


## Assessed type

Invalid Validation