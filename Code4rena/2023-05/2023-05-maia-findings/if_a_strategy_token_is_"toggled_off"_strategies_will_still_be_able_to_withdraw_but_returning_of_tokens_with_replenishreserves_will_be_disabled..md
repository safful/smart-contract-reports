## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [If a STRATEGY TOKEN is "Toggled off" STRATEGIES will still be able to withdraw but returning of tokens with replenishReserves will be disabled.](https://github.com/code-423n4/2023-05-maia-findings/issues/882) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchPort.sol#L158-L169
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchPort.sol#L172-L186


# Vulnerability details

## Impact
`BranchPort.manage` allows a registered Strategy to withdraw certain amounts of enabled strategy tokens. It validates access rights ie. if called by a strategy registered for the requested token. 
It however doesn't check if the token itself is currently enabled. 

Conversely `BranchPort.replenishTokens` allows to force withdraw managed tokens from a strategy. It however performs a check if the token is currently an active strategy token. 

Strategy token may be disabled by  `toggleStrategyToken()` even if there are active strategies managing it active. In such case these strategies will still be able to withdraw the tokens with calls to `manage()` while `replenishTokens` will not be callable on them and thus tokens won't be force returnable. 


## Tools Used
Pen and Paper.
## Recommended Mitigation Steps
1. Add check on enabled strategy token in `manage()`
2. Validate   `getPortStrategyTokenDebt[_strategy][_token] > 0` instead of `!isStrategyToken[_token]` in `replenishReserves()`


## Assessed type

Access Control