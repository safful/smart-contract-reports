## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-16

# [## vMaia is an ERC-4626 compliant but maxWithdraw & maxRedeem functions are not fully up to EIP-4626's specification](https://github.com/code-423n4/2023-05-maia-findings/issues/585) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/tokens/ERC4626PartnerManager.sol#L174


# Vulnerability details

## vMaia is an ERC-4626 compliant but maxWithdraw & maxRedeem functions are not fully up to EIP-4626's specification

maxWithdraw & maxRedeem functions should return the 0 during withdrawal is paused. But here it's returning balanceOf[user].

## Proof of Concept

vMaia Withdrawal is only allowed once per month during the 1st Tuesday (UTC+0) of the month.

It's  checked by the below function.

     102       function beforeWithdraw(uint256, uint256) internal override {
                /// @dev Check if unstake period has not ended yet, continue if it is the case.
                if (unstakePeriodEnd >= block.timestamp) return;
        
                uint256 _currentMonth = DateTimeLib.getMonth(block.timestamp);
                if (_currentMonth == currentMonth) revert UnstakePeriodNotLive();
        
                (bool isTuesday, uint256 _unstakePeriodStart) = DateTimeLib.isTuesday(block.timestamp);
                if (!isTuesday) revert UnstakePeriodNotLive();
        
                currentMonth = _currentMonth;
                unstakePeriodEnd = _unstakePeriodStart + 1 days;
    114        }

https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L102C1-L114C6


    173            function maxWithdraw(address user) public view virtual override returns (uint256) {
                      return balanceOf[user];
                  }
              
                  /// @notice Returns the maximum amount of assets that can be redeemed by a user.
                  /// @dev Assumes that the user has already forfeited all utility tokens.
                  function maxRedeem(address user) public view virtual override returns (uint256) {
                      return balanceOf[user];
    181              }


https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/tokens/ERC4626PartnerManager.sol#L173C3-L181C6

Other than that period (during the 1st Tuesday (UTC+0) of the month ), maxWithdraw & maxRedeem functions should return the 0
According to EIP-4626 specifications.

maxWithdraw

     MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST
     return 0.

maxRedeem

     MUST factor in both global and user-specific limits, like if redemption is entirely disabled (even temporarily) it MUST
     return 0.

https://eips.ethereum.org/EIPS/eip-4626

## Tools Used
Manual Auditing

## Recommended Mitigation Steps

Use if-else block & if the time period is within the 1st Tuesday (UTC+0) of the month return balanceOf[user] & else return 0.

For more information: https://blog.openzeppelin.com/pods-finance-ethereum-volatility-vault-audit-2#non-standard-erc-4626-vault-functionality



## Assessed type

ERC4626