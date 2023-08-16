## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [AbstractRewardMine - Re-entrancy attack during withdrawal](https://github.com/code-423n4/2021-11-malt-findings/issues/333) 

# Handle

ScopeLift


# Vulnerability details

## Impact

The internal `_withdraw` method does not follow the checks-effects-interactions pattern. A malicious token, or one that implemented transfer hooks, could re-enter the public calling function (such as `withdraw()`) before proper internal accounting was completed. Because the `earned` function looks up the `_userWithdrawn` mapping, which is not yet updated when the transfer occurs, it would be possible for a malicious contract to re-enter `_withdraw` repeatedly and drain the pool.

## Proof of Concept

N/A

## Tools Used

N/A

## Recommended Mitigation Steps

The internal accounting should be done before the transfer occurs:

```solidity
function _withdraw(address account, uint256 amountReward, address to) internal {
    _userWithdrawn[account] += amountReward;
    _globalWithdrawn += amountReward;f

   rewardToken.safeTransfer(to, amountReward);

    emit Withdraw(account, amountReward, to);
  }
```


