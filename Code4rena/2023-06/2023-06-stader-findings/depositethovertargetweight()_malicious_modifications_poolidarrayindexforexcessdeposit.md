## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- M-09

# [depositETHOverTargetWeight() malicious modifications poolIdArrayIndexForExcessDeposit](https://github.com/code-423n4/2023-06-stader-findings/issues/175) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/7566b5a35f32ebd55d3578b8bd05c038feb7d9cc/contracts/StaderStakePoolsManager.sol#L229


# Vulnerability details

## Impact
Malicious modification in favor of your own funds allocation rounds

## Proof of Concept
`poolIdArrayIndexForExcessDeposit` is used to save `depositETHOverTargetWeight()` which `Pool` is given priority for allocation in the next round

The current implementation rolls over to the next `pool` regardless of whether the current balance is sufficient or not

poolAllocationForExcessETHDeposit:
```solidity
    function poolAllocationForExcessETHDeposit(uint256 _excessETHAmount)
        external
        override
        returns (uint256[] memory selectedPoolCapacity, uint8[] memory poolIdArray)
    {
..
        for (uint256 j; j < poolCount; ) {
            uint256 poolCapacity = poolUtils.getQueuedValidatorCountByPool(poolIdArray[i]);
            uint256 poolDepositSize = ETH_PER_NODE - poolUtils.getCollateralETH(poolIdArray[i]);
            uint256 remainingValidatorsToDeposit = ethToDeposit / poolDepositSize;
            selectedPoolCapacity[i] = Math.min(
                poolAllocationMaxSize - selectedValidatorCount,
                Math.min(poolCapacity, remainingValidatorsToDeposit)
            );
            selectedValidatorCount += selectedPoolCapacity[i];
            ethToDeposit -= selectedPoolCapacity[i] * poolDepositSize;
@>          i = (i + 1) % poolCount;
            //For ethToDeposit < ETH_PER_NODE, we will be able to at best deposit one more validator
            //but that will introduce complex logic, hence we are not solving that
@>          if (ethToDeposit < ETH_PER_NODE || selectedValidatorCount >= poolAllocationMaxSize) {
@>              poolIdArrayIndexForExcessDeposit = i;
                break;
            }    
```

Suppose now the balance of `StaderStakePoolsManager` is 0 and poolIdArrayIndexForExcessDeposit = 1

If I have a `Validator` with funds to be allocated at pool = 2, I can maliciously transfer 1 wei and let `poolIdArrayIndexForExcessDeposit` roll over to 2

This way the next round of funding will be allocated in favor of my `Validator`.

Normally, if the current funds are not enough to allocate one `Validator`, then `poolIdArrayIndexForExcessDeposit` should not be rolled
This is more fair

Suggested: If `poolAllocationForExcessETHDeposit()` returns all 0, revert to avoid rolling `poolIdArrayIndexForExcessDeposit`.

## Tools Used

## Recommended Mitigation Steps

```solidity
    function depositETHOverTargetWeight() external override nonReentrant {
..

+       bool findValidator;
        for (uint256 i = 0; i < poolCount; i++) {
            uint256 validatorToDeposit = selectedPoolCapacity[i];
            if (validatorToDeposit == 0) {
                continue;
            }
+           findValidator = true;
            address poolAddress = IPoolUtils(poolUtils).poolAddressById(poolIdArray[i]);
            uint256 poolDepositSize = staderConfig.getStakedEthPerNode() -
                IPoolUtils(poolUtils).getCollateralETH(poolIdArray[i]);

            lastExcessETHDepositBlock = block.number;
            //slither-disable-next-line arbitrary-send-eth
            IStaderPoolBase(poolAddress).stakeUserETHToBeaconChain{value: validatorToDeposit * poolDepositSize}();
            emit ETHTransferredToPool(i, poolAddress, validatorToDeposit * poolDepositSize);
        } 
+       require(findValidator,"not valid validator");     
```


## Assessed type

Context