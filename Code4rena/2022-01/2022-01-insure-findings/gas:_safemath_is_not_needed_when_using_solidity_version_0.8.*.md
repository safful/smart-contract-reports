## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: SafeMath is not needed when using Solidity version 0.8.*](https://github.com/code-423n4/2022-01-insure-findings/issues/38) 

# Handle

Dravee


# Vulnerability details

## Impact  
Increased gas cost  
  
## Proof of Concept  
Solidity version 0.8.* already implements overflow and underflow checks by default. 
Using the SafeMath library from OpenZeppelin (which is more gas expensive than the 0.8.* overflow checks) is therefore redundant.  
  
Instances include: 
```  
mocks\ERC20.sol:4:import "@openzeppelin/contracts/utils/math/SafeMath.sol";
mocks\ERC20.sol:30:    using SafeMath for uint256;
mocks\TestPremiumModel.sol:3:import "@openzeppelin/contracts/utils/math/SafeMath.sol";
mocks\TestPremiumModel.sol:7:    using SafeMath for uint256;
PremiumModels\BondingPremium.sol:10:import "@openzeppelin/contracts/utils/math/SafeMath.sol";
```  
  
## Tools Used  
VS Code  
  
## Recommended Mitigation Steps  
Use the built-in checks instead of SafeMath and remove SafeMath from the dependencies


