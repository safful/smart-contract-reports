## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-07

# [Users can bypass distributions fees by ragequitting instead of using a formal distribution](https://github.com/code-423n4/2023-05-party-findings/issues/12) 

# Lines of code

https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L293-L347


# Vulnerability details

## Impact

Distribution fees can be bypassed by ragequitting instead of distributing

## Proof of Concept

https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernance.sol#L510-L515

        address payable feeRecipient_ = feeRecipient;
        uint16 feeBps_ = feeBps;
        if (tokenType == ITokenDistributor.TokenType.Native) {
            return
                distributor.createNativeDistribution{ value: amount }(this, feeRecipient_, feeBps_);
        }

When a distribution is created the distribution will pay fees to the feeRecipient.

https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L333-L345

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

On the other hand, when a user ragequits they are given the full amount without any fees being taken. If we assume that the party is winding down then users can bypass this fee by ragequitting instead of using a formal distribution. This creates value leakage at the fee recipient is not being paid the fees they would otherwise be due.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Charge distribution fees when ragequitting


## Assessed type

Other