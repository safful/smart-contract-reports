## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [purchaseArbitrageTokens 0 amount](https://github.com/code-423n4/2021-11-malt-findings/issues/359) 

# Handle

pauliax


# Vulnerability details

## Impact
Function purchaseArbitrageTokens should validate that amount > 0, otherwise it may be possible to spam accountCommitmentEpochs with 0 amounts:
```solidity
  if (auction.accountCommitments[msg.sender].commitment == 0) {
    accountCommitmentEpochs[msg.sender].push(currentAuctionId);
  }
```

## Recommended Mitigation Steps
require amount > 0

