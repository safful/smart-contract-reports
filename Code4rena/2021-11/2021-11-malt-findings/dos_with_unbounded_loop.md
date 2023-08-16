## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [DOS with unbounded loop](https://github.com/code-423n4/2021-11-malt-findings/issues/380) 

# Handle

Koustre


# Vulnerability details

## Impact
In UniswapHandler, in the function ```removeBuyer``` there is a for loop over an unbounded Buyers array, which if the buyers array gets too large can cause a denial of service and prevents the contract from being able to remove buyer roles from users/contracts. This would allow users/contracts to circumvent recovery mode and to continue to purchase and sell tokens using the contract.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

## Tools Used
- Manual Study
## Recommended Mitigation Steps
- remove unbounded for loop

