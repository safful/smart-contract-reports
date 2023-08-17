## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- M-03

# [Anyone can call AutoPxGmx.compound and perform sandwich attacks with control parameters](https://github.com/code-423n4/2022-11-redactedcartel-findings/issues/91) 

# Lines of code

https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L242-L278


# Vulnerability details

## Impact
AutoPxGmx.compound allows anyone to call to compound the reward and get the incentive.
However, AutoPxGmx.compound calls SWAP_ROUTER.exactInputSingle with some of the parameters provided by the caller, which allows the user to perform a sandwich attack for profit.
For example, a malicious user could provide the fee parameter to make the token swap occur in a small liquid pool, and could make the amountOutMinimum parameter 1 to make the token swap accept a large slippage, thus making it easier to perform a sandwich attack.
## Proof of Concept
https://github.com/code-423n4/2022-11-redactedcartel/blob/03b71a8d395c02324cb9fdaf92401357da5b19d1/src/vaults/AutoPxGmx.sol#L242-L278
## Tools Used
None
## Recommended Mitigation Steps
Consider using poolFee as the fee and using an onchain price oracle to calculate the amountOutMinimum