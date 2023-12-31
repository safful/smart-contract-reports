## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- valid

# [Missing upper limit definition in replaceLenderFee() of HomeFi.sol](https://github.com/code-423n4/2022-08-rigor-findings/issues/400) 

# Lines of code

https://github.com/code-423n4/2022-08-rigor/blob/main/contracts/Community.sol#L392-L394
https://github.com/code-423n4/2022-08-rigor/blob/main/contracts/HomeFi.sol#L184-L197


# Vulnerability details

# Missing upper limit definition in `replaceLenderFee()` of `HomeFi.sol`

## Impact
The admin of the `HomeFi` contract can set `lenderFee` to greater than 100%, forcing calls to `lendToProject()` to all projects created in the future to revert.

## Proof of Concept
Using the function `replaceLenderFee()`, admins of the `HomeFi` contract can set `lenderFee` to any arbitrary `uint256` value:
```solidity
 185:        function replaceLenderFee(uint256 _newLenderFee)
 186:            external
 187:            override
 188:            onlyAdmin
 189:        {
 190:            // Revert if no change in lender fee
 191:            require(lenderFee != _newLenderFee, "HomeFi::!Change");
 192:    
 193:            // Reset variables
 194:            lenderFee = _newLenderFee;
 195:    
 196:            emit LenderFeeReplaced(_newLenderFee);
 197:        }
```

New projects that are created will then get its `lenderFee` from the `HomeFi` contract. When communities wish to lend to these projects, it calls `lendToProject()`, which has the following calculation:
```solidity
 392:        // Calculate lenderFee
 393:        uint256 _lenderFee = (_lendingAmount * _projectInstance.lenderFee()) /
 394:            (_projectInstance.lenderFee() + 1000);
```
If `lenderFee` a large value, such as `type(uint256).max`, the calculation shown above to overflow. This prevents any community from lending to any new projects.


## Recommended Mitigation Steps
Consider adding a reasonable fee rate bounds checks in the `replaceLenderFee()` function. This would prevent potential griefing and increase the trust of users in the contract.