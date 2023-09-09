## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-24

# [Accrued perfomance fee calculation takes wrong assumptions for share decimals, leading to loss of shares or hyperinflation](https://github.com/code-423n4/2023-01-popcorn-findings/issues/306) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L529-L542
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L447-L460


# Vulnerability details

This issue applies to both `AdapterBase.sol` and `Vault.sol`. For the sake of simplicity and brevity, this POC will describe just the former.

## Impact
Fee calculation is wrong and it either takes too few or too many shares than what is supposed to be when calculating the `accruedPerformanceFee` and the shares decimals are not `18`. 
The former causes a loss of shares that the `FEE_RECIPIENT` should earn, but the latter causes hyperinflation, which makes users' shares worthless. 

## Proof of Concept
`accruedPerformanceFee` doesn't take into consideration the shares' decimals, and it supposes that it's always `1e18`.

This is supposed to be a percentage and it's calculated as the following, rounding down.

```solidity
function accruedPerformanceFee() public view returns (uint256) {
    uint256 highWaterMark_ = highWaterMark;
    uint256 shareValue = convertToAssets(1e18);
    uint256 performanceFee_ = performanceFee;

    return
        performanceFee_ > 0 && shareValue > highWaterMark_
            ? performanceFee_.mulDiv(
                (shareValue - highWaterMark_) * totalSupply(),
                1e36,
                Math.Rounding.Down
            )
            : 0;
}
```

This calculation is wrong because the assumption is: 
```
totalSupply (1e18) * performanceFee_ (1e18) = 1e36
```

which is not always true because the `totalSupply` decimals can be greater or less than that.

Let's see what would happen in this case.

### Best case scenario: `supply decimals < 1e18`

In this case, the fee calculation will always round to zero, thus the `FEE_RECIPIENT` will never get the deserved accrued fees.

### Worst case scenario: `supply decimals > 1e18`

The `FEE_RECIPIENT` will get a highly disproportionate number of shares. 

This will lead to share hyperinflation, which will also impact the users, making their shares worthless.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Modify the fee calculation so it's divided with the correct denominator, that takes into account the share decimals:

```solidity
performanceFee_ > 0 && shareValue > highWaterMark_
? performanceFee_.mulDiv(
    (shareValue - highWaterMark_) * totalSupply(),
    1e18 * (10 ** decimals()),
    Math.Rounding.Down
)
: 0;
``