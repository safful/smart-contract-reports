## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [Rage quitter loses his claimable share of distributed tokens](https://github.com/code-423n4/2023-05-party-findings/issues/22) 

# Lines of code

https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L293-L353


# Vulnerability details



## Impact
Rage quitter loses his claimable share of distributed tokens.

## Proof of Concept
[`PartyGovernanceNFT.rageQuit()`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L293-L353) burns a governance NFT and transfers its share of the balance of ETH and tokens:
```solidity
// Burn caller's party card. This will revert if caller is not the owner
// of the card.
burn(tokenId);

// Withdraw fair share of tokens from the party.
IERC20 prevToken;
for (uint256 j; j < withdrawTokens.length; ++j) {
    IERC20 token = withdrawTokens[j];

    // Prevent null and duplicate transfers.
    if (prevToken >= token) revert InvalidTokenOrderError();

    prevToken = token;

    // Check if token is ETH.
    if (address(token) == ETH_ADDRESS) {
        // Transfer fair share of ETH to receiver.
        uint256 amount = (address(this).balance * shareOfVotingPower) / 1e18;
        if (amount != 0) {
            payable(receiver).transferEth(amount);
        }
    } else {
        // Transfer fair share of tokens to receiver.
        uint256 amount = (token.balanceOf(address(this)) * shareOfVotingPower) / 1e18;
        if (amount != 0) {
            token.compatTransfer(receiver, amount);
        }
    }
}
```
The problem with this is that the governance NFT might also have tokens to [`claim()`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/distribution/TokenDistributor.sol#L138-L182) in the `TokenDistributor`. These cannot be claimed after the governance NFT has been burned.
The rage quitter cannot completely protect himself from this by calling `claim()` first, because the tokens might not yet have been distributed to the `TokenDistributor` until in a frontrun call to [`distribute()`](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernance.sol#L474-L520) just before his `rageQuit()`. This way the rage quitter might be robbed of his fair share.

## Recommended Mitigation Steps
Have `rageQuit()` call `TokenDistributor.claim()` before the governance NFT is burned.


## Assessed type

Context