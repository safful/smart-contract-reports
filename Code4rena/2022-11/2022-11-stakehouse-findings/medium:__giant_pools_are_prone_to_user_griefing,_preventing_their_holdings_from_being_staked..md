## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-30

# [Medium:  Giant pools are prone to user griefing, preventing their holdings from being staked.](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/415) 

# Lines of code

 https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/GiantMevAndFeesPool.sol#L105


# Vulnerability details

## Description

batchRotateLPTokens in GiantMevAndFeesPool allows any user to rotate LP tokens of stakingFundsVaults around.

```
function batchRotateLPTokens(
    address[] calldata _stakingFundsVaults,
    LPToken[][] calldata _oldLPTokens,
    LPToken[][] calldata _newLPTokens,
    uint256[][] calldata _amounts
) external {
    uint256 numOfRotations = _stakingFundsVaults.length;
    require(numOfRotations > 0, "Empty arrays");
    require(numOfRotations == _oldLPTokens.length, "Inconsistent arrays");
    require(numOfRotations == _newLPTokens.length, "Inconsistent arrays");
    require(numOfRotations == _amounts.length, "Inconsistent arrays");
    require(lpTokenETH.balanceOf(msg.sender) >= 0.5 ether, "No common interest");
    for (uint256 i; i < numOfRotations; ++i) {
        StakingFundsVault(payable(_stakingFundsVaults[i])).batchRotateLPTokens(_oldLPTokens[i], _newLPTokens[i], _amounts[i]);
    }
}
```

There is a check that sender has over 0.5 ether of lpTokenETH, to prevent griefing. However, this check is unsatisfactory as user can at any stage deposit ETH to receive lpTokenETH and burn it to receive back ETH. Their lpTokenETH holdings do not correlate with their interest in the vault funds.

Therefore, malicious users can keep bouncing LP tokens around and prevent them from being available for actual staking by liquid staking manager.

## Impact

Giant pools are prone to user griefing, preventing their holdings from being staked.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Three options:
1. batchRotateLPTokens should have logic to enforce that this specific rotation is logical
2. only DAO or some priviledged user can perform Giant pool operations
3. Make the caller have something to lose from behaving maliciously, unlike the current status.