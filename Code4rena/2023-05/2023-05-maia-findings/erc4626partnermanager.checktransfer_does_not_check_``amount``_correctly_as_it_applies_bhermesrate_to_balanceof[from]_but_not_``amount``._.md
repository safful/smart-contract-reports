## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-36

# [ERC4626PartnerManager.checkTransfer does not check ``amount`` correctly as it applies bHermesRate to balanceOf[from] but not ``amount``. ](https://github.com/code-423n4/2023-05-maia-findings/issues/268) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L325-L335


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
ERC4626PartnerManager.checkTransfer() does not check ``amount`` correctly as it applies bHermesRate to balanceOf[from] but not to ``amount``. 

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

``ERC4626PartnerManager.checkTransfer()`` is a modifier that will be called to ensure that the ``from`` account has sufficient funding to cover  ``userClaimedWeight[from]``, ``userClaimedBoost[from]``, ``userClaimedGovernance[from]``, and ``userClaimedPartnerGovernance[from]`` before transfer occurs:

[https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L325-L335](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/maia/tokens/ERC4626PartnerManager.sol#L325-L335)

However, ``bHermesRate`` is applied to ``balanceOf[from]`` but not to ``amount``. This is not right, since ``amount`` is not in the units of ``userClaimedWeight[from]``, ``userClaimedBoost[from]``, ``userClaimedGovernance[from]``, and ``userClaimedPartnerGovernance[from]``, but in the units of shares of ERC4626PartnerManager. 
 
The correct way to check would be to ensure  ``balanceOf[from]-amount *  bHermesRate`` >= ``userClaimedWeight[from]``, ``userClaimedBoost[from]``, ``userClaimedGovernance[from]``, and ``userClaimedPartnerGovernance[from]``. 


## Tools Used
VSCode

## Recommended Mitigation Steps

```diff
 modifier checkTransfer(address from, uint256 amount) virtual {
-        uint256 userBalance = balanceOf[from] * bHermesRate;
+        uint256 userBalance = (balanceOf[from] - amount) * bHermesRate;

-        if (
-            userBalance - userClaimedWeight[from] < amount || userBalance - userClaimedBoost[from] < amount
-                || userBalance - userClaimedGovernance[from] < amount
-                || userBalance - userClaimedPartnerGovernance[from] < amount
-        ) revert InsufficientUnderlying();

+        if (
+            userBalance < userClaimedWeight[from] || userBalance < userClaimedBoost[from] 
+                || userBalance < userClaimedGovernance[from]  || userBalance <  userClaimedPartnerGovernance[from] 
+        ) revert InsufficientUnderlying();


        _;
    }
```





## Assessed type

Math