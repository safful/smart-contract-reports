## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`proposal` declared as both a function and a Proposal in Factory](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/71) 

# Handle

loop


# Vulnerability details

`proposal` is declared as both a function name and the name for a Proposal object.

## Proof of Concept
Factory.sol line 35: `function proposal(uint256 proposalId) external override view returns (Proposal memory) {`
Factory.sol line 77: `Proposal memory proposal = Proposal({`

## Tools Used
Remix

## Recommended Mitigation Steps
Change function name to `getProposal` to avoid double naming and be more in line with other getter/setter functions used.

