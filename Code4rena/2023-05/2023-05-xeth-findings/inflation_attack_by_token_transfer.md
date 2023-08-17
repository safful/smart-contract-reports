## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor acknowledged
- M-06

# [Inflation attack by token transfer](https://github.com/code-423n4/2023-05-xeth-findings/issues/21) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/wxETH.sol#L212


# Vulnerability details



## Impact
The first staker can inflate the exchange rate by transferring tokens directly to the contract such that subsequent stakers get minted zero wxETH. Their stake can then be unstaked by the first staker, together with their own first stake and inflation investment. Effectively, the first staker can steal the second stake.
The attack exploits the rounding error in tokens minted, caused by the inflation. This vulnerability has a more general impact than just described, in that stakes might be partially stolen and that it also affects further stakers down the line, but the below example demonstrates the basic case.

## Proof of Concept
Alice is the first staker, so `totalSupply() == 0`.
She stakes `1` xETH by calling `stake(1)`.
```solidity
function stake(uint256 xETHAmount) external drip returns (uint256) {
    /// @dev calculate the amount of wxETH to mint
    uint256 mintAmount = previewStake(xETHAmount);

    /// @dev transfer xETH from the user to the contract
    xETH.safeTransferFrom(msg.sender, address(this), xETHAmount);

    /// @dev emit event
    emit Stake(msg.sender, xETHAmount, mintAmount);

    /// @dev mint the wxETH to the user
    _mint(msg.sender, mintAmount);

    return mintAmount;
}
```
and gets minted `previewStake(1)`
```solidity
function previewStake(uint256 xETHAmount) public view returns (uint256) {
    /// @dev if xETHAmount is 0, revert.
    if (xETHAmount == 0) revert AmountZeroProvided();

    /// @dev calculate the amount of wxETH to mint before transfer
    return (xETHAmount * BASE_UNIT) / exchangeRate();
}
```
which thus is `1 * BASE_UNIT / exchangeRate()`, where 
```solidity
function exchangeRate() public view returns (uint256) {
    /// @dev if there are no tokens minted, return the initial exchange rate
    uint256 _totalSupply = totalSupply();
    if (_totalSupply == 0) {
        return INITIAL_EXCHANGE_RATE;
    }

    /// @dev calculate the cash on hand by removing locked funds from the total xETH balance
    /// @notice this balanceOf call will include any lockedFunds,
    /// @notice as the locked funds are also in xETH
    uint256 cashMinusLocked = xETH.balanceOf(address(this)) - lockedFunds;

    /// @dev return the exchange rate by dividing the cash on hand by the total supply
    return (cashMinusLocked * BASE_UNIT) / _totalSupply;
}
```
so this works out to `1 * BASE_UNIT / INITIAL_EXCHANGE_RATE`. This is what Alice is minted and also the new `totalSupply()`.

Bob is about to stake `b` xETH (which Alice can frontrun).

Before Bob's stake, Alice transfers `a` xETH directly to the wxETH contract. The xETH balance of wxETH is now `1 + a + lockedFunds`; `1` from Alice's stake, `a` from her token transfer and whatever `lockedFunds` may have been added.

Now Bob stakes his `b` xETH by calling `stake(b)`. By reference to the above code, this mints him `previewStake(b)`, which is `n * BASE_UNIT / exchangeRate()`.
This time `totalSupply() > 0` so `exchangeRate()` this time is `(1 + a) * BASE_UNIT / (1 * BASE_UNIT / INITIAL_EXCHANGE_RATE)`, which simplifies to `(1 + a) * INITIAL_EXCHANGE_RATE`.
So Bob gets minted `b * BASE_UNIT / ((1 + a) * INITIAL_EXCHANGE_RATE)`. This may clearly be `< 1` which is therefore rounded down to `0` in Solidity. `BASE_UNIT` and `INITIAL_EXCHANGE_RATE` are both set to `1e18`, so Bob is minted `b/(1 + a)` tokens. In that case we simply have that if `a >= b` then Bob is minted `0` wxETH.

Now note in `stake()` above that whenever a non-zero amount is staked, those funds are transferred to the contract (even if nothing is minted).

Bob cannot unstake anything with `0` wxETH, so he has lost his `b` xETH.

`totalSupply()` is now `1` and the xETH balance of wxETH is `1 + a + b + lockedFunds`. `exchangeRate()` is therefore `(1 + a + b) * BASE_UNIT / 1`.

Alice owns the `1` wxETH ever minted so if she  unstakes it
```solidity
function unstake(uint256 wxETHAmount) external drip returns (uint256) {
    /// @dev calculate the amount of xETH to return
    uint256 returnAmount = previewUnstake(wxETHAmount);

    /// @dev emit event
    emit Unstake(msg.sender, wxETHAmount, returnAmount);

    /// @dev burn the wxETH from the user
    _burn(msg.sender, wxETHAmount);

    /// @dev return the xETH back to user
    xETH.safeTransfer(msg.sender, returnAmount);

    /// @dev return the amount of xETH sent to user
    return returnAmount;
}
```
she gets `previewUnstake(1)`
```solidity
function previewUnstake(uint256 wxETHAmount) public view returns (uint256) {
    /// @dev if wxETHAmount is 0, revert.
    if (wxETHAmount == 0) revert AmountZeroProvided();

    /// @dev calculate the amount of xETH to return
    return (wxETHAmount * exchangeRate()) / BASE_UNIT;
}
```
which thus is `1 * (1 + a + b) * BASE_UNIT / BASE_UNIT == (1 + a + b)`.

That is, Alice gets both hers and Bob's stakes.

## Recommended Mitigation Steps
Do not use `xETH.balanceOf(address(this))` when calculating the funds staked. Account only for funds transferred through `stake()` by keeping an internal accounting of the balance. Consider implementing a sweep function to access any unaccounted funds, or use them as locked funds if free (but unlikely) funds would be accepted as such.


## Assessed type

ERC4626