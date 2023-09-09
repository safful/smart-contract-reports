## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-16

# [The calculation of ````takeFees```` in ````Vault```` contract is incorrect](https://github.com/code-423n4/2023-01-popcorn-findings/issues/491) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L480-L494


# Vulnerability details

## Impact
The calculation of ````takeFees```` in ````Vault```` contract is incorrect, which will cause fee charged less than expected.

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L480-L494

## Proof of Concept
To simplify the problem, let's given the fee parameters are as follows
```solidity
// Fees are set in 1e18 for 100% (1 BPS = 1e14)
struct VaultFees {
    uint64 deposit; // 0
    uint64 withdrawal; // 0
    uint64 management; // 0.5e18 = 50%
    uint64 performance; // 0
}
```
And the initial asset token and share tokens in the vault are
```solidity
totalAsset = 100 $AST
totalShare = 100 $SHR
yieldEarnings = 0 $AST
```
The yield earnings is also set to 0.
As the yearly management fee is 50%, so the expected fee for one year is
```solidity
feeInAsset = 100 $AST * 0.5 = 50 $AST
```

Now let's calculate the actual fee.
The implementation of ````accruedManagementFee()```` is
```solidity
    function accruedManagementFee() public view returns (uint256) {
        uint256 managementFee = fees.management;
        return
            managementFee > 0
                ? managementFee.mulDiv(
                    totalAssets() * (block.timestamp - feesUpdatedAt),
                    SECONDS_PER_YEAR,
                    Math.Rounding.Down
                ) / 1e18
                : 0;
    }
```
So in this case, one year later, the calculation of first step for ````managementFee ```` will be
```solidity
managementFee = 0.5e18 * 100 $AST  * SECONDS_PER_YEAR / SECONDS_PER_YEAR  / 1e18 = 50 $AST
```
The implementation of ````takeFees()```` is
```solidity
    modifier takeFees() {
        uint256 managementFee = accruedManagementFee();
        uint256 totalFee = managementFee + accruedPerformanceFee();
        uint256 currentAssets = totalAssets();
        uint256 shareValue = convertToAssets(1e18);

        if (shareValue > highWaterMark) highWaterMark = shareValue;

        if (managementFee > 0) feesUpdatedAt = block.timestamp;

        if (totalFee > 0 && currentAssets > 0)
            _mint(feeRecipient, convertToShares(totalFee)); // @audit L491

        _;
    }
```
So, variables before L491 of ````takeFees()```` will be
```solidity
managementFee = 50 $AST
totalFee  = 50 $AST + 0 = 50 $AST
currentAssets  = 100 $AST
```
As the implementation of ````convertToShares()```` is
```solidity
    function convertToShares(uint256 assets) public view returns (uint256) {
        uint256 supply = totalSupply();

        return
            supply == 0
                ? assets
                : assets.mulDiv(supply, totalAssets(), Math.Rounding.Down);
    }
```
So the the second parameter for ````_mint()```` call at L491 is
```solidity
feeInShare = convertToShares(totalFee) = convertToShares(50 $AST) = 50 $AST * 100 $SHR / 100 $AST = 50 $SHR
```
After ````_mint()```` at L491, the variables will be
```solidity
shareOfUser = 100 $SHR
shareOfFeeRecipient = 50 $SHR
totalShare = 100 + 50  = 150 $SHR
totalAsset = 100 $AST
```

The implementation of ````convertToAssets()```` is
```solidity
    function convertToAssets(uint256 shares) public view returns (uint256) {
        uint256 supply = totalSupply(); 

        return
            supply == 0
                ? shares
                : shares.mulDiv(totalAssets(), supply, Math.Rounding.Down);
    }
```
Now we can get actual fee by calling ````convertToAsset()````, which is
```solidity
actualFeeInAsset = convertToAsset(feeInShare) = convertToAsset(50 $SHR) = 50 $SHR * 100 $AST / 150 $SHR = 33.3 $AST
```

We can see the actual fee is less than expected, the realistic parameters will be less than the give 0.5e18, but it will be always true that the actual fee charged is not enough.

## Tools Used
VS Code

## Recommended Mitigation Steps
Use the correct formula such as
```diff
    modifier takeFees() {
        uint256 managementFee = accruedManagementFee();
        uint256 totalFee = managementFee + accruedPerformanceFee();
        uint256 currentAssets = totalAssets();
        uint256 shareValue = convertToAssets(1e18);

        if (shareValue > highWaterMark) highWaterMark = shareValue;

        if (managementFee > 0) feesUpdatedAt = block.timestamp;

-       if (totalFee > 0 && currentAssets > 0)
-           _mint(feeRecipient, convertToShares(totalFee));
+       if (totalFee > 0 && currentAssets > 0) {
+           uint256 supply = totalSupply();
+           uint256 feeInShare = supply == 0
+               ? totalFee
+               : totalFee.mulDiv(supply, currentAssets - totalFee, Math.Rounding.Down);
+           _mint(feeRecipient, feeInShare);
+       }

        _;
    }
```