## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [Use of _mint in ReraiseETHCrowdfund#_contribute is incompatible with PartyGovernanceNFT#mint ](https://github.com/code-423n4/2023-04-party-findings/issues/42) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L256-L303


# Vulnerability details

## Impact

Misconfigured receiver could accidentally DOS party

## Proof of Concept

[ReraiseETHCrowdfund.sol#L238](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L238)

        if (previousVotingPower == 0) _mint(contributor); <- @audit-issue standard minting here

[ReraiseETHCrowdfund.sol#L374](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L374)

            uint256 tokenId = party.mint(contributor, votingPowerByCard[i], delegate); <- @audit-issue uses party.mint

[PartyGovernanceNFT.sol#L162](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernanceNFT.sol#L162)

        _safeMint(owner, tokenId); <- @audit-issue PartyGovernanceNFT#mint utilizes _safeMint

The issue at hand is that ReraiseETHCrowdfund#_contribute and PartyGovernanceNFT#mint use inconsistent minting methods. PartyGovernanceNFT uses safeMint whereas ReraiseETHCrowdfund uses the standard mint. This is problematic because this means that a contract that doesn't implement ERC721Receiver can receive a CrowdfundNFT but they can never claim because safeMint will always revert. This can cause a party to be inadvertently DOS'd because CrowdfundNFTs are soul bound and can't be transferred 

## Tools Used

Manual Review

## Recommended Mitigation Steps

Use _safeMint instead of _mint for ReraiseETHCrowdfund#_contribute