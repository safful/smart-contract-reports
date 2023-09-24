## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-02

# [Slippage controls for calling `bHermes` contract's `ERC4626DepositOnly.deposit` and `ERC4626DepositOnly.mint` functions are missing](https://github.com/code-423n4/2023-05-maia-findings/issues/901) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/erc-4626/ERC4626DepositOnly.sol#L32-L44
https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/erc-4626/ERC4626DepositOnly.sol#L47-L58
https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/ulysses-amm/UlyssesRouter.sol#L49-L56


# Vulnerability details

## Impact
https://eips.ethereum.org/EIPS/eip-4626#security-considerations mentions that "if implementors intend to support EOA account access directly, they should consider adding an additional function call for `deposit`/`mint`/`withdraw`/`redeem` with the means to accommodate slippage loss or unexpected deposit/withdrawal limits, since they have no other means to revert the transaction if the exact output amount is not achieved." Using the `bHermes` contract that inherits the `ERC4626DepositOnly` contract, EOAs can call the following `ERC4626DepositOnly.deposit` and `ERC4626DepositOnly.mint` functions directly. However, because no slippage controls can be specified when calling these functions, these functions' `shares` and `assets` outputs can be less than expected to these EOAs.

https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/erc-4626/ERC4626DepositOnly.sol#L32-L44
```solidity
    function deposit(uint256 assets, address receiver) public virtual returns (uint256 shares) {
        // Check for rounding error since we round down in previewDeposit.
        require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

        // Need to transfer before minting or ERC777s could reenter.
        address(asset).safeTransferFrom(msg.sender, address(this), assets);

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);

        afterDeposit(assets, shares);
    }
```

https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/erc-4626/ERC4626DepositOnly.sol#L47-L58
```solidity
    function mint(uint256 shares, address receiver) public virtual returns (uint256 assets) {
        assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.

        // Need to transfer before minting or ERC777s could reenter.
        address(asset).safeTransferFrom(msg.sender, address(this), assets);

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);

        afterDeposit(assets, shares);
    }
```

In contrast, the following `UlyssesRouter.addLiquidity` function does control the slippage by including the `minOutput` input and executing `amount = ulysses.deposit(amount, msg.sender)` and `if (amount < minOutput) revert OutputTooLow()`. Although such slippage control for an ERC-4626 vault exists in this protocol's other contract, it does not exist in the `bHermes` contract. As a result, EOAs can mint less bHermes shares than expected when calling the `bHermes` contract's `ERC4626DepositOnly.deposit` function and send and burn more HERMES tokens than expected when calling the `bHermes` contract's `ERC4626DepositOnly.mint` function.

https://github.com/code-423n4/2023-05-maia/blob/53c7fe9d5e55754960eafe936b6cb592773d614c/src/ulysses-amm/UlyssesRouter.sol#L49-L56
```solidity
    function addLiquidity(uint256 amount, uint256 minOutput, uint256 poolId) external returns (uint256) {
        UlyssesPool ulysses = getUlyssesLP(poolId);

        amount = ulysses.deposit(amount, msg.sender);

        if (amount < minOutput) revert OutputTooLow();
        return amount;
    }
```

## Proof of Concept
The following steps can occur for the described scenario involving the `bHermes` contract's `ERC4626DepositOnly.mint` function. The case involving the `bHermes` contract's `ERC4626DepositOnly.deposit` function is similar to this.
1. Alice wants to mint 1e18 bHermes shares in exchange for sending and burning 1e18 HERMES tokens.
2. Alice calls the `bHermes` contract's `ERC4626DepositOnly.mint` function with the `shares` input being 1e18.
3. Yet, such `ERC4626DepositOnly.mint` function call causes 1.2e18 HERMES tokens to be transferred from Alice.
4. Alice unexpectedly sends, burns, and loses 0.2e18 more HERMES tokens than expected for minting 1e18 bHermes shares.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `bHermes` contract can be updated to include a `deposit` function that allows `msg.sender` to specify the minimum bHermes shares to be minted for calling the corresponding `ERC4626DepositOnly.deposit` function; calling such `bHermes.deposit` function should revert if the corresponding `ERC4626DepositOnly.deposit` function's `shares` output is less than the specified minimum bHermes shares to be minted. Similarly, the `bHermes` contract can also include a `mint` function that allows `msg.sender` to specify the maximum HERMES tokens to be sent for calling the corresponding `ERC4626DepositOnly.mint` function; calling such `bHermes.mint` function should revert if the corresponding `ERC4626DepositOnly.mint` function's `assets` output is more than the specified maximum HERMES tokens to be sent.


## Assessed type

Other