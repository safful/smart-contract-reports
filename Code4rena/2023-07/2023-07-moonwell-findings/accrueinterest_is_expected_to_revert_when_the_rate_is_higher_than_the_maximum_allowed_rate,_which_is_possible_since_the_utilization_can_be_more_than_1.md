## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-16

# [accrueInterest is expected to revert when the rate is higher than the maximum allowed rate, which is possible since the utilization can be more than 1](https://github.com/code-423n4/2023-07-moonwell-findings/issues/40) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L403


# Vulnerability details

## Impact
accrueInterest is expected to revert when the rate is higher than the maximum allowed rate, which is possible since the utilization can be more than 1

accrueInterest is an essential function to keep updated of the global mToken interest payment from the borrower to the depositor. The function is called anytime there is a deposit/liquidate/borrow/repay/withdraw.

The function is set to revert when the borrowRateMantissa is bigger than the borrowRateMaxMantissa.
`require(borrowRateMantissa <= borrowRateMaxMantissa, "borrow rate is absurdly high");`.

With proper configuration, this check should be fine since the real borrowRate should be calibrated to be within the maxRate. However, since the utilization could actually be bigger than 1 (when reserve is bigger than cash [issue-37] , the actual rate could be bigger than the expected unreachable max rate:

Example:
base: 1%
multiplierPerTimestamp: 5% APR
jumpMultiplier: 50% APR
klink: 50%
We could reasonably assume the max APR is 1% + 5% * 0.5 + 50% * (1-0.5) = 28.5%

However when reserve is more than cash it would cause the utilization be bigger than 1, let'say utilization is 101%: such that the actual rate would  be:

1% + 5% * 0.5 + 50% * (1.01 - 0.5) = 29%

This would cause the accurueInterest to revert.

Impact: all deposit/withdraw/borrow/repay function would fail and severe brick the operation of the protocol and lock user funds. Interest accrual cannot work which impact both borrower for timely exit as well as deposit to collect their interest.

```solidity
    function accrueInterest() virtual override public returns (uint) {
        /* Remember the initial block timestamp */
        uint currentBlockTimestamp = getBlockTimestamp();
        uint accrualBlockTimestampPrior = accrualBlockTimestamp;

        /* Short-circuit accumulating 0 interest */
        if (accrualBlockTimestampPrior == currentBlockTimestamp) {
            return uint(Error.NO_ERROR);
        }

        /* Read the previous values out of storage */
        uint cashPrior = getCashPrior();
        uint borrowsPrior = totalBorrows;
        uint reservesPrior = totalReserves;
        uint borrowIndexPrior = borrowIndex;

        /* Calculate the current borrow interest rate */
        uint borrowRateMantissa = interestRateModel.getBorrowRate(cashPrior, borrowsPrior, reservesPrior);
        require(borrowRateMantissa <= borrowRateMaxMantissa, "borrow rate is absurdly high");
```

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/MToken.sol#L403

## Proof of Concept


## Tools Used

## Recommended Mitigation Steps
Consider to cap the real-time borrowRate as the borrowRateMaxMantissa instead of reverting.





## Assessed type

DoS