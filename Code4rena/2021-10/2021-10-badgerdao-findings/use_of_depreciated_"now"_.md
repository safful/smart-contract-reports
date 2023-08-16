## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [use of depreciated "now" ](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/83) 

# Handle

JMukesh


# Vulnerability details

## Impact
In updatePricePerShare() instead of "block.timestamp" , "now"  is used which is deprciated.  "block.timestamp" is way more explicit in showing the intent while "now"
 relates to the timestamp of the block controlled by the miner

more on this -> https://github.com/ethereum/solidity/issues/4020



## Tools Used
manual review

## Recommended Mitigation Steps
use block.timestamp

