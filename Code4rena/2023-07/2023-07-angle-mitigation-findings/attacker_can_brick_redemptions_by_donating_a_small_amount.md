## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- MR-M-02
- edited-by-warden

# [Attacker can brick redemptions by donating a small amount](https://github.com/code-423n4/2023-07-angle-mitigation-findings/issues/2) 

# Lines of code

https://github.com/AngleProtocol/angle-transmuter/blob/6f2ffcb1e89e3bba05c9aa2133ef94347aa42c28/contracts/transmuter/libraries/LibGetters.sol#L89


# Vulnerability details

## Impact

While the fix properly fixes the issue of collateralization ratio overflows (that can no longer occurs), it enables DoS attacks on the redemption mechanism:

## Issue description

Consider the example that was already provided (https://github.com/code-423n4/2023-06-angle-findings/issues/9
), where an attacker produces an overflow by donating a small amount:

> Let's take an extreme example and say that `stablecoinsIssued` is 1 (although the attack also works with larger values). If a user now transfer 1 USD of a stablecoin to the contract, `totalCollateralization` will be roughly 10**18 . Therefore, the system calculates:
> 
> ```
> collatRatio = uint64(10**18 * 10**9 / 1) = uint64(10**27);
> ```
> 
> Because `10**27 > type(uint64).max`, this overflows.


After the fix, any call to `getCollateralRatio` will revert because of the `SafeCast`. Therefore, any call to `_quoteRedemptionCurve` will also revert, meaning that redemptions fail.

Note that the attacker never directly interacted with the contract (as he donated the tokens), which means that the protection also does not work.

## Recommendation

It is recommended to set `collatRatio` to `type(uint64).max` instead of reverting when an overflow happens. Then, the problem would no longer occur.


## Assessed type

DoS