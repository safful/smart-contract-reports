## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-30

# [Incorrect flow of adding liquidity in UlyssesRouter.sol](https://github.com/code-423n4/2023-05-maia-findings/issues/201) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesRouter.sol#L49-L56


# Vulnerability details

## Impact
Usually router in AMM is stateless, i.e. it isn't supposed to contain any tokens, it is just wrapper of low-level pool functions to perform user-friendly interaction.
Current implementation of `addLiquidity()` assumes that user firstly transfers tokens to router and then router performs deposit to pool. However it is not atomic and requires two transactions.
Another user can break in after the first transaction and deposit someone else's tokens.

## Proof of Concept
Router calls deposit with msg.sender as receiver of shares:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-amm/UlyssesRouter.sol#L49-L56
```solidity
    function addLiquidity(uint256 amount, uint256 minOutput, uint256 poolId) external returns (uint256) {
        UlyssesPool ulysses = getUlyssesLP(poolId);

        amount = ulysses.deposit(amount, msg.sender);

        if (amount < minOutput) revert OutputTooLow();
        return amount;
    }
```
In deposit pool transfers tokens from msg.sender, which is router:
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/erc-4626/UlyssesERC4626.sol#L34-L45
```solidity
    function deposit(uint256 assets, address receiver) public virtual nonReentrant returns (uint256 shares) {
        // Need to transfer before minting or ERC777s could reenter.
        asset.safeTransferFrom(msg.sender, address(this), assets);

        shares = beforeDeposit(assets);

        require(shares != 0, "ZERO_SHARES");

        _mint(receiver, shares);

        emit Deposit(msg.sender, receiver, assets, shares);
    }
```

First user will lose tokens sent to router, if malicious user calls `addLiquidity()` after it

## Tools Used
Manual Review

## Recommended Mitigation Steps
Transfer tokens to router via `safeTransferFrom()`:
```solidity
    function addLiquidity(uint256 amount, uint256 minOutput, uint256 poolId) external returns (uint256) {
        UlyssesPool ulysses = getUlyssesLP(poolId);
        address(ulysses.asset()).safeTransferFrom(msg.sender, address(this), amount);

        amount = ulysses.deposit(amount, msg.sender);

        if (amount < minOutput) revert OutputTooLow();
        return amount;
    }
```





## Assessed type

Access Control