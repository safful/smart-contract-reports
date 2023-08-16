## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: No need to initialize variables with default values](https://github.com/code-423n4/2022-01-timeswap-findings/issues/120) 

# Handle

Dravee


# Vulnerability details

## Impact  
If a variable is not set/initialized, it is assumed to have the default value (0, false, 0x0 etc depending on the data type). Explicitly initializing it with its default value is an anti-pattern and wastes gas.
  
## Proof of Concept  
Instances include:  
```
Timeswap-V1-Convenience\contracts\libraries\NFTTokenURIScaffold.sol:119:        for(uint i = 0; i < lengthDiff; i++) {
Timeswap-V1-Convenience\contracts\libraries\NFTTokenURIScaffold.sol:147:        for(uint i = 0; i < lengthDiff; i++) {
Timeswap-V1-Convenience\contracts\libraries\NFTTokenURIScaffold.sol:201:        for (uint256 i = 0; i < data.length; i++) {
``` 
  
## Tools Used  
Manual Analysis  
  
## Recommended Mitigation Steps  
Remove explicit initialization for default values.


