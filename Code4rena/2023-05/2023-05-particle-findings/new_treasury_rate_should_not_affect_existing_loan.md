## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [New treasury rate should not affect existing loan](https://github.com/code-423n4/2023-05-particle-findings/issues/9) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L800


# Vulnerability details

## Impact
In the protocol, lenders have to pay a small treasury fee when they claim their interest. The contract owner can change this `_treasuryRate` at any time using the function `setTreasuryRate()`.

```solidity
// @audit treasury rate should not affect existing loan
function setTreasuryRate(uint256 rate) external onlyOwner {
    if (rate > MathUtils._BASIS_POINTS) {
        revert Errors.InvalidParameters();
    }
    _treasuryRate = rate;
    emit UpdateTreasuryRate(rate);
}

```

However, when the admin changes the rate, the new treasury rate will also be applied to active loans, which is not the agreed-upon term between the lenders and borrowers when they supplied the NFT and created the loan.

## Proof of Concept

Consider the following scenario:

1. Alice and Bob have an active loan with an accumulated interest of 1 ETH and `_treasuryRate = 5%`.
2. The admin suddenly changes the `_treasuryRate` to `50%`. Now, if Alice claims the interest, she needs to pay 0.5 ETH to the treasury and keep 0.5 ETH.
3. Alice can either accept it and keep 0.5 ETH interest or front-run the admin transaction and claim before the `_treasuryRate` is updated.
The point is that Alice only agreed to pay a `5%` treasury rate at the beginning, so the new rate should not apply to her.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider storing the `treasuryRate` in the loan struct. The loan struct is not kept in storage, so the gas cost will not increase significantly.

Alternatively, consider adding a timelock mechanism to prevent the admin from changing the treasury rate.



## Assessed type

Other