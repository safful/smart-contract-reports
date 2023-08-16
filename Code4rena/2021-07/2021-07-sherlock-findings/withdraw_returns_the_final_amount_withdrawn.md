## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [withdraw returns the final amount withdrawn](https://github.com/code-423n4/2021-07-sherlock-findings/issues/78) 

# Handle

pauliax


# Vulnerability details

## Impact
function withdraw in ILendingPool returns the actual withdrawn amount, however, function withdraw in AaveV2 strategy does not check this return value so e.g. function strategyWithdraw may actually withdraw less but still add the full amount to the staked balance: 
    ps.strategy.withdraw(_amount);
    ps.stakeBalance = ps.stakeBalance.add(_amount);

## Recommended Mitigation Steps
function withdraw in IStrategy should return uint indicating the actual withdrawn amount and functions that use it should account for that.

