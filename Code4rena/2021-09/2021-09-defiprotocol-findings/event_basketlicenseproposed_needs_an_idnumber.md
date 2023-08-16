## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [Event BasketLicenseProposed needs an idNumber](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/263) 

# Handle

0xsanson


# Vulnerability details

## Impact
The function `Factory.proposeBasketLicense` at the end emits `BasketLicenseProposed(msg.sender, tokenName)` and returns the id of the proposal.
This `id` should also be written to the log, since it's needed by the proposer (for createBasket), and they may not see the return value of an external function.

## Proof of Concept
https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Factory.sol#L87-L90

## Tools Used
editor

## Recommended Mitigation Steps
Consider redefining the event to contain the id of the proposal.

