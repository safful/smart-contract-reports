## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-10

# [No slippage control when minting and redeeming FPS](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/396) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Equity.sol#L241-L255


# Vulnerability details

## Impact
When minting and redeeming FPS in Equity, there is no slippage control. Since the price of FPS will change with the zchf reserve in the contract, users may suffer from sandwich attacks.

Consider the current contract has a zchf reserve of 1000 and a total supply of 1000.

Alice considers using 4000 zchf to mint FPS. Under normal circumstances, the contract reserve will rise to 5000 zchf, and the total supply will rise to (5000/1000)**(1/3)*1000 = 1710, that is, alice will get 1710 - 1000 = 710 FPS.

bob holds 400 FPS, and bob observes alice's transaction in MemPool, bob uses MEV to preemptively use 4000 zchf to mint 710 FPS.

When alice's transaction is executed, the contract reserve will increase from 5000 to 9000 zchf, and the total supply will increase from 1710 to (9000/5000)**(1/3)*1710 = 2080, that is, alice gets 2080-1710 = 370FPS.

Then bob will redeem 400 FPS, the total supply will drop from 2080 to 1680, and the contract reserve will drop from 9000 to (1689/2080)**3*9000 = 4742, that is, bob gets 9000-4742 = 4258 zchf.

bob's total profit is 310 FPS and 258 zchf.

## Proof of Concept
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Equity.sol#L241-L255
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Equity.sol#L266-L270
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Equity.sol#L275-L282
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Equity.sol#L290-L297
## Tools Used
None
## Recommended Mitigation Steps
Consider setting minFPSout and minZCHFout parameters to allow slippage control when minting and redeeming FPS