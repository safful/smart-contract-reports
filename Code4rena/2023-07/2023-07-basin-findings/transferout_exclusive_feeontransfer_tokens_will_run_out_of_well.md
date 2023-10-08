## Tags

- bug
- 2 (Med Risk)
- low quality report
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-10

# [Transferout exclusive feeOnTransfer tokens will run out of well](https://github.com/code-423n4/2023-07-basin-findings/issues/108) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Well.sol#L610
https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Well.sol#L304
https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/Well.sol#L370


# Vulnerability details

## Impact

The well does not check the actual transferout amount and the k value, which for exclusive feeOnTransfer tokens causes the well to have a portion of the token fee cut out of air every time the token is transferred, which is not included in the transfer amount. Attackers can take advantage of this to run out of well.

## Proof of Concept

The most common attack flow is through the skim function:
1. Attacker swaps to increase the price of exclusive feeOnTransfer token
2. Attacker transfers token to well and skim. This will cut some of the well funds
3. Repeat the above process to reduce the number of well tokens to 1
4. Attacker calls sync and swap to pair token

## Tools Used

Manual review

## Recommended Mitigation Steps

Check the correct amount every time you transfer out





## Assessed type

Uniswap