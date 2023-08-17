## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [The distribution logic will be broken after calling `rageQuit()`](https://github.com/code-423n4/2023-05-party-findings/issues/6) 

# Lines of code

https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/party/PartyGovernanceNFT.sol#L150


# Vulnerability details

## Impact
Malicious users might receive more distributed funds than they should with higher `distributionShare`.

## Proof of Concept
In `PartyGovernanceNFT.sol`, there is a `getDistributionShareOf()` function to calculate the distribution share of party NFT.

```solidity
    function getDistributionShareOf(uint256 tokenId) public view returns (uint256) {
        uint256 totalVotingPower = _governanceValues.totalVotingPower;

        if (totalVotingPower == 0) {
            return 0;
        } else {
            return (votingPowerByTokenId[tokenId] * 1e18) / totalVotingPower;
        }
    }
```

This function is used to calculate the claimable amount in [getClaimAmount()](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/distribution/TokenDistributor.sol#L234).

```solidity
    function getClaimAmount(
        ITokenDistributorParty party,
        uint256 memberSupply,
        uint256 partyTokenId
    ) public view returns (uint128) {
        // getDistributionShareOf() is the fraction of the memberSupply partyTokenId
        // is entitled to, scaled by 1e18.
        // We round up here to prevent dust amounts getting trapped in this contract.
        return
            ((uint256(party.getDistributionShareOf(partyTokenId)) * memberSupply + (1e18 - 1)) /
                1e18).safeCastUint256ToUint128();
    }
```

So after the party distributed funds by executing the distribution proposal, users can claim relevant amounts of funds using their party NFTs.

After the update, `rageQuit()` was added so that users can burn their party NFTs while taking their share of the party's funds.

So the below scenario would be possible.
1. Let's assume `totalVotingPower = 300` and the party has 3 party NFTs of 100 voting power. And `Alice` has 2 NFTs and `Bob` has 1 NFT.
2. They proposed a distribution proposal and executed it. Let's assume the party transferred 3 ether to the distributor.
3. They can claim the funds by calling [TokenDistributor.claim()](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/distribution/TokenDistributor.sol#L138) and `Alice` should receive 2 ether and 1 ether for `Bob`.(We ignore the distribution fee.)
4. But `Alice` decided to steal `Bob`'s funds so she claimed the distributed funds(3 / 3 = 1 ether) with the first NFT and called `rageQuit()` to take her share of the party's remaining funds.
5. After that, `Alice` calls `claim()` with the second NFT ,and `getDistributionShareOf()` will return 50% as the total voting power was decreased to 200. So `Alice` will receive `3 * 50% = 1.5 ether` and `Bob` will receive only 0.5 ether because of this [validation](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/distribution/TokenDistributor.sol#L167)
6. After all, `Alice` received 2.5 ether instead of 2 ether.

Even if `rageQuit` is disabled, `Alice` can burn using `burn()` her NFT directly if her share of the party's remaining funds are less than the stolen funds from `Bob`.

Here is a simple POC showing the distribution shares after `rageQuit()`.

```solidity
    function testWrongDistributionSharesAfterRageQuit() external {
        (Party party, , ) = partyAdmin.createParty(
            partyImpl,
            PartyAdmin.PartyCreationMinimalOptions({
                host1: address(this),
                host2: address(0),
                passThresholdBps: 5100,
                totalVotingPower: 300,
                preciousTokenAddress: address(toadz),
                preciousTokenId: 1,
                rageQuitTimestamp: 0,
                feeBps: 0,
                feeRecipient: payable(0)
            })
        );

        vm.prank(address(this));
        party.setRageQuit(uint40(block.timestamp) + 1);

        address user1 = _randomAddress();
        address user2 = _randomAddress();
        address user3 = _randomAddress();

        //3 users have the same voting power
        vm.prank(address(partyAdmin));
        uint256 tokenId1 = party.mint(user1, 100, user1);

        vm.prank(address(partyAdmin));
        uint256 tokenId2 = party.mint(user2, 100, user2);

        vm.prank(address(partyAdmin));
        uint256 tokenId3 = party.mint(user3, 100, user3);

        vm.deal(address(party), 1 ether);

        // Before calling rageQuit(), each user has the same 33.3333% shares
        uint256 expectedShareBeforeRageQuit = uint256(100) * 1e18 / 300;
        assertEq(party.getDistributionShareOf(tokenId1), expectedShareBeforeRageQuit);
        assertEq(party.getDistributionShareOf(tokenId2), expectedShareBeforeRageQuit);
        assertEq(party.getDistributionShareOf(tokenId3), expectedShareBeforeRageQuit);

        IERC20[] memory tokens = new IERC20[](1);
        tokens[0] = IERC20(ETH_ADDRESS);
        uint256[] memory tokenIds = new uint256[](1);
        tokenIds[0] = tokenId1;

        vm.prank(user1);
        party.rageQuit(tokenIds, tokens, user1);

        // After calling rageQuit() by one user, the second user has 50% shares and can claim more distribution
        uint256 expectedShareAfterRageQuit = uint256(100) * 1e18 / 200;
        assertEq(party.getDistributionShareOf(tokenId2), expectedShareAfterRageQuit);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
I think we shouldn't use `getDistributionShareOf()` for distribution shares.

Instead, we should remember `totalVotingPower` for each distribution separately in [_createDistribution()](https://github.com/code-423n4/2023-05-party/blob/f6f80dde81d86e397ba4f3dedb561e23d58ec884/contracts/distribution/TokenDistributor.sol#L310) so that each user can receive correct funds even after some NFTs are burnt.


## Assessed type

Governance