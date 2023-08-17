## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- upgraded by judge
- sponsor duplicate
- H-02

# [ProtocolDAO lacks a method to take out GGP](https://github.com/code-423n4/2022-12-gogopool-findings/issues/532) 

# Lines of code

https://github.com/code-423n4/2022-12-gogopool/blob/aec9928d8bdce8a5a4efe45f54c39d4fc7313731/contracts/contract/Staking.sol#L379-L383


# Vulnerability details

## Impact

ProtocolDAO implementation does not have a method to take out  GGP. So it can't handle ggp unless it updates ProtocolDAO
## Proof of Concept


recordStakingEnd() will pass the rewards of this reward
"If the validator is failing at their duties, their GGP will be slashed and used to compensate the loss to our Liquid Stakers"

At this point slashGGP() will be executed and the GGP will be transferred to "ProtocolDAO"

staking.slashGGP():
```solidity
    function slashGGP(address stakerAddr, uint256 ggpAmt) public onlySpecificRegisteredContract("MinipoolManager", msg.sender) {
        Vault vault = Vault(getContractAddress("Vault"));
        decreaseGGPStake(stakerAddr, ggpAmt);
        vault.transferToken("ProtocolDAO", ggp, ggpAmt);
    }
```
But the current ProtocolDAO implementation does not have a method to take out  GGP. So it can't handle ggp unless it updates ProtocolDAO


## Tools Used

## Recommended Mitigation Steps

1.transfer GGP to  ClaimProtocolDAO 
or
2.Similar to ClaimProtocolDAO, add spend method to retrieve GGP

```solidity
contract ProtocolDAO is Base {
...

+    function spend(
+        address recipientAddress,
+        uint256 amount
+    ) external onlyGuardian {
+        Vault vault = Vault(getContractAddress("Vault"));
+        TokenGGP ggpToken = TokenGGP(getContractAddress("TokenGGP"));
+
+        if (amount == 0 || amount > vault.balanceOfToken("ProtocolDAO", ggpToken)) {
+            revert InvalidAmount();
+        }
+
+        vault.withdrawToken(recipientAddress, ggpToken, amount);
+
+        emit GGPTokensSentByDAOProtocol(address(this), recipientAddress, amount);
+   }
```