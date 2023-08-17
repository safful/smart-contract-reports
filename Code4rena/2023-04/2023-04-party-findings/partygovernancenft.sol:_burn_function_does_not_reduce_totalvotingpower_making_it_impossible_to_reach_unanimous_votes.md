## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-06

# [PartyGovernanceNFT.sol: burn function does not reduce totalVotingPower making it impossible to reach unanimous votes](https://github.com/code-423n4/2023-04-party-findings/issues/10) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernanceNFT.sol#L201-L224


# Vulnerability details

## Impact
With the new version of the Party protocol the [`PartyGovernanceNFT.burn`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernanceNFT.sol#L201-L224) function has been introduced.  

This function is used to burn party cards.  

According to the sponsor the initial purpose of this function was to enable the [`InitialETHCrowdfund`](https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/InitialETHCrowdfund.sol) contract (the `burn` function is needed for refunds).  

Later on they decided to allow any user to call this function and to burn their party cards.  

The second use case when a regular user burns his party card is when the issue occurs.  

The `PartyGovernanceNFT.burn` function does not decrease `totalVotingPower` which makes it impossible to reach an unanimous vote after a call to this function and it makes remaining votes of existing users less valuable than they should be.  

## Proof of Concept
Let's look at the `PartyGovernanceNFT.burn` function:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernanceNFT.sol#L201-L224)  
```solidity
function burn(uint256 tokenId) external onlyDelegateCall {
    address owner = ownerOf(tokenId);
    if (
        msg.sender != owner &&
        getApproved[tokenId] != msg.sender &&
        !isApprovedForAll[owner][msg.sender]
    ) {
        // Allow minter to burn cards if the total voting power has not yet
        // been set (e.g. for initial crowdfunds) meaning the party has not
        // yet started.
        uint96 totalVotingPower = _governanceValues.totalVotingPower;
        if (totalVotingPower != 0 || !isAuthority[msg.sender]) {
            revert UnauthorizedToBurnError();
        }
    }


    uint96 votingPower = votingPowerByTokenId[tokenId].safeCastUint256ToUint96();
    mintedVotingPower -= votingPower;
    delete votingPowerByTokenId[tokenId];


    _adjustVotingPower(owner, -votingPower.safeCastUint96ToInt192(), address(0));


    _burn(tokenId);
}
```

It burns the party card specified by the `tokenId` parameter and makes the appropriate changes to the voting power of the owner (by calling `_adjustVotingPower`) and to `mintedVotingPower`.  

But it does not reduce `totalVotingPower` which remains untouched by this function.  

In case this function is called by `InitialETHCrowdfund` it is intended that `totalVotingPower` is not reduced. In this case the `burn` function is only called when the initial crowdfund is lost and `totalVotingPower` hasn't even been increased so it is still `0` (the initial value).  

But why is it an issue when a regular user calls this function?  

Let's consider the following scenario:  

```
Alice: 100 Votes
Bob: 100 Votes
Chris: 100 Votes

totalVotingPower = 300 Votes
```

Now Alice decides to burn half of her voting power:  

```
Alice: 50 Votes
Bob: 100 Votes
Chris: 100 Votes

totalVotingPower = 300 Votes
```

Now it is easy to see why it is a problem that `totalVotingPower` is not reduced.  

It is impossible to reach an unanimous vote because even if all users vote there is only a `(250/300) = ~83%` agreement.  

One vote only represents `1/300 = ~ 0.33%` of all votes even though it should represent `1/250 = 0.4%` of all votes. And thereby votes are less valuable than they should be.  

You can see in the following test that `totalVotingPower` stays unaffected even though `voter1` burns his party card which represents a third of all votes.  

(Add the test to the `PartyGovernanceNFTUnit.sol` test file and add this import: `import "../../contracts/party/PartyGovernance.sol";` to access the `GovernanceValues` struct ).  

```solidity
function test_canntReachUnanimousVoteAfterBurning() external {
    _initGovernance();
    address voter1 = _randomAddress();
    address voter2 = _randomAddress();
    address voter3 = _randomAddress();
    uint256 vp = defaultGovernanceOpts.totalVotingPower / 3;
    uint256 token1 = nft.mint(voter1, vp, voter1);
    uint256 token2 = nft.mint(voter2, vp, voter2);
    uint256 token3 = nft.mint(voter3, vp, voter3);

    assertEq(nft.mintedVotingPower(), vp*3);
    assertEq(nft.getCurrentVotingPower(voter1), vp);

    PartyGovernance.GovernanceValues memory gv = nft.getGovernanceValues();
    console.log(gv.totalVotingPower);

    vm.prank(voter1);
    nft.burn(token1);
    gv = nft.getGovernanceValues();
    // totalVotingPower stays the same
    console.log(gv.totalVotingPower);
}
```

The remaining two voters will not be able to reach unanimous vote since the `_isUnanimousVotes` function is called with `totalVotingPower` as the total votes with which to calculate the percentage.  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L988)  
```solidity
if (_isUnanimousVotes(pv.votes, _governanceValues.totalVotingPower)) {
```

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L1024)  
```solidity
if (_isUnanimousVotes(pv.votes, gv.totalVotingPower)) {
```

## Tools Used
VSCode

## Recommended Mitigation Steps
It is important to understand that when `InitialETHCrowdfund` calls the `burn` function it is intended that `totalVotingPower` is not reduced.  

So we need to differentiate these two cases.  

Fix:  
```diff
diff --git a/contracts/party/PartyGovernanceNFT.sol b/contracts/party/PartyGovernanceNFT.sol
index 9ccfa1f..d382d0e 100644
--- a/contracts/party/PartyGovernanceNFT.sol
+++ b/contracts/party/PartyGovernanceNFT.sol
@@ -200,6 +200,7 @@ contract PartyGovernanceNFT is PartyGovernance, ERC721, IERC2981 {
     /// @param tokenId The ID of the NFT to burn.
     function burn(uint256 tokenId) external onlyDelegateCall {
         address owner = ownerOf(tokenId);
+        uint96 totalVotingPower = _governanceValues.totalVotingPower;
         if (
             msg.sender != owner &&
             getApproved[tokenId] != msg.sender &&
@@ -208,7 +209,6 @@ contract PartyGovernanceNFT is PartyGovernance, ERC721, IERC2981 {
             // Allow minter to burn cards if the total voting power has not yet
             // been set (e.g. for initial crowdfunds) meaning the party has not
             // yet started.
-            uint96 totalVotingPower = _governanceValues.totalVotingPower;
             if (totalVotingPower != 0 || !isAuthority[msg.sender]) {
                 revert UnauthorizedToBurnError();
             }
@@ -218,6 +218,10 @@ contract PartyGovernanceNFT is PartyGovernance, ERC721, IERC2981 {
         mintedVotingPower -= votingPower;
         delete votingPowerByTokenId[tokenId];
 
+        if (totalVotingPower != 0 || !isAuthority[msg.sender]) {
+            _governanceValues.totalVotingPower = totalVotingPower - votingPower;
+        }
+
         _adjustVotingPower(owner, -votingPower.safeCastUint96ToInt192(), address(0));
 
         _burn(tokenId);
```

Also note that the `|| !isAuthority[msg.sender]` part of the condition is important.  
It ensures that if we are not yet in the governance phase, i.e. `totalVotingPower == 0` and a user calls the `burn` function he cannot burn his party card. This is because the `totalVotingPower - votingPower` subtraction results in an underflow.  

This ensures that in the pre-governance phase a user cannot accidentally burn his party card. He can only burn it via the `InitialETHCrowdfund` contract which ensures the user gets his ETH refund.  

