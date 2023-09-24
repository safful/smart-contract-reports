## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-42

# [UlyssesPool.sol does not match EIP4626 because of preview functions](https://github.com/code-423n4/2023-05-maia-findings/issues/98) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/UlyssesERC4626.sol#L96-L106


# Vulnerability details

## Impact
According to EIP4626 `previewDeposit()`, `previewRedeem()`, `previewMint()` must include fee in returned value:
1. previewDeposit "MUST be inclusive of deposit fees. Integrators should be aware of the existence of deposit fees."
2. previewRedeem "MUST be inclusive of withdrawal fees. Integrators should be aware of the existence of withdrawal fees."
3. previewMint "MUST be inclusive of deposit fees. Integrators should be aware of the existence of deposit fees."

## Proof of Concept
UlyssesPool.sol inherits UlyssesERC4626.sol with default implementation:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/UlyssesERC4626.sol#L96-L106
```solidity
    function previewDeposit(uint256 assets) public view virtual returns (uint256) {
        return assets;
    }

    function previewMint(uint256 shares) public view virtual returns (uint256) {
        return shares;
    }

    function previewRedeem(uint256 shares) public view virtual returns (uint256) {
        return shares;
    }
```
However deposit, redeem, mint in UlyssesPool.sol take fees:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesPool.sol#L1200-L1221
```solidity
    function beforeDeposit(uint256 assets) internal override returns (uint256 shares) {
        // Update deposit/mint
        shares = ulyssesAddLP(assets, true);
    }

    /**
     * @notice Performs the necessary steps to make after depositing.
     * @param assets to be deposited
     */
    function beforeMint(uint256 shares) internal override returns (uint256 assets) {
        // Update deposit/mint
        assets = ulyssesAddLP(shares, false);
    }

    /**
     * @notice Performs the necessary steps to take before withdrawing assets
     * @param shares to be burned
     */
    function afterRedeem(uint256 shares) internal override returns (uint256 assets) {
        // Update withdraw/redeem
        assets = ulyssesRemoveLP(shares);
    }
```
Further you can check that functions `ulyssesAddLP()` and `ulyssesRemoveLP()` take fee. I consider it overabundant in this submission

## Tools Used
Manual Review

## Recommended Mitigation Steps
Override preview functions in UlyssesPool.sol to include fee


## Assessed type

ERC4626