## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing zero address check which will put forfeited rewards at risk(ForefeitHandler.sol) ](https://github.com/code-423n4/2021-11-malt-findings/issues/216) 

# Handle

0xwags


# Vulnerability details

## Impact
Since users forfeited awards will be shared between either the treasury and the swing trader, there should be a zero address in the initialize() function to ensure rewards are not lost and thereby affecting  Malt's collateralisation and other such funding mechanism. 

This will have implications for safetransfer() functions in lines 50 & 54 in handleForfeit(). 

## Tools Used
Manual Analysis. 

## Recommended Mitigation Steps

require(treasuryMultisig&& swingTrader ! =address(0), "0x0");

