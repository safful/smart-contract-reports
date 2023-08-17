## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [Corruption of oracle data](https://github.com/code-423n4/2023-06-stader-findings/issues/202) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L270
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L285


# Vulnerability details

## Impact
Corruption of oracle data

## Proof of Concept
Block for lastReportedSDPriceData = 7200
Let current block = 21601
Now StaderOracle will have data for 14400 and 21600 both blocks being pushed by nodes and in prices array 
it will be all mixed up. Also, As soon as 14400 block is finalised data for block 21600 is all lost as well.

## Tools Used

## Recommended Mitigation Steps
add ```if (_sdPriceData.reportingBlockNumber == getSDPriceReportableBlock()) ``` to ensure it is always latest reportable block data
add ```mapping(uint256 => uint256[]) blockPrices``` to store prices array separately for each block being reported to avoid mixing and corruption of data
or have ```uint256 currentEpochBlock```
so that when new block data is pushed, previous data is deleting before pushing new data
```
if(_sdPriceData.reportingBlockNumber!=currentEpochBlock){
   delete prices;
}




## Assessed type

Oracle