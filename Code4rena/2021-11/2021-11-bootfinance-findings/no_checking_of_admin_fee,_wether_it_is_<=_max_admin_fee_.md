## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [No checking of admin_fee, wether it is <= max_admin_fee ](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/127) 

# Handle

JMukesh


# Vulnerability details

## Impact
due to lack of checking admin_fee, it can be greater than max_admin_fee

## Proof of Concept
https://github.com/code-423n4/2021-11-bootfinance/blob/7c457b2b5ba6b2c887dafdf7428fd577e405d652/core-contracts/contracts/sol/BTCPoolDelegator.sol#L57

https://github.com/code-423n4/2021-11-bootfinance/blob/7c457b2b5ba6b2c887dafdf7428fd577e405d652/core-contracts/contracts/sol/ETHPoolDelegator.sol#L58

https://github.com/code-423n4/2021-11-bootfinance/blob/7c457b2b5ba6b2c887dafdf7428fd577e405d652/core-contracts/contracts/sol/USDPoolDelegator.sol#L53

## Tools Used

manual review

## Recommended Mitigation Steps

add input validation for admin_fee

