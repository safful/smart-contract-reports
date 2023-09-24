## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-40

# [Governance relies on current totalSupply of bHermes when calculate `proposalThresholdAmount` and `quorumVotesAmount`](https://github.com/code-423n4/2023-05-maia-findings/issues/179) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L87-L93


# Vulnerability details

## Impact
As people mint bHermes, bHermesVotes' totalSupply grows. And `quorumVotesAmount` to execute proposal also grows. But it shouldn't, because new people can't vote for it. This behavior adds inconsistency to voting process, because changes threshold after creating proposal.

## Proof of Concept
Here you can see that Governance fetches current totalSupply:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/governance/GovernorBravoDelegateMaia.sol#L87-L93
```solidity
    function getProposalThresholdAmount() public view returns (uint256) {
        return govToken.totalSupply() * proposalThreshold / DIVISIONER;
    }

    function getQuorumVotesAmount() public view returns (uint256) {
        return govToken.totalSupply() * quorumVotes / DIVISIONER;
    }
```

bHermes is ERC4626DepositOnly and mints new govToken when user calls `deposit()` or `mint()`, thus increasing totalSupply:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/hermes/bHermes.sol#L123-L133
```solidity
    function _mint(address to, uint256 amount) internal virtual override {
        gaugeWeight.mint(address(this), amount);
        gaugeBoost.mint(address(this), amount);
        governance.mint(address(this), amount);
        super._mint(to, amount);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add parameter `totalSupply` to Proposal struct and use it instead of current totalSupply in functions `getProposalThresholdAmount()` and `getQuorumVotesAmount()`


## Assessed type

Governance