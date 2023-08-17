## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-03

# [Users wouldn't refund from the lost ETH crowdfunds due to the lack of ETH](https://github.com/code-423n4/2023-04-party-findings/issues/15) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L236


# Vulnerability details

## Impact
After the ETH crowdfunds are lost, contributors wouldn't refund their funds because the crowdfunds contract doesn't have enough ETH balance.

## Proof of Concept
The core flaw is `_calculateRefundAmount()` might return more refund amount than the original contribution amount.

```solidity
    function _calculateRefundAmount(uint96 votingPower) internal view returns (uint96 amount) {
        amount = (votingPower * 1e4) / exchangeRateBps;

        // Add back fee to contribution amount if applicable.
        address payable fundingSplitRecipient_ = fundingSplitRecipient;
        uint16 fundingSplitBps_ = fundingSplitBps;
        if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
            amount = (amount * 1e4) / (1e4 - fundingSplitBps_); //@audit might be greater than original contribution
        }
    }
```

When users contribute to the ETH crowdfunds, it subtracts the fee from the contribution amount.

```solidity
File: 2023-04-party\contracts\crowdfund\ETHCrowdfundBase.sol
226:         uint16 fundingSplitBps_ = fundingSplitBps;
227:         if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
228:             uint96 feeAmount = (amount * fundingSplitBps_) / 1e4;
229:             amount -= feeAmount;
230:         }
```

During the calculation, it calculates `feeAmount` first which is rounded down and subtracts from the contribution amount. It means the final amount after subtracting the fee would be rounded up.

So when we calculate the original amount using `_calculateRefundAmount()`, we might get a greater value.

This shows the detailed example and POC.

1. Let's assume `fundingSplitBps = 1e3(10%), exchangeRateBps = 1e4`. 
2. A user contributed `1e18 - 1` wei of ETH. After subtracting the fee, the voting power was `1e18 - 1 - (1e18 - 1) / 10 = 9 * 1e17`
2. Let's assume there are no other contributors and the crowdfund was lost.
3. When the user calls `refund()`, the refund amount will be `9 * 1e17 * 1e4 / 9000 = 1e18` in `_calculateRefundAmount()`
4. So it will try to transfer 1e18 wei of ETH from the crowdfund contract that contains 1e18 - 1 wei only. As a result, the transfer will revert and the user can't refund his funds.

```solidity
    function test_refund_reverts() public {
        InitialETHCrowdfund crowdfund = _createCrowdfund({
            initialContribution: 0,
            initialContributor: payable(address(0)),
            initialDelegate: address(0),
            minContributions: 0,
            maxContributions: type(uint96).max,
            disableContributingForExistingCard: false,
            minTotalContributions: 3 ether,
            maxTotalContributions: 5 ether,
            duration: 7 days,
            fundingSplitBps: 1000, //10% fee
            fundingSplitRecipient: payable(_randomAddress()) //recipient exists
        });
        Party party = crowdfund.party();

        uint256 ethAmount = 1 ether - 1; //contribute amount

        address member = _randomAddress();
        vm.deal(member, ethAmount);

        // Contribute
        vm.prank(member);
        crowdfund.contribute{ value: ethAmount }(member, "");
        assertEq(address(member).balance, 0);
        assertEq(address(crowdfund).balance, ethAmount); //crowdfund's balance = 1 ether - 1

        skip(7 days);

        assertTrue(crowdfund.getCrowdfundLifecycle() == ETHCrowdfundBase.CrowdfundLifecycle.Lost);

        // Claim refund
        vm.prank(member);
        uint256 tokenId = 1;
        crowdfund.refund(tokenId); //reverts as it tried to withdraw 1 ether
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
When we subtract the fee in [_processContribution()](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L227), we should calculate the final amount using `1e4 - fundingSplitBps` directly. Then there will be 2 rounds down in `_processContribution()` and `_calculateRefundAmount` and the refund amount won't be greater than the original amount.

```solidity
    if (fundingSplitRecipient_ != address(0) && fundingSplitBps_ > 0) {
        amount = (amount * (1e4 - fundingSplitBps_)) / 1e4;
    }
```