## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-06

# [Protocol will not benefit from slashing mechanism when remaining penalty bigger than minThreshold](https://github.com/code-423n4/2023-06-stader-findings/issues/292) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/SDCollateral.sol#L82


# Vulnerability details

## Impact

During withdraw process, the function `settleFunds()` get called, this function first calculate the operatorShare and the penaltyAmount, if the operatorShare < penaltyAmount, the function call `slashValidatorSD` in order to slash the operator and start new auction to cover the Loss. 

The issue here is that `slashValidatorSD` determine the amount to be reduced based on the smallest value between operator current SD balance and the `poolThreshold.minThreshold`, in case where the operatorShare(1ETH) is too small than the penaltyAmount (10ETH) the system should reduce an equivalent amount to cover the remaining ETH (9ETH_ however, the function choose the smallest value which could end up being 4e17. so in such cases the protocol will not favor, cause starting a new auction with SD amount = 4e17 surly will not end up with a 9ETH in exchange.

## Proof of Concept

1. Implementation of `settleFunds`
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L54

```
function settleFunds() external override {
        uint8 poolId = VaultProxy(payable(address(this))).poolId();
        uint256 validatorId = VaultProxy(payable(address(this))).id();
        IStaderConfig staderConfig = VaultProxy(payable(address(this))).staderConfig();
        address nodeRegistry = IPoolUtils(staderConfig.getPoolUtils()).getNodeRegistry(poolId);
        if (msg.sender != nodeRegistry) {
            revert CallerNotNodeRegistryContract();
        }
        (uint256 userSharePrelim, uint256 operatorShare, uint256 protocolShare) = calculateValidatorWithdrawalShare();

        uint256 penaltyAmount = getUpdatedPenaltyAmount(poolId, validatorId, staderConfig);

        if (operatorShare < penaltyAmount) {
            ISDCollateral(staderConfig.getSDCollateral()).slashValidatorSD(validatorId, poolId);
            penaltyAmount = operatorShare;
        }

        uint256 userShare = userSharePrelim + penaltyAmount;
        operatorShare = operatorShare - penaltyAmount;

        // Final settlement
        vaultSettleStatus = true;
        IPenalty(staderConfig.getPenaltyContract()).markValidatorSettled(poolId, validatorId);
        IStaderStakePoolManager(staderConfig.getStakePoolManager()).receiveWithdrawVaultUserShare{value: userShare}();
        UtilLib.sendValue(payable(staderConfig.getStaderTreasury()), protocolShare);
        IOperatorRewardsCollector(staderConfig.getOperatorRewardsCollector()).depositFor{value: operatorShare}(
            getOperatorAddress(poolId, validatorId, staderConfig)
        );
        emit SettledFunds(userShare, operatorShare, protocolShare);
    }
```

2. Let's suppose operatorShare is 1 ETH, penaltyAmount is 5ETH, in this case the fucntion will enter the If condition on L67
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L67

```
 ISDCollateral(staderConfig.getSDCollateral()).slashValidatorSD(validatorId, poolId);
```

3. Implementation of `slashValidatorSD` and `slashSD`
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/SDCollateral.sol#L78

```
function slashValidatorSD(uint256 _validatorId, uint8 _poolId) external override nonReentrant {
        address operator = UtilLib.getOperatorForValidSender(_poolId, _validatorId, msg.sender, staderConfig);
        isPoolThresholdValid(_poolId);
        PoolThresholdInfo storage poolThreshold = poolThresholdbyPoolId[_poolId];
        uint256 sdToSlash = convertETHToSD(poolThreshold.minThreshold);
        slashSD(operator, sdToSlash);
    }

    /// @notice used to slash operator SD, incase of operator default
    /// @dev do provide SD approval to auction contract using `maxApproveSD()`
    /// @param _operator which operator SD collateral to slash
    /// @param _sdToSlash amount of SD to slash
    function slashSD(address _operator, uint256 _sdToSlash) internal {
        uint256 sdBalance = operatorSDBalance[_operator];
        uint256 sdSlashed = Math.min(_sdToSlash, sdBalance);
        if (sdSlashed == 0) {
            return;
        }
        operatorSDBalance[_operator] -= sdSlashed;
        IAuction(staderConfig.getAuctionContract()).createLot(sdSlashed);
        emit SDSlashed(_operator, staderConfig.getAuctionContract(), sdSlashed);
    }
```

4. As you can see, on line 82 the function get the `minThreshold` and pass it to `slashSD`

5. on line, 92 it select the smallest value between current balance of the operator and the `minThreshold`:

```
uint256 sdSlashed = Math.min(_sdToSlash, sdBalance);
```

6. if the minThreshold < remaining penalty which is 4 ETH in this case, the function simply ignore that and it reduces the operator amount with "minThreshold" instead in case it's < current SD balance.

7. The function then start new auction with the smallest value 
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/SDCollateral.sol#L97

```
operatorSDBalance[_operator] -= sdSlashed;
        IAuction(staderConfig.getAuctionContract()).createLot(sdSlashed);
```

So despite how much the user should pay, auction will start with the min value and the penaltyAmount will not be paid in full.

## Tools Used

Manual

## Recommended Mitigation Steps

The function shouldn't use `minThreshold` but should catch the remaining penalty ( difference between operatorShare and penaltyAmount ) and use it to calculate the required SD amount to be slashed.


## Assessed type

Context