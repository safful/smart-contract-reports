## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [proposal get defeated even if forVotes == againstVotes in GovernorAlpha.sol](https://github.com/code-423n4/2021-08-notional-findings/issues/35) 

# Handle

JMukesh


# Vulnerability details

## Impact
proposal get defeated even if forVotes == againstVotes during the voting which impact the given proposals. Instead of this condition forVotes <= againstVotes, it should be 
forVotes < againstVotes

## Proof of Concept

https://github.com/code-423n4/2021-08-notional/blob/4b51b0de2b448e4d36809781c097c7bc373312e9/contracts/external/governance/GovernorAlpha.sol#L389

## Tools Used
manual review

## Recommended Mitigation Steps
change the condition for determining the states

