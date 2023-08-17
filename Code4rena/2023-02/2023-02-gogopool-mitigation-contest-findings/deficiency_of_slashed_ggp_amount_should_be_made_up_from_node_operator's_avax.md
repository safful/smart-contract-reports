## Tags

- 2 (Med Risk)
- primary issue
- selected for report
- edited-by-warden
- MR-H-06

# [Deficiency of slashed GGP amount should be made up from node operator's AVAX](https://github.com/code-423n4/2023-02-gogopool-mitigation-contest-findings/issues/6) 

# Lines of code

https://github.com/multisig-labs/gogopool/blob/9ad393c825e6ac32f3f6c017b4926806a9383df1/contracts/contract/MinipoolManager.sol#L731-L733


# Vulnerability details

## Impact
If staked GGP doesn't cover slash amount, slashing it all will not be fair to the liquid stakers. Slashing is rare, and that the current 14 day validation cycle which is typically 1/26 of the minimum amount of GGP staked is unlikely to bump into this situation unless there is a nosedive of GGP price in AVAX. The deficiency should nonetheless be made up from `avaxNodeOpAmt` should this unfortunate situation happen.

## Proof of Concept
[File: MinipoolManager.sol#L731-L733](https://github.com/multisig-labs/gogopool/blob/9ad393c825e6ac32f3f6c017b4926806a9383df1/contracts/contract/MinipoolManager.sol#L731-L733)

```solidity
		if (staking.getGGPStake(owner) < slashGGPAmt) {
			slashGGPAmt = staking.getGGPStake(owner);
		}
```
As can be seen from the code block above, in extreme and unforeseen cases, the difference between `staking.getGGPStake(owner)` and `slashGGPAmt` can be significant. Liquid stakers would typically and ultimately care about how they are going to be adequately compensated with, in AVAX preferably. 

## Tools Used
Manual inspection

## Recommended Mitigation Steps
Consider having the affected if block refactored as follows:

```diff
		Staking staking = Staking(getContractAddress("Staking"));
		if (staking.getGGPStake(owner) < slashGGPAmt) {
			slashGGPAmt = staking.getGGPStake(owner);

+			uint256 diff = slashGGPAmt - staking.getGGPStake(owner);
+			Oracle oracle = Oracle(getContractAddress("Oracle"));
+			(uint256 ggpPriceInAvax, ) = oracle.getGGPPriceInAVAX();
+			uint256 diffInAVAX = diff.mulWadUp(ggpPriceInAvax);
+                       staking.decreaseAVAXStake(owner, diffInAVAX);
+			Vault vault = Vault(getContractAddress("Vault"));
+			vault.transferAVAX("ProtocolDAO", diffInAVAX);
		
		}
```