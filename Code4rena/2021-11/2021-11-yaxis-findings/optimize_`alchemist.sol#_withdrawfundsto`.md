## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Optimize `Alchemist.sol#_withdrawFundsTo`](https://github.com/code-423n4/2021-11-yaxis-findings/issues/102) 

# Handle

0x0x0x


# Vulnerability details

## Proof of Concept
`L839-L843` is as follow:
```
            AlchemistVault.Data storage _activeVault = _vaults.last();
            (uint256 _withdrawAmount, uint256 _decreasedValue) = _activeVault.withdraw(
                _recipient,
                _remainingAmount
            );
```
It can be replaced by following code block, since there is no reason to save it to memory.
```
            (uint256 _withdrawAmount, uint256 _decreasedValue) = _vaults.last().withdraw(
                _recipient,
                _remainingAmount
            );
```
## Tools Used
Manual analysis

