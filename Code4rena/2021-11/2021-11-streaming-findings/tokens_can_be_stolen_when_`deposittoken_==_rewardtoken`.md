## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Tokens can be stolen when `depositToken == rewardToken`](https://github.com/code-423n4/2021-11-streaming-findings/issues/215) 

# Handle

cmichel


# Vulnerability details

The `Streaming` contract allows the `deposit` and `reward` tokens to be the same token.

> I believe this is intended, think Sushi reward on Sushi as is the case with `xSushi`.

The reward and deposit balances are also correctly tracked independently in `depositTokenAmount` and `rewardTokenAmount`.
However, when recovering tokens this leads to issues as the token is recovered twice, once for deposits and another time for rewards:

```solidity
function recoverTokens(address token, address recipient) public lock {
    // NOTE: it is the stream creators responsibility to save
    // tokens on behalf of their users.
    require(msg.sender == streamCreator, "!creator");
    if (token == depositToken) {
        require(block.timestamp > endDepositLock, "time");
        // get the balance of this contract
        // check what isnt claimable by either party
        // @audit-info depositTokenAmount updated on stake/withdraw/exit, redeemedDepositTokens increased on claimDepositTokens
        uint256 excess = ERC20(token).balanceOf(address(this)) - (depositTokenAmount - redeemedDepositTokens);
        // allow saving of the token
        ERC20(token).safeTransfer(recipient, excess);

        emit RecoveredTokens(token, recipient, excess);
        return;
    }
    
    if (token == rewardToken) {
        require(block.timestamp > endRewardLock, "time");
        // check current balance vs internal balance
        //
        // NOTE: if a token rebases, i.e. changes balance out from under us,
        // most of this contract breaks and rugs depositors. this isn't exclusive
        // to this function but this function would in theory allow someone to rug
        // and recover the excess (if it is worth anything)

        // check what isnt claimable by depositors and governance
        // @audit-info rewardTokenAmount increased on fundStream
        uint256 excess = ERC20(token).balanceOf(address(this)) - (rewardTokenAmount + rewardTokenFeeAmount);
        ERC20(token).safeTransfer(recipient, excess);

        emit RecoveredTokens(token, recipient, excess);
        return;
    }
    // ...
```

#### POC
Given `recoverTokens == depositToken`, `Stream` creator calls `recoverTokens(token = depositToken, creator)`.

- The `token` balance is the sum of deposited tokens (minus reclaimed) plus the reward token amount. `ERC20(token).balanceOf(address(this)) >= (depositTokenAmount - redeemedDepositTokens) + (rewardTokenAmount + rewardTokenFeeAmount)`
- `if (token == depositToken)` executes, the `excess` from the deposit amount will be the reward amount (`excess >= rewardTokenAmount + rewardTokenFeeAmount`). This will be transferred.
- `if (token == rewardToken)` executes, the new token balance is just the deposit token amount now (because the reward token amount has been transferred out in the step before). Therefore, `ERC20(token).balanceOf(address(this)) >= depositTokenAmount - redeemedDepositTokens`. If this is non-negative, the transaction does not revert and the creator makes a profit.

Example:
- outstanding redeemable deposit token amount: `depositTokenAmount - redeemedDepositTokens = 1000`
- funded `rewardTokenAmount` (plus `rewardTokenFeeAmount` fees): `rewardTokenAmount + rewardTokenFeeAmount = 500`

Creator receives `1500 - 1000 = 500` excess deposit and `1000 - 500 = 500` excess reward.

## Impact
When using the same deposit and reward token, the stream creator can steal tokens from the users who will be unable to withdraw their profit or claim their rewards.

## Recommended Mitigation Steps
One needs to be careful with using `.balanceOf` in this special case as it includes both deposit and reward balances.

Add a special case for `recoverTokens` when `token == depositToken == rewardToken` and then the excess should be `ERC20(token).balanceOf(address(this)) - (depositTokenAmount - redeemedDepositTokens) - (rewardTokenAmount + rewardTokenFeeAmount);`

