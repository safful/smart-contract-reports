## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [User can steal refunded underlying tokens from `initRange` operation inside `RangeManager`](https://github.com/code-423n4/2023-08-goodentry-findings/issues/254) 

# Lines of code

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/RangeManager.sol#L95-L102
https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/TokenisableRange.sol#L134-L163


# Vulnerability details

## Impact
After the owner of `RangeManager` create new range via `generateRange`, they can then call `initRange` to init the range and providing the initial underlying tokens for initial uniswap v3 mint amounts. However, after operation the refunded underlying tokens is not send back to the owner, this will allow user to steal this token by triggering `cleanup()`.

## Proof of Concept

When `initRange` is called, it will trigger `init` inside the `TokenisableRange` :

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/RangeManager.sol#L95-L102

```solidity
  function initRange(address tr, uint amount0, uint amount1) external onlyOwner {
    ASSET_0.safeTransferFrom(msg.sender, address(this), amount0);
    ASSET_0.safeIncreaseAllowance(tr, amount0);
    ASSET_1.safeTransferFrom(msg.sender, address(this), amount1);
    ASSET_1.safeIncreaseAllowance(tr, amount1);
    TokenisableRange(tr).init(amount0, amount1);
    ERC20(tr).safeTransfer(msg.sender, TokenisableRange(tr).balanceOf(address(this)));
  }
```

Inside `init`, it will try to mint Uniswap v3 NFT and provide initial liquidity based on desired underlying amount provided :

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/TokenisableRange.sol#L134-L163

```solidity
  function init(uint n0, uint n1) external {
    require(status == ProxyState.INIT_LP, "!InitLP");
    require(msg.sender == creator, "Unallowed call");
    status = ProxyState.READY;
    TOKEN0.token.safeTransferFrom(msg.sender, address(this), n0);
    TOKEN1.token.safeTransferFrom(msg.sender, address(this), n1);
    TOKEN0.token.safeIncreaseAllowance(address(POS_MGR), n0);
    TOKEN1.token.safeIncreaseAllowance(address(POS_MGR), n1);
    (tokenId, liquidity, , ) = POS_MGR.mint( 
      INonfungiblePositionManager.MintParams({
         token0: address(TOKEN0.token),
         token1: address(TOKEN1.token),
         fee: feeTier * 100,
         tickLower: lowerTick,
         tickUpper: upperTick,
         amount0Desired: n0,
         amount1Desired: n1,
         amount0Min: n0 * 95 / 100,
         amount1Min: n1 * 95 / 100,
         recipient: address(this),
         deadline: block.timestamp
      })
    );
    
    // Transfer remaining assets back to user
    TOKEN0.token.safeTransfer( msg.sender,  TOKEN0.token.balanceOf(address(this)));
    TOKEN1.token.safeTransfer(msg.sender, TOKEN1.token.balanceOf(address(this)));
    _mint(msg.sender, 1e18);
    emit Deposit(msg.sender, 1e18);
  }
```

Uniswap `mint` will always refund the unused underlying tokens, in this case, the `init` will send it back to `RangeManager`.

However, inside `initRange`, it will only transfer to `msg.sender` the minted `tr` token, the remaining underlying token will stay inside this `RangeManager`.

Now user that aware of this `initRange` operation called by admin, can back-run the operation with calling `removeAssetsFromStep` that will trigger `cleanup` function to sweep the token :

https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/RangeManager.sol#L190-L207

```solidity
  function cleanup() internal {
    uint256 asset0_amt = ASSET_0.balanceOf(address(this));
    uint256 asset1_amt = ASSET_1.balanceOf(address(this));
    
    if (asset0_amt > 0) {
      ASSET_0.safeIncreaseAllowance(address(LENDING_POOL), asset0_amt);
      LENDING_POOL.deposit(address(ASSET_0), asset0_amt, msg.sender, 0);
    }
    
    if (asset1_amt > 0) {
      ASSET_1.safeIncreaseAllowance(address(LENDING_POOL), asset1_amt);
      LENDING_POOL.deposit(address(ASSET_1), asset1_amt, msg.sender, 0);
    }
    
    // Check that health factor is not put into liquidation / with buffer
    (,,,,,uint256 hf) = LENDING_POOL.getUserAccountData(msg.sender);
    require(hf > 1.01e18, "Health factor is too low");
  }
```
NOTE : This scenario is not an admin mistake, as the uniswap `mint` operation is very likely to refund underlying tokens (https://docs.uniswap.org/contracts/v3/guides/providing-liquidity/mint-a-position#updating-the-deposit-mapping-and-refunding-the-calling-address)
 
## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider to add `cleanup` after the `initRange` call :

```diff
  function initRange(address tr, uint amount0, uint amount1) external onlyOwner {
    ASSET_0.safeTransferFrom(msg.sender, address(this), amount0);
    ASSET_0.safeIncreaseAllowance(tr, amount0);
    ASSET_1.safeTransferFrom(msg.sender, address(this), amount1);
    ASSET_1.safeIncreaseAllowance(tr, amount1);
    TokenisableRange(tr).init(amount0, amount1);
    ERC20(tr).safeTransfer(msg.sender, TokenisableRange(tr).balanceOf(address(this)));
+   cleanup();
  }
```


## Assessed type

Uniswap