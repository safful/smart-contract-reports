## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [function redeem should return 'redeemed' amount](https://github.com/code-423n4/2021-05-yield-findings/issues/58) 

# Handle

pauliax


# Vulnerability details

## Impact
function redeem in contract FYToken should return 'redeemed' amount. There return value is not used anywhere, but it's a mistake that it assigns 'redeemed' but returns 'amount'.

## Recommended Mitigation Steps
Remove return sentence or explicitly return 'redeemed'.

